---
layout: post
title: "Implementing a builder: Zero and Yield"
description: "Getting started with the basic builder methods"
date: 2013-01-25
nav: thinking-functionally
seriesId: "Computation Expressions"
seriesOrder: 6
---

バインドとコンティニュエーション、そしてラッパー型の使用について説明してきましたが、いよいよ「ビルダー」クラスに関連するメソッドの全容を解明する準備が整いました。

[MSDNドキュメント](http://msdn.microsoft.com/en-us/library/dd233182.aspx)を見ると、`Bind`や`Return`だけでなく、`Delay`や`Zero`といった奇妙な名前のメソッドも出てきます。これらのメソッドは何のためにあるのでしょうか？ この記事と次の数回の記事がその答えになります。

## 行動計画

ビルダークラスの作成方法を説明するために、ビルダーのすべてのメソッドを使用するカスタムワークフローを作成します。

しかし、最初からこれらのメソッドの意味を説明するのではなく、シンプルなワークフローから始めて、問題やエラーを解決するために必要なメソッドだけを追加するという、ボトムアップで作業を進めます。その過程で、F#がどのようにコンピュテーション式を処理しているのか、詳しく理解することができます。

このプロセスの概要は以下の通りです。

* Part 1: この最初のパートでは、基本的なワークフローに必要なメソッドを見ていきます。ここでは、`Zero`、`Yield`、`Combine`、`For`を紹介します。
* 第2部では、コードの実行を遅延させて、必要なときにだけコードが評価されるようにする方法を紹介します。`Delay`と`Run`を紹介し、遅延計算についても見ていきます。
* Part 3: 最後に、残りのメソッドである `While`, `Using`, そして例外処理について説明します。

## 始める前に

ワークフローの作成に入る前に、一般的なコメントをいくつか紹介します。

### computation expressionのドキュメントについて

まず、お気づきかもしれませんが、MSDN のコンピュテーション式に関するドキュメントはせいぜい貧弱で、不正確ではありませんが、誤解を招く恐れがあります。例えば、ビルダーメソッドのシグネチャは、見た目よりも柔軟性が高く、ドキュメントだけではわからない機能を実装するために使用することができます。この例は後で紹介します。

より詳細なドキュメントが必要な場合は、お勧めできるソースが2つあります。コンピュテーション式の背後にある概念の詳しい概要については、[paper "The F# Expression Zoo" by Tomas Petricek and Don Syme](http://tomasp.net/academic/papers/computation-zoo/computation-zoo.pdf)が素晴らしい資料です。また、最も正確な最新の技術文書としては、[F# language specification](http://research.microsoft.com/en-us/um/cambridge/projects/fsharp/manual/spec.pdf)を読むべきで、その中にコンピュテーション式に関する項目があります。

### 折り返し型と非折り返し型

ドキュメントに記載されているシグネチャを理解しようとするときには、これまで私が「アンラップ型」と呼んでいたものは通常 `'T` と書かれ、「ラップ型」は通常 `M<'T>` と書かれていることを思い出してください。つまり、`Return`メソッドのシグネチャーが`'T -> M<'T>`となっているのは、`Return`がラップされていない型を受け取り、ラップされた型を返すことを意味しているのだと思います。

これまでの連載でも、これらの型の関係を表すのに「アンラップ」と「ラップ」を使い続けてきましたが、今後はこれらの用語が限界まで伸びてしまうので、「ラップ型」の代わりに「計算型」などの別の用語も使うようにします。ここまで来ると、変更の理由が明確になって理解しやすいと思います。

また、私の例では、基本的には以下のようなコードを使って、物事をシンプルに保つようにしています。

```fsharp
let! x = ...wrapped type value...
```

しかし、これは実際には単純化しすぎです。正確には、"x "は単一の値だけでなく、どんな*パターン*でもよく、"ラップされた型 "の値は、もちろんラップされた型として評価される*式*でもよい。
もちろん、ラップされた型として評価される*式*も可能です。
MSDNのドキュメントでは、このより正確なアプローチを使用しています。MSDNのドキュメントでは、このようなより正確なアプローチを採用しています。例えば、「`let! pattern = expr in cexpr`」のように、定義に「pattern」と「expression」を使用しています。

ここでは，`maybe`というコンピュテーション式でパターンや式を使う例を紹介します．
ここで，`Option`はラップされた型で，右辺の式は`options`です．

```fsharp
// let! pattern = expr in cexpr
maybe {
    let! x,y = Some(1,2)
    let! head::tail = Some( [1;2;3] )
    // etc
    }
```

ここでは、ただでさえ複雑なテーマに、さらに複雑さを加えないように、単純化しすぎた例を使い続けます。

### ビルダークラスに特別なメソッドを実装する (もしくはしない)

MSDNのドキュメントによると、各特殊な操作(例えば、`for..in`や`yield`)は、ビルダークラスのメソッドの1つ以上の呼び出しに変換されています。

必ずしも1対1の対応ではありませんが、一般的には、特殊な操作の構文をサポートするためには、ビルダークラスに対応するメソッドを実装*しなければなりません*。

一方で、その構文が必要でない場合は、すべてのメソッドを実装する*必要はありません*。例えば、`maybe`のワークフローは、`Bind`と`Return`の2つのメソッドを実装するだけで、とてもうまく実装できています。`Delay`や`Use`などを使う必要がなければ、それらを実装する必要はありません。

メソッドを実装していないとどうなるかを見るために、`maybe`のワークフローで`for...in...do`構文を次のように使ってみましょう。

```fsharp
maybe { for i in [1;2;3] do i }
```

コンパイラエラーが発生します。

```text
This control construct may only be used if the computation expression builder defines a 'For' method
```

時々、舞台裏で何が起こっているのかを知らない限り、不可解なエラーが発生することがあります。
例えば、次のようにワークフローの中に`return`を入れ忘れた場合です。

```fsharp
maybe { 1 }
```

コンパイラエラーが発生します。

```text
This control construct may only be used if the computation expression builder defines a 'Zero' method
```

`Zoro`メソッドとは何でしょうか? そして、なぜそれが必要なのか？ その答えはすぐに出てきます。

### '!'を使った演算と使わない演算

当然のことながら、特殊な操作の多くは、「！」マークのあるものとないものがペアになっています。例えば、`let` と `let!` （発音は「レットバン」）、`return` と `return!`、`yield` と `yield!` などです。

この違いは、"!"が付いていない演算は常に右辺が*unwrapped*型であるのに対し、"!"が付いている演算は常に*wrapped*型であることを理解すると、簡単に覚えられます。

そのため，例えば，`maybe`というワークフローを使って，`Option`がラップされた型である場合，それぞれの構文を比較することができます．

```fsharp
let x = 1           // 1 is an "unwrapped" type
let! x = (Some 1)   // Some 1 is a "wrapped" type
return 1            // 1 is an "unwrapped" type
return! (Some 1)    // Some 1 is a "wrapped" type
yield 1             // 1 is an "unwrapped" type
yield! (Some 1)     // Some 1 is a "wrapped" type
```

ラップされた型は，同じ型の*別の*コンピュテーション式の結果になることがあるので，"!"バージョンは合成の際に特に重要です．

```fsharp
let! x = maybe {...)       // "maybe" returns a "wrapped" type

// bind another workflow of the same type using let!
let! aMaybe = maybe {...)  // create a "wrapped" type
return! aMaybe             // return it

// bind two child asyncs inside a parent async using let!
let processUri uri = async {
    let! html = webClient.AsyncDownloadString(uri)
    let! links = extractLinks html
    ... etc ...
    }
```

## 潜入 - ワークフローの最小限の実装を作る

さあ、始めましょう。まず、"maybe "ワークフローの最小限のバージョン（"trace "と改名します）を作成し、すべてのメソッドをインストゥルメント化します。この記事では、これをテストベッドとして使用します。

以下は、`trace`ワークフローの最初のバージョンのコードです。

```fsharp
type TraceBuilder() =
    member this.Bind(m, f) =
        match m with
        | None ->
            printfn "Binding with None. Exiting."
        | Some a ->
            printfn "Binding with Some(%A). Continuing" a
        Option.bind f m

    member this.Return(x) =
        printfn "Returning a unwrapped %A as an option" x
        Some x

    member this.ReturnFrom(m) =
        printfn "Returning an option (%A) directly" m
        m

// make an instance of the workflow
let trace = new TraceBuilder()
```

目新しいことは何もありませんよね。これらのメソッドはすべて以前に見たことがあります。

では、いくつかのサンプルコードを実行してみましょう。

```fsharp
trace {
    return 1
    } |> printfn "Result 1: %A"

trace {
    return! Some 2
    } |> printfn "Result 2: %A"

trace {
    let! x = Some 1
    let! y = Some 2
    return x + y
    } |> printfn "Result 3: %A"

trace {
    let! x = None
    let! y = Some 1
    return x + y
    } |> printfn "Result 4: %A"
```

全てが期待通りに動くはずです。特に、4番目の例で `None` を使うと、次の2行（`let! y = ... return x + y`）がスキップされて、式全体の結果が `None` になっていることがわかるはずです。

## "do!"の導入

この式は `let!` をサポートしていますが、`do!` はどうでしょうか?

通常のF#では、`do`は`let`と同じですが、式が有用なもの（つまり、単位の値）を返さないことが違います。

コンピュテーション式の中では、`do!`は非常によく似ています。let!`がラップされた結果を`Bind`メソッドに渡すのと同様に、`do!`もラップされた結果を`Bind`メソッドに渡します。ただし、`do!`の場合、「結果」はユニット値なので、バインドメソッドにはユニットの*ラップされた*バージョンが渡されます。

以下は、`trace`ワークフローを使った簡単なデモです。

```fsharp
trace {
    do! Some (printfn "...expression that returns unit")
    do! Some (printfn "...another expression that returns unit")
    let! x = Some (1)
    return x
    } |> printfn "Result from do: %A"
```

これが出力結果です。

```text
...expression that returns unit
Binding with Some(<null>). Continuing
...another expression that returns unit
Binding with Some(<null>). Continuing
Binding with Some(1). Continuing
Returning a unwrapped 1 as an option
Result from do: Some 1
```

各 `do!` の結果として `unit option` が `Bind` に渡されていることを自分で確認することができます。

## "Zero "の紹介

コンピュテーション式の中で最も小さいものは何でしょうか？何もしないことを試してみましょう。

```fsharp
trace {
    } |> printfn "Result for empty: %A"
```

すぐにエラーが発生します。

```text
This value is not a function and cannot be applied
```

まあ、いいでしょう。考えてみれば、コンピュテーション式の中に何もないというのは意味がありません。結局のところ、その目的は式を連鎖させることにあります。

次に，`let!`や`return`のないシンプルな式はどうでしょうか？

```fsharp
trace {
    printfn "hello world"
    } |> printfn "Result for simple expression: %A"
```

今度は別のエラーが発生します。

```text
This control construct may only be used if the computation expression builder defines a 'Zero' method
```

では、なぜ今は`Zero`メソッドが必要なのに、以前は必要なかったのでしょうか？その答えは、今回のケースでは明示的に何も返していないのに、コンピュテーション式全体としてはラップされた値を返さなければならないからです。では、どのような値を返すべきでしょうか？

実は、このような状況は、コンピュテーション式の戻り値が明示的に与えられていない場合には必ず発生します。同じことは，else節のない`if...then`式でも起こります．

```fsharp
trace {
    if false then return 1
    } |> printfn "Result for if without else: %A"
```

通常のF#のコードでは、"if...then "に "else "を付けないと単位の値になりますが、コンピュテーション式では、特定の戻り値はラップされた型のメンバーでなければならず、コンパイラはそれがどのような値であるかを知りません。

これを解決するには、コンパイラに何を使うかを指示する必要があり、それが `Zero` メソッドの目的です。

### Zeroにはどのような値を使用すべきか？

では、`Zero`にはどのような値を使うべきなのでしょうか。それは、どのようなワークフローを作成するかによります。

ここでは、いくつかのガイドラインをご紹介します。

* **ワークフローに「成功」や「失敗」の概念はありますか？** もしそうなら、`Zero`には「失敗」の値を使用してください。例えば、`trace`ワークフローでは、失敗を示すために`None`を使用していますので、`None`をゼロ値として使用することができます。
* **ワークフローには「逐次処理」という概念があるのでしょうか？** つまり、ワークフローでは、あるステップを行った後に別のステップを行い、その裏で何らかの処理を行うということです。 通常のF#コードでは、明示的に何も返さない式はunitと評価されます。ですから、このケースと並行して、`Zero`はUnitの*wrapped*バージョンでなければなりません。例えば、オプションベースのワークフローのバリエーションでは、`ゼロ`を意味する`Some ()`を使用することができます（ちなみに、これは常に`Return ()`と同じになります）。
* **ワークフローは主にデータ構造を操作することに関係していますか** そうであれば、`Zero`は「空」のデータ構造であるべきです。例えば、"リストビルダー "のワークフローでは、空のリストをゼロの値として使用します。

また、`ゼロ`の値は、ラップされた型を組み合わせる際にも重要な役割を果たします。次回はZeroについてご紹介しますので、お楽しみに。

### ゼロの実装

それでは、テストベッドクラスを拡張して、`None`を返す`Zero`メソッドを追加して、もう一度試してみましょう。

```fsharp
type TraceBuilder() =
    // other members as before
    member this.Zero() =
        printfn "Zero"
        None

// make a new instance
let trace = new TraceBuilder()

// test
trace {
    printfn "hello world"
    } |> printfn "Result for simple expression: %A"

trace {
    if false then return 1
    } |> printfn "Result for if without else: %A"
```

テストコードを見ると、裏では `Zero` が呼ばれていることがわかります。そして、`None`が式全体の戻り値となります。*Note: `None` は `<null>` としてプリントアウトされる可能性があります。これは無視して構いません*。

### 常にゼロが必要なのか？

`Zero`は必須ではありませんが、ワークフローの中で意味を成す場合にのみ必要となります。例えば、`seq`ではzeroは使えませんが、`async`では使えます。

```fsharp
let s = seq {printfn "zero" }    // Error
let a = async {printfn "zero" }  // OK
```


## "Yield "の紹介

C#には "yield "という文があり、イテレータの中では早く戻ってきて、戻ってきたときには前回の続きをするという使い方をします。

ドキュメントを見ると、F#のコンピュテーション式にも "yield "があります。これは何をするものなのでしょうか？試しに使ってみましょう。

```fsharp
trace {
    yield 1
    } |> printfn "Result for yield: %A"
```

そして、エラーが発生します。

```text
This control construct may only be used if the computation expression builder defines a 'Yield' method
```

当然ですね。では、"yield "メソッドの実装はどのようになっているのでしょうか？ MSDNのドキュメントによると、`'T -> M<'T>`というシグネチャを持っており、これは`Return`メソッドのシグネチャと全く同じです。このメソッドはラップされていない値を受け取り、それをラップしなければなりません。

そこで，`Return`と同じように実装して，テスト式を再試行してみましょう．

```fsharp
type TraceBuilder() =
    // other members as before

    member this.Yield(x) =
        printfn "Yield an unwrapped %A as an option" x
        Some x

// make a new instance
let trace = new TraceBuilder()

// test
trace {
    yield 1
    } |> printfn "Result for yield: %A"
```

これで動作するようになり、`return`の正確な代用として使用できるようになりました。

また、`ReturnFrom`メソッドと平行して、`YieldFrom`メソッドもあります。これも同じように、ラップされていない値ではなく、ラップされた値を返すことができます。

それでは、このメソッドもビルダーメソッドのリストに加えてみましょう。

```fsharp
type TraceBuilder() =
    // other members as before

    member this.YieldFrom(m) =
        printfn "Yield an option (%A) directly" m
        m

// make a new instance
let trace = new TraceBuilder()

// test
trace {
    yield! Some 1
    } |> printfn "Result for yield!: %A"
```

ここで疑問に思うかもしれません。「return」と「yield」が基本的に同じものであるならば、なぜ2つの異なるキーワードがあるのか？ その答えは主に、一方を実装して他方を実装しないことで、適切な構文を強制することができるからです。 例えば、以下のスニペットからわかるように、`seq`式は、`yield`を*許容します*が、`return`を*許容しません*。一方、`async`式は、`return`を許容しますが、`yield`を許容しません。

```fsharp
let s = seq {yield 1}    // OK
let s = seq {return 1}   // error

let a = async {return 1} // OK
let a = async {yield 1}  // error
```

実際には、`return` と `yield` で若干異なる動作をさせることができます。例えば、`return` を使用するとコンピュテーション式の残りの部分が評価されなくなりますが、`yield` では評価されません。

より一般的には、もちろん、`yield`はシーケンス/列挙セマンティクスに使用されるべきであり、`return`は通常、1つの式につき1回使用されます。(次の記事で `yield` を複数回使用する方法を見てみましょう。)

## "For "の再検討

前回の記事では、`for...in...do`構文について説明しました。今回は、前回説明した「リストビルダー」をもう一度見直して、追加のメソッドを追加してみましょう。前回の記事で、リストの `Bind` と `Return` を定義する方法を見ましたので、あとは追加のメソッドを実装するだけです。

* `Zero` メソッドは空のリストを返すだけです。
* `Yield` メソッドは `Return` と同じ方法で実装できます。
* `For` メソッドは `Bind` と同じように実装することができます。

```fsharp
type ListBuilder() =
    member this.Bind(m, f) =
        m |> List.collect f

    member this.Zero() =
        printfn "Zero"
        []

    member this.Return(x) =
        printfn "Return an unwrapped %A as a list" x
        [x]

    member this.Yield(x) =
        printfn "Yield an unwrapped %A as a list" x
        [x]

    member this.For(m,f) =
        printfn "For %A" m
        this.Bind(m,f)

// make an instance of the workflow
let listbuilder = new ListBuilder()
```

また、`let!`を使ったコードは以下の通りです。

```fsharp
listbuilder {
    let! x = [1..3]
    let! y = [10;20;30]
    return x + y
    } |> printfn "Result: %A"
```

また、`for`を使った同等のコードは以下の通りです。

```fsharp
listbuilder {
    for x in [1..3] do
    for y in [10;20;30] do
    return x + y
    } |> printfn "Result: %A"
```

どちらの方法でも同じ結果が得られることがわかります。

## まとめ

今回の記事では、簡単なコンピュテーション式の基本的なメソッドの実装方法を見てきました。

繰り返しになりますが、以下の点に注意してください。

* 簡単な式では、すべてのメソッドを実装する必要はありません。
* "!"のあるものは、右手側にラップされた型があります。
* "!"のないものは、右手側にラップされていない型を持ちます。
* 明示的に値を返さないワークフローが必要な場合は、`Zero`を実装する必要があります。
* `Yield` は基本的に `Return` と同等ですが、`Yield` はシーケンスや列挙のセマンティクスに使用する必要があります。
* `For` は基本的に、単純なケースでは `Bind` と同等です。

次回は、複数の値を結合する必要がある場合にどうなるかを見てみましょう。
