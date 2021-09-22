---
layout: post
title: "Designing with types: Conclusion"
description: "A before and after comparison"
date: 2013-01-19
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 8
categories: [Types, DDD]
---

このシリーズでは、設計プロセスの一環として型を使用する方法として、以下のようなものを見てきました。

* 大きな構造を小さな「原子」のコンポーネントに分解する。
大規模な構造を小さな「アトミック」なコンポーネントに分解すること * シングルケースユニオンを使用して、`EmailAddress`や`ZipCode`などの主要なドメインタイプに意味的な意味と検証を追加すること
* 型システムが有効なデータのみを表現できることを保証する（「違法な状態を表現できないようにする」）。
* 隠された要件を発見するための分析ツールとしての型の使用
* フラグや列挙型を単純なステートマシンに置き換えること。
* プリミティブな文字列を、さまざまな制約を保証する型に置き換える

この最後の投稿では、これらをすべて適用して見てみましょう。

## "before" code ##

このシリーズの[最初の投稿](/posts/designing-with-types-intro/)で始めたオリジナルの例は以下の通りです。

```fsharp
type Contact =
    {
    FirstName: string;
    MiddleInitial: string;
    LastName: string;

    EmailAddress: string;
    //true if ownership of email address is confirmed
    IsEmailVerified: bool;

    Address1: string;
    Address2: string;
    City: string;
    State: string;
    Zip: string;
    //true if validated against address service
    IsAddressValid: bool;
    }
```

上記のテクニックをすべて適用した後の最終結果と比較してみましょう。

## "after" code ##

まず、アプリケーションに依存しない型から始めましょう。 これらの型は、おそらく多くのアプリケーションで再利用できるでしょう。

