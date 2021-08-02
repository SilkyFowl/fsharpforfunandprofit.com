---
layout: post
title: "Computation expressions and wrapper types"
description: "Using types to assist the workflow"
date: 2013-01-23
nav: thinking-functionally
seriesId: "Computation Expressions"
seriesOrder: 4
---

前回の記事では、"maybe"ワークフローを紹介しました。このワークフローを使うと、オプションタイプを連結する際の面倒な作業を隠すことができます。

典型的な "maybe "ワークフローの使い方は、次のようなものです。

```fsharp
let result =
    maybe
        {
        let! anInt = expression of Option<int>
        let! anInt2 = expression of Option<int>
        return anInt + anInt2
        }
```

前に見たように、ここでは明らかにおかしな動作が行われています。

* `let!`の行では、equalsの*右*にある式は`int option`ですが、*左*にある値は単なる`int`です。`let!`では、オプションを値にバインドする前に、オプションを"アンラップ"しています。

* そして、`return`の行では、逆のことが起こります。返される式は `int` ですが、ワークフロー全体の値 (`result`) は `int option` です。つまり、`return`は生の値をオプションに"包み込んで"戻しているのです。

この記事では、これらの観察結果を追って、コンピュテーション式の主要な用途の1つである、ある種のラッパー型に格納された値を暗黙のうちにアンラップ、リラップすることにつながることを説明します。

## 別の例

別の例を見てみましょう。例えば、データベースにアクセスして、その結果をSuccess/Errorのユニオンタイプに格納したいとします。

```fsharp
type DbResult<'a> =
    | Success of 'a
    | Error of string
```

そして、この型をデータベースアクセスメソッドで使用します。ここでは、`DbResult`型の使用方法を知るために、非常にシンプルなスタブを紹介します。

```fsharp
let getCustomerId name =
    if (name = "")
    then Error "getCustomerId failed"
    else Success "Cust42"

let getLastOrderForCustomer custId =
    if (custId = "")
    then Error "getLastOrderForCustomer failed"
    else Success "Order123"

let getLastProductForOrder orderId =
    if (orderId  = "")
    then Error "getLastProductForOrder failed"
    else Success "Product456"
```


では、これらの呼び出しを連鎖させたいとしましょう。まず名前から顧客IDを取得し、次に顧客IDに対する注文を取得し、さらに注文から商品を取得します。

これが最も明確な方法です。ご覧のように、各ステップでパターンマッチングを行う必要があります。

```fsharp
let product =
    let r1 = getCustomerId "Alice"
    match r1 with
    | Error _ -> r1
    | Success custId ->
        let r2 = getLastOrderForCustomer custId
        match r2 with
        | Error _ -> r2
        | Success orderId ->
            let r3 = getLastProductForOrder orderId
            match r3 with
            | Error _ -> r3
            | Success productId ->
                printfn "Product is %s" productId
                r3
```

本当に醜いコードです。しかも、トップレベルのフローがエラー処理のロジックに埋もれてしまっています。

そこで、コンピュテーション式の出番です。 成功/エラーの分岐を裏で処理するコンピュテーション式を書けばいいのです。

```fsharp
type DbResultBuilder() =

    member this.Bind(m, f) =
        match m with
        | Error _ -> m
        | Success a ->
            printfn "\tSuccessful: %s" a
            f a

    member this.Return(x) =
        Success x

let dbresult = new DbResultBuilder()
```

このようなワークフローがあれば、全体像を把握し、よりクリーンなコードを書くことができます。

```fsharp
let product' =
    dbresult {
        let! custId = getCustomerId "Alice"
        let! orderId = getLastOrderForCustomer custId
        let! productId = getLastProductForOrder orderId
        printfn "Product is %s" productId
        return productId
        }
printfn "%A" product'
```

また、エラーが発生した場合は、ワークフローがうまくトラップして、以下の例のように、どこでエラーが発生したのかを教えてくれます。

```fsharp
let product'' =
    dbresult {
        let! custId = getCustomerId "Alice"
        let! orderId = getLastOrderForCustomer "" // error!
        let! productId = getLastProductForOrder orderId
        printfn "Product is %s" productId
        return productId
        }
printfn "%A" product''
```


## ワークフローにおけるラッパー型の役割

ここまで、2つのワークフロー（`maybe`ワークフローと`dbresult`ワークフロー）を見てきましたが、それぞれに対応するラッパータイプ（それぞれ`Option<T>`と`DbResult<T>`）があります。

これらは単なる特殊なケースではありません。実際には、すべてのコンピュテーション式には関連するラッパータイプが必要です。そして、そのラッパータイプは、管理したいワークフローと連動するように特別に設計されていることが多いのです。

上の例はこのことを明確に示しています。作成した `DbResult` 型は、単なる戻り値の型ではなく、ワークフローの現在の状態や、各ステップで成功しているか失敗しているかを"保存"することで、実際にワークフローで重要な役割を果たしています。型自体の様々なケースを利用することで、`dbresult`ワークフローは遷移を管理し、それらを見えないようにして、全体像に集中できるようにしてくれます。

