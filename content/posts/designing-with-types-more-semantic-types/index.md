---
layout: post
title: "Designing with types: Constrained strings"
description: "Adding more semantic information to a primitive type"
date: 2013-01-17
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 6
categories: [Types, DDD]
---

[以前の記事](/posts/designing-with-types-single-case-dus/)で、メールアドレス、郵便番号、州などに、プレーンなプリミティブ文字列を使わないようにすることをお話しました。
これらをシングルケースユニオンでラップすることで、(a)型を強制的に区別することができ、(b)検証ルールを追加することができます。

今回の記事では、この概念をさらに細かいレベルまで拡張できるかどうかを見てみましょう。

## 文字列が文字列でない場合とは？

単純な`PersonalName`という型を見てみましょう。

```fsharp
type PersonalName =
    {
    FirstName: string;
    LastName: string;
    }
```

型を見ると、ファーストネームは `string` となっています。しかし、本当にそれだけなのでしょうか？ 他に追加しなければならない制約はあるでしょうか？

そうですね、NULLであってはいけません。でも、それはF#では想定されています。

文字列の長さはどうでしょう？64K文字の長さの名前を持つことは許容されますか？そうでない場合は、何か最大の長さが許されるのでしょうか？

また，名前に改行文字やタブを含めることはできますか？ また、名前に改行文字やタブを含めることはできますか。また、名前の最初や最後に空白を含めることはできますか。

このように考えると、「一般的な」文字列であっても、かなり多くの制約があることがわかります。ここでは、そのうちのいくつかを紹介します。

* 最大の長さは何ですか？
* 複数の行にまたがることができるか？
* 先行または後続のホワイトスペースを持つことができますか？
* 非印刷文字を含むことができますか？

## これらの制約はドメインモデルの一部であるべきでしょうか？

では、いくつかの制約が存在することは認められるかもしれませんが、それらは本当にドメインモデル（およびそこから派生する対応する型）の一部であるべきでしょうか？
例えば、名字は100文字までという制約は、確かに特定の実装に特有のもので、ドメインの一部ではありません。

私は、論理モデルと物理モデルの間には違いがあると答えます。 論理モデルでは、これらの制約のいくつかは関係ないかもしれませんが、物理モデルでは確実に関係してきます。私たちがコードを書いているときは、常に物理モデルを扱っているのです。

制約条件をモデルに組み込むもう1つの理由は、モデルが多くの別々のアプリケーションで共有されることが多いからです。例えば、個人名はeコマースアプリケーションで作成され、データベースのテーブルに書き込まれ、CRMアプリケーションが受け取るためにメッセージキューに入れられ、次にメールテンプレートサービスを呼び出すといった具合です。

これらすべてのアプリケーションやサービスが、個人名の長さやその他の制約を含めて、個人名について同じ考えを持つことが重要です。モデルが制約を明示していない場合、サービスの境界を越えて移動する際にミスマッチが生じやすくなります。

例えば、文字列をデータベースに書き込む前に、その文字列の長さをチェックするコードを書いたことはありませんか？

```csharp
void SaveToDatabase(PersonalName personalName)
{
   var first = personalName.First;
   if (first.Length > 50)
   {
        // ensure string is not too long
        first = first.Substring(0,50);
   }

   //save to database
}
```

この時点で文字列が長すぎる場合、どうすればいいのでしょうか？黙って切り捨てる？例外を発生させますか？

より良い答えは、できればこの問題を完全に回避することです。文字列がデータベース層に到達してからでは遅すぎるのです。

この問題は、文字列が使用されたときではなく、文字列が最初に作成されたときに処理されるべきです。 言い換えれば、これは文字列の検証の一部であったはずです。

しかし、可能性のあるすべてのパスについて、検証が正しく行われたことをどうやって信頼できるでしょうか？ 答えはおわかりですね。

## 型で制約された文字列のモデリング

もちろん、その答えは、型に組み込まれた制約を持つラッパー型を作ることです。

そこで、以前使ったシングルケースユニオンのテクニックを使って、簡単なプロトタイプを作ってみましょう(/posts/designing-with-types-single-case-dus/)。

```fsharp
module String100 =
    type T = String100 of string
    let create (s:string) =
        if s <> null && s.Length <= 100
        then Some (String100 s)
        else None
    let apply f (String100 s) = f s
    let value s = apply id s

module String50 =
    type T = String50 of string
    let create (s:string) =
        if s <> null && s.Length <= 50
        then Some (String50 s)
        else None
    let apply f (String50 s) = f s
    let value s = apply id s

module String2 =
    type T = String2 of string
    let create (s:string) =
        if s <> null && s.Length <= 2
        then Some (String2 s)
        else None
    let apply f (String2 s) = f s
    let value s = apply id s
```