```fsharp
// ========================================
// WrappedString
// ========================================

/// Common code for wrapped strings
module WrappedString =

    /// An interface that all wrapped strings support
    type IWrappedString =
        abstract Value : string

    /// Create a wrapped value option
    /// 1) canonicalize the input first
    /// 2) If the validation succeeds, return Some of the given constructor
    /// 3) If the validation fails, return None
    /// Null values are never valid.
    let create canonicalize isValid ctor (s:string) =
        if s = null
        then None
        else
            let s' = canonicalize s
            if isValid s'
            then Some (ctor s')
            else None

    /// Apply the given function to the wrapped value
    let apply f (s:IWrappedString) =
        s.Value |> f

    /// Get the wrapped value
    let value s = apply id s

    /// Equality
    let equals left right =
        (value left) = (value right)

    /// Comparison
    let compareTo left right =
        (value left).CompareTo (value right)

    /// Canonicalizes a string before construction
    /// * converts all whitespace to a space char
    /// * trims both ends
    let singleLineTrimmed s =
        System.Text.RegularExpressions.Regex.Replace(s,"\s"," ").Trim()

    /// A validation function based on length
    let lengthValidator len (s:string) =
        s.Length <= len

    /// A string of length 100
    type String100 = String100 of string with
        interface IWrappedString with
            member this.Value = let (String100 s) = this in s

    /// A constructor for strings of length 100
    let string100 = create singleLineTrimmed (lengthValidator 100) String100

    /// Converts a wrapped string to a string of length 100
    let convertTo100 s = apply string100 s

    /// A string of length 50
    type String50 = String50 of string with
        interface IWrappedString with
            member this.Value = let (String50 s) = this in s

    /// A constructor for strings of length 50
    let string50 = create singleLineTrimmed (lengthValidator 50)  String50

    /// Converts a wrapped string to a string of length 50
    let convertTo50 s = apply string50 s

    /// map helpers
    let mapAdd k v map =
        Map.add (value k) v map

    let mapContainsKey k map =
        Map.containsKey (value k) map

    let mapTryFind k map =
        Map.tryFind (value k) map

// ========================================
// 電子メールアドレス（アプリケーションに依存しない
// ========================================

module EmailAddress =

    type T = EmailAddress of string with
        interface WrappedString.IWrappedString with
            member this.Value = let (EmailAddress s) = this in s

    let create =
        let canonicalize = WrappedString.singleLineTrimmed
        let isValid s =
            (WrappedString.lengthValidator 100 s) &&
            System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        WrappedString.create canonicalize isValid EmailAddress

    /// Converts any wrapped string to an EmailAddress
    let convert s = WrappedString.apply create s

// ========================================
// ZipCode (アプリケーションに依存しない)
// ========================================

module ZipCode =

    type T = ZipCode of string with
        interface WrappedString.IWrappedString with
            member this.Value = let (ZipCode s) = this in s

    let create =
        let canonicalize = WrappedString.singleLineTrimmed
        let isValid s =
            System.Text.RegularExpressions.Regex.IsMatch(s,@"^\d{5}$")
        WrappedString.create canonicalize isValid ZipCode

    /// Converts any wrapped string to a ZipCode
    let convert s = WrappedString.apply create s

// ========================================
// StateCode （アプリケーションに依存しない
// ========================================

module StateCode =

    type T = StateCode  of string with
        interface WrappedString.IWrappedString with
            member this.Value = let (StateCode  s) = this in s

    let create =
        let canonicalize = WrappedString.singleLineTrimmed
        let stateCodes = ["AZ";"CA";"NY"] //etc
        let isValid s =
            stateCodes |> List.exists ((=) s)

        WrappedString.create canonicalize isValid StateCode

    /// Converts any wrapped string to a StateCode
    let convert s = WrappedString.apply create s

// ========================================
// PostalAddress (アプリケーションに依存しない)
// ========================================

module PostalAddress =

    type USPostalAddress =
        {
        Address1: WrappedString.String50;
        Address2: WrappedString.String50;
        City: WrappedString.String50;
        State: StateCode.T;
        Zip: ZipCode.T;
        }

    type UKPostalAddress =
        {
        Address1: WrappedString.String50;
        Address2: WrappedString.String50;
        Town: WrappedString.String50;
        PostCode: WrappedString.String50;   // todo
        }

    type GenericPostalAddress =
        {
        Address1: WrappedString.String50;
        Address2: WrappedString.String50;
        Address3: WrappedString.String50;
        Address4: WrappedString.String50;
        Address5: WrappedString.String50;
        }

    type T =
        | USPostalAddress of USPostalAddress
        | UKPostalAddress of UKPostalAddress
        | GenericPostalAddress of GenericPostalAddress

// ========================================
// PersonalName (アプリケーションに依存しない)
// ========================================

module PersonalName =
    open WrappedString

    type T =
        {
        FirstName: String50;
        MiddleName: String50 option;
        LastName: String100;
        }

    /// 新しい値の作成
    let create first middle last =
        match (string50 first),(string100 last) with
        | Some f, Some l ->
            Some {
                FirstName = f;
                MiddleName = (string50 middle)
                LastName = l;
                }
        | _ ->
            None

    /// 名前を連結して
    /// 生の文字列を返す
    let fullNameRaw personalName =
        let f = personalName.FirstName |> value
        let l = personalName.LastName |> value
        let names =
            match personalName.MiddleName with
            | None -> [| f; l |]
            | Some middle -> [| f; (value middle); l |]
        System.String.Join(" ", names)

    /// 名前を連結します。
    /// 長すぎる場合はNoneを返す
    let fullNameOption personalName =
        personalName |> fullNameRaw |> string100

    /// 名前を連結します。
    /// 長すぎる場合は切り捨てる
    let fullNameTruncated personalName =
        // helper function
        let left n (s:string) =
            if (s.Length > n)
            then s.Substring(0,n)
            else s

        personalName
        |> fullNameRaw  // concat
        |> left 100     // truncate
        |> string100    // wrap
        |> Option.get   // this will always be ok
```

そして、アプリケーション固有の型です。

