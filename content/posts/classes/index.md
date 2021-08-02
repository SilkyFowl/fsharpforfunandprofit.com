---
layout: post
title: "Classes"
description: ""
date: 2012-07-14
nav: fsharp-types
seriesId: "Object-oriented programming in F#"
seriesOrder: 2
categories: [Object-oriented, Classes]
---

この記事と次の記事では、F#でクラスとメソッドを作成して使用するための基本的な方法を説明します。

## クラスの定義

F#の他のデータ型と同様に、クラスの定義は `type` キーワードで始まります。

他のデータ型との違いは、クラスはコンストラクタで生成される際に必ずパラメータが渡されるため、クラス名の後には必ず*括弧が付くことです。

また、他の型とは異なり、クラスには必ず*関数をメンバーとして付けなければなりません。この記事ではクラスに関数を付ける方法を説明しますが、他の型に関数を付けることについての一般的な議論は[型の拡張についての記事](/posts/type-extensions/)を参照してください。

ですから、例えば、`CustomerName`というクラスがあって、それを構築するために3つのパラメータを必要とする場合、次のように記述します。

```fsharp
type CustomerName(firstName, middleInitial, lastName) =
    member this.FirstName = firstName
    member this.MiddleInitial = middleInitial
    member this.LastName = lastName
```

これをC#の場合と比較してみましょう。

```csharp
public class CustomerName
{
    public CustomerName(string firstName,
       string middleInitial, string lastName)
    {
        this.FirstName = firstName;
        this.MiddleInitial = middleInitial;
        this.LastName = lastName;
    }

    public string FirstName { get; private set; }
    public string MiddleInitial { get; private set; }
    public string LastName { get; private set; }
}
```

F#版では、プライマリ コンストラクタはクラス宣言そのものに組み込まれていることがわかります。つまり、独立したメソッドではありません。 つまり、クラス宣言はコンストラクタと同じパラメータを持ち、そのパラメータは自動的に不変のプライベートフィールドとなり、渡された元の値を保存します。

つまり、上記の例では、`CustomerName`クラスを次のように宣言したため、次のようになります。

```fsharp
type CustomerName(firstName, middleInitial, lastName)
```

したがって、`firstName`, `middleInitial`, `lastName` は自動的に不変のプライベートフィールドになります。

### コンストラクタでの型の指定

気づかなかったかもしれませんが、上記で定義した`CustomerName`クラスは、C#版とは異なり、パラメータを文字列に拘束していません。 一般的には、使用方法からの型推論により、値は文字列になると思われますが、明示的に型を指定する必要がある場合には、通常の方法でコロンに続いて型名を指定することができます。

以下は，コンストラクタで明示的に型を指定したバージョンのクラスです。

```fsharp
type CustomerName2(firstName:string,
                   middleInitial:string, lastName:string) =
    member this.FirstName = firstName
    member this.MiddleInitial = middleInitial
    member this.LastName = lastName
```

F#のちょっとした癖として、タプルをコンストラクタのパラメータとして渡す必要がある場合は、明示的にアノテーションを付けなければなりません。

```fsharp
type NonTupledConstructor(x:int,y: int) =
    do printfn "x=%i y=%i" x y

type TupledConstructor(tuple:int * int) =
    let x,y = tuple
    do printfn "x=%i y=%i" x y

// 呼び出しは同じに見える
let myNTC = new NonTupledConstructor(1,2)
let myTC = new TupledConstructor(1,2)
```

### クラスのメンバー

上の例のクラスは、3つの読み取り専用のインスタンスプロパティを持っています。F#ではプロパティもメソッドも `member` キーワードを使います。

また、上の例では、各メンバー名の前に「`this`」という言葉があります。 これは、クラスの現在のインスタンスを参照するために使用できる「自己識別子」です。 静的でないメンバーは、たとえそれが使われていなくても(上記のプロパティのように)、自己識別子を持たなければなりません。特定の単語を使用する必要はありませんが、一貫性があれば問題ありません。this "や "self"、"me"など、一般的に自己参照を示す言葉を使うことができます。

