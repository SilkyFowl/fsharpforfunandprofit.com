---
layout: post
title: "Function associativity and composition"
description: "Building new functions from existing ones"
date: 2012-05-07
nav: thinking-functionally
seriesId: "Thinking functionally"
seriesOrder: 7
categories: [Functions]
image: "/posts/function-composition/Functions_Composition.png"
---

## 関数の結合性 ##

関数が連鎖している場合、どのように結合されるのでしょう?

例えば、これは何を意味しているのでしょうか?

```fsharp
let F x y z = x y z
```

関数yにパラメータzを代入し、その結果をxに適用するということですか?その場合は、次のようになります:

```fsharp
let F x y z = x (y z)
```

それとも、関数xにパラメータyを代入し、その結果得られた関数をパラメータzで評価するということでしょうか?その場合は、次のようになります:

```fsharp
let F x y z = (x y) z
```

答えは後者です。関数適用は*左結合*です。つまり、`x y z`を評価することは、` (x y) z`を評価することと同義です。また、`w x y z`を評価することは、` ( (w x) y) z`を評価することと同義です。これは驚くべきことではありません。これが部分適用の仕組みであることはすでに見ました。xを2パラメータ関数と考える場合、` (x y) z`は最初のパラメータを部分適用した後、z引数を中間関数に渡した結果です。

適切に結合するには、明示的に括弧を使用するか、パイプを使用します。次の3つの形式は同等です。

```fsharp
let F x y z = x (y z)
let F x y z = y z |> x    // 順方向パイプを使用
let F x y z = x <| y z    // 逆方向パイプを使用
```

練習として、これらの関数のシグネチャを実際に評価せずに考えてみてください！

## 関数の合成 ##

関数合成はこれまで何度も言及してきましたが、実際には何を意味するのでしょうか?最初はかなり威圧的に見えるかもしれませんが、実際はとても単純です。

型「T1」から型「T2」 への写像である関数「f」があり、型「T2」から型「T3」への写像である関数「g」があるとします。すると、「f」の出力を「g」の入力に接続し、型「T1」から型「T3」への写像である新しい関数を生成することができます。

![](./Functions_Composition.png)

以下はその例です。

```fsharp
let f (x:int) = float x * 3.0  // f is int->float
let g (x:float) = x > 4.0      // g is float->bool
```

「f」の出力を受け取り、それを「g」の入力として使用する新しい関数 「h」 を作成できます。

```fsharp
let h (x:int) =
    let y = f(x)
    g(y)                   // return output of g
```

もっとコンパクトな方法はこれです。

```fsharp
let h (x:int) = g ( f(x) ) // h is int->bool

//test
h 1
h 2
```

これでは簡単です。面白いのは、「compose」という新しい関数を定義できることです。「compose」は、関数 「f」と「g」 を引数にとり、そのシグネチャを知らないまま、これらをこのように結合します。

```fsharp
let compose f g x = g ( f(x) )
```

これを評価すると、コンパイラは 「`f`」 がジェネリック型「`'a`」からジェネリック型「`'b`」への関数である場合、 「`g`」 は入力としてジェネリック型「`'b`」を持つように制約します。全体的なシグネチャは次のとおりです:

```fsharp
val compose : ('a -> 'b) -> ('b -> 'c) -> 'a -> 'c
```

(この汎用的な合成処理は、すべての関数の入出力が1つであるため可能となるのです。この方法は、関数型でない言語ではできないでしょう)

これまで見てきたように、実際のcomposeは 「`>>`」 記号で定義されています。

```fsharp
let (>>) f g x = g ( f(x) )
```

この定義により、既存の関数から新しい関数を構築するために合成を使用することができます。

```fsharp
let add1 x = x + 1
let times2 x = x * 2
let add1Times2 x = (>>) add1 times2 x

//test
add1Times2 3
```

この明示的なスタイルはかなり雑然としています。使いやすく、理解しやすくするための手順は以下のとおりです。

まず、xパラメータを省略して、合成演算子が部分適用を返すようにします。

```fsharp
let add1Times2 = (>>) add1 times2
```

これで2項演算となり、演算子を中間に置くことができます。

```fsharp
let add1Times2 = add1 >> times2
```

これで出来上がりです。合成演算子を使うことで、コードをより簡潔で分かりやすくできます。

