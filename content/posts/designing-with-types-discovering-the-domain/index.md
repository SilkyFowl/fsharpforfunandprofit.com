---
layout: post
title: "Designing with types: Discovering new concepts"
description: "Gaining deeper insight into the domain"
date: 2013-01-15
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 4
categories: [Types, DDD]
---

前回の記事では、型を使ってビジネス・ルールを表現する方法を紹介しました。

そのルールとは *"A contact must have an email or a postal address "*.

そして、私たちが設計した型は

```fsharp
type ContactInfo =
    | EmailOnly of EmailContactInfo
    | PostOnly of PostalContactInfo
    | EmailAndPost of EmailContactInfo * PostalContactInfo
```

ここで、ビジネス上、電話番号もサポートする必要があると判断したとします。 新しいビジネス・ルールは以下の通りです。*"A contact must have at least one of the following: an email, a postal address, a home phone, or a work phone "*.

これをどのように表現すればよいのでしょうか。

少し考えてみると、この4つの連絡手段の組み合わせは15通りあることがわかります。15の選択肢を持つ組合員のケースを作りたくないのではないでしょうか？何か良い方法はないだろうか？

その考えは保留にして、別の、しかし関連する問題を見てみましょう。

## 要件が変わったときに壊れやすい変更を強いる

問題はここにあります。例えば、以下のようなメールアドレスと住所のリストを含むコンタクト構造があるとします。

```fsharp
type ContactInformation =
    {
    EmailAddresses : EmailContactInfo list;
    PostalAddresses : PostalContactInfo list
    }
```

また、これらの情報をループしてレポートに出力する`printReport`関数を作成したとします。

```fsharp
// mock code
let printEmail emailAddress =
    printfn "Email Address is %s" emailAddress

// mock code
let printPostalAddress postalAddress =
    printfn "Postal Address is %s" postalAddress

let printReport contactInfo =
    let {
        EmailAddresses = emailAddresses;
        PostalAddresses = postalAddresses;
        } = contactInfo
    for email in emailAddresses do
         printEmail email
    for postalAddress in postalAddresses do
         printPostalAddress postalAddress
```

粗削りですが、シンプルでわかりやすいですね。

さて、もし新しいビジネスルールが有効になったら、電話番号用の新しいリストを持つように構造を変更することにしましょう。 更新後の構造は以下のようになります。

```fsharp
type PhoneContactInfo = string // dummy for now

type ContactInformation =
    {
    EmailAddresses : EmailContactInfo list;
    PostalAddresses : PostalContactInfo list;
    HomePhones : PhoneContactInfo list;
    WorkPhones : PhoneContactInfo list;
    }
```

この変更を行った場合、連絡先情報を処理するすべての関数が、新しい電話番号のケースにも対応できるように更新されていることを確認してください。

確かに、パターンマッチが崩れると修正せざるを得ないでしょう。しかし、多くの場合、新しいケースを処理する必要はありません。

例えば、新しいリストに対応するために更新された`printReport`は以下の通りです。

```fsharp
let printReport contactInfo =
    let {
        EmailAddresses = emailAddresses;
        PostalAddresses = postalAddresses;
        } = contactInfo
    for email in emailAddresses do
         printEmail email
    for postalAddress in postalAddresses do
         printPostalAddress postalAddress
```

わざとらしいミスがわかりますか？はい、電話を処理する関数を変更するのを忘れていました。レコードの新しいフィールドのおかげで、コードが壊れることは全くありませんでした。新しいケースを処理することを覚えているという保証はありません。忘れてしまうのはあまりにも簡単です。

繰り返しになりますが、このような状況が簡単に起こらないように型を設計することができるか、という課題があります。

## ドメインへの深い洞察

この例をもう少し深く考えてみると、「木を見て森を見ず」になっていることに気づくでしょう。

私たちの最初のコンセプトは *「お客様に連絡を取るためには、可能なメールのリストと、可能なアドレスのリストなどが必要」*です。

