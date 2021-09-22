---
layout: post
title: "Designing with types: Making state explicit"
description: "Using state machines to ensure correctness"
date: 2013-01-16
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 5
categories: [Types, DDD]
---

今回の記事では、ステートマシンを使って暗黙の状態を明示し、そのステートマシンをユニオン型でモデル化することを考えます。

## 背景 ##

このシリーズの[以前の記事](/posts/designing-with-types-single-case-dus/)では、メールアドレスのような型のラッパーとして、シングルケースユニオンを見ました。

```fsharp
モジュール EmailAddress =

    type T = EmailAddress of string

    let create (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^_S+@\S+\.\S+$")
            then Some (EmailAddress s)
            else None
```

このコードでは、アドレスが有効か無効かを想定しています。もし有効でなければ、完全に拒否し、有効な値の代わりに `None` を返します。

しかし、有効性には程度があります。例えば、無効なメールアドレスを単に拒否するのではなく、残しておきたい場合はどうすればよいでしょうか。 この場合は、いつものように型システムを使って、有効なアドレスと無効なアドレスが混ざらないようにしたいと思います。

そのためには、ユニオン型を使うのが当然です。
```fsharp
module EmailAddress =

    type T =
        | ValidEmailAddress of string
        | InvalidEmailAddress of string

    let create (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then ValidEmailAddress s    // change result type
            else InvalidEmailAddress s  // change result type

    // test
    let valid = create "abc@example.com"
    let invalid = create "example.com"
```

これらの型を使って、有効なメールだけが送信されるようにすることができます。

```fsharp
let sendMessageTo t =
    match t with
    | ValidEmailAddress email ->
         // send email
    | InvalidEmailAddress _ ->
         // ignore
```

ここまではいいでしょう。このような設計は、もうあなたには明らかでしょう。

しかし、このアプローチは思ったよりも広く適用できるものです。 多くの場面で、明示されていない似たような「状態」があり、コードの中でフラグや列挙型、条件付きロジックで処理されています。

## ステートマシン ##

上の例では、"有効 "と "無効 "のケースは相互に両立しません。つまり、有効なメールが無効になることはなく、その逆もまた然りです。