```fsharp
let add1 x = x + 1
let times2 x = x * 2

//old style
let add1Times2 x = times2(add1 x)

//new style
let add1Times2 = add1 >> times2
```

## 実際に合成演算子を使用する ##

合成演算子 は (他の中置演算子と同様に) 通常の関数適用よりも優先順位が低いです。つまり、合成で使用される関数は、括弧を使わずに引数を持つことができます。

たとえば、「add」 関数と 「times」 関数に追加のパラメータがある場合、これを合成時に引数として渡すことができます。

```fsharp
let add n x = x + n
let times n x = x * n
let add1Times2 = add 1 >> times 2
let add5Times3 = add 5 >> times 3

//test
add5Times3 1
```

入力と出力が一致しているならば、その関数にはどんな値でも入力できます。たとえば、関数を2回実行する次の例を考えてみます:

```fsharp
let twice f = f >> f    //signature is ('a -> 'a) -> ('a -> 'a)
```

コンパイラは、関数fが入出力の両方で同一型を使用すると推定していることに注目してください。

ここで、「`+`」のような関数を考えてみましょう。前述したように、入力は 「`int`」 ですが、出力は実際には部分的に適用された関数 「` (int->int) `」 です。したがって、「`+`」の出力は、「`twice`」 の入力として使用できます。次のように記述します:

```fsharp
let add1 = (+) 1           // signature is (int -> int)
let add1Twice = twice add1 // signature is also (int -> int)

//test
add1Twice 9
```

一方で，次のような書き方はできません。

```fsharp
let addThenMultiply = (+) >> (*)
```

なぜなら、「*」への入力は、`int`値でなければなりません、`int->int`関数ではありません (これは加算の出力となります) 。

しかし、最初の関数の出力が `int`だけになるように調整すれば、動作します。

```fsharp
let add1ThenMultiply = (+) 1 >> (*)
// (+) 1 はシグネチャ (int->int) を持ち、出力は'int'

//test
add1ThenMultiply 2 7
```

必要に応じて、「`<<`」演算子を使用して逆方向に合成することもできます。

```fsharp
let times2Add1 = add 1 << times 2
times2Add1 3
```

逆合成は、主にコードをより英語らしくするために使用されます。次に簡単な例を示します:

```fsharp
let myList = []
myList |> List.isEmpty |> not    // straight pipeline

myList |> (not << List.isEmpty)  // using reverse composition
```

## 合成 vs. パイプライン ##

ここでは、合成演算子とパイプライン演算子が非常によく似ているように見えるため、どのような違いがあるのか疑問に思うかもしれません。

まず、パイプライン演算子の定義をもう一度見てみましょう。

```fsharp
let (|>) x f = f x
```

これにより、関数の引数を関数の後ではなく前に配置できるようになります。それだけです。関数にパラメータが複数ある場合は、パイプ入力された値が最後のパラメータになります。前に使用した例を次に示します:

```fsharp
let doSomething x y z = x+y+z
doSomething 1 2 3       // all parameters after function
3 |> doSomething 1 2    // last parameter piped in
```

合成は別物で、パイプの代替にはなりません。次の例では、数値3は関数ではないため、その「出力」を`doSomething`に渡すことはできません。

```fsharp
3 >> doSomething 1 2     // not allowed
// f >> g は g(f(x))と同じなので、次のように書き換えます:
doSomething 1 2 ( 3(x) ) // つまり、3は関数でなければなりません!
// error FS0001: This expression was expected to have type 'a->'b
//               but here has type int
```

コンパイラは、「3」が何らかの関数`'a->'b`であるべきだと不満を言っています。

これを、3つの引数を取る合成関数の定義と比較してください。最初の2つは関数である必要があります。

```fsharp
let (>>) f g x = g ( f(x) )

let add n x = x + n
let times n x = x * n
let add1Times2 = add 1 >> times 2
```

パイプを代用することもできません。次の例では、「`add1`」は`int->int`型の (部分的な) 関数であり、「`times 2`」の2番目のパラメータとして使用することはできません。

```fsharp
let add1Times2 = add 1 |> times 2   // not allowed
// x |> fは f(x) と同じなので、次のように書き換えます:
let add1Times2 = times 2 (add 1)    // add1はintである必要があります
// error FS0001: Type mismatch. 'int -> int' does not match 'int'
```

コンパイラは、「`times2`」が`int->int`パラメータ、つまり`(int->int) ->'a`型を取るべきだと文句を言ってきます。