## クラスのシグネチャを理解する

クラスがコンパイルされると(またはエディタで定義をオーバーホバーすると)、そのクラスの「クラスシグネチャ」が表示されます。例えば、次のようなクラス定義があります。

```fsharp
type MyClass(intParam:int, strParam:string) =
    member this.Two = 2
    member this.Square x = x * x
```

これに対応するシグネチャは

```fsharp
type MyClass =
  class
    new : intParam:int * strParam:string -> MyClass
    member Square : x:int -> int
    member Two : int
  end
```

クラスのシグネチャには、そのクラスのすべてのコンストラクタ、メソッド、プロパティのシグネチャが含まれています。 これらのシグネチャの意味を理解することは価値があります。
なぜなら、関数と同じように、シグネチャを見ることでクラスが何をしているかを理解できるからです。また、抽象メソッドやインターフェースを作成する際にもこれらのシグネチャを記述する必要があるため、重要です。


### メソッドシグネチャ

メソッドのシグネチャは [独立した関数のシグネチャ](/posts/how-types-work-with-functions/) によく似ていますが、パラメータ名がシグネチャ自体の一部である点が異なります。

この場合、メソッドのシグネチャは次のようになります:

```fsharp
member Square : x:int -> int
```

また、比較のために、スタンドアロンの関数に対応するシグネチャは次のようになります。

```fsharp
val Square : int -> int
```

### コンストラクタのシグネチャ

コンストラクタのシグネチャは、常に `new` と呼ばれますが、それ以外はメソッドのシグネチャと同じです。

コンストラクタシグネチャは，常にタプル型の値を唯一のパラメータとして受け取ります。この場合のタプルの型は、予想通り `int * string` です。戻り値の型はクラスそのもので、これも予想通りです。

このコンストラクタのシグネチャを、同様のスタンドアロン関数と比較してみましょう。

```fsharp
// クラスのコンストラクタのシグネチャ
new : intParam:int * strParam:string -> MyClass

// スタンドアロン関数のシグネチャ
val new : int * string -> MyClass
```

### プロパティシグネチャ

最後に，`member Two : int`のようなプロパティシグネチャは，明示的な値が与えられないことを除けば，スタンドアロンの単純な値のシグネチャと非常によく似ています。

```fsharp
// メンバープロパティ
member Two : int

// 単体の値
val Two : int = 2
```

{{< book_page_pdf >}}

## "let" バインディングを使ったプライベートフィールドと関数

クラス宣言の後には、オプションで "let"バインディングのセットを持つことができます。これは通常、プライベートフィールドや関数を定義するのに使われます。

以下にそのサンプルコードを示します。

```fsharp
type PrivateValueExample(seed) =

    // 不変のプライベート値
    let privateValue = seed + 1

    // ミュータブルなプライベート値
    let mutable mutableValue = 42

    // プライベート関数の定義
    let privateAddToSeed input =
        seed + input

    // プライベート関数のパブリックラッパー
    member this.AddToSeed x =
        privateAddToSeed x

    // ミュータブルバリューのパプリック・ラッパー
    member this.SetMutableValue x =
        mutableValue <- x

// テスト
let instance = new PrivateValueExample(42)
printf "%i" (instance.AddToSeed 2)
instance.SetMutableValue 43
```

上の例では，3つのletバインディングがあります．

* `privateValue` には，初期シードに 1 を加えた値が設定されます．
* `mutableValue` は 42 に設定されます。
* `privateAddToSeed` 関数は，初期シードにパラメータを加えたものを使用しています。

これらはletバインディングなので，自動的にプライベートになるので，外部からアクセスするためには，ラッパーとして機能するパブリックなメンバーが必要になります．

なお、コンストラクタに渡される `seed` の値も let バインドの値と同様にプライベートフィールドとして利用できます。

### Mutable コンストラクタのパラメータ

