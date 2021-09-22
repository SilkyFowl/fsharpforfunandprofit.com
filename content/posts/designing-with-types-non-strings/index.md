---
layout: post
title: "Designing with types: Non-string types"
description: "Working with integers and dates safely"
date: 2013-01-18
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 7
categories: [Types, DDD]
---

このシリーズでは、文字列をラップするためのシングルケース判別式ユニオンの使い方をたくさん見てきました。

このテクニックを、数字や日付などの他のプリミティブな型に使えない理由はありません。 いくつかの例を見てみましょう。

## シングルケースユニオン

多くの場合、異なる種類の整数を誤って混ぜてしまうことは避けたいものです。2つのドメインオブジェクトが（整数を使って）同じ表現をしていても、決して混同してはいけません。

例えば、`OrderId`と`CustomerId`があり、両方ともintとして格納されているかもしれません。しかし、それらは本当の意味での整数ではありません。例えば、`CustomerId`に42を加えることはできません。
そして、`CustomerId(42)` は `OrderId(42)` と等しくありません。実際のところ、これらは比較することさえ許されていません。

もちろん、型が助けてくれます。

```fsharp
type CustomerId = CustomerId of int
type OrderId = OrderId of int

let custId = CustomerId 42
let orderId = OrderId 42

// コンパイラエラー
printfn "cust is equal to order? %b" (custId = orderId)
```

同様に、意味的に異なる日付の値を型で囲むことで、それらの値が混ざらないようにしたい場合もあるでしょう。(`DateTimeKind`はこの試みですが、必ずしも信頼できるものではありません。)

```fsharp
type LocalDttm = LocalDttm of System.DateTime
type UtcDttm = UtcDttm of System.DateTime
```

これらの型を使えば、常に正しい種類のdatetimeをパラメータとして渡すことができます。さらに、これはドキュメントとしても機能します。

```fsharp
let SetOrderDate (d:LocalDttm) =
    () // do something

let SetAuditTimestamp (d:UtcDttm) =
    () // do something
```

## 整数の制約

`String50`や `ZipCode`のような型に対する検証や制約があったように、整数に対する制約が必要な場合にも同じアプローチをとることができます。

例えば、在庫管理システムやショッピングカートでは、ある種の数値が常に正の値であることが求められるかもしれません。 `NonNegativeInt`という型を作ることで、これを保証することができます。

```fsharp
module NonNegativeInt =
    type T = NonNegativeInt of int

    let create i =
        if (i >= 0 )
        then Some (NonNegativeInt i)
        else None

module InventoryManager =

    // example of NonNegativeInt in use
    let SetStockQuantity (i:NonNegativeInt.T) =
        //set stock
        ()
```

## ビジネスルールを型に埋め込む

先ほど、ファーストネームの長さが64K文字になることがあるかどうか疑問に思ったように、ショッピングカートに99999個のアイテムを追加することは本当にできるのでしょうか？

![状態遷移図: パッケージ配送](./AddToCart.png)

制約付きの型を使ってこの問題を回避する価値はあるでしょうか？実際のコードを見てみましょう。

ここでは、数量に標準的な `int` 型を使用した、非常にシンプルなショッピングカートマネージャを紹介します。関連するボタンがクリックされると、数量が増加または減少します。明らかなバグを見つけられるでしょうか？

```fsharp
モジュール ShoppingCartWithBug =

    let mutable itemQty = 1 // 家ではやらないでください!

    let incrementClicked() =
        itemQty <- itemQty + 1

    let decrementClicked() =
        itemQty <- itemQty - 1
```

もし、すぐにバグを見つけられないのであれば、おそらく、どんな制約でももっと明示的にすることを検討すべきでしょう。