検証に失敗した場合には、結果としてオプション型を使うことですぐに対応しなければならないことに注意してください。 これは作成をより困難にしていますが、後で利益を得たいのであれば避けることはできません。

例えば、長さ2の良い文字列と悪い文字列があるとします。

```fsharp
let s2good = String2.create "CA"
let s2bad = String2.create "California"

match s2bad with
| Some s2 -> // update domain object
| None -> // handle error
```

`String2`の値を使用するためには、作成時に`Some`か`None`かをチェックする必要があります。

### この設計の問題点

1つの問題点は、多くの重複したコードがあることです。実際には、典型的なドメインでは数十個の文字列型しかないので、それほど無駄なコードはないでしょう。しかし、それでも、もっと良い方法があるはずです。

もうひとつの深刻な問題は、比較が難しくなることです。`String50`は `String100`とは異なる型なので，直接比較することはできません。

```fsharp
let s50 = String50.create "John"
let s100 = String100.create "Smith"

let s50' = s50.Value
let s100' = s100.Value

let areEqual = (s50' = s100') // コンパイラエラー
```

このようなことがあると、辞書やリストの扱いが難しくなります。

{{< book_page_pdf >}}

### リファクタリング

ここで、F#がサポートしているインターフェースを利用して、すべてのラップされた文字列がサポートしなければならない共通のインターフェースと、いくつかの標準的な関数を作成することができます。

```fsharp
module WrappedString =

    /// An interface that all wrapped strings support
    type IWrappedString =
        abstract Value : string

    /// ラップされた値のオプションを作成する
    /// 1) まず入力を正規化する
    /// 2) 検証が成功した場合，与えられたコンストラクタの Some を返す
    /// 3) 検証に失敗した場合は、None を返します。
    /// Null 値は決して有効ではありません。
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

    /// Equality test
    let equals left right =
        (value left) = (value right)

    /// Comparison
    let compareTo left right =
        (value left).CompareTo (value right)
```

鍵となるのは `create` という関数で、コンストラクタ関数を受け取り、バリデーションに合格したときだけ、その関数を使って新しい値を作成します。

これがあると、新しい型を定義するのがとても簡単になります。

```fsharp
モジュールWrappedString =

    // ... 上のコード ...

    /// 文字列を構築する前に、正規化します。
    /// * すべてのホワイトスペースをスペース文字に変換します。
    /// * 両端をトリムする
    let singleLineTrimmed s =
        System.Text.RegularExpressions.Regex.Replace(s,"\s"," ").Trim()

    /// 長さに基づいた検証関数
    let lengthValidator len (s:string) =
        s.Length <= len

    /// 長さ100の文字列
    type String100 = String100 of string with
        interface IWrappedString with
            member this.Value = let (String100 s) = this in s

    /// 長さ100の文字列のコンストラクタ
    let string100 = create singleLineTrimmed (lengthValidator 100) String100

    /// 折り返した文字列を長さ100の文字列に変換します。
    let convertTo100 s = apply string100 s

    /// 長さ50の文字列
    type String50 = String50 of string with
        interface IWrappedString with
            member this.Value = let (String50 s) = this in s

    /// 長さ50の文字列のためのコンストラクタ
    let string50 = create singleLineTrimmed (lengthValidator 50)  String50

    /// 折り返した文字列を長さ 50 の文字列に変換します。
    let convertTo50 s = apply string50 s
```

文字列の種類ごとに、次のようにすればよいのです。

* 型を作成する（例：`String100`）。
* その型のための `IWrappedString` の実装
* そして、その型のパブリックコンストラクタ（例：`string100`）です。

(上のサンプルでは、ある型から別の型に変換するために、便利な `convertTo` も投入しています)。

型はこれまで見てきたように、単純なラップ型です。

IWrappedStringの`Value`メソッドの実装は，以下のように複数の行を使って書くことができました．

```fsharp
member this.Value =
    let (String100 s) = this
    s
```

しかし、私はワンライナーのショートカットを使うことにしました。

```fsharp
member this.Value = let (String100 s) = this in s
```

コンストラクタ関数も非常にシンプルです。canonicalize関数は`singleLineTrimmed`、validator関数は長さをチェックし、コンストラクタは`String100`関数（シングルケースに関連する関数で、同名の型と混同しないように）です。