コンストラクタに渡されるパラメータを変更可能にしたい場合があります。パラメータ自体には指定できないので、以下のように mutable な let-bound 値を作成して、パラメータから代入するのが標準的な手法です。

```fsharp
type MutableConstructorParameter(seed) = 以下のようになります。
    let mutable mutableSeed = seed

    // ミュータブル値のパブリックラッパー
    member this.SetSeed x =
        mutableSeed <- x
```

このようなケースでは、次のように、パラメータ自体と同じ名前をミュータブル値に与えることがよくあります。

```fsharp
type MutableConstructorParameter2(seed) = 以下のようになります。
    let mutable seed = seed // パラメータをシャドウ化する

    // ミュータブル値のパブリックラッパー
    member this.SetSeed x =
        seed <- x
```

## "do" ブロックによるコンストラクタの追加動作

先ほどの `CustomerName` の例では、コンストラクタはいくつかの値が渡されるのを許可するだけで、他には何もしませんでした。 しかし、場合によっては、コンストラクタの一部として何らかのコードを実行する必要があるかもしれません。これには `do` ブロックを使用します。

以下にその例を示します。

```fsharp
type DoExample(seed) =
    let privateValue = seed + 1

    //コンストラクション時に行われる追加コード
    do printfn "the privateValue is now %i" privateValue

// テスト
new DoExample(42)
```

do "コードは、次の例のように、その前に定義されたlet-bound関数も呼び出すことができます。

```fsharp
type DoPrivateFunctionExample(seed) =
    let privateValue = seed + 1

    // コンストラクション時に行われるいくつかのコード
    do printfn "hello world"

    // これを呼び出すdoブロックの前になければならない
    let printPrivateValue() =
        do printfn "the privateValue is now %i" privateValue

    // 構築時に行うべき追加コード
    do printPrivateValue()

// テスト
new DoPrivateFunctionExample(42)
```

### doブロック内での "this"によるインスタンスへのアクセス

