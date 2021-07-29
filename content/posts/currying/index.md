---
layout: post
title: "Currying"
description: "Breaking multi-parameter functions into smaller one-parameter functions"
date: 2012-05-05
nav: thinking-functionally
seriesId: "Thinking functionally"
seriesOrder: 5
categories: [Currying]
---

基本型について少し説明した後で、改めて関数について説明します。ここで前述のパズルを思い出してください。数学関数が1つのパラメータしか持てないのであれば、F#関数が複数のパラメータを持つことできるのは何故でしょう?

その答えは非常に単純です。複数のパラメータを持つ関数は、それぞれ1つのパラメータだけを持つ新しい関数の連続に書き換えられます。これはコンパイラによって自動的に行われます。関数型プログラミングの開発に大きな影響を与えた数学者Haskell Curry氏にちなんで「**カリー化**」 と呼ばれています。

これが実際にどのように機能するか見るために、2つの数値を出力する非常に基本的な例を見てみましょう:

```fsharp
//normal version
let printTwoParameters x y =
   printfn "x=%i y=%i" x y
```

内部的には、コンパイラーはこれを次のようなものに書き換えます。

```fsharp
//explicitly curried version
let printTwoParameters x  =    // only one parameter!
   let subFunction y =
      printfn "x=%i y=%i" x y  // new function with one param
   subFunction                 // return the subfunction
```

詳しく見てみましょう。

1. 「`printTwoParameters`」という名前の関数を*1つの*パラメータ 「x」 のみで構築します。
2. その中で、*1つの*パラメータ 「y」 のみを持つ副関数を定義します。この内部関数は 「x」 パラメータを使用しますが、xはパラメータとして明示的に渡されないことに注意してください。「x」パラメータはスコープ内にあるため、内部関数はこれを参照し、渡されなくても使用できます。
3. 最後に、新しく作成した副関数を返します。
4. この戻された関数は、後で「y」に対して使用されます。「x」パラメータが組み込まれているため、返される関数に必要なのは、関数ロジックを完了させるための 「y」 パラメータだけです。

このように書き換えることで、コンパイラはすべての関数が必要に応じてパラメータを1つだけ持つことを保証します。したがって、「`printTwoParameters`」を使用すると、2つのパラメータ関数を使用しているように思われるかもしれませんが、実際には1つのパラメータ関数です。2つではなく1つの引数を渡すだけなのは、自分で確認できます:

```fsharp
// eval with one argument
printTwoParameters 1

// get back a function!
val it : (int -> unit) = <fun:printTwoParameters@286-3>
```

1つの引数で評価すると、エラーは発生せずに関数が返されます。

したがって、2つの引数を指定して`printTwoParameters`を呼び出すと、実際には次のようになります。

* 最初の引数 (x) を指定して`printTwoParameters`を呼び出します。
* `printTwoParameters`は、「x」が組み込まれた新しい関数を返します。
* 次に、2番目の引数 (y) で新しい関数を呼び出します。

ここでは、ステップ・バイ・ステップ・バージョンと通常バージョンの例を示します。

```fsharp
// step by step version
let x = 6
let y = 99
let intermediateFn = printTwoParameters x  // return fn with
                                           // x "baked in"
let result  = intermediateFn y

// inline version of above
let result  = (printTwoParameters x) y

// normal version
let result  = printTwoParameters x y
```

別の例を示します。

```fsharp
//normal version
let addTwoParameters x y =
   x + y

//explicitly curried version
let addTwoParameters x  =      // only one parameter!
   let subFunction y =
      x + y                    // new function with one param
   subFunction                 // return the subfunction

// now use it step by step
let x = 6
let y = 99
let intermediateFn = addTwoParameters x  // return fn with
                                         // x "baked in"
let result  = intermediateFn y

// normal version
let result  = addTwoParameters x y
```

ここでも、「2パラメータ関数」は実際には、中間関数を返す1パラメータ関数です。

でもちょっと待ってください「`+`」操作自体はどうでしょう?これは2つのパラメータを必要とするバイナリ操作ですよね?いいえ、他のすべての機能と同じようにカリー化されています。上記の`addTwoParameters`のように、1つのパラメータを取り、新しい中間関数を返す「`+`」という関数があります。

ステートメント`x+y`を記述すると、コンパイラはコードを並べ替えて接頭辞を削除し、2つのパラメータで呼び出される`+`という名前の関数である` (+) x y`に変換します。「+」という名前の関数は、中接演算子としてではなく通常の関数名として使用されることを示すために、括弧で囲む必要があることに注意してください。

最終的に、`+`という名前の2つのパラメータ関数は、他のパラメータ関数と同様に扱われます:

