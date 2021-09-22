---
layout: post
title: "Designing with types: Making illegal states unrepresentable"
description: "Encoding business logic in types"
date: 2013-01-14
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 3
categories: [Types, DDD]
---

この記事では、F#の重要な利点である、型システムを使って「違法な状態を表現できないようにする」（[Yaron Minsky](https://blog.janestreet.com/effective-ml-revisited/)から借りた言葉です）ことについて見ていきます。

それでは、`Contact`という型を見てみましょう。先ほどのリファクタリングのおかげで，非常にシンプルになりました．

```fsharp
type Contact =
    {
    Name: Name;
    EmailContactInfo: EmailContactInfo;
    PostalContactInfo: PostalContactInfo;
    }
```

さて、次のような単純なビジネス・ルールがあるとしましょう。*"A contact must have an email or a postal address "*. 私たちの型はこのルールに適合しているでしょうか？

答えはノーです。このビジネス・ルールは、コンタクトが電子メール・アドレスを持っていても、郵便のアドレスを持っていないかもしれない、あるいはその逆かもしれないことを意味しています。しかし、現状では、私たちの型は、コンタクトが常に*両方*の情報を持っていなければならないことを要求しています。

答えは簡単で、次のようにアドレスをオプションにします。

```fsharp
タイプ Contact =
    {
    Name: PersonalName;
    EmailContactInfo: EmailContactInfo オプション。
    PostalContactInfo: PostalContactInfo オプション。
    }
```

しかし今、私たちは逆に行き過ぎています。このデザインでは、あるコンタクトがどちらのタイプの住所も全く持っていないということもあり得ます。しかしビジネス・ルールでは少なくとも1つの情報が存在しなければならないとされています。

解決策は何ですか?

## 違法な状態を表現できないようにする

ビジネスルールをよく考えてみると、3つの可能性があることがわかります。

* 連絡先には電子メールアドレスしかありません
* 連絡先には郵便番号しかない
* 連絡先がメールアドレスと郵便番号の両方を持っている場合

このように考えると、解決策は明らかです。各可能性に対応するケースを持つユニオン型を使用します。

```fsharp
type ContactInfo =
    | EmailOnly of EmailContactInfo
    | PostOnly of PostalContactInfo
    | EmailAndPost of EmailContactInfo * PostalContactInfo

type Contact =
    {
    Name: Name;
    ContactInfo: ContactInfo;
    }
```

このデザインは要件を完全に満たしています。3つのケースがすべて明示的に表現されており、4番目に考えられるケース（メールアドレスや郵便番号がまったくない）は許されません。

なお、「電子メールと郵便」のケースでは、とりあえずタプル型を使ってみました。必要なことはこれで十分です。

### ContactInfoの構築

それでは、実際にどのように使用するかを考えてみましょう。まず、新規のコンタクトを作成してみます。

```fsharp
let contactFromEmail name emailStr =
    let emailOpt = EmailAddress.create emailStr
    // handle cases when email is valid or invalid
    match emailOpt with
    | Some email ->
        let emailContactInfo =
            {EmailAddress=email; IsEmailVerified=false}
        let contactInfo = EmailOnly emailContactInfo
        Some {Name=name; ContactInfo=contactInfo}
    | None -> None

let name = {FirstName = "A"; MiddleInitial=None; LastName="Smith"}
let contactOpt = contactFromEmail name "abc@example.com"
```

このコードでは、名前とEメールを渡して新しいコンタクトを作成するシンプルなヘルパー関数`contactFromEmail`を作成しました。
しかし、Eメールが有効でない場合もあるので、この関数は両方のケースを処理する必要があり、`Contact`ではなく`Contact option`を返すことで対応しています。

### ContactInfoの更新

ここで、既存の `ContactInfo` に郵便番号を追加する必要がある場合、3つの可能なケースをすべて処理するしかありません。

* これまでメールアドレスしか持っていなかったコンタクトが、メールアドレスと郵便番号の両方を持つようになったので、`EmailAndPost`ケースを使ってコンタクトを返します。
* コンタクトが以前は郵便住所しか持っていなかった場合、`PostOnly`ケースを使ってコンタクトを返し、既存の住所を置き換えます。
* 連絡先が以前に電子メールアドレスと郵便住所の両方を持っていた場合、`EmailAndPost`ケースを使って連絡先を返し、既存の住所を置き換えます。

ここでは、郵便番号を更新するヘルパーメソッドを紹介します。それぞれのケースを明示的に処理しているのがわかります。

```fsharp
let updatePostalAddress contact newPostalAddress =
    let {Name=name; ContactInfo=contactInfo} = contact
    let newContactInfo =
        match contactInfo with
        | EmailOnly email ->
            EmailAndPost (email,newPostalAddress)
        | PostOnly _ -> // ignore existing address
            PostOnly newPostalAddress
        | EmailAndPost (email,_) -> // ignore existing address
            EmailAndPost (email,newPostalAddress)
    // make a new contact
    {Name=name; ContactInfo=newContactInfo}
```

そして、使用しているコードは以下の通りです。

```fsharp
let contact = contactOpt.Value   // see warning about option.Value below
let newPostalAddress =
    let state = StateCode.create "CA"
    let zip = ZipCode.create "97210"
    {
        Address =
            {
            Address1= "123 Main";
            Address2="";
            City="Beverly Hills";
            State=state.Value; // see warning about option.Value below
            Zip=zip.Value;     // see warning about option.Value below
            };
        IsAddressValid=false
    }
let newContact = updatePostalAddress contact newPostalAddress
```

*注意：このコードでは、オプションの内容を抽出するのに、`option.Value`を使っています。
これは、インタラクティブに遊んでいるときはいいのですが、プロダクションコードでは非常に悪い習慣です。オプションの両方のケースを処理するには、常にマッチングを使用する必要があります。


## わざわざこんな複雑な型を作る必要があるのか？

この時点で、あなたは私たちが不必要に物事を複雑にしていると言うかもしれません。私は以下の点を指摘します。

まず、ビジネスロジックは複雑です。それを避けるための簡単な方法はありません。もしあなたのコードがここまで複雑でなければ、すべてのケースを適切に処理していないことになります。

第二に、ロジックが型で表現されていれば、自動的に文書化されます。以下のユニオンケースを見れば、ビジネスルールが何であるかすぐにわかります。他のコードを分析するために時間を費やす必要はありません。

```fsharp
type ContactInfo =
    | EmailContactInfoのEmailOnly
    | PostalContactInfoのPostOnly
    | EmailAndPost of EmailContactInfo * PostalContactInfo
```

最後に、ロジックがタイプで表現されている場合、ビジネス・ルールに変更があった場合、即座にブレーク・チェンジが発生しますが、これは一般的には良いことです。

次の記事では、最後のポイントをより深く掘り下げていきます。型を使ってビジネスロジックを表現してみると、突然、ドメインについて全く新しい知見を得ることができるかもしれません。

{{< book_page_ddd_img >}}
