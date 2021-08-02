---
layout: post
title: "Inheritance and abstract classes"
description: ""
date: 2012-07-15
nav: fsharp-types
seriesId: "Object-oriented programming in F#"
seriesOrder: 3
categories: [Object-oriented, Classes]
---

これは、[クラスに関する前回の投稿](/posts/classes/)に続くものです。この記事では、F#における継承と、抽象クラスやインターフェイスの定義と使用方法に焦点を当てます。

## 継承

あるクラスが他のクラスを継承することを宣言するには、次のような構文を使います。

```fsharp
type DerivedClass(param1, param2) =
   inherit BaseClass(param1)
```

`inherit`キーワードは、`DerivedClass`が`BaseClass`を継承していることを示します。さらに、いくつかの `BaseClass` のコンストラクタが同時に呼ばれなければなりません。

ここで、F#とC#を比較してみるといいかもしれません。ここでは，非常に単純なクラスのペアに関するC#のコードを紹介します．

```csharp
public class MyBaseClass
{
    public MyBaseClass(int param1)
    {
        this.Param1 = param1;
    }
    public int Param1 { get; private set; }
}

public class MyDerivedClass: MyBaseClass
{
    public MyDerivedClass(int param1,int param2): base(param1)
    {
        this.Param2 = param2;
    }
    public int Param2 { get; private set; }
}
```

継承宣言である`class MyDerivedClass: MyBaseClass`は、`base(param1)`を呼び出すコンストラクタとは別のものです。

では，F#版を見てみましょう。

```fsharp
type BaseClass(param1) =
   member this.Param1 = param1

type DerivedClass(param1, param2) =
   inherit BaseClass(param1)
   member this.Param2 = param2

// テスト
let derived = new DerivedClass(1,2)
printfn "param1=%O" derived.Param1
printfn "param2=%O" derived.Param2
```

C#とは異なり、宣言の継承部分である`inherit BaseClass(param1)`には、継承するクラスとそのコンストラクタの両方が含まれています。

## 抽象メソッドと仮想メソッド

継承の目的の一つは、抽象メソッドや仮想メソッドなどを持てることです。

### ベースクラスでの抽象メソッドの定義

C#では、抽象メソッドは `abstract` キーワードとメソッドのシグネチャで示されます。F#でも同じ概念ですが、F#の関数シグネチャの書き方はC#とはかなり異なります。

```fsharp
// 具体的な関数定義
let Add x y = x + y

// 関数シグネチャ
// val Add : int -> int -> int
```

つまり，抽象メソッドを定義するには，シグネチャ構文と，`abstract member`キーワードを使います．

```fsharp
type BaseClass() =
   abstract member Add: int -> int -> int
```

等号の代わりにコロンが使われていることに注目してください。これは，等号が値の束縛に使われるのに対し，コロンが型の注釈に使われることから予想されることです。

さて、上記のコードをコンパイルしようとすると、エラーが発生します! コンパイラは、このメソッドの実装がないことを訴えます。これを修正するには、次のようにする必要があります。

* メソッドのデフォルトの実装を提供する、または
* クラス全体が抽象的であることをコンパイラに伝える。

以下では、この2つの方法について説明します。

### 抽象プロパティの定義

抽象的な不変のプロパティも同様に定義します。シグネチャは単純な値の場合と同じです。

```fsharp
type BaseClass() =
   abstract member Pi : float
```

抽象的なプロパティが読み書き可能であれば，get/setキーワードを追加します．

```fsharp
type BaseClass() =
   abstract Area : float with get,set
```

### デフォルトの実装(ただし、仮想メソッドはない)

ベースクラスの抽象メソッドのデフォルト実装を提供するには、`member`キーワードの代わりに`default`キーワードを使います。

```fsharp
// デフォルトの実装を持つ
type BaseClass() =
   // 抽象メソッド
   abstract member Add: int -> int -> int
   // 抽象的なプロパティ
   abstract member Pi : float

   // デフォールト
   default this.Add x y = x + y
   default this.Pi = 3.14
```

defaultメソッドは、`member`の代わりに`default`を使用している以外は、通常の方法で定義されていることがわかります。

F#とC#の大きな違いは、C#では抽象的な定義とデフォルトの実装を、`virtual`キーワードを使って1つのメソッドにまとめることができることです。F#ではそれができません。抽象的なメソッドとデフォルトの実装を別々に宣言しなければなりません。`abstract member`にはシグネチャを、`default`には実装を記述します。

### 抽象クラス

少なくとも1つの抽象メソッドがデフォルトの実装を持たない場合は、クラス全体が抽象クラスとなりますので、`AbstractClass`属性でアノテーションしてその旨を示す必要があります。

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

このようにしておけば、コンパイラが実装の不足を訴えることはなくなります。

### サブクラスでのメソッドのオーバーライド

抽象メソッドやプロパティをサブクラスでオーバーライドするには、`member`キーワードの代わりに`override`キーワードを使用します。 この変更以外では、オーバーライドされたメソッドは通常の方法で定義されます。

```fsharp
[<AbstractClass>]
type Animal() =
   abstract member MakeNoise: unit -> unit

type Dog() =
   inherit Animal()
   override this.MakeNoise () = printfn "woof"

// テスト
// let animal = new Animal() // ABC作成エラー
let dog = new Dog()
dog.MakeNoise()
```

また、ベースメソッドを呼び出すには、C#と同じように、`base`キーワードを使います。

```fsharp
type Vehicle() =
   abstract member TopSpeed: unit -> int
   default this.TopSpeed() = 60

type Rocket() =
   inherit Vehicle()
   override this.TopSpeed() = base.TopSpeed() * 10

// テスト
let vehicle = new vehicle()
printfn "vehicle.TopSpeed = %i" <| vehicle.TopSpeed()
let rocket = new Rocket()
printfn "rocket.TopSpeed = %i" <| rocket.TopSpeed()
```

### 抽象メソッドのまとめ

抽象メソッドは基本的に単純で、C#に似ています。C#に慣れている場合は、次の2つの領域に注意が必要です。

* 関数シグネチャとその構文を理解する必要があります。詳細については [関数シグネチャに関する投稿](/posts/function-signatures/) を参照してください。
* 一体型の仮想メソッドはありません。抽象メソッドとデフォルト実装は別々に定義する必要があります。