```fsharp

// ========================================
// EmailContactInfo -- ステートマシン
// ========================================

module EmailContactInfo =
    open System

    // UnverifiedData = just the EmailAddress
    type UnverifiedData = EmailAddress.T

    // VerifiedData = EmailAddress plus the time it was verified
    type VerifiedData = EmailAddress.T * DateTime

    // set of states
    type T =
        | UnverifiedState of UnverifiedData
        | VerifiedState of VerifiedData

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

// ========================================
// PostalContactInfo -- ステートマシン
// ========================================

module PostalContactInfo =
    open System

    // InvalidData = just the PostalAddress
    type InvalidData = PostalAddress.T

    // ValidData = PostalAddress plus the time it was verified
    type ValidData = PostalAddress.T * DateTime

    // set of states
    type T =
        | InvalidState of InvalidData
        | ValidState of ValidData

    let create address =
        // invalid on creation
        InvalidState address

    // handle the "validated" event
    let validated postalContactInfo dateValidated =
        match postalContactInfo with
        | InvalidState address ->
            // construct a new info in the valid state
            ValidState (address, dateValidated)
        | ValidState _ ->
            // ignore
            postalContactInfo

    let contactValidationService postalContactInfo =
        let dateIsTooLongAgo (d:DateTime) =
            d < DateTime.Today.AddYears(-1)

        match postalContactInfo with
        | InvalidState address ->
            printfn "contacting the address validation service"
        | ValidState (address,date) when date |> dateIsTooLongAgo  ->
            printfn "last checked a long time ago."
            printfn "contacting the address validation service again"
        | ValidState  _ ->
            printfn "recently checked. Doing nothing."

// ========================================
// コンタクトメソッドとコンタクト
// ========================================

type ContactMethod =
    | Email of EmailContactInfo.T
    | PostalAddress of PostalContactInfo.T

type Contact =
    {
    Name: PersonalName.T;
    PrimaryContactMethod: ContactMethod;
    SecondaryContactMethods: ContactMethod list;
    }

```

{{< book_page_ddd_img >}}。


## Conclusion ##

フーッ!  新しいコードは、オリジナルのコードよりもずっとずっと長いです。もちろん、元のバージョンでは必要なかった多くのサポート機能を持っていますが、それにしてもたくさんの余分な仕事をしているように思えます。では、それだけの価値があったのでしょうか？

答えは「イエス」だと思います。その理由をいくつか挙げてみましょう。

**新しいコードはより明確になった**

元の例を見てみると、フィールド間のアトミック性、検証ルール、長さの制約、間違った順序でフラグを更新するのを止めるものが何もありませんでした、などなど。

データ構造は「ダム」で、すべてのビジネス・ルールはアプリケーション・コードの中で暗黙の了解となっていました。
そのアプリケーションには、ユニットテストにも現れないような微妙なバグがたくさんある可能性があります。 (*メールアドレスが更新されたすべての場所で、アプリケーションが `IsEmailVerified` フラグを false にリセットしていることを確認していますか *)

その一方で、新しいコードは細部まで非常に明確になっています。型以外のすべてのものを取り除くと、ビジネスルールとドメイン制約が何であるかについて非常に良いアイデアが得られるでしょう。

**新しいコードでは、エラー処理を先延ばしにすることはできません**。

新しい型で動作するコードを書くということは、長すぎる名前を処理したり、コンタクトメソッドを提供できなかったりと、うまくいかない可能性があるあらゆることを処理しなければならないということです。
そして、これを構築時に前もって行わなければなりません。これを後回しにすることはできません。

このようなエラー処理のコードを書くのは煩わしくて面倒ですが、一方で、それはほとんど自力で書いたものです。これらのタイプで実際にコンパイルされるコードを書く方法は本当に一つしかありません。

**新しいコードは正しい可能性が高い**。

新しいコードの*大きな*利点は、おそらくバグがないことです。ユニットテストを書かなくても、データベースで名前を `varchar(50)` に書き込んだときに、名前が切り捨てられることはないと確信できますし、検証用のメールを誤って2回送信することもないと確信できます。

また、コード自体についても、開発者が忘れずに対処しなければならない（あるいは対処し忘れてしまう）ことの多くが完全に排除されています。nullチェックも、キャストも、`switch`文のデフォルトがどうなっているかを心配する必要もありません。また、サイクロマティックな複雑さをコード品質の指標として使うのが好きな方は、350行の中に3つの`if`文しかないことに気づくかもしれません。

**警告の言葉...**

最後に、注意してください。このスタイルの型ベースの設計に慣れてしまうと、あなたに陰湿な影響を与えることになります。厳密に型付けされていないコードを見るたびに、被害妄想を抱くようになるでしょう。(メールアドレスはどのくらいの長さにすればいいのだろうか？そうなったら、あなたは完全に教団の一員になったと言えるでしょう。ようこそ!


*このシリーズがお気に召したのなら、同じトピックの多くをカバーするスライドデッキをどうぞ。ビデオもありますよ[（こちら）](/ddd/)*

{{< slideshare A4ay4HQqJgu0Q "domain-driven-design-with-the-f-type-system-functional-londoners-2014" "Domain Driven Design with the F# type System -- F#unctional Londoners 2014" >}}

