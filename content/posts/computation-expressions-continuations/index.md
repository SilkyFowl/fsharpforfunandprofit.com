---
layout: post
title: "Understanding continuations"
description: "How 'let' works behind the scenes"
date: 2013-01-21
nav: thinking-functionally
seriesId: "Computation Expressions"
seriesOrder: 2
---

前回の記事では、複雑なコードをコンピュテーション式を使って凝縮する方法を紹介しました。

以下は、コンピュテーション式を使う前のコードです。

```fsharp

let log p = printfn "expression is %A" p

let loggedWorkflow =
    let x = 42
    log x
    let y = 43
    log y
    let z = x + y
    log z
    //return
    z
```

また，コンピュテーション式を使った後の同じコードを示します．

```fsharp
let loggedWorkflow =
    logger
        {
        let! x = 42
        let! y = 43
        let! z = x + y
        return z
        }
```

通常の `let` ではなく `let!` を使用していることが重要です。 何が起こっているのかを理解するために、これを自分でエミュレートすることはできますか？ しかし、その前に連続性を理解する必要があります。

## 継続

命令型プログラミングでは、関数から「戻る」という概念があります。関数を呼び出すときには、スタックを押したり出したりするように、「入って」から「出て」きます。

以下は，このように動作するC#の典型的なコードです。`return`キーワードの使用に注目してください。

```csharp
public int Divide(int top, int bottom)
{
    if (bottom==0)
    {
        throw new InvalidOperationException("div by 0");
    }
    else
    {
        return top/bottom;
    }
}

public bool IsEven(int aNumber)
{
    var isEven = (aNumber % 2 == 0);
    return isEven;
}
```

これは何度も見たことがあると思いますが、このアプローチには、考えもしなかった微妙なポイントがあります。それは、*呼び出された関数が常に何をすべきかを決める*ということです。

例えば、`Divide`の実装は、例外を投げることを決定しています。 しかし、もし例外が欲しくない場合はどうでしょうか？もしかしたら`nullable<int>`が欲しいかもしれませんし、「#DIV/0」として画面に表示するつもりかもしれません。なぜ、すぐにキャッチしなければならないような例外を投げるのでしょうか？ 言い換えれば，何が起こるべきかを呼び出し側ではなく，呼び出し元に決定させればよいのです。

同様に、`IsEven`の例では、ブール値の戻り値をどうすればいいのでしょうか？分岐するのでしょうか？それとも、レポートに出力するのでしょうか？わかりませんが、呼び出し側が処理しなければならない真偽値を返すよりも、呼び出し側が次に何をすべきかを呼び出し側に伝えればいいのではないでしょうか。

これが継続というものです。 **継続**とは、単純に、他の関数に次の動作を伝えるために渡す関数のことです。

以下は同じC#のコードを、呼び出し側が関数を渡すことができるように書き換えたもので、呼び出し側はそれぞれのケースを処理するために着呼側が使用します。参考になればと思いますが、これはビジターパターンに似ていると考えられます。そうではないかもしれませんが。

```csharp
public T Divide<T>(int top, int bottom, Func<T> ifZero, Func<int,T> ifSuccess)
{
    if (bottom==0)
    {
        return ifZero();
    }
    else
    {
        return ifSuccess( top/bottom );
    }
}

public T IsEven<T>(int aNumber, Func<int,T> ifOdd, Func<int,T> ifEven)
{
    if (aNumber % 2 == 0)
    {
        return ifEven(aNumber);
    }
    else
    {   return ifOdd(aNumber);
    }
}
```

C#の関数は一般的な`T`を返すように変更されていて、どちらの継続も`T`を返す`Func`であることに注意してください。

さて、C#ではたくさんの`Func`パラメータを渡すことは、常にかなり見苦しいので、あまり行われません。 しかし、F#では関数を渡すのは簡単なので、このコードがどのように移植されるか見てみましょう。

これが「前」のコードです。

