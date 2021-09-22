---
layout: post
title: "Designing with types: Single case union types"
description: "Adding meaning to primitive types"
date: 2013-01-13
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 2
categories: [Types, DDD]
---

前回の記事の最後で、メールアドレスや郵便番号などの値を、次のように定義しました。

```fsharp

EmailAddress: string;
State: string;
Zip: string;

```

これらはすべて単純な文字列として定義されています。 しかし、本当にただの文字列なのでしょうか？ メールアドレスは、郵便番号や州の略語と互換性があるのでしょうか？

ドメイン・ドリブンなデザインでは、これらは単なる文字列ではなく、確かに別個のものです。ですから、理想的には、それらが誤って混ざってしまわないように、別々のタイプをたくさん用意したいと思います。

これは昔から[グッドプラクティスとして知られていること](http://codemonkeyism.com/never-never-never-use-string-in-java-or-at-least-less-often/)です。
しかし、C#やJavaのような言語では、このような小さな型を何百個も作るのは苦痛であり、いわゆる["primitive obsession"](http://sourcemaking.com/refactoring/primitive-obsession)というコードの臭いにつながります。

しかし、F#にはそのような言い訳はありません。シンプルなラッパー型を作るのは簡単です。

## プリミティブ型のラッピング

別の型を作る最も簡単な方法は、基礎となる文字列型を別の型で包むことです。

シングルケースのユニオン型を使って、次のように行うことができます。

```fsharp
type EmailAddress = EmailAddress of string
type ZipCode = 文字列のZipCode
type StateCode = 文字列のStateCode
```

あるいは，次のように，1つのフィールドを持つレコードタイプを使うこともできます。

```fsharp
type EmailAddress = { EmailAddress: string }
type ZipCode = { ZipCode: string }
type StateCode = { StateCode: string}
```

文字列やその他のプリミティブな型のラッパー型を作るには、どちらの方法も使えますが、どちらの方法が良いのでしょうか？

答えは、一般的にシングルケース・ディスクリミネーテッド・ユニオンです。 ユニオンケース」は実際にはそれ自体が適切なコンストラクタ関数であるため、「ラップ」と「アンラップ」がはるかに簡単です。アンラップはインラインのパターンマッチを使って行うことができます。

以下に、`EmailAddress`型がどのように構築され、どのように解体されるかの例を示します。

```fsharp
type EmailAddress = EmailAddress of string

// コンストラクタを関数として使用
"a" |> EmailAddress
["a"; "b"; "c"] |> List.map EmailAddress

// インライン・デコンストラクション
let a' = "a" |> EmailAddress
let (EmailAddress a'') = a'

let addresses =
    ["a"; "b"; "c"]
    |> List.map EmailAddress

let addresses' =
    addresses
    |> List.map (fun (EmailAddress e) -> e)
```

レコード型ではこのようなことは簡単にはできません。

そこで、このユニオン型を使うようにコードを再度リファクタリングしてみましょう。 現在は次のようになっています。

```fsharp
type PersonalName =
    {
    FirstName: string;
    MiddleInitial: string option;
    LastName: string;
    }

type EmailAddress = EmailAddress of string

type EmailContactInfo =
    {
    EmailAddress: EmailAddress;
    IsEmailVerified: bool;
    }

type ZipCode = ZipCode of string
type StateCode = StateCode of string

type PostalAddress =
    {
    Address1: string;
    Address2: string;
    City: string;
    State: StateCode;
    Zip: ZipCode;
    }

type PostalContactInfo =
    {
    Address: PostalAddress;
    IsAddressValid: bool;
    }

type Contact =
    {
    Name: PersonalName;
    EmailContactInfo: EmailContactInfo;
    PostalContactInfo: PostalContactInfo;
    }
```

ユニオン型のもう一つの良い点は、後述するように、実装をモジュール・シグネチャでカプセル化できることです。


## シングルケースユニオンの "ケース "のネーミング

上の例では、caseに型と同じ名前を使っています。

```fsharp
type EmailAddress = EmailAddress of string
type ZipCode = ZipCode of string
type StateCode = StateCode of string
```

最初は混乱するかもしれませんが，実際にはこれらは異なるスコープにあるので，名前の衝突はありません。1つは型で、もう1つは同じ名前のコンストラクタ関数です。

つまり、次のような関数のシグネチャがあるとします。

```fsharp
val f: string -> EmailAddress
```

これは，型の世界のものを指しており，`EmailAddress`は型を指しています。

一方，次のようなコードがあるとします。

```fsharp
let x = EmailAddress y
```

これは、値の世界のものを参照しているので、`EmailAddress`はコンストラクタ関数を参照しています。

## single case unionsの構築

メールアドレスや郵便番号のように特別な意味を持つ値の場合、一般的には特定の値しか許されません。 すべての文字列がメールや郵便番号として認められるわけではありません。

これは、どこかの時点で検証を行う必要があることを意味しており、構築時よりも良いタイミングがあるでしょう。結局のところ、一度構築された値は不変なので、後から誰かに修正される心配はありません。

上記のモジュールを拡張して、コンストラクタ関数を追加する方法を紹介します。

```fsharp

... types as above ...

let CreateEmailAddress (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then Some (EmailAddress s)
        else None

let CreateStateCode (s:string) =
    let s' = s.ToUpper()
    let stateCodes = ["AZ";"CA";"NY"] //etc
    if stateCodes |> List.exists ((=) s')
        then Some (StateCode s')
        else None
```

これでコンストラクタをテストすることができます。

```fsharp
CreateStateCode "CA"
CreateStateCode "XX"

CreateEmailAddress "a@example.com"
CreateEmailAddress "example.com"
```

## コンストラクタでの無効な入力の処理 ###.

このようなコンストラクタ関数の場合、すぐに問題になるのが、無効な入力をどう処理するかという問題です。
例えば、メールアドレスのコンストラクタに「abc」を渡した場合、どうすればよいのでしょうか。

これに対処する方法はいくつかあります。

まず、例外を発生させることができます。これは醜いし、想像力に欠けると思うので、私はこの方法を即座に却下します。

次に、入力が有効でないことを意味する `None` を含むオプションタイプを返すことができます。 これは、上のコンストラクタ関数で行っていることです。

これは一般的に最も簡単な方法です。しかし、値が有効でない場合には、呼び出し側が明示的に処理しなければならないという利点があります。

例えば，上の例の呼び出し側のコードは次のようになります。
```fsharp
match (CreateEmailAddress "a@example.com") with
| Some email -> ... do something with email
| None -> ... ignore?
```

欠点は、複雑なバリデーションでは、何が悪かったのかがはっきりしないかもしれないということです。メールが長すぎたのか、'@'マークがなかったのか、ドメインが無効だったのか。私たちにはわかりません。

より詳細な情報が必要な場合は、エラーケースでより詳細な説明を含んだ型を返すとよいでしょう。

次の例では、失敗のケースでエラーを示すために `CreationResult` 型を使用しています。

```fsharp
type EmailAddress = EmailAddress of string
type CreationResult<'T> = Success of 'T | Error of string

let CreateEmailAddress2 (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then Success (EmailAddress s)
        else Error "Email address must contain an @ sign"

// test
CreateEmailAddress2 "example.com"
```

最後に、最も一般的な方法として、連続性を利用します。つまり，2つの関数を渡すのです．1つは成功した場合（新しく構築されたメールをパラメータとして受け取る），もう1つは失敗した場合（エラー文字列をパラメータとして受け取る）です．

```fsharp
type EmailAddress = EmailAddress of string

let CreateEmailAddressWithContinuations success failure (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then success (EmailAddress s)
        else failure "Email address must contain an @ sign"
```

success関数はEmailをパラメータとして受け取り、error関数は文字列を受け取ります。どちらの関数も同じ型を返さなければなりませんが，その型は自由です。

以下に簡単な例を示します。どちらの関数もprintfを行い、何も返しません（つまりユニット）。

```fsharp
let success (EmailAddress s) = printfn "success creating email %s" s
let failure msg = printfn "error creating email: s" msg
CreateEmailAddressWithContinuations success failure "example.com"
CreateEmailAddressWithContinuations success failure "x@example.com"
```

コンティニュエーションを使えば、他のどのようなアプローチも簡単に再現することができます。例えば、オプションを作成する方法です。この場合、どちらの関数も、`EmailAddress option`を返します。

```fsharp
let success e = Some e
let failure _  = None
CreateEmailAddressWithContinuations success failure "example.com"
CreateEmailAddressWithContinuations success failure "x@example.com"
```

また、エラー時に例外を投げる方法は以下の通りです。

```fsharp
let success e = e
let failure _  = failwith "bad email address"
CreateEmailAddressWithContinuations success failure "example.com"
CreateEmailAddressWithContinuations success failure "x@example.com"
```

このコードは非常に面倒に見えますが、実際には、長ったらしい関数の代わりに、部分的に適用される関数をローカルに作成して使用することになるでしょう。

```fsharp
// 部分適用関数の設定
let success e = Some e
let failure _ = None
let createEmail = CreateEmailAddressWithContinuations success failure

// 部分的に適用された関数を使う
createEmail "x@example.com"
createEmail "example.com"
```

{{< book_page_ddd >}}


## ラッパータイプ用モジュールの作成 ###

これらのシンプルなラッパータイプは、バリデーションを追加することでより複雑になってきており、タイプに関連づけたい他の機能も発見できるでしょう。

そこで、ラッパータイプごとにモジュールを作成し、タイプと関連する関数をそこに配置するのがよいでしょう。

```fsharp
module EmailAddress =

    type T = EmailAddress of string

    // wrap
    let create (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then Some (EmailAddress s)
            else None

    // unwrap
    let value (EmailAddress e) = e
```

型のユーザーは、モジュールの関数を使って型を作成したり、アンラップしたりします。例えば、以下のようになります。

```fsharp

// メールアドレスの作成
let address1 = EmailAddress.create "x@example.com"
let address2 = EmailAddress.create "example.com"

// unwrap an email address
match address1 with
| Some e -> EmailAddress.value e |> printfn "the value is %s"
| None -> ()
```

## コンストラクターの使用を強制する ##

ひとつの問題は、呼び出し側にコンストラクタの使用を強制できないことです。誰かが検証を回避して直接型を作成することができます。

実際には問題にならないことが多いです。 単純なテクニックとしては、命名規則を使って「プライベート」型であることを示して
そして、「ラップ」と「アンラップ」の関数を用意して、クライアントがその型を直接操作する必要がないようにすることです。

以下にその例を示します。

```fsharp

module EmailAddress =

    // private type
    type _T = EmailAddress of string

    // wrap
    let create (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then Some (EmailAddress s)
            else None

    // unwrap
    let value (EmailAddress e) = e
```

もちろんこの場合、型は本当の意味でのプライベートではありませんが、呼び出し側には常に「公開された」関数を使うように促しています。

もし、本当に型の内部をカプセル化して、呼び出し側にコンストラクタ関数を使わせたいのであれば、モジュール署名を使うことができます。

以下は，メールアドレスの例の署名ファイルです．

```fsharp
// FILE: EmailAddress.fsi

module EmailAddress

// encapsulated type
type T

// wrap
val create : string -> T option

// unwrap
val value : T -> string
```

(ただし，モジュール署名はコンパイル済みのプロジェクトでのみ動作し，インタラクティブなスクリプトでは動作しないので，これをテストするには，F#プロジェクトの中に，ここに示すようなファイル名で3つのファイルを作成する必要があります)。

ここでは，実装ファイルを紹介します．

```fsharp
// FILE: EmailAddress.fs

module EmailAddress

// encapsulated type
type T = EmailAddress of string

// wrap
let create (s:string) =
    if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        then Some (EmailAddress s)
        else None

// unwrap
let value (EmailAddress e) = e

```

そして，これがクライアントです。

```fsharp
// FILE: EmailAddressClient.fs

module EmailAddressClient

open EmailAddress

// code works when using the published functions
let address1 = EmailAddress.create "x@example.com"
let address2 = EmailAddress.create "example.com"

// code that uses the internals of the type fails to compile
let address3 = T.EmailAddress "bad email"

```

モジュールのシグネチャによってエクスポートされた型 `EmailAddress.T` は不透明なので、クライアントは内部にアクセスすることができません。

ご覧のように、この方法ではコンストラクタの使用が強制されます。型を直接作成しようとすると（`T.EmailAddress "bad email"`）、コンパイルエラーが発生します。


## シングルケースユニオンを "ラップ "する場合 ###

さて、ラッパー型ができたところで、どのような場合にラッパー型を作成すればよいのでしょうか？

一般的には、サービスの境界（例えば、[六角アーキテクチャ](http://alistair.cockburn.us/Hexagonal+architecture))でのみ必要になります。

このアプローチでは、ラッピングはUIレイヤーや、永続化レイヤーからの読み込み時に行われ、ラッピングされた型が作成されると、ドメインレイヤーに渡され、不透明な型として「全体」が操作されます。
ドメイン自体を操作する際に、実際にラップされたコンテンツを直接必要とすることは驚くほど稀です。

コンストラクションの一環として、呼び出し側が独自の検証ロジックを行うのではなく、提供されたコンストラクタを使用することが重要です。これにより、「悪い」値がドメインに入ることはありません。

例えば、以下のコードは、UIが独自の検証を行っていることを示しています。

```fsharp
let processFormSubmit () =
    let s = uiTextBox.Text
    if (s.Length < 50)
        then // ドメインオブジェクトにメールを設定する
        else // 検証エラーメッセージを表示する
```

より良い方法は、先ほどのようにコンストラクタにやらせることです。

```fsharp
let processFormSubmit () =
    let emailOpt = uiTextBox.Text |> EmailAddress.create
    match emailOpt with
    | Some email -> // ドメインオブジェクトにメールを設定する
    | None -> // 検証エラーメッセージを表示する
```

## When to "unwrap" single case unions ###

では、どのような場合にアンラップが必要なのでしょうか？繰り返しになりますが、一般的にはサービスの境界でのみ必要になります。例えば、メールをデータベースに永続化する場合や、UI要素やビューモデルにバインドする場合などです。

明示的なアンラップを避けるためのヒントとしては、継続的なアプローチを用いて、ラップされた値に適用される関数を渡すことです。

つまり、"unwrap "関数を明示的に呼び出すのではなく、次のようにします。

```fsharp
address |> EmailAddress.value |> printfn "the value is %s"
```

次のように、内側の値に適用される関数を渡すのです。

```fsharp
address |> EmailAddress.apply (printfn "the value is %s")
```

これをまとめると、完全な`EmailAddress`モジュールになります。

```fsharp
module EmailAddress =

    type _T = EmailAddress of string

    // create with continuation
    let createWithCont success failure (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then success (EmailAddress s)
            else failure "Email address must contain an @ sign"

    // create directly
    let create s =
        let success e = Some e
        let failure _  = None
        createWithCont success failure s

    // unwrap with continuation
    let apply f (EmailAddress e) = f e

    // unwrap directly
    let value e = apply id e

```

`create`と`value`の関数は厳密には必要ではありませんが、呼び出し側の利便性のために追加しています。

## これまでのコード ###

それでは、新しいラッパータイプとモジュールを追加して、`Contact`のコードをリファクタリングしてみましょう。

```fsharp
module EmailAddress =

    type T = EmailAddress of string

    // create with continuation
    let createWithCont success failure (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
            then success (EmailAddress s)
            else failure "Email address must contain an @ sign"

    // create directly
    let create s =
        let success e = Some e
        let failure _  = None
        createWithCont success failure s

    // unwrap with continuation
    let apply f (EmailAddress e) = f e

    // unwrap directly
    let value e = apply id e

module ZipCode =

    type T = ZipCode of string

    // create with continuation
    let createWithCont success failure  (s:string) =
        if System.Text.RegularExpressions.Regex.IsMatch(s,@"^\d{5}$")
            then success (ZipCode s)
            else failure "Zip code must be 5 digits"

    // create directly
    let create s =
        let success e = Some e
        let failure _  = None
        createWithCont success failure s

    // unwrap with continuation
    let apply f (ZipCode e) = f e

    // unwrap directly
    let value e = apply id e

module StateCode =

    type T = StateCode of string

    // create with continuation
    let createWithCont success failure  (s:string) =
        let s' = s.ToUpper()
        let stateCodes = ["AZ";"CA";"NY"] //etc
        if stateCodes |> List.exists ((=) s')
            then success (StateCode s')
            else failure "State is not in list"

    // create directly
    let create s =
        let success e = Some e
        let failure _  = None
        createWithCont success failure s

    // unwrap with continuation
    let apply f (StateCode e) = f e

    // unwrap directly
    let value e = apply id e

type PersonalName =
    {
    FirstName: string;
    MiddleInitial: string option;
    LastName: string;
    }

type EmailContactInfo =
    {
    EmailAddress: EmailAddress.T;
    IsEmailVerified: bool;
    }

type PostalAddress =
    {
    Address1: string;
    Address2: string;
    City: string;
    State: StateCode.T;
    Zip: ZipCode.T;
    }

type PostalContactInfo =
    {
    Address: PostalAddress;
    IsAddressValid: bool;
    }

type Contact =
    {
    Name: PersonalName;
    EmailContactInfo: EmailContactInfo;
    PostalContactInfo: PostalContactInfo;
    }

```

ところで、3つのラッパー・タイプ・モジュールの中には、かなり多くの重複したコードがあることに気がつきました。このコードを削除する、あるいは少なくともすっきりさせるには、どのような方法があるでしょうか？

## まとめ ###

差別化されたユニオンの使い方をまとめると、以下のようなガイドラインになります。

* ドメインを正確に表す型を作成するために、シングルケースの判別付きユニオンを使用する。
* ラップされた値に検証が必要な場合は、検証を行うコンストラクタを用意し、その使用を強制すること。
* 検証に失敗した場合に何が起こるかを明確にする。単純なケースでは、オプションタイプを返します。より複雑なケースでは、成功と失敗のハンドラを呼び出し側に渡すようにします。
* ラップされた値に多くの関連する関数がある場合、それを独自のモジュールに移すことを検討してください。
* カプセル化を強制する必要がある場合は、署名ファイルを使用します。

リファクタリングはまだ終わっていません。 違法な状態を表現できないように、コンパイル時にビジネスルールを強制するように型の設計を変更することができます。

{{< linktarget "update" >}}

## アップデート ##

多くの方から、`EmailAddress`のような制約のある型が、検証を行う特別なコンストラクタを通してのみ作成されるようにする方法について、より詳しい情報を求められました。
そこで、他の方法の詳細な例をまとめた[gist here](https://gist.github.com/swlaschin/54cfff886669ccab895a)を作成しました。