```fsharp
// using plus as a single value function
let x = 6
let y = 99
let intermediateFn = (+) x     // return add with x baked in
let result  = intermediateFn y

// using plus as a function with two parameters
let result  = (+) x y

// normal version of plus as infix operator
let result  = x + y
```

これは，他のすべての演算子やprintfのような組み込み関数にも使えます．

```fsharp
// normal version of multiply
let result  = 3 * 5

// multiply as a one parameter function
let intermediateFn = (*) 3   // return multiply with "3" baked in
let result  = intermediateFn 5

// normal version of printfn
let result  = printfn "x=%i y=%i" 3 5

// printfn as a one parameter function
let intermediateFn = printfn "x=%i y=%i" 3  // "3" is baked in
let result  = intermediateFn 5
```

## カリー化された関数の特徴 ##)

カリー化された関数がどのように機能するかを理解しましたが、そのシグネチャはどのようなものなのでしょうか?

最初の例「`printTwoParameters`」に戻ると、1つの引数を取り、中間関数を返すことがわかりました。中間関数も引数を1つ取り、何も返しません(つまり、unit) 。したがって、中間関数の型は`int->unit`になります。つまり、`printTwoParameters`の定義域は`int`で、値域は`int->unit`です。これをまとめると、最終的なシグネチャは次のようになります。

```fsharp
val printTwoParameters : int -> (int -> unit)
```

明示的にカリー化された処理を評価すると、前述のようにシグネチャ内に括弧が表示されますが、暗黙的にカリー化された通常の処理を評価すると、次のように括弧が省略されます。

```fsharp
val printTwoParameters : int -> int -> unit
```

括弧は省略できます。関数シグネチャの意味を把握したいなら、頭の中で関数シグネチャを追加しなおすと良いでしょう。

ここで疑問に思うかもしれません、中間関数を返す関数と通常の2パラメータ関数の違いは何でしょうか?

中間関数を返す1パラメータ関数を次に示します。

```fsharp
let add1Param x = (+) x
// signature is = int -> (int -> int)
```

単純な値を返す2パラメータ関数を次に示します。

```fsharp
let add2Params x y = (+) x y
// signature is = int -> int -> int
```

シグネチャは少し異なりますが、実際には*違いは*ありません*、2番目の関数が自動的にカリー化されるだけです。

## 2つ以上のパラメータを持つ関数 ##

2パラメータ以上の関数では、どのような処理が行われるのでしょう?これまでと全く同じ方法で、最後のパラメータ以外の各パラメータに対して、関数は前のパラメータを組み込んだ中間関数を返します。

この意図的な例を考えましょう。パラメータ型を明示的に指定しましたが、関数自体は何もしません。

```fsharp
let multiParamFn (p1:int)(p2:bool)(p3:string)(p4:float)=
   ()   //何もしない

let intermediateFn1 = multiParamFn 42
   // intermediateFn1はboolを引数として
   // 新しい関数 (string -> float -> unit) を返します
let intermediateFn2 = intermediateFn1 false
   // intermediateFn2はstringを引数として、
   // 新しい関数 (float -> unit) を返します
let intermediateFn3 = intermediateFn2 "hello"
   // intermediateFn3はfloatを引数として、
   // 単純な値 (unit) を返します
let finalResult = intermediateFn3 3.141
```

関数全体のシグネチャは次のとおりです。

```fsharp
val multiParamFn : int -> bool -> string -> float -> unit
```

中間関数のシグネチャは次のとおりです。

```fsharp
val intermediateFn1 : (bool -> string -> float -> unit)
val intermediateFn2 : (string -> float -> unit)
val intermediateFn3 : (float -> unit)
val finalResult : unit = ()
```

関数シグネチャによって、関数のパラメータ数を知ることができます。括弧の外側にある矢印の数を数えるだけです。関数が他の関数パラメータを引数に取るか返す場合、括弧内に他の矢印が表示されますが、これらは無視できます。次に例を示します。

```fsharp
int->int->int      // 2つのintを引数として、intを返します

string->bool->int  // 最初の引数はstring、2番目の引数はbool、
                   // 返り値はint

int->string->bool->unit // 3つのパラメータ (int、string、bool)
                        // 返り値はなし (unit)

(int->string)->int      // パラメータは1つだけ、
                        // (intを引数にとりstringを返す)関数値
                        // 返り値はint1つ

(int->string)->(int->bool) // 一つの関数値(引数int、返り値string)を引数として
                           // (引数int、返り値bool)の関数値を返します
```


## 複数のパラメータの問題 ##