```fsharp
let divide top bottom =
    if (bottom=0)
    then invalidOp "div by 0"
    else (top/bottom)

let isEven aNumber =
    aNumber % 2 = 0
```

そして、これが "後 "のコードです。

```fsharp
let divide ifZero ifSuccess top bottom =
    if (bottom=0)
    then ifZero()
    else ifSuccess (top/bottom)

let isEven ifOdd ifEven aNumber =
    if (aNumber % 2 = 0)
    then aNumber |> ifEven
    else aNumber |> ifOdd
```

いくつか注意点があります。まず、パラメータリスト中に追加関数 (`ifZero`など) を、C#の例のように最後ではなく*最初*に置いたことが分かると思います。なぜ?おそらく [部分適用] (/posts/partial-application/) を使いたいからです。

また、`isEven`の例では、`aNumber|>ifEven`および`aNumber|>ifOdd`と記述しました。これにより、現在の値を継続にパイプしていることが明確になります。つまり、継続が評価されるのは常に最後です。*この記事の後半でもまったく同じパターンを使用するので、ここで何が起こっているかを理解しておいてください。*

### 継続の例

継続の力を利用すれば、同じ `divide` 関数でも、呼び出し側の要求に応じて、3つの全く異なる方法で使用することができます。

ここでは、すぐに作成できる3つのシナリオを紹介します。

* 結果をメッセージにパイプして、それを出力する。
* 悪い場合は `None` を、良い場合は `Some` を使って、結果をoptionに変換する。
* あるいは、悪いケースでは例外を発生させ、良いケースでは結果を返す。

```fsharp
// シナリオ1：結果をメッセージにパイプする
// ----------------------------------------
// メッセージを表示するように関数を設定する
let ifZero1 () = printfn "bad"
let ifSuccess1 x = printfn "good %i" x

// 部分適用をする
let divide1 = divide ifZero1 ifSuccess1

//テスト
let good1 = divide1 6 3
let bad1 = divide1 6 0

// シナリオ2：結果をoptionに変換する
// ----------------------------------------
// optionを返すように関数を設定する
let ifZero2() = None
let ifSuccess2 x = Some x
let divide2 = divide ifZero2 ifSuccess2

//テスト
let good2 = divide2 6 3
let bad2 = divide2 6 0

// シナリオ3：悪いケースでは例外を投げる
// ----------------------------------------
// 例外を発生させる関数の設定
let ifZero3() = failwith "div by 0"
let ifSuccess3 x = x
let divide3  = divide ifZero3 ifSuccess3

//テスト
let good3 = divide3 6 3
let bad3 = divide3 6 0
```

このアプローチでは、呼び出し元が`divide`から例外をキャッチする必要は*決して*ありません。例外をスローするかどうかは、呼出し先ではなく呼出し元が決定します。そのため、 `divide`関数がさまざまな状況で大幅に再利用可能になっただけでなく、循環的複雑さのレベルも下がっています。

`isEven`の実装に同様の3つのシナリオを適用できます:

```fsharp
// シナリオ 1: 結果をメッセージにパイプする
// ----------------------------------------
// メッセージを表示するための関数を設定する
let ifOdd1 x = printfn "isOdd %i" x
let ifEven1 x = printfn "isEven %i" x

// 部分適用をする
let isEven1 = isEven ifOdd1 ifEven1

//テスト
let good1 = isEven1 6
let bad1 = isEven1 5

// シナリオ2：結果をoptionに変換する
// ----------------------------------------
// optionを返すように関数を設定する
let ifOdd2 _ = None
let ifEven2 x = Some x
let isEven2 = isEven ifOdd2 ifEven2

//テスト
let good2 = isEven2 6
let bad2 = isEven2 5

// シナリオ3：悪いケースでは例外を投げる
// ----------------------------------------
// 例外を投げるための関数を設定する
let ifOdd3 _ = failwith "assert failed"
let ifEven3 x = x
let isEven3 = isEven ifOdd3 ifEven3

//テスト
let good3 = isEven3 6
let bad3 = isEven3 5
```

