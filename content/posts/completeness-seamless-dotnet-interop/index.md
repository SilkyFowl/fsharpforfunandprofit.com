---
layout: post
title: "Seamless interoperation with .NET libraries"
description: "Some convenient features for working with .NET libraries"
date: 2012-04-28
nav: why-use-fsharp
seriesId: "Why use F#?"
seriesOrder: 28
categories: [Completeness]
---


F#が.NETライブラリと一緒に使われている例は、`System.Net.WebRequest`や`System.Text.RegularExpressions`を使ったものなど、すでにたくさん見てきました。そして、その統合は実にシームレスでした。

より複雑な要求に対しては、F#は.NETのクラス、インターフェイス、構造体をネイティブにサポートしているので、相互運用は非常にわかりやすいものになっています。 例えば、C#で`ISomething`というインターフェイスを書き、その実装をF#で行うことができます。

しかし、F#は既存の.NETコードを呼び出すことができるだけでなく、ほとんどすべての.NET APIを他の言語に公開することができます。例えば、F#でクラスやメソッドを書いて、それをC#やVB、COMに公開することができます。 例えば、F#でクラスやメソッドを書いて、それをC#やVB、COMに公開することができますし、上の例を逆にして、F#で`ISomething`インターフェイスを定義し、その実装をC#で行うこともできます。 これらの利点は、既存のコードベースを捨てる必要がないということです。

緊密な統合に加えて、F#にはいくつかの優れた機能があり、.NETライブラリの作業をC#よりも便利にしてくれることがあります。私のお気に入りをいくつか紹介します。

* `TryParse`や`TryGetValue`を "out "パラメータを渡すことなく使用することができます。
* メソッドのオーバーロードを引数名で解決することができ、これは型推論にも役立ちます。
* アクティブパターン」を使用して、.NETのAPIをよりフレンドリーなコードに変換することができます。
* 具象クラスを作成することなく、`IDisposable`のようなインターフェイスからオブジェクトを動的に作成することができます。
* 「純粋な」F#オブジェクトと既存の.NET APIを混ぜて使うことができます。

## TryParse and TryGetValue ##

値や辞書に対する `TryParse` や `TryGetValue` 関数は、余分な例外処理を避けるためによく使われます。しかし、C#の構文は少々不便です。F#からこれらの関数を使うと、F#が自動的に関数をタプルに変換してくれるので、よりエレガントになります。

```fsharp
//using an Int32
let (i1success,i1) = System.Int32.TryParse("123");
if i1success then printfn "parsed as %i" i1 else printfn "parse failed"

let (i2success,i2) = System.Int32.TryParse("hello");
if i2success then printfn "parsed as %i" i2 else printfn "parse failed"

//using a DateTime
let (d1success,d1) = System.DateTime.TryParse("1/1/1980");
let (d2success,d2) = System.DateTime.TryParse("hello");

//using a dictionary
let dict = new System.Collections.Generic.Dictionary<string,string>();
dict.Add("a","hello")
let (e1success,e1) = dict.TryGetValue("a");
let (e2success,e2) = dict.TryGetValue("b");
```

## 型推論を助ける名前付き引数

C#（そして一般的な.NET）では、多くの異なるパラメータを持つオーバーロードされたメソッドを持つことができます。F#ではこれが問題になります。例えば、`StreamReader`を作成しようとすると、次のようになります。

```fsharp
let createReader fileName = new System.IO.StreamReader(fileName)
// error FS0041: A unique overload for method 'StreamReader'
//               could not be determined
```

問題は、引数が文字列なのかストリームなのか、F#がわからないことです。引数の型を明示的に指定することもできますが、それはF#のやり方ではありません。

代わりに、F#では、.NETライブラリのメソッドを呼び出すときに、名前付きの引数を指定できるという事実を利用して、素晴らしい回避策が可能です。

```fsharp
let createReader2 fileName = new System.IO.StreamReader(path=fileName)
```

上の例のように、多くの場合、引数名を使うだけで型の問題は解決します。また、明示的な引数名を使用することで、コードをより見やすくすることができます。

## .NET関数のアクティブパターン ##

.NETの型に対してパターンマッチを使用したいが、ネイティブライブラリはこれをサポートしていないという状況が多々あります。先ほど、「アクティブパターン」と呼ばれるF#の機能について簡単に触れましたが、これはマッチする選択肢を動的に作成することができるものです。この機能は、.NETとの統合に非常に役立ちます。

よくあるケースとしては、.NET ライブラリのクラスに、互いに排他的な `isSomething` や `isSomethingElse` といったメソッドがいくつもあり、見た目が悪い if-else 文のカスケードでテストしなければならない場合です。アクティブパターンでは、このような醜いテストをすべて隠し、残りのコードにはより自然なアプローチを採用することができます。

例えば、以下は`System.Char`の様々な`isXXX`メソッドをテストするコードです。

```fsharp
let (|Digit|Letter|Whitespace|Other|) ch =
   if System.Char.IsDigit(ch) then Digit
   else if System.Char.IsLetter(ch) then Letter
   else if System.Char.IsWhiteSpace(ch) then Whitespace
   else Other
```

選択肢が定義されると、通常のコードは簡単になります。

```fsharp
let printChar ch =
  match ch with
  | Digit -> printfn "%c is a Digit" ch
  | Letter -> printfn "%c is a Letter" ch
  | Whitespace -> printfn "%c is a Whitespace" ch
  | _ -> printfn "%c is something else" ch

// print a list
['a';'b';'1';' ';'-';'c'] |> List.iter printChar
```