```fsharp
let string100 = create singleLineTrimmed (lengthValidator 100) String100
```

もし、異なる制約を持つ他の型を持ちたい場合は、簡単に追加することができます。例えば、複数行や埋め込みタブをサポートし、トリミングされない`Text1000`という型を持ちたいとします。

```fsharp
モジュールWrappedString =

    // ... 上のコード ...

    /// 長さ1000の複数行のテキスト
    type Text1000 = Text1000 of string with
        interface IWrappedString with
            member this.Value = let (Text1000 s) = this in s

    /// 長さ1000のマルチライン文字列のコンストラクタ
    let text1000 = create id (lengthValidator 1000) Text1000
```

### WrappedStringモジュールで遊ぶ

それでは、このモジュールがどのように動作するか、インタラクティブに遊んでみましょう。

```fsharp
let s50 = WrappedString.string50 "abc" |> Option.get
printfn "s50 is %A" s50
let bad = WrappedString.string50 null｜> Option.get
printfn "bad is %A" bad
let s100 = WrappedString.string100 "abc" |> Option.get
printfn "s100 is %A" s100

// モジュール関数を使った同等性は真
printfn "s50 is equal to s100 using module equals? %b" (WrappedString.equals s50 s100)

// オブジェクト・メソッドを使った等値性は偽です
printfn "s50はObject.Equalsを使ってs100と同じですか？%b" (s50.Equals s100)

// 直接的な等値性はコンパイルされません
printfn "s50 is equal to s100? %b" (s50 = s100) // コンパイラエラー
```

マップのように生の文字列を使用するタイプを操作する必要がある場合は、新しいヘルパー関数を簡単に作成することができます。

例えば，マップを扱うためのヘルパーをいくつか紹介します．

```fsharp
モジュールWrappedString =

    // ... 上のコード ...

    /// マップヘルパー
    let mapAdd k v map =
        Map.add (value k) v map

    let mapContainsKey k map =
        Map.containsKey (value k) map

    let mapTryFind k map =
        Map.tryFind (value k) map
```

これらのヘルパーが実際にどのように使われるかというと、次のようになります。

```fsharp
let abc = WrappedString.string50 "abc" |> Option.get
let def = WrappedString.string100 "def" |> Option.get
let map =
    Map.empty
    |> WrappedString.mapAdd abc "value for abc"
    |> WrappedString.mapAdd def "value for def"

printfn "Found abc in map? %A" (WrappedString.mapTryFind abc map)

let xyz = WrappedString.string100 "xyz" |> Option.get
printfn "Found xyz in map? %A" (WrappedString.mapTryFind xyz map)
```

以上、全体的に見て、この「WrappedString」モジュールは、 あまり干渉することなく、きれいに型付けされた文字列を作成することができます。では、実際の場面で使ってみましょう。

## ドメインで新しい文字列型を使う

型ができたので、`PersonalName`型の定義を変更して、それを使うようにしましょう。

```fsharp
module PersonalName =
    open WrappedString

    type T =
        {
        FirstName: String50;
        LastName: String100;
        }

    /// create a new value
    let create first last =
        match (string50 first),(string100 last) with
        | Some f, Some l ->
            Some {
                FirstName = f;
                LastName = l;
                }
        | _ ->
            None
```

この型のモジュールを作成し、文字列のペアを `PersonalName` に変換する作成関数を追加しました。

ここで、入力された文字列の *どちらか* が無効な場合にどうするかを決めなければならないことに注意してください。繰り返しになりますが、この問題を後回しにすることはできませんので、構築時に対処しなければなりません。

この場合、失敗を示すためにNoneを持つオプションタイプを作成するというシンプルな方法を使います。

ここではそれを使っています。

```fsharp
let name = PersonalName.create "John" "Smith"
```

また，モジュールの中で追加のヘルパー関数を提供することもできます。

例えば、名字と名前を結合して返す`fullname`関数を作りたいとしましょう。

ここでもまた、決定しなければならないことがあります。

* 生の文字列を返すべきか、それともラップされた文字列を返すべきか。
  後者の利点は、呼び出し側が文字列の長さを正確に知ることができることと、他の類似した型との互換性があることです。

* もし、ラップされた文字列（例えば、`String100`）を返すとしたら、結合された長さが長すぎる場合、どのように処理するのでしょうか？(名字と名前のタイプの長さに基づいて、最大で151文字になる可能性があります)。組み合わせた長さが長すぎる場合は、オプションを返すか、強制的に切り捨てることができます。