この場合、利点はわずかですが同様です。呼出し元が`if/then/else`でブール値を処理する必要はありません。複雑さが軽減され、エラーが発生する可能性も低くなります。

些細な違いのように思えるかもしれませんが、このように関数を渡せば、合成、部分適用といった、お気に入りの関数技法をすべて使うことができます。

また、 [型による設計](/posts/designing-with-types-single-case-dus/) のシリーズの中でも継続を見てきました。これらを使用することで、呼び出し側は、単に例外をスローするのではなく、コンストラクタで検証エラーが発生した場合に何が起こるか決定できることを見てきました。
```fsharp
type EmailAddress = EmailAddress of string

let CreateEmailAddressWithContinuations success failure (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then success (EmailAddress s)
        else failure "Email address must contain an @ sign"
```

success関数はEmailをパラメータとして受け取り、error関数は文字列を受け取ります。どちらの関数も同じ型を返さなければなりませんが、型は自由に決められます。

そして、ここでは継続を使った簡単な例を紹介します。どちらの関数もprintfを行い、何も返しません（つまりunit）。

```fsharp
// 関数の設定
let success (EmailAddress s) = printfn "success creating email %s" s
let failure msg = printfn "error creating email: s" msg
let createEmail = CreateEmailAddressWithContinuations success failure

// テスト
let goodEmail = createEmail "x@example.com"
let badEmail = createEmail "example.com"
```

### 継続渡しスタイル