以下は同じシンプルなショッピングカートマネージャで、代わりに型付けされた数量を使用しています。これでバグを見つけられるでしょうか？ (ヒント: コードをF#スクリプトファイルに貼り付けて実行してください)

```fsharp
module ShoppingCartQty =

    type T = ShoppingCartQty of int

    let initialValue = ShoppingCartQty 1

    let create i =
        if (i > 0 && i < 100)
        then Some (ShoppingCartQty i)
        else None

    let increment t = create (t + 1)
    let decrement t = create (t - 1)

module ShoppingCartWithTypedQty =

    let mutable itemQty = ShoppingCartQty.initialValue

    let incrementClicked() =
        itemQty <- ShoppingCartQty.increment itemQty

    let decrementClicked() =
        itemQty <- ShoppingCartQty.decrement itemQty
```

このような些細な問題に対して、これはやりすぎだと思うかもしれません。しかし、DailyWTFに載ることを避けたいのであれば、検討する価値はあるかもしれません。

{{< book_page_ddd >}}

## 日付の制約

すべてのシステムがすべての可能な日付を扱えるわけではありません。1980年1月1日までの日付しか保存できないシステムもあれば、2038年までの未来の日付しか保存できないシステムもあります (私は、月/日の順序に関する米国/英国の問題を避けるために、2038年1月1日を最大の日付として使用しています)。

整数と同じように、型に有効な日付の制約を組み込んでおくと、境界を越えた問題に後から対処するのではなく、構築時に対処することができて便利かもしれません。

```fsharp
type SafeDate = SafeDate of System.DateTime

let create dttm =
    let min = new System.DateTime(1980,1,1)
    let max = new System.DateTime(2038,1,1)
    if dttm < min || dttm > max
    then None
    else Some (SafeDate dttm)
```


## ユニオンタイプと単位の比較

この時点で疑問に思うかもしれません。[units of measure](/posts/units-of-measure/)ってどうなの？この目的のために使われるものではないのか？

はい、そして違います。 単位は、異なる種類の数値が混ざらないようにするために使用することができ、これまで使用してきたシングルケースユニオンよりもはるかに強力です。

その一方で、単位はカプセル化されておらず、制約を持つことができません。誰もがintの単位を`<kg>`として作成することができ、最小値も最大値もありません。

多くの場合、どちらのアプローチもうまくいきます。 例えば、.NETライブラリにはタイムアウトを使用する部分が多くありますが、タイムアウトは秒単位で設定されることもあれば、ミリ秒単位で設定されることもあります。
どっちがどっちだか覚えられないことがよくあります。本当は1000ミリ秒のタイムアウトを指定しているのに、誤って1000秒のタイムアウトを指定してしまうことは絶対に避けたいものです。

このような事態を避けるために、私は秒とミリ秒に別々の型を作りたいと思っています。

ここでは、シングルケースユニオンを使った型ベースのアプローチを紹介します。

```fsharp
type TimeoutSecs = TimeoutSecs of int
type TimeoutMs = TimeoutMs of int

let toMs (TimeoutSecs secs)  =
    TimeoutMs (secs * 1000)

let toSecs (TimeoutMs ms) =
    TimeoutSecs (ms / 1000)

/// sleep for a certain number of milliseconds
let sleep (TimeoutMs ms) =
    System.Threading.Thread.Sleep ms

/// timeout after a certain number of seconds
let commandTimeout (TimeoutSecs s) (cmd:System.Data.IDbCommand) =
    cmd.CommandTimeout <- s
```

また、単位を使って同じことをしてみましょう。

```fsharp
[<Measure>] type sec
[<Measure>] type ms

let toMs (secs:int<sec>) =
    secs * 1000<ms/sec>

let toSecs (ms:int<ms>) =
    ms / 1000<ms/sec>

/// sleep for a certain number of milliseconds
let sleep (ms:int<ms>) =
    System.Threading.Thread.Sleep (ms * 1<_>)

/// timeout after a certain number of seconds
let commandTimeout (s:int<sec>) (cmd:System.Data.IDbCommand) =
    cmd.CommandTimeout <- (s * 1<_>)
```

どちらのアプローチが良いでしょうか？

もし、たくさんの算術演算（足し算、掛け算など）をするのであれば、測定単位のアプローチの方がはるかに便利ですが、そうでなければ、どちらを選ぶべきかはあまりありません。