もうひとつのよくあるケースは、テキストやエラーコードを解析して、例外や結果のタイプを判断しなければならない場合です。ここでは、アクティブパターンを使って `SqlExceptions` に関連するエラー番号を解析し、よりわかりやすくする例を紹介します。

まず、エラー番号にマッチするアクティブパターンを設定します。

```fsharp
open System.Data.SqlClient

let (|ConstraintException|ForeignKeyException|Other|) (ex:SqlException) =
   if ex.Number = 2601 then ConstraintException
   else if ex.Number = 2627 then ConstraintException
   else if ex.Number = 547 then ForeignKeyException
   else Other
```

これで，SQLコマンドを処理するときに，これらのパターンを使うことができるようになりました。

```fsharp
let executeNonQuery (sqlCommmand:SqlCommand) =
    try
       let result = sqlCommmand.ExecuteNonQuery()
       // handle success
    with
    | :?SqlException as sqlException -> // if a SqlException
        match sqlException with         // nice pattern matching
        | ConstraintException  -> // handle constraint error
        | ForeignKeyException  -> // handle FK error
        | _ -> reraise()          // don't handle any other cases
    // all non SqlExceptions are thrown normally
```

## インターフェイスから直接オブジェクトを作成する ##

F#にはもう一つ「オブジェクト式」という便利な機能があります。これは、具象クラスを最初に定義することなく、インターフェイスや抽象クラスから直接オブジェクトを作成する機能です。

以下の例では、`makeResource`ヘルパー関数を使って、`IDisposable`を実装したオブジェクトをいくつか作成しています。

```fsharp
// create a new object that implements IDisposable
let makeResource name =
   { new System.IDisposable
     with member this.Dispose() = printfn "%s disposed" name }

let useAndDisposeResources =
    use r1 = makeResource "first resource"
    printfn "using first resource"
    for i in [1..3] do
        let resourceName = sprintf "\tinner resource %d" i
        use temp = makeResource resourceName
        printfn "\tdo something with %s" resourceName
    use r2 = makeResource "second resource"
    printfn "using second resource"
    printfn "done."
```

この例では、「`use`」キーワードが、リソースがスコープ外になったときに、自動的にリソースを処分する様子も示しています。以下はその出力です。


	最初のリソースを使用
		内側のリソース1で何かをする
		内部リソース1が破棄される
		内部リソース 2 で何かをする
		インナーリソース2が廃棄される
		インナーリソース3で何かをする
		処分されるインナーリソース3
	2番目のリソースを使う
	されます。
	2つ目のリソースが廃棄されました
	第一のリソースが破棄される

## .NETインターフェースと純粋なF#型の混合 ##

インターフェイスのインスタンスをその場で作成できるということは、既存のAPIのインターフェイスと純粋なF#型を簡単に混ぜ合わせることができるということです。

例えば、以下のような `IAnimal` インターフェースを使った既存のAPIがあるとします。

```fsharp
type IAnimal =
   abstract member MakeNoise : unit -> string

let showTheNoiseAnAnimalMakes (animal:IAnimal) =
   animal.MakeNoise() |> printfn "Making noise %s"
```

しかし，パターン・マッチングなどの利点をすべて享受したいので，クラスの代わりに猫と犬用の純粋なF#型を作りました。

```fsharp
type Cat = Felix | Socks
type Dog = Butch | Lassie
```

しかし、この純粋なF#のアプローチを使うと、猫と犬を直接、`showTheNoiseAnAnimalMakes`関数に渡すことができません。

しかし，`IAnimal` を実装するために，新たに具象クラスを作成する必要はありません．代わりに，純粋なF#の型を拡張することで，`IAnimal`のインターフェイスを動的に作成することができます．

```fsharp
// now mixin the interface with the F# types
type Cat with
   member this.AsAnimal =
        { new IAnimal
          with member a.MakeNoise() = "Meow" }

type Dog with
   member this.AsAnimal =
        { new IAnimal
          with member a.MakeNoise() = "Woof" }
```

以下はテストコードです。

```fsharp
let dog = Lassie
showTheNoiseAnAnimalMakes (dog.AsAnimal)

let cat = Felix
showTheNoiseAnAnimalMakes (cat.AsAnimal)
```

この方法では、両方の良いところを得ることができます。内部的には純粋なF#の型でありながら、必要に応じてライブラリとのインタフェースに変換することができるのです。

## リフレクションを使ってF#の型を調べる ##

F#は、.NETのリフレクションシステムの恩恵を受けています。これは、言語自体の構文では直接利用できない、あらゆる種類の興味深いことができることを意味します。 Microsoft.FSharp.Reflection`名前空間には、F#の型に特化した機能が多数あります。

例えば、レコード型のフィールドやユニオン型の選択肢を出力する方法は以下の通りです。

```fsharp
open System.Reflection
open Microsoft.FSharp.Reflection

// create a record type...
type Account = {Id: int; Name: string}

// ... and show the fields
let fields =
    FSharpType.GetRecordFields(typeof<Account>)
    |> Array.map (fun propInfo -> propInfo.Name, propInfo.PropertyType.Name)

// create a union type...
type Choices = | A of int | B of string

// ... and show the choices
let choices =
    FSharpType.GetUnionCases(typeof<Choices>)
    |> Array.map (fun choiceInfo -> choiceInfo.Name)
```
