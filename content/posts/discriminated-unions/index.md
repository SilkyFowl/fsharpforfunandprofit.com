---
layout: post
title: "Discriminated Unions"
description: "Adding types together"
date: 2012-06-06
nav: fsharp-types
seriesId: "Understanding F# types"
seriesOrder: 6
categories: [Types]
---


タプルやレコードは、既存の型を"掛け合わせる"ことで新しい型を作る例です。 連載の冒頭で、新しい型を作るもう1つの方法は、既存の型を"合計"することだと述べました。これはどういうことでしょうか？

例えば、整数やブーリアンを扱う関数を定義し、それらを文字列に変換したいとします。 しかし、他の型（浮動小数点数や文字列など）を受け入れないように厳格にしたいとします。このような関数の図を示します。

![function from int union bool](./fun_int_union_bool.png)

この関数の定義域を表すにはどうしたらよいでしょうか？

必要なのは、可能な限りの整数と可能な限りのブーリアンを表す型です。

![int union bool](./int_union_bool.png)

つまり、"和 (sum) " タイプです。この場合、新しい型はinteger型にboolean型を加えた"sum"です。

F#では、sum型は "判別共用体" と呼ばれます。各構成要素の型 (*共用体のケース*) にはラベル (*ケース識別子*または*タグ*) を付けて、区別 ("判別") できるようにする必要があります。ラベルには任意の識別子を使用できますが、先頭は大文字にする必要があります。

上記の型を定義する方法を次に示します:

```fsharp
type IntOrBool =
  | I of int
  | B of bool
```

"I"と"B"は単なる任意のラベルです。意味のある他のラベルを使用することもできました。

小さな型の場合、定義を1行にまとめることができます。

```fsharp
type IntOrBool = int of int | B of bool
```

コンポーネント型には、タプル、レコード、その他の共用体など、任意の型を指定できます。

```fsharp
type Person = {first:string; last:string}  // レコード型を定義します
type IntOrBool = I of int | B of bool

type MixedType =
  | Tup of int * int  // タプル
  | P of Person       // 前述のレコード型を使用
  | L of int list     // intのリスト
  | U of IntOrBool    // 前述の共用体を使用
```

再帰的な型、つまり自身を参照する型も定義できます。一般に木構造の定義に使用されます。再帰型については、後で詳しく説明します。

### Sum型とC++の共用体、およびVBのバリアントとの比較

一見すると、sum型はC++のunion型やVisual Basicのvariant型に似ていますが、重要な違いがあります。C++のunion型は型安全ではなく、その型に格納されたデータには、任意のタグを使用してアクセスできます。F#の判別共用体は型安全であり、データにアクセスする方法は1つのみです。単なるデータのオーバーレイではなく、2つの型の合計 (図を参照) と考えると非常に便利です。

## 判別共用体に関する重要ポイント

判別共用体について知っておくべき重要なことは、次のとおりです:

* 最初の構成要素の前にある縦線は省略可能であるため、インタラクティブウィンドウの出力を確認するとわかるように、次の定義は全て等価です:

```fsharp
type IntOrBool = I of int | B of bool     //  先頭の棒を省略
type IntOrBool = | I of int | B of bool   // 先頭の棒を配置
type IntOrBool =
   | I of int
   | B of bool      // 各行頭に棒を配置
```

* タグやラベルは大文字で始まる必要があります。したがって、次の場合はエラーになります:

```fsharp
type IntOrBool = int of int| bool of bool
//  error FS0053: 判別された共用体ケースと例外のラベルは、
//                大文字の識別子にする必要があります
```

* 他の名前付き型 (`Person`や`IntOrPool`など) は、判別共用体の外部で事前に定義しておく必要があります。次のように"インライン" で定義することはできません:

```fsharp
type MixedType =
  | P of  {first:string; last:string}  // error
```

または

```fsharp
type MixedType =
  | U of (I of int | B of bool)  // error
```

* ラベルには、任意の識別子 (構成要素型自体の名前を含む) を使用できますが、知らない人は非常に混乱するでしょう。たとえば、 (`System`名前空間の) `Int32`型や`Boolean`型が使われ、ラベルの名前も同じである場合、次のような定義は完全に有効です:

```fsharp
open System
type IntOrBool = Int32 of Int32 | Boolean of Boolean
```

この"重複した名前付け" は、コンポーネントの型が正確に記述されているため、よく使用されます。

{{< book_page_pdf >}}。

## 判別共用体の値を構築する