"do"束縛と"let"束縛の違いの1つは、"do"束縛はインスタンスにアクセスできるが、"let"束縛はアクセスできないことです。これは、"let"束縛が実際にはコンストラクタ自体の前に評価される (C#のフィールド初期化子と同様) ので、ある意味でのインスタンスはまだ存在しないからです。

"do"ブロックからインスタンスのメンバーを呼び出す必要がある場合は、インスタンス自体を参照する何らかの方法が必要です。これも"自己識別子"を使用して行われますが、今回はクラス宣言自体に付加されます。

```fsharp
type DoPublicFunctionExample(seed) as this =
    // 宣言の中の "this"キーワードに注意してください。

    let privateValue = seed + 1

    // コンストラクション時に行われる追加コード
    do this.PrintPrivateValue()

    // メンバー
    member this.PrintPrivateValue() =
        do printfn "the privateValue is now %i" privateValue

// テスト
new DoPublicFunctionExample(42)
```

一般的に、コンストラクタからメンバーを呼び出すことは、必要がない限り(例えば、仮想メソッドを呼び出す場合など)、ベストプラクティスではありません。プライベートのlet-bound関数を呼び出し、必要に応じてパブリックのメンバーが同じプライベート関数を呼び出すようにするのが良いでしょう。

## メソッド

メソッドの定義は関数の定義とよく似ていますが、`let`キーワードだけではなく、`member`キーワードと自己識別子を持っている点が異なります。

以下にいくつかの例を示します。

```fsharp
type MethodExample() =

    // スタンドアロンのメソッド
    member this.AddOne x =
        x + 1

    // 別のメソッドを呼び出す
    member this.AddTwo x =
        this.AddOne x |> this.AddOne

    // パラメータのないメソッド
    member this.Pi() =
        3.14159

// テスト
let me = new MethodExample()
printfn "%i" <| me.AddOne 42
printfn "%i" <| me.AddTwo 42
printfn "%f" <| me.Pi()
```

通常の関数と同様に、メソッドはパラメータを持ったり、他のメソッドを呼び出したり、パラメータレス(正確には[unit parameter](/posts/how-types-work-with-functions/#parameterless-functions)を取る)にできることがわかります。

### タプル形式とカリー形式の比較

通常の関数とは異なり、複数のパラメータを持つメソッドは、2つの異なる方法で定義できます。

* カリー形式では、パラメータをスペースで区切り、部分適用をサポートします。(なぜ「カリー形式」なのか？[カリー化の説明](/posts/currying/)をご覧ください)。
* タプル形式では、すべてのパラメータがカンマで区切られて1つのタプルとして同時に渡されます。

カリー化されたアプローチはより関数的で、タプルのアプローチはよりオブジェクト指向的です。それぞれのアプローチに対応したメソッドを持つクラスの例を示します。

```fsharp
type TupleAndCurriedMethodExample() =

    // カリー形式
    member this.CurriedAdd x y =
        x + y

    // タプル形式
    member this.TupleAdd(x,y) =
        x + y

// テスト
let tc = new TupleAndCurriedMethodExample()
printfn "%i" <| tc.CurriedAdd 1 2
printfn "%i" <| tc.TupleAdd(1,2)

// 部分適用を使う
let addOne = tc.CurriedAdd 1
printfn "%i" <| addOne 99
```

では、どちらのアプローチを使うべきでしょうか？

タプル形式の利点は次のとおりです。

* 他の.NETコードとの互換性
* 名前付きパラメータおよびオプションのパラメータをサポート
* メソッドのオーバーロードをサポートしていること(同じ名前で関数のシグネチャだけが異なる複数のメソッド)。

一方、タプル形式の短所は以下の通りです。

* 部分適用に対応していません。
* 高次関数との相性が悪い。
* 型推論がうまくいかない。

タプル形式とcurried形式の比較については、[type extensions](/posts/type-extensions/#tuple-form)の投稿を参照してください。


### クラスメソッドと連携したlet-bound関数

よくあるパターンは、すべての処理を行うlet-bound関数を作成し、publicメソッドがこれらの内部関数を直接呼び出すというものです。これは、関数型のコードの方がメソッドよりも型推論がうまくいくという利点があります。

以下にその例を示します。

```fsharp
type LetBoundFunctions() =

    let listReduce reducer list =
        list |> List.reduce reducer

    let reduceWithSum sum elem =
        sum + elem

    let sum list =
        list |> listReduce reduceWithSum

    // 最後にパブリックラッパー
    member this.Sum  = sum

// テスト
let lbf = new LetBoundFunctions()
printfn "Sum is %i" <| lbf.Sum [1..10]
```

この方法の詳細については、[この議論](/posts/type-extensions/#attaching-existing-functions)を参照してください。

### 再帰的なメソッド

通常のlet-bound関数とは異なり、再帰的なメソッドには特別な`rec`キーワードは必要ありません。 ここでは、退屈なほどおなじみのフィボナッチ関数をメソッドにしてみました。

```fsharp
type MethodExample() =

    // "rec"キーワードのない再帰的メソッド
    member this.Fib x =
        match x with
        | 0 | 1 -> 1
        | _ -> this.Fib (x-1) + this.Fib (x-2)

// テスト
let me = new MethodExample()
printfn "%i" <| me.Fib 10
```

### メソッドの型アノテーション

通常、メソッドのパラメータや戻り値の型はコンパイラが推測してくれますが、指定する必要がある場合は、標準的な関数と同じように指定します。

```fsharp
type MethodExample() =
    // 明示的な型アノテーション
    member this.AddThree (x:int) :int =
        x + 3
```


## プロパティ

プロパティは3つのグループに分けられます。

* Immutable(不変)プロパティ："get"はあっても "set"はない。
* Mutable(変更可能)プロパティ："get"と(おそらくプライベートな)"set"の両方が存在する。
* 書き込み専用のプロパティ："set"はあるが "get"はない。 これらのプロパティは非常に珍しいので、ここでは説明しませんが、必要に応じてMSDNドキュメントにその構文が記載されています。

不変型プロパティと可変型プロパティの構文は若干異なります。

不変プロパティの場合、構文は単純です。標準的な "let"値結合に似た "get"メンバーがあります。結合の右辺の式は、標準的な式であれば何でもよく、通常はコンストラクタのパラメータ、プライベートの let-bound フィールド、およびプライベート関数の組み合わせです。

以下にその例を示します。

```fsharp
type PropertyExample(seed) =
    // 不変的なプロパティ
    // コンストラクタのパラメータを使用
    member this.Seed = seed
```

しかし、変更可能なプロパティの場合、構文はより複雑になります。1つは取得、もう1つは設定のための2つの関数を用意する必要があります。これは、次のような構文で行います。

```fsharp
with get() = ...
and set(value) = ...
```

以下に例を示します。

```fsharp
type PropertyExample(seed) =
    // プライベートなミュータブル値
    let mutable myProp = seed

    // ミュータブルプロパティ
    // プライベートなミュータブル値を変更する
    member this.MyProp
        with get() = myProp
        and set(value) =  myProp <- value
```

セット関数をプライベートにするには、代わりにキーワード `private set` を使います。

### 自動プロパティ

VS2012 から F# は自動プロパティをサポートしており、自動プロパティ用に別のバッキングストアを作成する必要がなくなりました。

不変の自動プロパティを作成するには、次のような構文を使います。

```fsharp
member val MyProp = initialValue
```

ミュータブルな自動プロパティを作成するには、次のような構文を使います。

```fsharp
member val MyProp = initialValue with get,set
```

この構文では、新しいキーワード `val` が追加され、自己識別子がなくなっていることに注意してください。

### プロパティの完全な例

ここでは、すべてのプロパティタイプを示す完全な例を紹介します。

```fsharp
type PropertyExample(seed) =
    // プライベートなミュータブル値
    let mutable myProp = seed

    // プライベート関数
    let square x = x * x

    // 不変のプロパティ
    // コンストラクタのパラメータを使用
    member this.Seed = seed

    // 不変的なプロパティ
    // プライベート関数を使用
    member this.SeedSquared = square seed

    // 可変型プロパティ
    // プライベートな可変値の変更
    member this.MyProp
        with get() = myProp
        そして set(value) = myProp <-value

    // プライベートなセットを持つ mutable プロパティ
    member this.MyProp2
        with get() = myProp
        and private set(value) =  myProp <- value

    // 自動不変プロパティ(VS2012の場合)
    member val ReadOnlyAuto = 1

    // 自動的に変更可能なプロパティ(VS2012の場合)
    member val ReadWriteAuto = 1 with get,set

// テスト
let pe = new PropertyExample(42)
printfn "%i" <| pe.Seed
printfn "%i" <| pe.SeedSquared
printfn "%i" <| pe.MyProp
printfn "%i" <| pe.MyProp2

// セットを呼び出してみる
pe.MyProp <- 43 // OK
printfn "%i" <| pe.MyProp

// プライベートセットの呼び出しを試す
pe.MyProp2 <- 43 // エラー
```

### プロパティとパラメータレスメソッドの比較

ここまでくると、プロパティとパラメータレスメソッドの違いに戸惑うかもしれません。一見すると同じように見えますが、微妙な違いがあります。「パラメータレス」メソッドは実際にはパラメータレスではなく、必ずユニットパラメータを持ちます。

ここでは、定義と使用方法の違いを示す例を示します。

```fsharp
type ParameterlessMethodExample() =
    member this.MyProp = 1 // 括弧がない！？
    member this.MyFunc() = 1 // ()に注意してください。

// 使用中
let x = new ParameterlessMethodExample()
printfn "%i" <| x.MyProp // 括弧がありません!
printfn "%i" <| x.MyFunc() // ()に注意してください。
```

また，クラス定義のシグネチャを見れば，その違いがわかります。

クラス定義は次のようになっています。

```fsharp
type ParameterlessMethodExample =
  class
    new : unit -> ParameterlessMethodExample
    member MyFunc : unit -> int
    member MyProp : int
  end
```

メソッドのシグネチャは `MyFunc : unit -> int` で、プロパティのシグネチャは `MyProp : int` です。

これは、関数とプロパティがどのクラスからも独立して宣言されている場合のシグネチャと非常によく似ています。

```fsharp
let MyFunc2() = 1
let MyProp2 = 1
```

これらのシグネチャは次のようになります。

```fsharp
val MyFunc2 : unit -> int
val MyProp2 : int = 1
```

これはほとんど同じです．

この違いや，なぜ関数にユニットパラメータが必要なのかが不明な場合は， [discussion of parameterless methods](/posts/how-types-work-with-function/#parameterless-functions)を読んでください．


## 追加コンストラクタ

クラスの宣言に組み込まれたプライマリ コンストラクタに加えて、クラスは追加のコンストラクタを持つことができます。 これらのコンストラクタは `new` キーワードで示され、最後の式としてプライマリ コンストラクタを呼び出さなければなりません。

```fsharp
type MultipleConstructors(param1, param2) =
    do printfn "Param1=%i Param12=%i" param1 param2

    // 追加コンストラクタ
    new(param1) =
        MultipleConstructors(param1,-1)

    // 追加コンストラクタ
    new() =
        printfn "構築中..."
        MultipleConstructors(13,17)

// テスト
let mc1 = new MultipleConstructors(1,2)
let mc2 = new MultipleConstructors(42)
let mc3 = new MultipleConstructors()
```

## スタティックメンバー

C#と同様に、クラスはスタティックなメンバーを持つことができ、これは`static`キーワードで示されます。`static`修飾子はmemberキーワードの前に付きます。

静的なメンバーは "this"のような自己識別子を持つことができません。なぜなら参照するインスタンスがないからです。

```fsharp
type StaticExample() =
    member this.InstanceValue = 1
    static member StaticValue = 2 // "this"がない。

// テスト
let instance = new StaticExample()
printf "%i" instance.InstanceValue
printf "%i" StaticExample.StaticValue
```

## スタティックコンストラクタ

F#にはスタティックコンストラクタに相当するものはありませんが、クラスが最初に使用されたときに実行されるスタティックなlet-bound値やスタティックなdo-blockを作成することができます。

```fsharp
type StaticConstructor() =

    // 静的フィールド
    static let rand = new System.Random()

    // 静的実行
    static do printfn "Class initialization!"

    // 静的フィールドにアクセスするインスタンスメンバー
    member this.GetRand() = rand.Next()
```

## メンバーのアクセシビリティ

メンバーのアクセシビリティは、.NET標準のキーワードである`public`、`private`、`internal`で制御することができます。 アクセシビリティの修飾子は、`member`キーワードの後、メンバー名の前にきます。

C#とは異なり，すべてのクラスメンバはデフォルトではprivateではなくpublicです。これはプロパティとメソッドの両方を含みます。 しかし、非メンバー(let宣言など)はprivateであり、publicにすることはできません。

以下にその例を示します。

```fsharp
type AccessibilityExample() =
    member this.PublicValue = 1
    member private this.PrivateValue = 2
    member internal this.InternalValue = 3
// テスト
let a = new AccessibilityExample();
printf "%i" a.PublicValue
printf "%i" a.PrivateValue // アクセスできない
```

プロパティの場合、setとgetが異なるアクセシビリティを持つ場合、各部分に別々のアクセシビリティ修飾子をタグ付けすることができます。

```fsharp
type AccessibilityExample2() =
    let mutable privateValue = 42
    member this.PrivateSetProperty
        with get() =
            privateValue
        and private set(value) =
            privateValue <- value

// テスト
let a2 = new AccessibilityExample2();
printf "%i" a2.PrivateSetProperty // 読んでもOK
a2.PrivateSetProperty <- 43 // 書いても大丈夫ではない
```

実際には、C#では一般的な「public get, private set」の組み合わせは、F#では一般的には必要ありません。なぜなら、不変のプロパティは前述のようにもっとエレガントに定義できるからです。

## ヒント: 他の.NETコードで使用されるクラスの定義

他の.NETコードとの相互運用が必要なクラスを定義する場合は、モジュール内で定義してはいけません。代わりに、モジュール外の名前空間で定義してください。

その理由は、F#モジュールはスタティッククラスとして公開されており、モジュール内で定義されたF#クラスはスタティッククラス内のネストされたクラスとして定義されるため、相互運用性が損なわれる可能性があるからです。 例えば、ユニットテストランナーの中には、スタティッククラスを好まないものがあります。

モジュールの外で定義されたF#クラスは、通常のトップレベルの.NETクラスとして生成され、これはおそらくあなたが望むものです。 しかし、(以前の記事](/posts/organizing-function/)で説明したように、名前空間を明確に宣言しないと、クラスは自動的に生成されたモジュールに配置され、知らないうちにネストされてしまうことを覚えておいてください。

以下は、2つのF#クラスの例です。1つはモジュールの外側で定義され、もう1つは内側で定義されています。

```fsharp
// 注意：このコードは.FSXスクリプトでは動作しません。
// .FSのソースファイルでのみ動作します。
namespace MyNamespace

type TopLevelClass() =
    let nothing = 0

module MyModule =

    type NestedClass() =
        let nothing = 0
```

また、同じコードをC#で書くと、次のようになります。

```csharp
namespace MyNamespace
{
  public class TopLevelClass
  {
  // code
  }

  public static class MyModule
  {
    public class NestedClass
    {
    // code
    }
  }
}
```

## クラスの構築と使用

さて、クラスを定義した後は、どのようにしてそれを利用すればよいのでしょうか。

クラスのインスタンスを生成する一つの方法は、C#のように単純で、`new`キーワードを使い、コンストラクタに引数を渡すだけです。

```fsharp
type MyClass(intParam:int, strParam:string) =
    member this.Two = 2
    member this.Square x = x * x

let myInstance = new MyClass(1,"hello")
```

しかし、F#では、コンストラクタは単なる関数の一つと考えられているので、通常は、次のように、`new`をなくして、コンストラクタ関数を単独で呼び出すことができます。

```fsharp
let myInstance2 = MyClass(1, "hello")
let point = System.Drawing.Point(1,2) // .NETクラスでも動作します。
```

`IDisposible`を実装したクラスを作成する場合、`new`を使わないとコンパイラの警告が出ます。

```fsharp
let sr1 = System.IO.StringReader("") // 警告
let sr2 = new System.IO.StringReader("") // OK
```

これは、使い捨てのために `let` キーワードの代わりに `use` キーワードを使うことを思い出すのに役立ちます。詳しくは[the post on `use`](/posts/let-use-do/#use)を参照してください。

### メソッドやプロパティの呼び出し

また、インスタンスを取得したら、標準的な方法でインスタンスに「ドットイン」して、任意のメソッドやプロパティを使用することができます。

```fsharp
myInstance.Two
myInstance.Square 2
```

以上、メンバーの使い方について多くの例を見てきましたが、これについてはあまり多くを語る必要はありません。

上で説明したように、タプル形式のメソッドとcurried形式のメソッドは、別々の方法で呼び出せることを覚えておいてください。

```fsharp
type TupleAndCurriedMethodExample() =
    member this.TupleAdd(x,y) = x + y
    member this.CurriedAdd x y = x + y

let tc = TupleAndCurriedMethodExample()
tc.TupleAdd(1,2) // 括弧付きで呼ばれる
tc.CurriedAdd 1 2 // 括弧なしで呼ばれる
2 |> tc.CurriedAdd 1 // 部分適用
```
