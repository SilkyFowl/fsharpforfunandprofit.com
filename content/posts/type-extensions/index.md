---
layout: post
title: "Attaching functions to types"
description: "Creating methods the F# way"
date: 2012-05-11
nav: thinking-functionally
seriesId: "Thinking functionally"
seriesOrder: 11
---

これまでは純粋な関数型スタイルに焦点を当ててきましたが、オブジェクト指向スタイルに切り替えると便利な場合もあります。
このオブジェクト指向方式の重要な特徴の1つに、クラスに関数を添付し、そのクラスに 「ドット・イン」 して目的の動作を取得するという機能があります。

F#では、これは「型拡張」と呼ばれる機能を使って実現します。さらに、クラス以外のあらゆるF#型についても関数を添付することができます。

これはレコード型に関数を添付する例です:

```fsharp
module Person =
    type T = {First:string; Last:string} with
        // 型宣言で定義されたメンバー
        member this.FullName =
            this.First + " " + this.Last

    // コンストラクタ
    let create first last =
        {First=first; Last=last}

// テスト
let person = Person.create "John" "Doe"
let fullname = person.FullName
```

重要な点は次のとおりです。

* `with`キーワードは、メンバーのリストの先頭を示します。
* `member`キーワードは、これがメンバー関数(つまりメソッド) であることを示します
* `this`という単語は、ドットで区切られるオブジェクトのプレースホルダです (「自己識別子」と呼ばれます) 。プレースホルダは関数名の前に付けられ、関数本体は現在のインスタンスを参照する必要がある場合には同様のプレースホルダを用います。
一貫性さえあれば、どの単語を使っても構いません。`this`、`self`、`me`、あるいはその他の一般的に自己参照を示す単語を使用できます。

型宣言と同時にメンバーを追加する必要はなく、同じモジュール内で後から追加することもできます。

```fsharp
module Person =
    type T = {First:string; Last:string} with
       // 型宣言と同時に定義されたメンバー
        member this.FullName =
            this.First + " " + this.Last

    // コンストラクタ
    let create first last =
        {First=first; Last=last}

    // 後から追加される別のメンバー
    type T with
        member this.SortableName =
            this.Last + ", " + this.First
// test
let person = Person.create "John" "Doe"
let fullname = person.FullName
let sortableName = person.SortableName
```


これらの例は、「組み込みの型拡張」 と呼ばれるものです。型自体にコンパイルされ、型が使用される際は常に使用可能です。リフレクションを使用した場合にも表示されます。

組み込みの型拡張では、すべてのコンポーネントが同じ名前空間を使用し、同じアセンブリ内にコンパイルされている限り、複数ファイルに分割された型定義を作成することも可能です。
C#の部分クラスと同様に、生成コードをオーサリングされたコードから分離するのに便利です。

## 省略可能な型拡張

