---
layout: post
title: "Interfaces"
description: ""
date: 2012-07-16
nav: fsharp-types
seriesId: "Object-oriented programming in F#"
seriesOrder: 4
categories: [Object-oriented, Interfaces]
---

F#ではインターフェイスが利用でき、完全にサポートされていますが、その使い方はC#で慣れ親しんだものとは異なる重要な点がいくつかあります。

### インターフェースの定義

インターフェースの定義は、抽象クラスの定義に似ています。インターフェイスの定義は、抽象クラスの定義と似ていますが、非常に似ているため、混同しやすいかもしれません。

以下にインターフェースの定義を示します。

```fsharp
type MyInterface = // 抽象クラス
   // 抽象メソッド
   abstract member Add: int -> int -> int

   // 抽象的な不変のプロパティ
   abstract member Pi : float

   // 抽象的な読み書き可能なプロパティ
   abstract member Area : float with get,set
```

また、同等の抽象ベースクラスの定義は以下の通りです。

```fsharp
[<AbstractClass>]
type AbstractBaseClass() =
   // 抽象メソッド
   abstract member Add: int -> int -> int

   // 抽象的な不変のプロパティ
   abstract member Pi : float

   // 抽象的な読み書き可能なプロパティ
   abstract member Area : float with get,set
```

では、何が違うのでしょうか？いつものように、すべての抽象メンバーはシグネチャのみで定義されています。唯一の違いは `[<AbstractClass>]` 属性がないことのようです。

しかし，先ほどの抽象メソッドの説明では，`[<AbstractClass>]` 属性が必要であることを強調しました。そうしないと，コンパイラがメソッドには実装がないと文句を言うからです。では、インターフェイスの定義はどのようにして守られているのでしょうか？

答えは些細なことですが、微妙です。*インターフェイスにはコンストラクタがありません*。つまり，インターフェイス名の後に括弧をつけていないのです。

```fsharp
type MyInterface = // <- 括弧がない!
```

これで終わりです。 括弧を削除すると、クラス定義がインターフェイスに変換されます!

### 明示的・暗黙的なインターフェイスの実装

クラスにインターフェイスを実装する場合、F#はC#とかなり異なります。 C#では、クラス定義にインターフェースのリストを追加して、暗黙的にインターフェースを実装することができます。

F#ではそうではありません。F#では、すべてのインターフェイスは*明示的に*実装しなければなりません。

明示的なインターフェイスの実装では、インターフェイスのメンバーは、インターフェイスのインスタンスを通してのみアクセスできます（例えば、クラスをインターフェイスの型にキャストすることで）。インターフェースのメンバーは、クラス自体の一部としては見えません。

C#では、明示的なインターフェース実装と暗黙的なインターフェース実装の両方がサポートされていますが、ほとんどの場合、暗黙的なアプローチが使用されており、多くのプログラマーは[C#における明示的なインターフェース](http://msdn.microsoft.com/en-us/library/ms173157.aspx)を意識していません。


### F#でのインターフェイスの実装 ###

では、F#でインターフェイスを実装するにはどうすればよいのでしょうか。 抽象的な基底クラスのように、単に「継承」することはできません。 以下のように、`interface XXX with`という構文を使って、インターフェイスの各メンバーに明示的な実装を行う必要があります。

```fsharp
type IAddingService =
    abstract member Add: int -> int -> int

type MyAddingService() =

    interface IAddingService with
        member this.Add x y =
            x + y

    interface System.IDisposable with
        member this.Dispose() =
            printfn "disposed"
```

上記のコードは，クラス `MyAddingService` がどのようにして `IAddingService` と `IDisposable` インターフェースを明示的に実装しているかを示しています。必要な `interface XXX with` セクションの後、メンバーは通常の方法で実装されています。

(余談ですが、`MyAddingService()`にはコンストラクタがありますが、`IAddingService`にはないことに再度注意してください)。

### インターフェースの利用

それでは、追加サービスのインターフェースを使ってみましょう。

```fsharp
let mas = new MyAddingService()
mas.Add 1 2 // エラー
```

すぐにエラーが発生します。どうやら、このインスタンスは`Add`メソッドを全く実装していないようです。もちろん、これが意味するところは、まず `:>` 演算子を使ってインターフェイスにキャストする必要があるということです。

```fsharp
// インターフェースへのキャスト
let mas = new MyAddingService()
let adder = mas :> IAddingService
adder.Add 1 2 // OK
```

これは非常に厄介に思えるかもしれませんが、ほとんどの場合、暗黙のうちにキャストが行われるので、実際には問題ありません。

例えば，インターフェイスのパラメータを指定した関数にインスタンスを渡すことがよくあります。この場合、キャスティングは自動的に行われます。

```fsharp
// インターフェースを必要とする関数
let testAddingService (adder:IAddingService) =
    printfn "1+2=%i" <| adder.Add 1 2 // OK

let mas = new MyAddingService()
testAddingService mas // 自動的にキャスト
```

また、`IDisposable`の特殊なケースでは、`use`キーワードも必要に応じて自動的にインスタンスをキャストします。

```fsharp
let testDispose =
    use mas = new MyAddingService()
    printfn "testing"
    // ここでDispose()が呼ばれる
```