優れたラッパー型の設計方法については後ほどご紹介しますが、まずはラッパー型がどのように操作されるのかを見てみましょう。


## Bind and Return とラッパータイプ

コンピュテーション式の `Bind` と `Return` メソッドの定義をもう一度見てみましょう。

まずは、簡単な方の `Return` から始めましょう。[MSDNに掲載されている](http://msdn.microsoft.com/en-us/library/dd233182.aspx)`Return`のシグネチャは以下の通りです。

```fsharp
member Return : 'T -> M<'T>
```

言い換えれば、ある型`T`に対して、`Return`メソッドはそれをラッパー型でラップするだけです。

*`M<int>` は `int` に適用されるラッパー型、`M<string>` は `string` に適用されるラッパー型、というようにです。*

そして、この使い方の2つの例を見てきました。`maybe`ワークフローは，オプション型である`Some`を返し，`dbresult`ワークフローは，`DbResult`型の一部である`Success`を返しています．

```fsharp
// 多分ワークフローのリターン
member this.Return(x) =
    Some x

// dbresultワークフローでのリターン
member this.Return(x) =
    Success x
```

次に、`Bind`を見てみましょう。 Bind`のシグネチャーは以下の通りです。

```fsharp
member Bind : M<'T> * ('T -> M<'U>) -> M<'U>
```

複雑そうなので、分解してみましょう。 これはタプル`M<'T> * ('T -> M<'U>)`を受け取り、`M<'U>`を返します。`M<'U>`は型`U`に適用されるラッパー型を意味します。

このタプルには2つの部分があります。

* `M<'T>` はタイプ `T` のラッパーです。
* `'T -> M<'U>` は *アンラップされた* `T` を受け取り、*ラップされた* `U` を作成する関数です。

言い換えると、`Bind`は次のことを行います。

* *ラッピング*された値を引数に取ります。
* ラップを解除し、特別な"舞台裏の"ロジックを実行します。
* 次に、オプションで関数を*アンラップ*された値に適用して、新しい*ラップされた*値を作成します。
* 関数が*適用されていない*場合でも、`Bind`は*ラッピングされた*`U`を返す必要があります。

このことを理解した上で、すでに見た`Bind`メソッドを以下に示します:

```fsharp
// return for the maybe workflow
member this.Bind(m,f) =
   match m with
   | None -> None
   | Some x -> f x

// return for the dbresult workflow
member this.Bind(m, f) =
    match m with
    | Error _ -> m
    | Success x ->
        printfn "\tSuccessful: %s" x
        f x
```

このコードを見て、なぜこれらのメソッドが上述のパターンに従っているのかを理解してください。

最後に、絵があると便利です。ここでは、さまざまなタイプと機能の図を示します。

![bindの図](./bind.png)

* `Bind` では、ラップされた値 (ここでは `m`) から始めて、それをアンラップして `T` 型の生の値にして、それから (たぶん) 関数 `f` を適用して `U` 型のラップされた値を得ます。
* `Return` については、値 (ここでは `x`) から始めて、それを単純にラップします。


### 型ラッパーはジェネリック

すべての関数は、ラッパー型以外のジェネリック型（`T`と`U`）を使用していることに注意してください。例えば，`maybe`というバインディング関数が，`int`を受け取って`Option<string>`を返したり，`string`を受け取って`Option<bool>`を返したりすることを妨げるものは何もありません． 唯一の条件は，常に `Option<something>` を返すことです。

このことを理解するために，上の例をもう一度見てみましょう．しかし，どこでも文字列を使用するのではなく，顧客ID，注文ID，商品IDのために特別な型を作成します．つまり、チェーンの各ステップで異なる型を使用することになります。

まず、型から始めて、今度は`CustomerId`などを定義します。

```fsharp
type DbResult<'a> =
    | Success of 'a
    | Error of string

type CustomerId =  CustomerId of string
type OrderId =  OrderId of int
type ProductId =  ProductId of string
```

`Success`の行で新しい型を使っていることを除けば，コードはほとんど同じです。

```fsharp
let getCustomerId name =
    if (name = "")
    then Error "getCustomerId failed"
    else Success (CustomerId "Cust42")

let getLastOrderForCustomer (CustomerId custId) =
    if (custId = "")
    then Error "getLastOrderForCustomer failed"
    else Success (OrderId 123)

let getLastProductForOrder (OrderId orderId) =
    if (orderId  = 0)
    then Error "getLastProductForOrder failed"
    else Success (ProductId "Product456")
```


ここでもう一度、長文のバージョンを紹介します。


```fsharp
let product =
    let r1 = getCustomerId "Alice"
    match r1 with
    | Error e -> Error e
    | Success custId ->
        let r2 = getLastOrderForCustomer custId
        match r2 with
        | Error e -> Error e
        | Success orderId ->
            let r3 = getLastProductForOrder orderId
            match r3 with
            | Error e -> Error e
            | Success productId ->
                printfn "Product is %A" productId
                r3
```

議論する価値のある変更点がいくつかあります。

* まず、一番下の `printfn` では、"%s"ではなく"%A"というフォーマット指定子を使用しています。これは、`ProductId`型がユニオン型になったために必要になりました。
* もっと微妙なところでは、エラー行に不要なコードがあるようです。なぜ `| Error e -> Error e` と書くのでしょうか。 その理由は、マッチされる入力エラーは `DbResult<CustomerId>` または `DbResult<OrderId>` 型ですが、*リターン* 値は `DbResult<ProductId>` 型でなければならないからです。つまり、2つの"Error"は同じように見えても、実際には異なるタイプのものなのです。

次にビルダーですが、`| Error e -> Error e`の行を除いては、全く変わっていません。

```fsharp
type DbResultBuilder() =

    member this.Bind(m, f) =
        match m with
        | Error e -> Error e
        | Success a ->
            printfn "\tSuccessful: %A" a
            f a

    member this.Return(x) =
        Success x

let dbresult = new DbResultBuilder()
```

最後に，先ほどのワークフローを使ってみましょう。

```fsharp
let product' =
    dbresult {
        let! custId = getCustomerId "Alice"
        let! orderId = getLastOrderForCustomer custId
        let! productId = getLastProductForOrder orderId
        printfn "Product is %A" productId
        return productId
        }
printfn "%A" product'
```

それぞれの行では、返される値の型が異なりますが（`DbResult<CustomerId>`、`DbResult<OrderId>`など）、ラッパーの型が共通しているため、期待通りのバインドが行われています。

最後に、エラーケースを想定したワークフローを紹介します。

```fsharp
let product'' =
    dbresult {
        let! custId = getCustomerId "Alice"
        let! orderId = getLastOrderForCustomer (CustomerId "") //エラー
        let! productId = getLastProductForOrder orderId
        printfn "Product is %A" productId
        return productId
        }
printfn "%A" product''
```


## コンピュテーション式の構成

すべてのコンピュテーション式には、必ず関連するラッパー型が必要であることを説明しました。このラッパータイプは `Bind` と `Return` の両方で使用され、これが重要な利点につながります。

* このラッパータイプは `Bind` と `Return` の両方で使用されます。

言い換えれば、ワークフローはラッパー型を返し、`let!` はラッパー型を消費するので、`let!` 式の右辺に"子"ワークフローを置くことができます。

例えば、`myworkflow`というワークフローがあるとします。その場合、以下のように書くことができます。

```fsharp
let subworkflow1 = myworkflow { return 42 }.
let subworkflow2 = myworkflow { return 43 }.

let aWrappedValue =
    myworkflow {
        let! unwrappedValue1 = subworkflow1
        let! unwrappedValue2 = subworkflow2
        return unwrappedValue1 + unwrappedValue2
        }
```

また、次のように"インライン"にすることもできます。

```fsharp
let aWrappedValue =
    myworkflow {
        let! unwrappedValue1 = myworkflow {
            let! x = myworkflow { return 1 }
            return x
            }
        let! unwrappedValue2 = myworkflow {
            let! y = myworkflow { return 2 }
            return y
            }
        return unwrappedValue1 + unwrappedValue2
        }
```

非同期ワークフローには通常、他の非同期ワークフローが組み込まれているので、`async`ワークフローを使用したことがある人は、すでにこの作業を行っているでしょう。

```fsharp
let a =
    async {
        let! x = doAsyncThing  // nested workflow
        let! y = doNextAsyncThing x // nested workflow
        return x + y
    }
```

## "ReturnFrom" の紹介

これまで、ラップされていない戻り値を簡単にラップする方法として、`return`を使ってきました。

しかし、時には、すでにラップされた値を返す関数があって、それを直接返したいことがあります。 しかし、`return` はラップされていない型を入力として必要とするため、これには適していません。

解決策は、`return`の変形である`return!`で、入力として*ラップされた型*を受け取り、それを返します。

ビルダー"クラスの対応するメソッドは、`ReturnFrom`と呼ばれます。通常、実装では、ラップされた型を"そのまま"返すだけです（もちろん、舞台裏で常に追加のロジックを追加することはできますが）。

ここでは、"maybe "のワークフローを変形して、どのように使用できるかを示します。

```fsharp
type MaybeBuilder() =
    member this.Bind(m, f) = Option.bind f m
    member this.Return(x) =
        printfn "Wrapping a raw value into an option"
        Some x
    member this.ReturnFrom(m) =
        printfn "Returning an option directly"
        m

let maybe = new MaybeBuilder()
```

通常の`return`と比較してみると、以下のようになります。

```fsharp
// return an int
maybe { return 1  }

// return an Option
maybe { return! (Some 2)  }
```

より現実的な例として、`return!`を`divideBy`と組み合わせて使ってみましょう。

```fsharp
// returnを使う
maybe
    {
    let! x = 12 |> divideBy 3
    let! y = x |> divideBy 2
    return y // 整数を返す
    }

// return!
maybe
    {
    let! x = 12 |> divideBy 3
    return! x |> divideBy 2 // オプションを返す
    }
```

## まとめ

今回は、ビルダークラスのコアメソッドである `Bind`, `Return`, `ReturnFrom` に関連するラッパータイプを紹介しました。

次の記事では、リストをラッパー型として使用することを含め、引き続きラッパー型について見ていきます。