カリー化の背後にあるロジックを理解するまでは、予期しない結果が生じることがあります。関数に渡される引数が予想より少ない場合でも、エラーは発生ないことに注意してください。代わりに、部分適用された関数が返されます。値を期待するコンテキストでこの部分的適用された関数を使用すると、コンパイラから不明瞭なエラーメッセージが出力されます。

以下に、一見無害な関数を示します。

```fsharp
// 関数を作成します
let printHello() = printfn "hello"
```

以下のように呼ぶと何が起こると思いますか?コンソールに「hello」 と表示されますか?実行する前に予想してみてください。これがヒントです: 必ず関数シグネチャを確認してください。

```fsharp
// 呼び出します
printHello
```

期待通りは呼ばれ*ません*。元の関数は引数としてunitを要求していますが指定されていません、そのため部分適用された関数が返されます (上記の場合は引数なし) 。

これはどうでしょうか?コンパイルされますか?

```fsharp
let addXY x y =
    printfn "x=%i y=%i" x
    x + y
```

評価してみると、printfn行についてコンパイラが文句を言っているのがわかります。

```fsharp
printfn "x=%i y=%i" x
//^^^^^^^^^^^^^^^^^^^^^
//warning FS0193: この式は関数値です (つまり、引数が足りません)。
//型は  ^a -> unit です。
```

もしあなたがカリー化を理解してないなら、このメッセージは非常に不可解でしょう!こういった単独で評価される全ての式 (つまり、戻り値として使用されず、"let"で束縛されていない式) は、*必ず*unit値として評価されなければなりません。この場合はunit値とは*評価されず*関数と評価されてしまいました。これは、 「printfn」 の引数が足りないという長々しい表現です。

このようなエラーは通常、.NETライブラリとの連携時に発生します。たとえば、`TextReader`の`ReadLine`メソッドはunitパラメータを引数として要求します。このことを忘れて括弧を省略してしまうことがよくあります。このような場合、コンパイルエラーはすぐには発生せず、結果を文字列として処理しようとしたときに発生します。

```fsharp
let reader = new System.IO.StringReader("hello");

let line1 = reader.ReadLine        // 間違っていますが、コンパイラは文句を言いません

printfn "The line is %s" line1     // ここでコンパイラエラーが発生します！
// ==> error error FS0001: この式に必要な型は 'string' ですが、
// ここでは次の型が指定されています 'unit -> string'

let line2 = reader.ReadLine()      // 正しい書き方
printfn "The line is %s" line2     // コンパイラエラーなし
```

上記のコードでは、`line1`は単なる`Readline`メソッドへのポインタまたはデリゲートであり、期待した文字列ではありません。`()`を使用した`reader.ReadLine()`は、実際に関数を実行します。

## Too many parameters ##

パラメータの数が多すぎる場合も、同様の不可解なメッセージが表示されます。以下はprintfにパラメータを多く渡しすぎた例です。

```fsharp
printfn "hello" 42
// ==> error FS0001: この式に必要な型は''a -> 'b'ですが、
//                   ですが、ここでは次の型が指定されています 'unit'

printfn "hello %i" 42 43
// ==> Error FS0001: 型が一致しません。 ''a -> 'b -> 'c'という指定が必要ですが、
//                   ''a -> unit'が指定されました。型 ''a -> 'b' は型 'unit' と一致しません

printfn "hello %i %i" 42 43 44
// ==> Error FS0001: 型が一致しません。 ''a -> 'b -> 'c -> 'd'という指定が必要ですが、
//                   ''a -> 'b -> unit'が指定されました。型 ''a -> 'b' は型 'unit' と一致しません
```

たとえば、最後のケースでは、コンパイラはformat引数に3つのパラメータがあることを期待している (シグネチャ`'a->'b->'c->'d`には3つのパラメータがある) が、2つしか指定されていない (シグネチャ`'a->'b->unit`には2つのパラメータがある) と言っています。

`printf`を使用していない場合、パラメータを渡しすぎると、単純な値が返された後、更にパラメータを渡そうとすることになります。コンパイラは、単純な値は関数ではないとエラーを出力します。

```fsharp
let add1 x = x + 1
let x = add1 2 3
// ==>   error FS0003: この値は関数ではないため、
//                     適用できません。
```

先ほどのように、この呼び出しを一連の明示的な中間関数に分割すると、何が問題になっているのかがよくわかります。

```fsharp
let add1 x = x + 1
let intermediateFn = add1 2   //単純な値を返す
let x = intermediateFn 3      //intermediateFnは関数ではありません！
// ==>   error FS0003: この値は関数ではないため、
//                     適用できません。
```
