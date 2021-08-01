---
layout: post
title: "Worked example: Parsing command line arguments"
description: "Pattern matching in practice"
date: 2012-06-29
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 11
categories: [Patterns, Worked Examples]
---

match式の仕組みがわかったところで、実際にいくつかの例を見てみましょう。その前に、設計手法について説明します。

## F#でのアプリケーション設計 ##

汎用関数が入力を受け取り、結果を出力することを見てきました。ある意味では、このアプローチは*あらゆる*レベルの関数型コードに適用されるものであり、トップレベルも例外ではありません。

実際、関数型の*アプリケーション*は入力を受け取り、それを変換し、結果を出力します。

![](./function_transform1.png)

理想的には、この変換は、ドメインをモデル化するために作成した純粋な型安全な世界の中で機能するべきですが、残念ながら実世界は型定義されていません。
つまり、入力は単純な文字列またはバイトである可能性が高く、出力も同様です。

どうすればこの問題を解決できるでしょうか?解決策としては、入力を純粋な内部モデルに変換する段階と、内部モデルを出力に変換する段階を別々に用意するという方法があります。

![](./function_transform2.png)

これにより、実世界の混沌をアプリケーションの中核から隠すことができます。この"モデルを純粋に保つ"アプローチは、大規模な場合は ["Hexagonal Architecture"](http://alistair.cockburn.us/Hexagonal+architecture) の概念、小規模な場合はMVCパターンに似ています。

この記事と [その次](/posts/roman-numeric) で、簡単な例をいくつか紹介します。

## 例: コマンドラインの解析

[前の投稿](/posts/match-expression)でマッチ式の一般的な話をしましたが、それが役立つ実際の例、つまりコマンドラインの解析を見てみましょう。

ここでは、基本的な内部モデルと、いくつかの改良を加えた2つの異なるバージョンを設計・実装します。

### 要件

ここでは、3つのコマンドラインオプションがあるとします。ここでは、3つのコマンドラインオプション、"verbose"、"subdirectories"、"orderby "があるとします。
"verbose "と "subdirectories "はフラグで、"orderby "は2つの選択肢があります。"by size "と "by name "です。

So the command line params would look like

	MYAPP [/V] [/S] [/O order]
	/V    verbose
	/S    include subdirectories
	/O    order by. Parameter is one of
			N - order by name.
			S - order by size

## 最初のバージョン

上記のデザインルールに従うと、次のようになります。

* 入力は文字列の配列（またはリスト）で、各引数に1つずつ対応します。
* 内部モデルは、（小さな）ドメインをモデル化した型のセットになります。
* 出力は、この例では範囲外です。

そこで、まずパラメータの内部モデルを作成し、次に入力を解析して内部モデルで使用する型に変換する方法を見ていきます。

ここでは、最初にモデルを作ってみましょう。

```fsharp
// 後で使われる定数
let OrderByName = "N"
let OrderBySize = "S"

// オプションを表す型を設定します
type CommandLineOptions = {
    verbose: bool;
    subdirectories: bool;
    orderby: string;
    }
```

OK、これで大丈夫そうですね。では、引数を解析してみましょう。

この解析ロジックは、前の記事の `loopAndSum` の例と非常によく似ています。

* 引数のリストで再帰的なループを作ります。
* ループに入るたびに、1つの引数を解析します。
* これまでに解析されたオプションは，パラメータとして各ループに渡されます（"accumulator"パターン）．

```fsharp
let rec parseCommandLine args optionsSoFar =
    match args with
    // 空のリストは終了を意味します。
    | [] ->
        optionsSoFar

    // verboseフラグにマッチ
    | "/v"::xs ->
        let newOptionsSoFar = { optionsSoFar with verbose=true}
        parseCommandLine xs newOptionsSoFar

    // subdirectoriesフラグにマッチ
    | "/s"::xs ->
        let newOptionsSoFar = { optionsSoFar with subdirectories=true}
        parseCommandLine xs newOptionsSoFar

    // orderByにフラグでマッチ
    | "/o"::xs ->
        //次の引数でサブマッチを開始する
        match xs with
        | "S"::xss ->
            let newOptionsSoFar = { optionsSoFar with orderby=OrderBySize}
            parseCommandLine xss newOptionsSoFar

        | "N"::xss ->
            let newOptionsSoFar = { optionsSoFar with orderby=OrderByName}
            parseCommandLine xss newOptionsSoFar

        // 認識されないオプションを処理してループを続ける
        | _ ->
            eprintfn "OrderBy needs a second argument"
            parseCommandLine xs optionsSoFar

    // 認識されないオプションを処理してループを続ける
    | x::xs ->
        eprintfn "Option '%s' is unrecognized" x
        parseCommandLine xs optionsSoFar
```

このコードはわかりやすいと思います。

各マッチは `option::restOfList` パターンで構成されています。
オプションにマッチすると、新しい `optionsSoFar` の値が作成され、残りのリストで、リストが空になるまでループを繰り返します。
この時点でループを終了し、最終結果として `optionsSoFar` の値を返すことができます。

2つの特別なケースがあります。

* "orderBy" オプションにマッチすると、残りのリストの最初のアイテムを検索するサブマッチパターンが作成され、見つからない場合は 2 番目のパラメータがないことを訴えます。
* メインの `match..with` の一番最後のマッチは、ワイルドカードではなく、"bind to value"です。ワイルドカードと同じように、これは常に成功しますが、値にバインドしているので、問題のあるマッチしない引数を印刷することができます。
* エラーを表示するには、`printf`ではなく`eprintf`を使うことに注意してください。これにより、STDOUTではなくSTDERRに書き込まれます。

それでは、これをテストしてみましょう。

```fsharp
parseCommandLine ["/v"; "/s"]
```

おっと！うまくいきませんでした。最初の `optionsSoFar` 引数を渡す必要があります。もう一度やってみましょう。

```fsharp
// 渡すべきデフォルトを定義する
let defaultOptions = {
    verbose = false;
    subdirectories = false;
    orderby = ByName
    }

// テストする
parseCommandLine ["/v"] defaultOptions
parseCommandLine ["/v"; "/s"] defaultOptions
parseCommandLine ["/o"; "S"] defaultOptions
```

期待通りの出力が得られるかどうかをチェックします。

そして、エラーケースも確認しましょう。

```fsharp
parseCommandLine ["/v"; "xyz"] defaultOptions
parseCommandLine ["/o"; "xyz"] defaultOptions
```

このような場合は、エラーメッセージが表示されます。

この実装を完了する前に、厄介な問題を解決しましょう。
私たちはこれらのデフォルトオプションを毎回渡しています--それらを取り除くことはできないのでしょうか?

これは非常によくあるケースです。 "アキュムレータ"パラメータを受け取る再帰関数がありますが、常に初期値を渡したくはありません。

その答えは簡単で、デフォルトで再帰関数を呼び出す別の関数を作成するというものです。

通常、この2番目の関数は "public"関数であり、再帰的な関数は隠されているため、次のようにコードを書き直します。

* `parseCommandLine` の名前を `parseCommandLineRec` に変更します。他にも、`parseCommandLine'`にチェックマークを付けたり、`innerParseCommandLine`のような命名規則を使うこともできます。
* デフォルトで `parseCommandLineRec` を呼び出す、新しい `parseCommandLine` を作成します。

```fsharp
// "ヘルパー "再帰関数を作成します。
let rec parseCommandLineRec args optionsSoFar =
	// 上記のような実装

// "パブリック"なパース関数を作る
let parseCommandLine args =
    // 既定値の作成
    let defaultOptions = {
        verbose = false;
        subdirectories = false;
        orderby = OrderByName
        }

    // 初期オプションで再帰的なものを呼び出す
    parseCommandLineRec args defaultOptions
```

この場合、ヘルパー関数はそれだけで成り立ちます。しかし，本当に隠したいのであれば，`parseCommandLine`自体の定義の中に，入れ子になったサブ関数として置くことができます．

```fsharp
// "public" parse関数を作成します。
let parseCommandLine args =
    // デフォルトの設定
    let defaultOptions =
		// 上記のような実装

	// 内部の再帰関数
	let rec parseCommandLineRec args optionsSoFar = // 上記のように実装します。
		// 上記のような実装

    // 初期オプションで再帰的なものを呼び出す
    parseCommandLineRec args defaultOptions
```

この場合、ただ単に複雑になるだけだと思うので、別々にしています。

そこで、すべてのコードを一度にモジュールにまとめてみました。

```fsharp
module CommandLineV1 =

    // 後で使われる定数
    let OrderByName = "N"
    let OrderBySize = "S"

    // オプションを表す型を設定する
    type CommandLineOptions = {
        verbose: bool;
        subdirectories: bool;
        orderby: string;
        }

    // "ヘルパー "再帰関数の作成
    let rec parseCommandLineRec args optionsSoFar =
        match args with
        // 空のリストは終了を意味します。
        | [] ->
            optionsSoFar

        // verboseフラグにマッチ
        | "/v"::xs ->
            let newOptionsSoFar = { optionsSoFar with verbose=true}
            parseCommandLineRec xs newOptionsSoFar

        // subdirectoriesフラグにマッチ
        | "/s"::xs ->
            let newOptionsSoFar = { optionsSoFar with subdirectories=true}
            parseCommandLineRec xs newOptionsSoFar

        // orderByにフラグでマッチ
        | "/o"::xs ->
            //次の引数でサブマッチを開始する
            match xs with
            | "S"::xss ->
                let newOptionsSoFar = { optionsSoFar with orderby=OrderBySize}
                parseCommandLineRec xss newOptionsSoFar

            | "N"::xss ->
                let newOptionsSoFar = { optionsSoFar with orderby=OrderByName}
                parseCommandLineRec xss newOptionsSoFar

            // 認識されないオプションを処理してループを続ける
            | _ ->
                eprintfn "OrderBy needs a second argument"
                parseCommandLineRec xs optionsSoFar

        // 認識されないオプションを処理してループを続ける
        | x::xs ->
            eprintfn "Option '%s' is unrecognized" x
            parseCommandLineRec xs optionsSoFar

    // "パブリック "な解析関数を作る
    let parseCommandLine args =
        // 既定値の作成
        let defaultOptions = {
            verbose = false;
            subdirectories = false;
            orderby = OrderByName
            }

        // 初期オプションで再帰を呼び出す
        parseCommandLineRec args defaultOptions


// ハッピーパス
CommandLineV1.parseCommandLine ["/v"]
CommandLineV1.parseCommandLine ["/v"; "/s"]
CommandLineV1.parseCommandLine ["/o"; "S"]

// エラー処理
CommandLineV1.parseCommandLine ["/v"; "xyz"] // エラー処理
CommandLineV1.parseCommandLine ["/o"; "xyz"] // エラー処理
```

## 第二弾

最初のモデルでは、可能な値を表すのに、boolとstringを使いました。

```fsharp
type CommandLineOptions = {
    verbose: bool;
    subdirectories: bool;
    orderby: string;
    }
```

これには2つの問題があります。

* **ドメインを*本当に*表現してるわけではありません。**例えば、`orderby`は本当に*あらゆる*文字列ですか?これを"ABC"に設定するとコードは壊れてしまいませんか?

* **値は自己文書化されていません。**たとえば、verboseの値はboolです。boolが"詳細"を表しているのは、*状況* (`verbose`という名前のフィールド) にあるからです。
もし、その状況から外れてしまったら、このboolが何を意味しているのか分かりません。以下のような多数のbooleanパラメータを持つC#関数を見たことがあるはずです:

```csharp
myObject.SetUpComplicatedOptions(true,false,true,false,false);
```

boolはドメイン・レベルでは何も表現しないので、間違いを起こしやすいのです。

これらの問題を解決するため、ドメインを定義する際は可能な限り具体的にします。つまり、特別な型をたくさん生成するのです。

これが新しいバージョンの `CommandLineOptions`です:

```fsharp
type OrderByOption = OrderBySize | OrderByName
type SubdirectoryOption = IncludeSubdirectories | ExcludeSubdirectories
type VerboseOption = VerboseOutput | TerseOutput

type CommandLineOptions = {
    verbose: VerboseOption;
    subdirectories: SubdirectoryOption;
    orderby: OrderByOption
    }
```

注目すべき点がいくつかあります:

* boolやstringはどこにもありません。
* 名前は非常に明示的です。これは、値が独立して取得される場合のドキュメントとして機能します。
また、その名前が一意であることも意味します。これは型推論に役立ち、明示的な型注釈を避けるのに役立ちます。

ドメインに変更を加えると、構文解析ロジックを簡単に修正できます。

では、 "v2"モジュールにまとめられた、改訂版のコードを以下に示します:

```fsharp
module CommandLineV2 =

    type OrderByOption = OrderBySize | OrderByName
    type SubdirectoryOption = IncludeSubdirectories | ExcludeSubdirectories
    type VerboseOption = VerboseOutput | TerseOutput

    type CommandLineOptions = {
        verbose: VerboseOption;
        subdirectories: SubdirectoryOption;
        orderby: OrderByOption
        }

    // create the "helper" recursive function
    let rec parseCommandLineRec args optionsSoFar =
        match args with
        // empty list means we're done.
        | [] ->
            optionsSoFar

        // match verbose flag
        | "/v"::xs ->
            let newOptionsSoFar = { optionsSoFar with verbose=VerboseOutput}
            parseCommandLineRec xs newOptionsSoFar

        // match subdirectories flag
        | "/s"::xs ->
            let newOptionsSoFar = { optionsSoFar with subdirectories=IncludeSubdirectories}
            parseCommandLineRec xs newOptionsSoFar

        // match sort order flag
        | "/o"::xs ->
            //start a submatch on the next arg
            match xs with
            | "S"::xss ->
                let newOptionsSoFar = { optionsSoFar with orderby=OrderBySize}
                parseCommandLineRec xss newOptionsSoFar
            | "N"::xss ->
                let newOptionsSoFar = { optionsSoFar with orderby=OrderByName}
                parseCommandLineRec xss newOptionsSoFar
            // handle unrecognized option and keep looping
            | _ ->
                printfn "OrderBy needs a second argument"
                parseCommandLineRec xs optionsSoFar

        // handle unrecognized option and keep looping
        | x::xs ->
            printfn "Option '%s' is unrecognized" x
            parseCommandLineRec xs optionsSoFar

    // create the "public" parse function
    let parseCommandLine args =
        // create the defaults
        let defaultOptions = {
            verbose = TerseOutput;
            subdirectories = ExcludeSubdirectories;
            orderby = OrderByName
            }

        // call the recursive one with the initial options
        parseCommandLineRec args defaultOptions

// ==============================
// tests

// happy path
CommandLineV2.parseCommandLine ["/v"]
CommandLineV2.parseCommandLine ["/v"; "/s"]
CommandLineV2.parseCommandLine ["/o"; "S"]

// error handling
CommandLineV2.parseCommandLine ["/v"; "xyz"]
CommandLineV2.parseCommandLine ["/o"; "xyz"]
```

## 再帰の代わりにfoldを使う？

前回の投稿では、可能な限り再帰を避け、`map`や`fold`のような`List`モジュールに組み込まれた関数を使用することが望ましいと述べました。

さて、このアドバイスに従って、このコードを修正してみましょう。

あいにく簡単ではありません。問題は、リスト関数は一般に一度に1つの要素に対して動作するのに対し、"orderby"オプションには"lookahead"引数が必要なことです。

これを`fold`などで動作させるには、先読みモードであるかどうかを示す"解析モード"フラグを作成する必要があります。
しかし、それでは上記の単純な再帰バージョンに比べて、複雑さが増すだけだと思います。

そして実務的な状況において、これよりも複雑なものは [FParsec](http://www.quanttec.com/fparsec/) のような適切な構文解析システムに切り替える必要があることを意味します。

とはいえ、`fold`を使うとどうなるかはお見せしましょう:

```fsharp
module CommandLineV3 =

    type OrderByOption = OrderBySize | OrderByName
    type SubdirectoryOption = IncludeSubdirectories | ExcludeSubdirectories
    type VerboseOption = VerboseOutput | TerseOutput

    type CommandLineOptions = {
        verbose: VerboseOption;
        subdirectories: SubdirectoryOption;
        orderby: OrderByOption
        }

    type ParseMode = TopLevel | OrderBy

    type FoldState = {
        options: CommandLineOptions ;
        parseMode: ParseMode;
        }

    // トップレベルの引数を解析する
    // 新しいFoldStateを返す
    let parseTopLevel arg optionsSoFar =
        match arg with

        // 詳細フラグにマッチ
        | "/v" ->
            let newOptionsSoFar = {optionsSoFar with verbose=VerboseOutput}
            {options=newOptionsSoFar; parseMode=TopLevel}

        // サブディレクトリにマッチするフラグ
        | "/s"->
            let newOptionsSoFar = { optionsSoFar with subdirectories=IncludeSubdirectories}
            {options=newOptionsSoFar; parseMode=TopLevel}

        // マッチのソート順フラグ
        | "/o" ->
            {options=optionsSoFar; parseMode=OrderBy}

        // 認識されないオプションを処理し、ループを続ける
        | x ->
            printfn "Option '%s' is unrecognized" x
            {options=optionsSoFar; parseMode=TopLevel}

    // orderByの引数を解析する
    // 新しいFoldStateを返す
    let parseOrderBy arg optionsSoFar =
        match arg with
        | "S" ->
            let newOptionsSoFar = { optionsSoFar with orderby=OrderBySize}
            {options=newOptionsSoFar; parseMode=TopLevel}
        | "N" ->
            let newOptionsSoFar = { optionsSoFar with orderby=OrderByName}
            {options=newOptionsSoFar; parseMode=TopLevel}
        // 認識されないオプションを処理してループを続ける
        | _ ->
            printfn "OrderBy needs a second argument"
            {options=optionsSoFar; parseMode=TopLevel}

    // 補助的な fold 関数を作成する
    let foldFunction state element  =
        match state with
        | {options=optionsSoFar; parseMode=TopLevel} ->
            // 新しい状態を返す
            parseTopLevel element optionsSoFar

        | {options=optionsSoFar; parseMode=OrderBy} ->
            // 新しい状態を返す
            parseOrderBy element optionsSoFar

    // "パブリック "な解析関数の作成
    let parseCommandLine args =

        let defaultOptions = {
            verbose = TerseOutput;
            subdirectories = ExcludeSubdirectories;
            orderby = OrderByName
            }

        let initialFoldState =
            {options=defaultOptions; parseMode=TopLevel}

        // 初期状態の fold を呼び出す
        args |> List.fold foldFunction initialFoldState

// ==============================
// テスト

// ハッピーパス
CommandLineV3.parseCommandLine ["/v"]
CommandLineV3.parseCommandLine ["/v"; "/s"]
CommandLineV3.parseCommandLine ["/o"; "S"]

// エラー処理
CommandLineV3.parseCommandLine ["/v"; "xyz"] // エラー処理
CommandLineV3.parseCommandLine ["/o"; "xyz"] // エラー処理
```

ところで、このバージョンでは、微妙に動作が変わっているのがわかりますか？

これまでのバージョンでは、"orderBy"オプションにパラメータがない場合、再帰ループは次回も解析していました。
しかし、"fold"バージョンでは、このトークンが飲み込まれて失われてしまいます。

これを見るために、2つの実装を比較してみましょう。

```fsharp
// verboseセット
CommandLineV2.parseCommandLine ["/o"; "/v"]

// verboseがセットされていない!
CommandLineV3.parseCommandLine ["/o"; "/v"]
```

これを修正するには、さらに手間がかかります。繰り返しになりますが、これは第二の実装が最もデバッグとメンテナンスが容易であることを主張しています。

## まとめ

この記事では、パターンマッチを実際の例に適用する方法を見てきました。

さらに重要なことは、どんなに小さなドメインであっても、適切に設計された内部モデルを作ることがいかに簡単かということです。そして、この内部モデルは、stringやboolのようなプリミティブな型を使うよりも、型の安全性と文書化を提供します。

次の例では、さらに多くのパターンマッチングを行います。