判別共用体の値を構築するには、考えられる共用体のケースを1つだけ参照する 「コンストラクター」 を使用します。コンストラクタは定義の形式に従い、ケース識別子を関数のように使用します。`IntOrPool`の例の場合、次のように記述します:

```fsharp
type IntOrBool = I of int | B of bool

let i = I 99    // "I"のコンストラクタ
// val i : IntOrBool = I 99

let b = B true // "B"のコンストラクタ
// val b : IntOrBool = B true
```

結果の値は、構成要素の型とともにケース識別子とともに出力されます。

```fsharp
val [値の名前] : [型]    = [ケース識別子] [構成要素の型の出力]
val i            : IntOrBool = I       99
val b            : IntOrBool = B       true
```

ケースのコンストラクタに複数の "パラメータ"がある場合は、関数を呼び出すのと同じ方法で構築します。

```fsharp
type Person = {first:string; last:string}

type MixedType =
  | Tup of int * int
  | P of Person

let myTup = Tup (2,99)    // "Tup"のコンストラクタ
// val myTup : MixedType = Tup (2,99)

let myP = P {first=“Al”; last=“Jones” } // "P"のコンストラクタ
// val myP : MixedType = P {first = "Al";last = "Jones";}
```

判別共用体のケース・コンストラクターは通常の関数なので、関数が期待される場所であればどこでも使用することができます。たとえば、`List.map`では次のようになります:

```fsharp
type C = Circle of int | Rectangle of int * int

[1..10]
|> List.map Circle

[1..10]
|> List.zip [21..30]
|> List.map Rectangle
```

### 名前の衝突

特定のケースに一意の名前がある場合、構築される型は明白です。

しかし、同じ識別子のケースが2種類ある場合はどうなるのでしょうか?

```fsharp
type IntOrBool1 = I of int | B of bool
type IntOrBool2 = I of int | B of bool
```

この場合、通常は最後に定義されたものが使用されます。

```fsharp
let x = I 99                // val x : IntOrBool2 = I 99
```

しかし、以下のように型を明示的に指定する方がはるかに良いでしょう。

```fsharp
let x1 = IntOrBool1.I 99    // val x1 : IntOrBool1 = I 99
let x2 = IntOrBool2.B true  // val x2 : IntOrBool2 = B true
```

型が異なるモジュールに由来する場合は、モジュール名を使うこともできます。

```fsharp
module Module1 =
  type IntOrBool = I of int | B of bool

module Module2 =
  type IntOrBool = I of int | B of bool

module Module3=
  let x = Module1.IntOrBoot.I 99 // val x : Module1.IntOrBool = I 99
```


### 判別共用体のマッチング

タプルとレコードに関しては、"分解"するために値を構成するのと同じ型を使うことを見てきました。これは判別共用体にも当てはまりますが、複雑な問題があります。どのケースを分解すべきなのでしょうか?

これこそ "match"式の目的です。お気付きのように、match式の構文は、判別共用体の定義方法と似ています。

```fsharp
// 判別共用体の定義
type MixedType =
  | Tup of int * int
  | P of Person

// 判別共用体の "分解"
let matcher x =
  match x with
  | Tup (x, y) ->
        printfn "Tuple matched with %i %i" x y
  | P {first=f, last=l} ->
        printfn "Person matched with %s %s" f l

let myTup = Tup (2,99)                 // "Tup"のコンストラクタ
matcher myTup

let myP = P {first="Al", last="Jones"} // "P"のコンストラクタ
matcher myP
```

ここで何が起こっているかを分析してみましょう。

* マッチ式全体の各 "枝"は、対応する共用体型のケースにマッチするように設計されたパターン式です。
* パターンは特定のケースのタグで始まり、パターンの残りの部分は通常の方法で該当ケースの型を分解します。
* パターンの後には、矢印"->"と実行されるコードが続きます。


## 空のケース

判別共用体のケース識別子は、そのあとに型を指定する必要はありません。以下の例にすべての有効な判別共用体のです:

```fsharp
type Directory =
  | Root                   // ルートに名前を付ける必要はありません
  | Subdirectory of string // 他のディレクトリには名前を付ける必要があります

type Result =
  | Success                // 成功の状態に文字列は必要ありません
  | ErrorMessage of string // エラーメッセージが必要です
```

*全て*のケースが空の場合、"列挙形式"の判別共用体になります。

```fsharp
type Size = Small | Medium | Large
type Answer = Yes | No | Maybe
```