以下に，3つのオプションを示すコードを示します．

```fsharp
モジュール PersonalName =

    // ... 上のコード ...

    /// 姓と名を連結します。
    /// 生の文字列を返す
    let fullNameRaw personalName =
        let f = personalName.FirstName |> value
        let l = personalName.LastName |> value
        f + " " + l

    /// ファーストネームとラストネームを連結する
    /// 長すぎる場合はNoneを返す
    let fullNameOption personalName =
        personalName |> fullNameRaw |> string100

    /// 名字と名前を連結する
    /// 長すぎる場合は切り捨てる
    let fullNameTruncated personalName =
        // ヘルパー関数
        let left n (s:string) =
            if (s.Length > n)
            then s.Substring(0,n)
            else s

        パーソナルネーム
        |> fullNameRaw // concat
        |> left 100 // 切り捨て
        |> string100 // 折り返し
        |> Option.get // これは常にOKです。

```

`fullName`の実装にどのようなアプローチをとるかは、あなた次第です。 しかし、このスタイルの型指向設計についての重要なポイントを示しています。これらの決定は、コードを作成するときに *前もって* 行わなければなりません。 後回しにすることはできません。

これは時に非常に厄介なことですが、全体としては良いことだと思います。

## メールアドレスと郵便番号のタイプを見直す

このWrappedStringモジュールを利用して、`EmailAddress`や`ZipCode`の型を再実装することができます。

```fsharp
モジュール EmailAddress =

    type T = EmailAddress of string with
        interface WrappedString.IWrappedString with
            member this.Value = let (EmailAddress s) = this in s

    let create =
        let canonicalize = WrappedString.singleLineTrimmed
        let isValid s =
            (WrappedString.lengthValidator 100 s) &&
            System.Text.RegularExpressions.Regex.IsMatch(s,@"^\S+@\S+\.\S+$")
        WrappedString.create canonicalize isValid EmailAddress

    /// 任意のWrappedStringをEmailAddressに変換します。
    let convert s = WrappedString.apply create s

module ZipCode =

    type T = ZipCode of string with
        interface WrappedString.IWrappedString with
            member this.Value = let (ZipCode s) = this in s

    let create =
        let canonicalize = WrappedString.singleLineTrimmed
        let isValid s =
            System.Text.RegularExpressions.Regex.IsMatch(s,@"^\d{5}$")
        WrappedString.create canonicalize isValid ZipCode

    /// 任意のWrappedStringをZipCodeに変換します。
    let convert s = WrappedString.apply create s
```

## その他の折り返し文字列の使い方

文字列をラップするこの方法は、文字列タイプを不用意に混ぜたくない場合にも使えます。

ぱっと思いつくのは、Webアプリケーションで文字列を安全にクォートしたりアンクォートしたりする場合です。

例えば、文字列をHTMLに出力したいとします。 その文字列はエスケープされているべきでしょうか？
すでにエスケープされている場合はそのままにしておきたいですが、エスケープされていない場合はエスケープしたいですよね。

これは厄介な問題です。Joel Spolsky氏は命名規則の使用について [こちら](http://www.joelonsoftware.com/articles/Wrong.html)で説明していますが、もちろんF#では代わりにタイプベースのソリューションを求めています。

型ベースのソリューションでは、おそらく、「安全な」（すでにエスケープされている）HTMLの文字列（`HtmlString`と言います）のための型、安全なJavascriptの文字列（`JsString`）のための型、安全なSQLの文字列（`SqlString`）のための型、などを使用するでしょう。
そうすれば、これらの文字列を安全に混ぜ合わせることができ、誤ってセキュリティ問題を引き起こすこともありません。

ここでは解決策を作りませんが（いずれにしてもRazorのようなものを使うことになるでしょう）、もし興味があれば、[Haskellのアプローチ](http://blog.moertel.com/articles/2006/10/18/a-type-based-solution-to-the-strings-problem)や[F#への移植](http://stevegilham.blogspot.co.uk/2011/12/approximate-type-based-solution-to.html)について読むことができます。


## アップデート ##

多くの方から、`EmailAddress`のような制約のある型が、検証を行う特別なコンストラクタを通してのみ作成されることを保証する方法について、より詳しい情報を求められました。
そこで、他の方法の詳細な例をまとめた[gist here](https://gist.github.com/swlaschin/54cfff886669ccab895a)を作成しました。

{{< book_page_ddd_img >}}