別の方法として、全く別のモジュールから別のメンバを追加することもできます。
これらは「省略可能な型拡張」と呼ばれます。その型自体にコンパイルされるのではなく、その型が動作できるスコープ内に別のモジュールが必要です (この動作はC#拡張メソッドと同じです) 。

たとえば、`Person`タイプが定義されているとします:

```fsharp
module Person =
    type T = {First:string; Last:string} with
       // 型宣言で定義されたメンバー
        member this.FullName =
            this.First + " " + this.Last

    // コンストラクタ
    let create first last =
        {First=first; Last=last}

    // 後から追加される別のメンバー
    type T with
        member this.SortableName =
            this.Last + ", " + this.First
```

次の例では、`UppercaseName`という型拡張を別のモジュールから追加する方法を示します:

```fsharp
// 別のモジュールで
module PersonExtensions =

    type Person.T with
    member this.UppercaseName =
        this.FullName.ToUpper()
```

それでは、この拡張機能をテストしてみましょう。

```fsharp
let person = Person.create "John" "Doe"
let uppercaseName = person.UppercaseName
```

おっと、エラーが出ました。何が問題かというと、`PersonExtensions`がスコープに入っていないことです。
C#と同じように、拡張機能を使用するためには、スコープに入れなければなりません。

それができれば、すべてがうまくいきます。

```fsharp
// まず拡張機能をスコープに入れよう
open PersonExtensions

let person = Person.create "John" "Doe"
let uppercaseName = person.UppercaseName
```


## システム型の拡張

.NETライブラリに含まれる型も拡張することができます。ただし、拡張する際には型の省略形ではなく、実際の型名を使用しなければならないことに注意してください。

例えば、`int`を拡張しようとすると、`int`は型の本当の名前ではないので、失敗します。

```fsharp
type int with
    member this.IsEven = this % 2 = 0
```

代わりに，`System.Int32`を使う必要があります．

```fsharp
type System.Int32 with
    member this.IsEven = this % 2 = 0

//test
let i = 20
if i.IsEven then printfn "'%i' is even" i
```

## 静的なメンバ

メンバ関数を静的にするには、次のようにします:

* キーワード`static`を追加
* `this`プレースホルダーを削除する

```fsharp
module Person =
    type T = {First:string; Last:string} with
        // 型宣言で定義されたメンバー
        member this.FullName =
            this.First + " " + this.Last

        // 静的コンストラクタ
        static member Create first last =
            {First=first; Last=last}

// test
let person = Person.T.Create "John" "Doe"
let fullname = person.FullName
```

また、システム型に対しても、静的なメンバーを作成することができます。

```fsharp
type System.Int32 with
    static member IsOdd x = x % 2 = 1

type System.Double with
    static member Pi = 3.141

//test
let result = System.Int32.IsOdd 20
let pi = System.Double.Pi
```

{{< linktarget "attaching-existing-functions" >}}


## 既存の関数を添付する

よくあるパターンは、既存の独立した関数を型に添付することです。これにはいくつかの利点があります。

* 開発中に、ほかの単体関数を参照する単体関数を作成できます。型推論は、関数型コードの方が、オブジェクト型コード (「ドット・イン」) よりも遥かに優れているため、プログラミングが容易になります。
* ただし一部の重要な機能については、型に添付することもできます。これにより、ユーザは関数スタイルとオブジェクト指向スタイルのどちらを使用するかを選択できます。

F#ライブラリの例として、リストの長さを計算する関数があります。これは、`List`モジュールの独立した関数として使用できますが、Listインスタンスのメソッドとしても使用できます。

```fsharp
let list = [1..10]

// functional style
let len1 = List.length list

// OO style
let len2 = list.Length
```

次の例では、最初にメンバーのない型を定義し、次にいくつかの関数を定義し、最後に型に`fullName`関数を添付しています。

```fsharp
module Person =
    // 最初はメンバーのいない型
    type T = {First:string; Last:string}

    // コンストラクタ
    let create first last =
        {First=first; Last=last}

    // スタンドアロン関数
    let fullName {First=first; Last=last} =
        first + " " + last

    // 既存の関数をメンバーとして添付する
    type T with
        member this.FullName = fullName this

// test
let person = Person.create "John" "Doe"
let fullname = Person.fullName person  // functional style
let fullname2 = person.FullName        // OO style
```

独立した`fullName`関数には、personというパラメータが1つあります。添付されたメンバーでは、このパラメータは `this`自己参照から取得されます。

### 複数パラメータを持つ既存の関数を添付する

以前に定義した関数に複数のパラメータがある場合、`this`パラメータが最初であれば、添付を行う際にすべてのパラメータを再指定する必要はありません。

次の例では、`hasSameFirstAndLastName`関数は3つのパラメータを持ちます。しかし、私たちがそれを添付する時、私たちが指定する必要があるのは一つだけです！

```fsharp
module Person =
    // 最初はメンバーを持たない型
    type T = {First:string; Last:string}.

    // コンストラクタ
    let create first last =
        {First=first; Last=last}

    // スタンドアロン関数
    let hasSameFirstAndLastName (person:T) otherFirst otherLast =
        person.First = otherFirst && person.Last = otherLast

    // 既存の関数をメンバーとしてアタッチ
    type T with
        member this.HasSameFirstAndLastName = hasSameFirstAndLastName this

// test
let person = Person.create "John" "Doe"
let result1 = Person.hasSameFirstAndLastName person "bob" "smith" // functional style
let result2 = person.HasSameFirstAndLastName "bob" "smith" // OO style
```


なぜこのようなことができるのでしょうか？ヒント: カーリングと部分適用について考えてみましょう！


{{< linktarget "tuple-form" >}}


## タプル形式のメソッド

複数のパラメータを持つメソッドを持つようになったら、次のような判断をしなければなりません。

* 標準の (カリー化された) 形式を使用した場合、パラメータはスペースで区切られ、部分適用がサポートされます。
* パラメータを*全て*一度にカンマ区切り、単一のタプルとして渡すことができます。

「カリー化」形式はより関数的で、「タプル」形式はよりオブジェクト指向的です。

タプル形式は、F#が標準の.NETライブラリと連携する方法でもあるので、このアプローチをもっと詳しく調べてみましょう。

実験台として、ここに2つのメソッドを持つProduct型を用意し、それぞれのメソッドは一方のアプローチを用いて実装します。
`CurriedTotal`メソッドと`TupleTotal`メソッドはそれぞれ同じことを行います: 与えられた数量と割引の合計価格の計算です。

```fsharp
type Product = {SKU:string; Price: float} with

    // カリー化
    member this.CurriedTotal qty discount =
        (this.Price * float qty) - discount

    // タプル
    member this.TupleTotal(qty,discount) =
        (this.Price * float qty) - discount
```

そして、テストコードです:

```fsharp
let product = {SKU="ABC"; Price=2.0}
let total1 = product.CurriedTotal 10 1.0
let total2 = product.TupleTotal(10,1.0)
```

今のところ違いはありません。

カリー化版は部分適用できることがわかっています。

```fsharp
let totalFor10 = product.CurriedTotal 10
let discount = [1.0..5.0].
let totalForDifferentDiscounts
    = discounts |> List.Map totalFor10
```

しかし、タプルを使ったアプローチには、カリー化によるアプローチでは不可能なことがいくつかあります。

* 名前付きパラメータ
* 省略可能なパラメータ
* オーバーロード

### タプルスタイル・パラメータの名前付きパラメータ

タプル形式のアプローチでは、名前付きパラメータが使用できます。

```fsharp
let product = {SKU="ABC"; Price=2.0}
let total3 = product.TupleTotal(qty=10,discount=1.0)
let total4 = product.TupleTotal(discount=1.0, qty=10)
```

このように、名前を使用すると、パラメータの順序を変更できます。

注意: 名前付きパラメータと無名パラメータが混在している場合、名前付きパラメータは常に最後に指定する必要があります。

###タプルスタイル・パラメータの省略可能なパラメータ

タプル形式のメソッドでは、パラメータ名の前に疑問符を付けることで、省略可能なパラメータを指定できます。

* パラメータが設定されている場合は、`Some value`
* パラメータが設定されていない場合は、`None`となります。

次に例を示します:

```fsharp
type Product = {SKU:string; Price: float} with

    // 任意の割引
    member this.TupleTotal2(qty,?discount) =
        let extPrice = this.Price * float qty
        match discount with
        | None -> extPrice
        | Some discount -> extPrice - discount
```

テストをしてみましょう:

```fsharp
let product = {SKU="ABC"; Price=2.0}

// 割引が指定されていない
let total1 = product.TupleTotal2(10)

// 割引の指定
let total2 = product.TupleTotal2(10,1.0)
```

この `None`と`Some`の明示的なマッチングは面倒な作業ですが、省略可能なパラメータを処理するのに、もう少し洗練された方法があります。

関数`defaultArg`は、パラメータを第1引数とし、第2引数はデフォルト値として使用します。パラメータが設定されている場合は、その値が返されます。
そうでない場合は、デフォルト値が返されます。

同じコードを`defaultArg`を使用して書き直してみましょう。

```fsharp
type Product = {SKU:string; Price: float} with

    // 任意の割引
    メンバー this.TupleTotal2(qty,?discount) =
        let extPrice = this.Price * float qty
        let discount = defaultArg discount 0.0
        //戻り値
        extPrice - discount
```

{{< linktarget "method-overloading" >}}。


### メソッドのオーバーロード

C#では、同じ名前を持つ複数のメソッドのうち、関数シグネチャのみが異なるもの(例えば、異なる型のパラメータやパラメータ数など) を持つことができます。

純粋な関数型では、それは意味をなしません ―― 関数は特定の定義域型と特定の値域型で動作します。
同じ関数を異なる定義域および地域に対して使用することはできません。

しかし、F#はメソッドのオーバーロードを*サポートしています*、ただしメソッド (つまり型に付随する関数) とその中でタプル型のパラメータ渡しを利用しているメソッドだけが対象です。

以下に例を示しますが、`TupleTotal`メソッドにはさらに別のバリエーションがあります！

```fsharp
type Product = {SKU:string; Price: float} with

    // 割引なし
    member this.TupleTotal3(qty) =
        printfn "using non-discount method"
        this.Price * float qty

    // 割引あり
    member this.TupleTotal3(qty, discount) =
        printfn "using discount method"
        (this.Price * float qty) - discount
```

通常、F#コンパイラは同じ名前を持つ2つのメソッドがあると文句を言いますが、今回の場合、それらはタプルベースであり、シグネチャも異なるため、問題ありません。
(どちらが呼び出されているかを明確にするために、小さなデバッグメッセージを追加しました。)

これがテストです:


```fsharp
let product = {SKU="ABC"; Price=2.0}

// 割引が指定されていない
let total1 = product.TupleTotal3(10)

// 割引の指定
let total2 = product.TupleTotal3(10,1.0)
```


{{< linktarget "downsides-of-methods" >}}。

## ちょっと待った！...メソッドを使うことのデメリット

オブジェクト指向の経験がある人は、どこにでもメソッドを使いたくなるかもしれません。
しかし、メソッドを使用することには大きな欠点もあることを知っておいてください。

* メソッドは型推論との相性が良くない
* メソッドは高階関数では適切に動作しない

実際、メソッドを多用すると、F#のプログラミングで最も強力で有用な側面を不必要に無視することになります。

どういうことか考えてみましょう。

### メソッドは型推論がうまく動作しない

Personの例に戻りましょう。この例では、同じロジックが独立した関数とメソッドの両方で実装されています。

```fsharp
module Person =
    // 最初はメンバーのいない型
    type T = {First:string; Last:string}

    // コンストラクタ
    let create first last =
        {First=first; Last=last}

    // スタンドアロン関数
    let fullName {First=first; Last=last} =
        first + " " + last

    // 関数をメンバーとして添付
    type T with
        member this.FullName = fullName this
```

では、それぞれがどのように型推論されるかを見てみましょう。例えば、フルネームを出力したいので、personをパラメータとする`printFullName`関数を定義します。

モジュールレベルの独立関数を使用するコードを次に示します。

```fsharp
人物を開く

// スタンドアロン関数を使う
let printFullName person =
    printfn "Name is %s" (fullName person)

// 型推論は動作しました:
// val printFullName : Person.T -> unit
```

これは問題なくコンパイルでき、型推論はパラメータがpersonであることを正しく推定しました。

それでは、「ドット」 バージョンを試してみましょう。

```fsharp
open Person

//「ドット・イン」 メソッドを使用
let printFullName2 person =
    printfn "Name is %s" (person.FullName)
```

型推論はパラメータを推論するのに十分な情報を持たないため、これは全くコンパイルされることがありません。*任意の*オブジェクトの`.FullName` ―― これ以上先に行くには不十分です。

もちろん、パラメーター型を使用して関数に注釈を付けることもできますが、これでは型推論の目的そのものが損なわれてしまいます。

### メソッドは高階関数との相性が悪い

同様の問題は高階関数でも起こります。たとえば、とある人物リストから、全員のフルネームを取得したいとします。

独立関数では、これは簡単です:

```fsharp
open Person

let list = [
    Person.create "Andy" "Anderson";
    Person.create "John" "Johnson";
    Person.create "Jack" "Jackson"]

// すべてのフルネームを一度に取得
list |> List.map fullName
```

オブジェクト・メソッドでは、いたるところに特殊なラムダを作成しなければなりません。

```fsharp
open Person

let list = [
    Person.create "Andy" "Anderson";
    Person.create "John" "Johnson";
    Person.create "Jack" "Jackson"]

// すべてのフルネームを一度に取得
list |> List.map (fun p -> p.FullName)
```

これは単純な例に過ぎません、オブジェクトメソッドは適切に合成できず、パイプ処理が困難になります。

関数プログラミングを始めたばかりの人にお願いします。可能な場合は、メソッドを避けてください。特に学んでいるときは一切使わないでください。
これは、関数型プログラミングの恩恵を完全に享受することを妨げる杖です。いでください。
メソッドは、関数型プログラミングから最大限の利益を得ることを妨げる松葉づえのようなものです。