この"列挙形式"の判別共用体は、後述する本当のC#の列挙型と*同じではない*ことに注意してください。

空のケースを作成するには、パラメータなしのコンストラクタとしてケース識別子を使用します。

```fsharp
let myDir1 = Root
let myDir2 = Subdirectory "bin"

let myResult1 = Success
let myResult2 = ErrorMessage "not found"

let mySize1 = Small
let mySize2 = Medium
```

{{< linktarget "single-case" >}}。

## シングルケース

1つのケースのみを持つ判別共用体を作成すると便利です。価値を付加しているようには見えないので、無意味に思えるかもしれません。しかし実際には、これは型安全性*を実現する非常に有用な手法です。

{{}}
今後のシリーズでは、モジュール・シグニチャーと組み合わせて、単一ケースの判別共用体がデータの隠蔽やケイパビリティ・ベースのセキュリティーに役立つことを説明します。
{{}}

たとえば、顧客IDと注文IDの両方が整数で表されているが、互いに割り当ててはならないとします。

前述したように、型エイリアスの手法ではうまくいきません。なぜなら、型エイリアスは単なるシノニムであり、異なる型を生成するわけではないからです。以下は型エイリアスを使った方法です:

```fsharp
type CustomerId = int   // 型エイリアスを定義します
type OrderId = int      // 別のエイリアスを定義します

let printOrderId (orderId:OrderId) =
   printfn "The orderId is %i" orderId

// 試してみます
let custId = 1          // 顧客IDを作成する
printOrderId custId   // ああ!
```

たとえ明示的に `orderId`パラメータに`OrderId`型の注釈を付けたとしても、顧客IDが誤って渡されないことを保証することはできません。

一方、単一の判別共用体を作成すると、型の識別を簡単に強制できます。

```fsharp
type CustomerId = CustomerId of int   // 判別共用体を定義します
type OrderId = OrderId of int         // 別の判別共用体を定義します

let printOrderId (OrderId orderId) =  // paramセクションでデコンストラト
   printfn "The orderId is %i" orderId

// 試してみます
let custId = CustomerId 1             // 顧客IDを作成する
printOrderId custId                   // Good! 今度はコンパイラエラーです。
```

このアプローチはC#やJavaでも実現可能ですが、型ごとに特別なクラスを作成して管理するというオーバーヘッドがあるため、ほとんど使われていません。 F#ではこのアプローチは軽量なので、よく使われています。

シングルケースの判別共用体の便利な点は、完全な `match-with` 式を使わなくても、値に対して直接パターンマッチできることです。

```fsharp
// paramでデコンストラクト
let printCustomerId (CustomerId customerIdInt) =
   printfn "The CustomerId is %i" customerIdInt

// または let 文で明示的にデコンストラストします。
let printCustomerId2 custId =
   let (CustomerId customerIdInt) = custId // ここでデコンストラクトします。
   printfn "The CustomerId is %i" customerIdInt

// 試してみる
let custId = CustomerId 1 // 顧客IDを作成します。
printCustomerId custId
printCustomerId2 custId
```

ただ、よくある "落とし穴"は、パターンマッチの際に括弧を付けなければならない場合があることです。そうしないと、コンパイラは関数を定義していると判断してしまいます！

```fsharp
let custId = CustomerId 1
let (CustomerId customerIdInt) = custId // 正しいパターンマッチ
let CustomerId customerIdInt = custId // 間違っています! 新しい関数？
```

同様に、単一ケースの列挙型形式の判別共用体を定義する必要がある場合、型定義は縦棒から始める必要があります。そうしないと、コンパイラは型エイリアスを作成していると判断してしまいます。

```fsharp
type TypeAlias = A // 型エイリアス!
type SingleCase = | A // シングルケースの判別共用体
```


## 判別共用体の等式 ##

他のF#型と同様に、判別共用体には自動的に定義された等価演算があります。2つの判別共用体が同一型で同じケースを持ち、そのケースの値が等しい場合、2つの判別共用体は等価です。

```fsharp
type Contact = Email of string | Phone of int

let email1 = Email "bob@example.com"
let email2 = Email "bob@example.com"

let areEqual = (email1=email2)
```


## 判別共用体の表記 ##

は判別共用体には適切なデフォルトの文字列表現があり、簡単にシリアライズできます。しかし、タプルとは異なり、ToString() による表現は役に立ちません。

```fsharp
type Contact = Email of string | Phone of int
let email = Email "bob@example.com"
printfn "%A" email // いいね
printfn "%O" email // 醜い!
```