しかし、多くの場合、何らかのイベントをきっかけにして、あるケースから別のケースに移行することが可能です。この場合、各ケースが「状態」を表し、ある状態から別の状態に移行することが「遷移」である、["ステートマシン"](http://en.wikipedia.org/wiki/Finite-state_machine)ができあがります。

いくつかの例を挙げましょう。

* メールアドレスには「Unverified」と「Verified」という状態があり、ユーザーに確認メールのリンクをクリックしてもらうことで、「Unverified」の状態から「Verified」の状態へと移行することができます。
![状態遷移図：Verified Email](./State_VerifiedEmail.png)

* ショッピングカートには、「Empty」「Active」「Paid」という状態があり、「Empty」の状態から、カートに商品を追加することで「Active」の状態に、お金を支払うことで「Paid」の状態に遷移することができます。
![状態遷移図：ショッピングカート](./State_ShoppingCart.png)

* チェスなどのゲームでは、「WhiteToPlay」、「BlackToPlay」、「GameOver」という状態があり、白がゲームを終了させない手を打つことで「WhiteToPlay」状態から「BlackToPlay」状態に移行したり、チェックメイトの手を打つことで「GameOver」状態に移行したりします。
![状態遷移図: チェスゲーム](./State_Chess.png)

これらのケースでは、状態のセット、遷移のセット、および遷移を引き起こすイベントがあります。
ステートマシンは多くの場合、ショッピングカートのような表で表されます。

{{<rawtable>}}
<table class="table table-condensed">
<thead>
<tr>
<th>現在の状態</th>
<th>イベント</th>
<th>アイテムの追加</th>
<th>アイテムの削除</th>
<th>支払う</th>
</tr>
</thead>
<tbody>
<tr>
<th>空の状態</th>
<td></td>
<td>新しい状態 = アクティブ</td>
<td>該当なし</td>
<td>該当なし</td>
</tr>
<tr>
<th>アクティブ</th>
<td></td>
<td>新しい状態 = アクティブ</td>
<td>new state =<br>アイテム数に応じて</br>、ActiveまたはEmpty</td>
<td>新しい状態 = 有料</td>
</tr>
<tr>
<th>Paid</th>
<td></td>
<td>n/a</td>
<td>n/a</td>
<td>n/a</td>
</tr>
</tbody>
</table>
{{</rawtable>}}

このようなテーブルがあれば、システムが特定の状態にあるときに、各イベントに対して何が起こるべきかを素早く正確に把握することができます。

{{< linktarget "why-use" >}}

## なぜステートマシンを使うのか？

このようなケースでステートマシンを使用することには、多くの利点があります。

**各状態には異なる許容される動作を持たせることができます。**

メール認証の例では、パスワードのリセットは認証済みのメールアドレスにのみ送信でき、未認証のアドレスには送信できないというビジネスルールがあるでしょう。
また、ショッピングカートの例では、アクティブなカートだけが支払い可能であり、支払い済みのカートは追加できません。

**すべての状態が明示的に文書化されています**

暗黙的でありながら文書化されていない重要な状態を持つことは、あまりにも簡単です。

例えば、「空のカート」は「アクティブなカート」とは異なる動作をしますが、これがコード上で明示的に文書化されていることは稀でしょう。

**これは、起こりうるすべての可能性を考慮することを強いる設計ツールです。**

エラーの原因として、ある種のエッジケースが処理されていないことが挙げられますが、ステートマシンではすべてのケースを考えなければなりません。

例えば、既に検証済みのメールを検証しようとするとどうなるか？
空のショッピングカートから商品を削除しようとするとどうなるか？
状態が「BlackToPlay」のときに白がプレイしようとしたらどうなるか？などなど。


## 簡単なステートマシンをF#で実装する方法 ##

言語パーサーや正規表現で使われるような複雑なステートマシンには慣れているだろう。 この種のステートマシンは、ルールセットや文法から生成され、非常に複雑です。

しかし、ここでお話しするステートマシンは、もっともっとシンプルなものです。せいぜいいくつかのケースがあり、遷移の数も少ないので、複雑なジェネレータを使う必要はありません。

では、このようなシンプルなステートマシンを実装するには、どのような方法があるのでしょうか。

一般的には、それぞれの状態に関連するデータ（もしあれば）を格納するために、それぞれの状態に応じた型を持ち、状態のセット全体をユニオンクラスで表現します。

ここでは、ショッピングカートのステートマシンの例を紹介します。

```fsharp
type ActiveCartData = { UnpaidItems: string list }
type PaidCartData = { PaidItems: string list; Payment: float }

type ShoppingCart =
    | EmptyCart  // no data
    | ActiveCart of ActiveCartData
    | PaidCart of PaidCartData
```

なお、`EmptyCart`の状態はデータがないので、特別な型は必要ありません。

各イベントは、ステートマシン全体（ユニオン型）を受け取り、ステートマシンの新しいバージョン（これもユニオン型）を返す関数で表されます。

以下は、ショッピングカートのイベントを2つ使った例です。

```fsharp
let addItem cart item =
    match cart with
    | EmptyCart ->
        // create a new active cart with one item
        ActiveCart {UnpaidItems=[item]}
    | ActiveCart {UnpaidItems=existingItems} ->
        // create a new ActiveCart with the item added
        ActiveCart {UnpaidItems = item :: existingItems}
    | PaidCart _ ->
        // ignore
        cart

let makePayment cart payment =
    match cart with
    | EmptyCart ->
        // ignore
        cart
    | ActiveCart {UnpaidItems=existingItems} ->
        // create a new PaidCart with the payment
        PaidCart {PaidItems = existingItems; Payment=payment}
    | PaidCart _ ->
        // ignore
        cart
```

呼び出し側から見ると、状態の集合は一般的な操作（`ShoppingCart`型）のために「一つのもの」として扱われていますが、内部でイベントを処理する際には、それぞれの状態が別々に扱われていることがわかります。

### イベント処理関数の設計

ガイドライン ガイドライン: *イベント処理関数は常にステートマシン全体を受け取り、返すべきである*。

なぜイベント処理関数にショッピングカート全体を渡す必要があるのか、という疑問があるかもしれません。例えば、`makePayment`イベントは、カートがActive状態のときにのみ意味を持つので、次のように明示的にActiveCartタイプを渡せばいいのではないでしょうか。

```fsharp
let makePayment2 activeCart payment =
    let {UnpaidItems=existingItems} = activeCart
    {PaidItems = existingItems; Payment=payment}
```

関数のシグネチャを比較してみましょう。

```fsharp
// the original function
val makePayment : ShoppingCart -> float -> ShoppingCart

// the new more specific function
val makePayment2 :  ActiveCartData -> float -> PaidCartData
```

オリジナルの`makePayment`関数はカートを受け取って結果を得るのに対し、新しい関数は`ActiveCartData`を受け取って結果を`PaidCartData`にするので、より関連性が高いと思われることがわかります。

しかし、このようにした場合、カートが空や支払い済みなどの異なる状態になったときに、同じイベントをどのように処理するのでしょうか。 3つの状態のイベントをどこかで処理しなければなりません。このビジネスロジックを関数内にカプセル化する方が、呼び出し元の言いなりになるよりもずっと良いでしょう。

### 「生」の状態を扱う

場合によっては、1つのステートを独立した別のエンティティとして扱い、単独で使用する必要があります。各ステートはタイプでもあるので、通常は簡単にできます。

例えば、すべての支払い済みカートについてレポートする必要がある場合、`PaidCartData`のリストを渡すことができます。

```fsharp
let paymentReport paidCarts =
    let printOneLine {Payment=payment} =
        printfn "Paid %f for items" payment
    paidCarts |> List.iter printOneLine
```

`ShoppingCart`自体ではなく、`PaidCartData`のリストをパラメータとして使用することで、誤って未払いのカートをレポートすることがないようにしています。

これを行う場合は、イベントハンドラをサポートする関数の中で行うべきで、イベントハンドラそのものではありません。

{{< linktarget "replace-flags" >}}


## boolean フラグを置き換えるための明示的な状態の使用 ##

このアプローチを実際の例に適用する方法を見てみましょう。

[以前の記事](/posts/designing-with-types-intro/)で紹介した`Contact`の例では、顧客がメールアドレスを認証したかどうかを示すフラグがありました。
その型は次のようなものでした。

```fsharp
type EmailContactInfo =
    {
    EmailAddress: EmailAddress.T;
    IsEmailVerified: bool;
    }
```

このようなフラグを目にするときは、おそらく状態を扱っているのでしょう。今回のケースでは、2つの状態があることを示すためにブール値が使われています。「Unverified と "Verified "です。

前述したように、それぞれの状態で許容されるものには様々なビジネスルールがあるでしょう。例えば、以下のようなものがあります。

* ビジネスルール。*「検証メールは、未検証のメールアドレスを持つ顧客にのみ送信されるべきである」*。
* ビジネスルール *"パスワードリセットのメールは、確認済みのメールアドレスを持つ顧客にのみ送信されるべきである "*

前述のように、コードがこれらのルールに準拠していることを確認するために型を使用することができます。

それでは、`EmailContactInfo`の型をステートマシンを使って書き換えてみましょう。また、これをモジュールに入れてみましょう。

まず、2つの状態を定義します。

* "Unverified "状態では、メールアドレスのみを保持する必要があります。
* "検証済み"の状態では、メールアドレスに加えて、検証された日付や最近のパスワードリセットの回数など、いくつかの追加データを保持する必要があります。これらのデータは「Unverified」の状態には関係ありません（見えないようにしてください）。

```fsharp
module EmailContactInfo =
    open System

    // placeholder
    type EmailAddress = string

    // UnverifiedData = just the email
    type UnverifiedData = EmailAddress

    // VerifiedData = email plus the time it was verified
    type VerifiedData = EmailAddress * DateTime

    // set of states
    type T =
        | UnverifiedState of UnverifiedData
        | VerifiedState of VerifiedData

```

なお、`UnverifiedData`型については、単に型のエイリアスを使っただけです。今はこれ以上複雑なことをする必要はありませんが、型のエイリアスを使うことで目的が明確になり、リファクタリングにも役立ちます。

では、新しいステートマシンの構築と、イベントを処理してみましょう。

* 構築は*常に*検証されていないメールになるので、それは簡単です。
* ある状態から別の状態に移行するイベントは1つだけです: "verified "イベントです。

```fsharp
module EmailContactInfo =

    // types as above

    let create email =
        // unverified on creation
        UnverifiedState email

    // handle the "verified" event
    let verified emailContactInfo dateVerified =
        match emailContactInfo with
        | UnverifiedState email ->
            // construct a new info in the verified state
            VerifiedState (email, dateVerified)
        | VerifiedState _ ->
            // ignore
            emailContactInfo
```

[ここで議論されているように](/posts/match-expression/)、マッチのすべての分岐は同じ型を返さなければならないので、検証済みの状態を無視する場合でも、渡されたオブジェクトなどの何かを返さなければならないことに注意してください。

最後に、2つのユーティリティー関数 `sendVerificationEmail` と `sendPasswordReset` を書いてみましょう。

```fsharp
module EmailContactInfo =

    // types and functions as above

    let sendVerificationEmail emailContactInfo =
        match emailContactInfo with
        | UnverifiedState email ->
            // send email
            printfn "sending email"
        | VerifiedState _ ->
            // do nothing
            ()

    let sendPasswordReset emailContactInfo =
        match emailContactInfo with
        | UnverifiedState email ->
            // ignore
            ()
        | VerifiedState _ ->
            // ignore
            printfn "sending password reset"
```

{{< book_page_ddd >}}


## 明示的なケースを使ってcase/switch文を置き換える ##

状態を示すのに単純なブール値のフラグだけではない場合があります。 C#やJavaでは、`int`や`enum`を使って一連の状態を表現するのが一般的です。

例えば、以下は配送システムのパッケージの状態を表す簡単な状態図で、パッケージには3つの可能な状態があります。

![状態遷移図: パッケージ配送](./State_Delivery.png)

この図から、いくつかの明らかなビジネスルールが見えてきます。


* *Rule: "すでに配達されている荷物にサインをすることはできません" *

といった具合です。

さて、ユニオン型を使わずに、このデザインを表現するには、次のように状態を表す列挙型を使うとよいでしょう。

```fsharp
open System

type PackageStatus =
    | Undelivered
    | OutForDelivery
    | Delivered

type Package =
    {
    PackageId: int;
    PackageStatus: PackageStatus;
    DeliveryDate: DateTime;
    DeliverySignature: string;
    }
```

そして、"putOnTruck "と "signedFor "のイベントを処理するコードは次のようになります。

```fsharp
let putOnTruck package =
    {package with PackageStatus=OutForDelivery}

let signedFor package signature =
    let {PackageStatus=packageStatus} = package
    if (packageStatus = Undelivered)
    then
        failwith "package not out for delivery"
    else if (packageStatus = OutForDelivery)
    then
        {package with
            PackageStatus=OutForDelivery;
            DeliveryDate = DateTime.UtcNow;
            DeliverySignature=signature;
            }
    else
        failwith "package already delivered"
```

このコードには、いくつかの微妙なバグがあります。

* "putOnTruck "イベントを処理する際に、ステータスが既に*OutForDelivery`または`Delivered`である場合にはどうすべきか。このコードはそれについて明示的ではありません。
* signedFor "イベントを処理する際、他の状態を処理しますが、最後の else ブランチは 3 つの状態しかないと仮定しているため、それをテストすることをわざわざ明示していません。このコードは、新しい状態を追加した場合には正しくありません。
* 最後に、`DeliveryDate`と`DeliverySignature`は基本構造の中にあるので、ステータスが`Delivered`ではないにもかかわらず、誤ってそれらを設定することができるでしょう。

しかし，いつものように，慣用的で，より型安全なF#のアプローチは，データ構造の中にステータス値を埋め込むのではなく，全体的なユニオン型を使用することです。

```fsharp
open System

type UndeliveredData =
    {
    PackageId: int;
    }

type OutForDeliveryData =
    {
    PackageId: int;
    }

type DeliveredData =
    {
    PackageId: int;
    DeliveryDate: DateTime;
    DeliverySignature: string;
    }

type Package =
    | Undelivered of UndeliveredData
    | OutForDelivery of OutForDeliveryData
    | Delivered of DeliveredData
```

そして、イベントハンドラはすべてのケースを処理しなければなりません*。

```fsharp
let putOnTruck package =
    match package with
    | Undelivered {PackageId=id} ->
        OutForDelivery {PackageId=id}
    | OutForDelivery _ ->
        failwith "package already out"
    | Delivered _ ->
        failwith "package already delivered"

let signedFor package signature =
    match package with
    | Undelivered _ ->
        failwith "package not out"
    | OutForDelivery {PackageId=id} ->
        Delivered {
            PackageId=id;
            DeliveryDate = DateTime.UtcNow;
            DeliverySignature=signature;
            }
    | Delivered _ ->
        failwith "package already delivered"
```

*注：エラー処理に `failWith` を使用しています。本番システムでは、このコードはクライアント駆動のエラーハンドラで置き換えるべきです。
[Post about single case DU](/posts/designing-with-types-single-case-dus/)のコンストラクタのエラー処理の議論を参考にしてみてください。

## Using explicit case to replace implicit conditional code ##

最後に、システムに状態があっても、条件付きコードでは暗黙の了解になっている場合がよくあります。

例えば、注文を表す型を以下に示します。

```fsharp
オープン システム

type Order =
    {
    OrderId: int;
    PlacedDate: DateTime;
    PaidDate: DateTime option;
    PaidAmount: float option;
    ShippedDate: DateTime option;
    ShippingMethod: string option;
    ReturnedDate: DateTime option;
    ReturnedReason: string option;
    }
```

注文は「新規」、「支払い済み」、「出荷済み」、「返品済み」のいずれかであり、それぞれの遷移に対してタイムスタンプや追加の情報を持っていることが推測できますが、これは構造上明示されていません。

オプション型は、この型が多くのことをやろうとしすぎていることを示す手掛かりとなります。 少なくともF#ではオプションを使うことを強制しています。C#やJavaではこれらは単なるヌルで、必須かどうかは型の定義からはわからないでしょう。

それでは、これらのオプション型をテストして、注文の状態を確認するような、醜いコードを見てみましょう。

繰り返しになりますが、注文の状態に依存する重要なビジネスロジックがありますが、様々な状態や遷移がどのようなものであるかは、どこにも明示的に書かれていません。

```fsharp
let makePayment order payment =
    if (order.PaidDate.IsSome)
    then failwith "order is already paid"
    //return an updated order with payment info
    {order with
        PaidDate=Some DateTime.UtcNow
        PaidAmount=Some payment
        }

let shipOrder order shippingMethod =
    if (order.ShippedDate.IsSome)
    then failwith "order is already shipped"
    //return an updated order with shipping info
    {order with
        ShippedDate=Some DateTime.UtcNow
        ShippingMethod=Some shippingMethod
        }
```

*Note: C#プログラムが `null` をテストする方法をそのまま移植して、オプション値が存在するかどうかをテストするために `IsSome` を追加しました。しかし、`IsSome`は醜くて危険です。 使わないでください！*。

ここでは，状態を明示的にする型を使った，より良いアプローチを紹介します。

```fsharp
open System

type InitialOrderData =
    {
    OrderId: int;
    PlacedDate: DateTime;
    }
type PaidOrderData =
    {
    Date: DateTime;
    Amount: float;
    }
type ShippedOrderData =
    {
    Date: DateTime;
    Method: string;
    }
type ReturnedOrderData =
    {
    Date: DateTime;
    Reason: string;
    }

type Order =
    | Unpaid of InitialOrderData
    | Paid of InitialOrderData * PaidOrderData
    | Shipped of InitialOrderData * PaidOrderData * ShippedOrderData
    | Returned of InitialOrderData * PaidOrderData * ShippedOrderData * ReturnedOrderData
```

そして、イベントハンドリングのメソッドは以下の通りです。

```fsharp
let makePayment order payment =
    match order with
    | Unpaid i ->
        let p = {Date=DateTime.UtcNow; Amount=payment}
        // return the Paid order
        Paid (i,p)
    | _ ->
        printfn "order is already paid"
        order

let shipOrder order shippingMethod =
    match order with
    | Paid (i,p) ->
        let s = {Date=DateTime.UtcNow; Method=shippingMethod}
        // return the Shipped order
        Shipped (i,p,s)
    | Unpaid _ ->
        printfn "order is not paid for"
        order
    | _ ->
        printfn "order is already shipped"
        order
```

*注：ここでは、エラー処理に `printfn` を使用しています。本番環境では別の方法を使ってください。


## この方法を使ってはいけない場合

どのようなテクニックを学んでも、それを[金づち](http://en.wikipedia.org/wiki/Law_of_the_instrument)のように扱うことには注意が必要です。

このアプローチは複雑さを増しますので、使い始める前に、メリットがコストを上回ることを確認してください。

要約すると、単純なステートマシンを使用することが有益な条件は以下の通りです。

* 相互に排他的な状態があり、それらの間に遷移がある。
* 遷移は外部イベントによって引き起こされる。
* 状態が網羅的である。つまり、他の選択肢はなく、常にすべてのケースを処理しなければなりません。
* 各状態には、システムが別の状態にあるときにはアクセスできないような関連データがあるかもしれません。
* 状態に適用される静的なビジネスルールがあります。
 
これらのガイドラインが適用されない例を見てみましょう。

**ステートはドメイン内では重要ではありません。**

ブログを作成するアプリケーションを考えてみましょう。通常、各ブログ記事は、「Draft」、「Published」などの状態になります。そして、これらの状態の間には、イベント（「公開」ボタンのクリックなど）によって駆動されるトランジションが明らかにあります。

しかし、このためにステートマシンを作る価値があるでしょうか？一般的には、私はそうは思いません。

確かに状態の遷移はありますが、それによって本当にロジックが変化するのでしょうか？ オーサリングの観点から見ると、ほとんどのブログアプリでは、状態に基づく制限はありません。
公開済みの記事をオーサリングするのと全く同じ方法で、下書きの記事をオーサリングすることができます。

システムの中で状態を気にする唯一の部分は表示エンジンであり、ドメインに到達する前にデータベース層で下書きをフィルタリングします。

状態を気にする特別なドメインロジックはないので、おそらく不要です。

**状態遷移はアプリケーションの外で発生します。**

顧客管理アプリケーションでは、顧客を「見込み客」、「アクティブ」、「非アクティブ」などに分類するのが一般的です。

![状態遷移図: 顧客の状態](./State_Customer.png)

アプリケーションでは、これらの状態はビジネス上の意味を持ち、型システム（ユニオン型など）によって表現されるべきです。 しかし、状態の*遷移*は、一般的にアプリケーション自体の中では発生しません。例えば、ある顧客が6ヶ月間何も注文していない場合、その顧客を非アクティブと分類するかもしれません。そして、このルールは、夜間のバッチジョブによって、またはデータベースから顧客レコードが読み込まれたときに、データベース内の顧客レコードに適用されるかもしれません。 しかし、私たちのアプリケーションの視点では、遷移はアプリケーション内では起こらないので、特別なステートマシンを作成する必要はありません。

**ダイナミックなビジネスルール**

上のリストの最後の箇条書きは、「静的な」ビジネスルールを指しています。これは、ルールがゆっくりと変化するため、コード自体に埋め込むべきだという意味です。

一方で、ルールが動的で頻繁に変更される場合は、わざわざ静的な型を作る価値はないでしょう。

このような場合には、アクティブパターンや適切なルールエンジンの使用を検討する必要があります。

## まとめ

この記事では、明示的なフラグ（"IsVerified"）やステータスフィールド（"OrderStatus"）を持つデータ構造や、暗黙的な状態（過剰な数のnullable型やoption型が手がかりとなる）がある場合、ドメインオブジェクトをモデル化するためにシンプルなステートマシンの使用を検討する価値があることを見てきました。 ほとんどの場合、余分な複雑さは、状態の明示的な文書化と、すべての可能なケースを処理しないことによるエラーの排除によって補われる。