しかし、これは間違っています。もっと良いコンセプトがあります。*「お客様に連絡を取るには、連絡方法のリストが必要です。それぞれの連絡方法には、電子メール、郵便番号、電話番号などがあります。」*。

これは、ドメインがどのようにモデル化されるべきかについての重要な洞察です。 これにより、「ContactMethod」という全く新しい型が作成され、問題が一挙に解決されます。

この新しい概念を使うために、すぐに型をリファクタリングすることができます。

```fsharp
type ContactMethod =
    | Email of EmailContactInfo
    | PostalAddress of PostalContactInfo
    | HomePhone of PhoneContactInfo
    | WorkPhone of PhoneContactInfo

type ContactInformation =
    {
    ContactMethods  : ContactMethod list;
    }
```

そして、レポーティングのコードも新しいタイプを扱うように変更する必要があります。

```fsharp
// mock code
let printContactMethod cm =
    match cm with
    | Email emailAddress ->
        printfn "Email Address is %s" emailAddress
    | PostalAddress postalAddress ->
         printfn "Postal Address is %s" postalAddress
    | HomePhone phoneNumber ->
        printfn "Home Phone is %s" phoneNumber
    | WorkPhone phoneNumber ->
        printfn "Work Phone is %s" phoneNumber

let printReport contactInfo =
    let {
        ContactMethods=methods;
        } = contactInfo
    methods
    |> List.iter printContactMethod
```

これらの変更にはいくつかの利点があります。

まず、モデリングの観点からは、新しい型はドメインをより良く表現し、変化する要求に対応できるようになっています。

また、開発の観点からは、型をユニオンに変更することで、新しいケースを追加（または削除）しても、非常にわかりやすい方法でコードを壊すことができ、すべてのケースを処理することをうっかり忘れてしまうことがより難しくなります。

{{< book_page_ddd >}}

## 15通りの組み合わせがあるビジネスルールに戻る

さて、元の例に戻りましょう。ビジネスルールをコード化するためには、さまざまな連絡方法の15通りの組み合わせを作らなければならないのではないかと考えていました。

しかし、報告問題から得られた新たな知見は、ビジネスルールの理解にも影響を与えます。

「連絡方法」という概念が頭に入っていれば、要件を次のように言い換えることができます。*「お客様は、少なくとも1つの連絡方法を持っている必要があります。連絡方法には、電子メール、住所、電話番号があります。」*

そこで、コンタクトメソッドのリストを持つように`Contact`タイプを再設計しましょう。

```fsharp
type Contact =
    {
    Name: PersonalName;
    ContactMethods: ContactMethod list;
    }
```

しかし、これはまだ正しいとは言えません。リストは空かもしれません。 少なくとも1つのコンタクトメソッドが存在しなければならないというルールをどのようにして適用すればよいのでしょうか。

最も簡単な方法は、次のように必須の新しいフィールドを作ることです。

```fsharp
type Contact =
    {
    Name: PersonalName;
    PrimaryContactMethod: ContactMethod;
    SecondaryContactMethods: ContactMethod list;
    }
```

このデザインでは、`PrimaryContactMethod`が必須で、セカンダリーコンタクトメソッドはオプションとなっていますが、これはまさにビジネスルールが要求していることです。

今回のリファクタリングでは、いくつかのヒントを得ることができました。 プライマリ」と「セカンダリ」のコンタクトメソッドの概念は、他の領域のコードを明確にし、洞察とリファクタリングの連鎖的な変化を生み出すかもしれません。

## まとめ

今回の記事では、ビジネスルールのモデル化に型を使用することで、実際にドメインをより深いレベルで理解することができることを見てきました。

エリック・エヴァンス氏は、*Domain Driven Design*という本の中で、セクション全体と特に2つの章（第8章と第9章）を割いて、[refactoring towards deeper insight](http://dddcommunity.org/wp-content/uploads/files/books/evans_pt03.pdf)の重要性を論じています。 今回の例はそれに比べると簡単なものですが、このような洞察がモデルとコードの正しさの両方を向上させるのに役立つことを示していると思います。

次回は、細かい状態を表現するのに型がどのように役立つかを見ていきます。