このように継続を使用するプログラミングスタイルは、「[継続渡しスタイル](https://ja.wikipedia.org/wiki/%E7%B6%99%E7%B6%9A%E6%B8%A1%E3%81%97%E3%82%B9%E3%82%BF%E3%82%A4%E3%83%AB)」（またはCPS）と呼ばれます。これは、*すべての*関数に「次に何をするか」という関数パラメータを追加して呼び出すものです。

この違いを理解するために、標準的なダイレクトスタイルのプログラミングを見てみましょう。

ダイレクトスタイルでは、次のように関数を「入れたり」「出したり」します。

```text
関数を呼び出す ->
   <- その関数から戻る
別の関数を呼び出す ->
   <- その関数から戻る
さらに別の関数を呼び出す ->
   <- その関数から戻る
```

一方、継続的に渡すスタイルでは、次のような関数の連鎖ができてます。

```text
何かを評価して、それを -> に渡す
   何かを評価して、それを -> に渡す関数
      何かを評価してそれを -> に渡す別の関数
         何かを評価してそれを渡す別の関数 -> 何かを評価してそれを渡す別の関数
            ...etc...
```

2つのスタイルには明白な違いがあります。

ダイレクトスタイルには、関数の階層があります。トップレベルの関数は一種の"マスターコントローラー"で、あるサブルーチンを呼び出してから別のサブルーチンを呼び出し、いつ分岐するか、どの時点でループするかを決定し、概して制御フローを明示的に調整します。

しかし,継越渡しスタイルには “マスターコントローラ”という概念はありません。その代わり、データではなく制御フローに "パイプライン"のようなものがあり、実行ロジックがパイプを流れると"担当関数"が変更されます。

GUIのボタンクリックにイベントハンドラをアタッチや、 [BeginInvoke](http://msdn.microsoft.com/en-us/library/2e08f6yc.aspx)でコールバックを使用したことがある場合は、このスタイルを意識せずに使用しています。このスタイルこそが“async”ワークフローを理解するための鍵となります。このワークフローについては、このシリーズの後半で説明します。


## 続行と 'let' ##

では、`let`とはどういう関係なのでしょうか?

前に戻って、 `let`が実際に行うことを [再確認](/posts/let-use-do/) しましょう。
(トップレベルでない) "let"は決して単独で使用することはできず、常により大きなコードブロックの一部でなければなりません。

つまり、以下のような式の場合:

```fsharp
let x = someExpression
```

実際に行っているのはこうです:

```fsharp
let x = someExpression in [xを含む式]
```

そして、2番目の式 (本体式) の中に`x`があるたびに、それを最初の式 (`someExpression`) で置き換えています。

たとえば、次のような式があるならば:

```fsharp
let x = 42
let y = 43
let z = x + y
```

実際には次のようになります (冗長な`in`キーワードを使用) :

```fsharp
let x = 42 in
  let y = 43 in
    let z = x + y in
       z    // 結果
```

面白いことに、ラムダは `let` と非常によく似ています。

```fsharp
fun x -> [xを含む式]
```

そして、`x`の値もパイプで入力すると、次のようになります。

```fsharp
someExpression |> (fun x -> [xを含む式] )
```

これは `let` に似ていると思いませんか？letとlambdaを並べてみましょう。

```fsharp
// let
let x = someExpression in [xを含む式]

// 値をラムダにパイプする
someExpression |> (fun x -> [xを含む式] )
```

どちらも `x` と `someExpression` を持っていて、ラムダの本体に `x` があるところはすべて `someExpression` に置き換えています。
確かに、ラムダの場合は `x` と `someExpression` が逆になりますが、それ以外は基本的に `let` と同じです。

このテクニックを使うと，元の例を次のようなスタイルに書き換えることができます．

```fsharp
42 |> (fun x ->
  43 |> (fun y ->
     x + y |> (fun z ->
       z)))
```

このように書くと、 `let`スタイルが継続渡しスタイルに変化したことがわかります!

* 最初の行には 「42」 という値があります--これを使って何をしますか?先ほどの 「isEven」 関数と同様、継続に渡しましょう。そして、継続の文脈においては、 `42`の名前を`x`というようにします。
* 2行目には 「43」 という値があります--これをどうしますか?これも、継続に渡して、そのコンテキストで は`y`と呼びましょう。
* 3行目では、xとyを加算して新しい値を作成します。それで何をしましょ?さらに別の継続へ、別のラベル (`z`) を追加します。
* 最後の行で`z`と評価され、処理が終了しました。

### 関数で継続をラップする

明示的なパイプを取り除き、このロジックをラップする小さな関数を作成しましょう。これは予約語であるため、「レット」と呼ぶことはできません。さらに重要なのは、パラメータが'let'から逆になっていることです。
右に"x"、左に"someExpression"があります。そこで、これを `pipeInto`と呼ぶことにします。

`pipeInto`の定義はとても明白です。

```fsharp
let pipeInto (someExpression,lambda) =
    someExpression |> lambda
```

*ここでは、空白で区切られた2つの異なるパラメータとしてではなく、タプルを使用して両方のパラメータを一度に渡していることに注意してください。必ずペアでお届けします。*

この`pipeInto`関数を使用して、例を再度次のように書き直すことができます:

```fsharp
pipeInto (42, fun x ->
  pipeInto (43, fun y ->
    pipeInto (x + y, fun z ->
       z)))
```

あるいは、インデントをなくして、次のように書くこともできます。

```fsharp
pipeInto (42, fun x ->
pipeInto (43, fun y ->
pipeInto (x + y, fun z ->
z)))
```

あなたはこう思うかもしれません：だから何？なぜわざわざパイプを関数にまとめるの？

その答えは、`pipeInto`関数の中へコンピュテーション式のように「裏」で何かをするための *追加コード* を追加できるからです。

### 「ロギング」の例の再検討 ###

次のように `pipeInto` を再定義して、ちょっとしたロギング機能を追加してみましょう。

```fsharp
let pipeInto (someExpression,lambda) =
   printfn "expression is %A" someExpression
   someExpression |> lambda
```

さて、このコードをもう一度実行してみましょう。

```fsharp
pipeInto (42, fun x ->
pipeInto (43, fun y ->
pipeInto (x + y, fun z ->
z
)))
```

出力結果は？

```text
expression is 42
expression is 43
expression is 85
```

これは、以前の実装で得られたものとまったく同じ出力です。 これで、自作のコンピュテーション式ワークフローが完成しました。

これをコンピュテーション式バージョンと並べて比較してみると、パラメータを逆にしていることと、継続のための明示的な矢印があることを除いて、自作バージョンは `let!` と非常によく似ていることがわかります。

![computation expression: logging](./compexpr_logging.png)

### 「安全な分割」の例の再検討 ###

同じことを「安全な割り算」の例でやってみましょう。オリジナルのコードは以下の通りです。

```fsharp
let divideBy bottom top =
    if bottom = 0
    then None
    else Some(top/bottom)

let divideByWorkflow x y w z =
    let a = x |> divideBy y
    match a with
    | None -> None  // give up
    | Some a' ->    // keep going
        let b = a' |> divideBy w
        match b with
        | None -> None  // give up
        | Some b' ->    // keep going
            let c = b' |> divideBy z
            match c with
            | None -> None  // give up
            | Some c' ->    // keep going
                //return
                Some c'
```

この "段階的"なスタイルは、本当に継続を使うべきだという明白な手がかりであることがおわかりいただけると思います。

では、`pipeInto`にマッチングを行うコードを追加できるかどうか見てみましょう。必要なロジックは以下の通りです。

* `someExpression` パラメータが `None` であれば、継続ラムダを呼び出さない。
* `someExpression` パラメータが `Some` であれば、`Some` の内容を渡して、継続ラムダを呼び出します。

これがそうです:

```fsharp
let pipeInto (someExpression,lambda) =
   match someExpression with
   | None ->
       None
   | Some x ->
       x |> lambda
```

この新しいバージョンの`pipeInto`を使うと、元のコードを次のように書き換えることができます。

```fsharp
let divideByWorkflow x y w z =
    let a = x |> divideBy y
    pipeInto (a, fun a' ->
        let b = a' |> divideBy w
        pipeInto (b, fun b' ->
            let c = b' |> divideBy z
            pipeInto (c, fun c' ->
                Some c' //return
                )))
```

これはかなりきれいにすることができます。

まず、`a`、`b`、`c`を削除して、`divideBy`式で直接置き換えることができます。つまり，次のようになります。

```fsharp
let a = x |> divideBy y
pipeInto (a, fun a' ->
```

これをこうします:

```fsharp
pipeInto (x |> divideBy y, fun a' ->
```

ここで、`a'`をただの`a`とラベル付けし直し、さらに段差のあるインデントを削除すると、次のようになります。

```fsharp
let divideByResult x y w z =
    pipeInto (x |> divideBy y, fun a ->
    pipeInto (a |> divideBy w, fun b ->
    pipeInto (b |> divideBy z, fun c ->
    Some c //return
    )))
```

最後に、結果をoptionにまとめるために、`return'`という小さなヘルパー関数を作ります。すべてをまとめると、コードは次のようになります。

```fsharp
let divideBy bottom top =
    if bottom = 0
    then None
    else Some(top/bottom)

let pipeInto (someExpression,lambda) =
   match someExpression with
   | None ->
       None
   | Some x ->
       x |> lambda

let return' c = Some c

let divideByWorkflow x y w z =
    pipeInto (x |> divideBy y, fun a ->
    pipeInto (a |> divideBy w, fun b ->
    pipeInto (b |> divideBy z, fun c ->
    return' c
    )))

let good = divideByWorkflow 12 3 2 1
let bad = divideByWorkflow 12 3 0 1
```

もう一度、コンピュテーション式バージョンと並べて比較してみると、自作バージョンは意味が同じであることがわかります。シンタックスだけが違うのです。

![computation expression: logging](./compexpr_safedivide.png)

###まとめ

この記事では、継続と継続の渡し方、そして `let` を継続を裏で行うための良い構文と考えることができることについて説明しました。

これで、*独自*バージョンの `let` を作り始めるために必要なものがすべて揃いました。次の記事では、この知識を実際に使ってみましょう。

