---
layout: post
title: "Worked example: A stack based calculator"
description: "Using combinators to build functionality"
date: 2012-07-07
nav: thinking-functionally
seriesId: "Thinking functionally"
seriesOrder: 12
categories: [Combinators, Functions, Worked Examples]
---

この記事では、シンプルなスタックベースの計算機を実装します（"逆ポーランド"記法としても知られています）。この実装は、ほとんどが関数で行われており、特殊な型は1つだけで、パターンマッチも全くないので、このシリーズで紹介する概念を試すのに最適なものとなっています。

スタックベースの計算機に慣れていない方のために説明すると、次のような仕組みになっています。数字はスタックにプッシュされ、足し算や掛け算などの演算は、スタックから数字を取り出し、その結果を再びスタックにプッシュします。

以下は、スタックを使った簡単な計算の図です。

![スタック式計算機の図](./stack-based-calculator.png)

このようなシステムを設計する最初のステップは、それがどのように使われるかを考えることです。Forthのような構文にしたがって、それぞれのアクションにラベルをつけていきます。

    EMPTY ONE THREE ADD TWO MUL SHOW

正確な構文にはならないかもしれませんが、どこまで近づけることができるか見てみましょう。

## スタックのデータ型

まず、スタックのデータ構造を定義する必要があります。ここではシンプルにするために、floatのリストを使います。

```fsharp
type Stack = float list
```

でもちょっと待ってください、 [シングルケースの判別共用体](/posts/discriminated-unions/#single-case)でラップして、より分かりやすくしましょう。

```fsharp
type Stack = StackContents of float list
```

なぜこれが良いのかについての詳細は、 [この投稿](/posts/discriminated-unions/#single-case) のシングルケースの判別共用体の話を読んでください。

さて、新しいスタックを作るには、コンストラクタとして `StackContents` を使います:

```fsharp
let newStack = StackContents [1.0;2.0;3.0]
```

また，既存のStackの内容を取り出すには，`StackContents`を使ってパターンマッチします:

```fsharp
let (StackContents contents) = newStack

// "contents"の値を次のように設定します。
// float list = [1.0; 2.0; 3.0].
```


## Push関数

次に、数値をスタックにプッシュする方法が必要です。これは単純に、"`::`"演算子を使って、新しい値をリストの先頭に置くことになります。

ここでは、プッシュ関数を紹介します:

```fsharp
let push x aStack =
    let (StackContents contents) = aStack
    let newContents = x::contents
    StackContents newContents
```

この関数には、検討すべき点がいくつかあります。

まず、list構造は不変であるため、この関数は既存のスタックを受け取り、新しいスタックを返す必要があるという点です。既存のスタックを変更することはできません。事実、このサンプルの関数はすべて、次のような形式になります:

    入力:スタックとその他のパラメータ
    出力:新しいスタック

次はパラメータの順序です。スタックのパラメータを先頭にすべきか、最後にすべきか?[部分適用のための関数設計](/posts/partial-application) の話を覚えていれば、最も変わりやすいものは最後に置くべきだということも覚えているでしょう。このガイドラインは間もなく発表されるでしょう。

最後に、関数本体で `let`を使用するのではなく、関数パラメータ自体でパターンマッチングすることで、関数をより簡潔にすることができます。

こちらが書き直したものです:

```fsharp
let push x (StackContents contents) =
    StackContents (x::contents)
```

とても良くなりました！

ところで、この素敵なシグネチャを見てください:

```fsharp
val push : float -> Stack -> Stack
```

[以前の記事](/posts/function-signatures)から分かるように、シグネチャは関数について多くのことを教えてくれます。
この場合、関数の名前が"push"であることを知らなくても、シグネチャだけで何をするのか推測できます。
これが、明示的な型名を使用することが望ましい理由の1つです。スタック型が単にfloatのリストであったなら、自己文書化ではなかったでしょう。

とにかく、テストしてみましょう:

```fsharp
let emptyStack = StackContents []
let stackWith1 = push 1.0 emptyStack
let stackWith2 = push 2.0 stackWith1
```

うまくいきました！

## "push"の上での構築

このシンプルな関数を使用すると、特定の番号をスタックにプッシュする操作を簡単に定義できます。

```fsharp
let ONE stack = push 1.0 stack
let TWO stack = push 2.0 stack
```

でもちょっと待ってください!`stack`パラメータが両側で使用されていますね?実は、このパラメータに触れる必要はありません。代わりに、`stack`パラメータを省略し、以下のように部分適用を用いて関数を定義できます:

```fsharp
let ONE = push 1.0
let TWO = push 2.0
let THREE = push 3.0
let FOUR = push 4.0
let FIVE = push 5.0
```

`push`のパラメータの順序が違うなら、このようにはできなかったでしょう。

ここで、空のスタックを作成する関数も定義しましょう:

```fsharp
let EMPTY = StackContents []
```

では、すべてをテストしてみましょう:

```fsharp
let stackWith1 = ONE EMPTY
let stackWith2 = TWO stackWith1
let stackWith3  = THREE stackWith2
```

このような中間スタックは煩わしいものです――取り除くことはできないでしょうか?出来ます!これらの関数ONE、TWO、THREEはすべて同じシグネチャを持つことに注目してください:

```fsharp
Stack -> Stack
```

ということは、きれいに繋げられます！以下のように出力を次の入力へと送ることができます:

```fsharp
let result123 = EMPTY |> ONE |> TWO |> THREE
let result312 = EMPTY |> THREE |> ONE |> TWO
```


# スタックのポップ

これでスタックにプッシュすることができます ―― 続いて`pop`関数はいかがでしょうか?

スタックをポップすると、当然スタックの先頭が返されますが、それだけでしょうか?

オブジェクト指向では、 [その答えは"はい"です](http://msdn.microsoft.com/en-us/library/system.collections.stack.pop.aspx) 。OO方式では、スタック自体を背後で*変更*して先頭の要素を削除します。

しかし、関数型のスタイルでは、スタックは不変です。最上位要素を削除する唯一の方法は、その要素を削除した*新しいスタック*を作成することです。
呼び出し側が小さくなった新しいスタックにアクセスできるようにするには、先頭の要素といっしょに返す必要があります。

言い換えると、`pop`関数は*2つの*値を返さなければなりません。F#でこれを行う最も簡単な方法は、タプルを使うことです。

実装を次に示します:

```fsharp
/// スタックから値をポップして、
/// 新しいスタックとのタプルとして返す
let pop (StackContents contains) =
    match contains with
    | top::rest ->
        let newStack = StackContents rest
        (top, newStack)
```

この関数も非常に単純です。

これまでと同様に、 `contents`はパラメータから直接抽出されます。

次に、`match..with`式を使用して内容を調べます。

続いて、先頭の要素を残りの要素から切り分け、残りの要素から新しいスタックを作成し、最後にそのペアをタプルとして返します。

上記のコードを試して、何が起こるかを見てください。コンパイラエラーが表示されます。
コンパイラーが、見落としていたケースを見つけました--スタックが空の場合はどうなりますか?

これをどう扱うか決めなければなりません

* Option 1: ["Why use F#?"シリーズでの投稿](/posts/correctness-exhaustive-pattern-matching/) で行ったように、特別な"Success"または"Error"状態を返します。
* Option 2: 例外をスローします。

通常、筆者はエラーケースを使用しますが、この場合は例外を使用します。空の場合を処理するように変更された`pop`コードを次に示します:

```fsharp
/// スタックから値をポップして、
/// 新しいスタックとのタプルとして返す
let pop (StackContents contents) =
    match contents with
    | top::rest ->
        let newStack = StackContents rest
        (top,newStack)
    | [] ->
        failwith "Stack underflow"
```

では、テストしてみましょう:

```fsharp
let initialStack = EMPTY |> ONE |> TWO
let popped1, poppedStack = pop initialStack
let popped2, poppedStack2 = pop poppedStack
```

そして，アンダーフローをテストします:

```fsharp
let _ = pop EMPTY
```

## 数学関数の記述

pushとpopができたので、今度はaddとmultiplyの関数を書いてみましょう。

```fsharp
let ADD stack =
   let x,s = pop stack //スタックの先頭をポップする
   let y,s2 = pop s    //結果のスタックをポップする
   let result = x + y  //計算をする
   push result s2      //二重にポップされたスタックにプッシュバックする

let MUL stack =
   let x,s = pop stack //スタックの先頭をポップする
   let y,s2 = pop s    //結果のスタックをポップする
   let result = x * y  //計算を行う
   push result s2      //二重にポップされたスタックにプッシュバックする
```

これらをインタラクティブにテストします。

```fsharp
let add1and2 = EMPTY |> ONE |> TWO |> ADD
let add2and3 = EMPTY |> TWO |> THREE |> ADD
let mult2and3 = EMPTY |> TWO |> THREE |> MUL
```

うまくいきました。

### リファクタリングの時間...

この2つの関数の間には、明らかにコードの重複があります。どのようにリファクタリングするべきでしょうか?

どちらの関数もスタックから2つの値を取り出し、何らかの二項関数を適用し、結果をスタックに戻します。そのため、共通コードをリファクタリングして、2パラメーターの算術関数をパラメーターとして取る "バイナリ"関数にしましょう:

```fsharp
let binary mathFn stack =
    // スタックの先頭をポップする
    let y,stack' = pop stack
    // 再びスタックの先頭をポップする
    let x,stack'' = pop stack'
    // 計算を行う
    let z = mathFn x y
    // 結果の値を二重にポップされたスタックに戻す
    push z stack''
```

*この実装では、 "同じ"オブジェクトの変更された状態を表すために、数字の添字ではなくティック{'}を使っていることに注意してください。数字の添字では混乱を招きがちです。*

問: なぜパラメータは `mathFN`ではなく`stack`の後にあるのですか?

これで `binary'ができたので、ADDとその仲間たちをより簡潔に定義することができます。

以下は、新しい`binary`ヘルパーを使ったADDの最初の試行例です:

```fsharp
let ADD aStack = binary (fun x y->x+y)  aStack
```

しかし、このラムダは組み込みの`+`関数の定義と*まったく*同じなので取り除くことができます！その結果です:

```fsharp
let ADD aStack = binary (+) aStack
```

繰り返しますが、部分適用を使ってスタックパラメータを隠すことができます。最終的な定義は次のとおりです:

```fsharp
let ADD = binary (+)
```

他の算術関数の定義は次のとおりです。

```fsharp
let SUB = binary (-)
let MUL = binary (*)
let DIV = binary (/)
```

もう一度インタラクティブにテストしてみましょう。

```fsharp
let threeDivTwo = EMPTY |> THREE |> TWO |> DIV   // Answer: 1.5
let twoSubtractFive = EMPTY |> TWO |> FIVE |> SUB  // Answer: -3.0
let oneAddTwoSubThree = EMPTY |> ONE |> TWO |> ADD |> THREE |> SUB // Answer: 0.0
```

同様に、単項関数用のヘルパー関数を作成することもできます。

```fsharp
let unary f stack =
    let x,stack' = pop stack  // スタックの先頭をポップする
    push (f x) stack'         // スタックに関数値をプッシュする
```

次に、単項関数をいくつか定義します:

```fsharp
let NEG = unary (fun x -> -x)
let SQUARE = unary (fun x -> x * x)
```

インタラクティブに再テストしましょう:

```fsharp
let neg3 = EMPTY |> THREE |> NEG
let square2 = EMPTY |> TWO |> SQUARE
```

## すべてをまとめあげる

元の要件では、結果を表示できるようにしたいと述べていたので、SHOW関数を定義しましょう。

```fsharp
let SHOW stack =
    let x,_ =pop stack
    printfn "The answer is %f" x
    stack  // 同じスタックを使い続ける
```

この場合、元のスタックはポップされますが、縮小バージョンは無視されます。関数の最終結果は、元のスタックになり、まるでポップされなかったようになります。

これでようやく、元の要件を満たすコード例を書くことができます。

```fsharp
EMPTY |> ONE |> THREE |> ADD |> TWO |> MUL |> SHOW // (1+3)*2 = 8
```

### 今後の展開

楽しいですね -- 他に何ができるでしょう?

では、さらにいくつかのコアヘルパー関数を定義しましょう:

```fsharp
/// スタックの先頭の値を複製する
let DUP stack =
    // スタックの先頭を取得
    let x,_ = pop stack
    // これをスタックにプッシュします
    push x stack

/// 上位2つの値を入れ替える
let SWAP stack =
    let x,s = pop stack
    let y,s' = pop s
    push y (push x s')

/// 始点を明確にする
let START = EMPTY
```

これらの関数を追加することで、良い例を書くことができます:

```fsharp
START
    |> ONE |> TWO |> SHOW

START
    |> ONE |> TWO |> ADD |> SHOW
    |> THREE |> ADD |> SHOW

START
    |> THREE |> DUP |> DUP |> MUL |> MUL // 27

START
    |> ONE |> TWO |> ADD |> SHOW  // 3
    |> THREE |> MUL |> SHOW       // 9
    |> TWO |> DIV |> SHOW         // 9 div 2 = 4.5
```

## パイプの代わりに合成を用いる

でもこれだけではありません、実際は、他の非常に興味深い方法でこれらの関数を考えることができます。

先に述べたように、これらはすべて同じシグネチャを持っています。

```fsharp
Stack -> Stack
```

したがって、入力型と出力型は同じであるため、これらの関数は、パイプで連結されるだけでなく、合成演算子`>>`を使用して合成も可能です。

次に例を示します:

```fsharp
// 新しい関数を定義する
let ONE_TWO_ADD =
    ONE >> TWO >> ADD

// それをテストする
START |> ONE_TWO_ADD |> SHOW

// 新しい関数を定義する
let SQUARE =
    DUP >> MUL

// テストする
スタート｜> two｜> square｜> show

// 新しい関数を定義する
let CUBE =
    DUP >> DUP >> MUL >> MUL

// テストする
START |> THREE |> CUBE |> SHOW

// 新しい関数を定義する
let SUM_NUMBERS_UPTO =
    DUP      // n, n      スタック上の2アイテム
    >> ONE   // n, n, 1   スタック上の3つのアイテム
    >> ADD   // n, (n+1)  スタック上の2つのアイテム
    >> MUL   // n(n+1)    スタック上のアイテム
    >> TWO   // n(n+1), 2 スタックの2項目
    >> DIV   // n(n+1)/2  スタック上の1つのアイテム

// 9までの数字の合計でテストする
START |> THREE |> SQUARE |> SUM_NUMBERS_UPTO |> SHOW  // 45
```

いずれの場合も、他の関数を合成して新しい関数を構築することによって、別の関数を定義します。これは関数を構築する "コンビネータ"という手法の良い例です。

## パイプ vs コンポジション

このスタックベースのモデルを利用するため、2つの異なる方法を見てきました;パイプと合成でです。これらの違いは何でしょう?一方が他方より優れている点は何でしょうか?

この違いは、パイプはある意味"リアルタイム変換"操作であるという点にあります。パイプを使用すると、特定のスタックを渡しながら実際に操作を実行することになります。

その一方で、合成とはある種の "計画"です。いくつかの部分から全体の機能を構築しますが、それを実際に動かしているわけではありません。

たとえば、小さな演算を組み合わせて数値を二乗する "計画"を作成できます:

```fsharp
let COMPOSED_SQUARE = DUP >> MUL
```

これはパイピングによる手法ではできません。

```fsharp
let PIPED_SQUARE = DUP |> MUL
```

コンパイルエラーが発生します。これを動作させるには、何らかの具体的なスタックインスタンスが必要です。

```fsharp
let stackWith2 = EMPTY |> TWO
let twoSquared = stackWith2 |> DUP |> MUL
```

その場合でも、COMPOSED_SQUAREの例のように、すべての入力可能な値に対する計画ではなく、この特定の入力に対する答えしか得られません。

"計画"を作成するもう1つの方法は、冒頭に示したようにラムダをよりプリミティブな関数へ明示的に渡すことです。

```fsharp
let LAMBDA_SQUARE = unary (fun x -> x * x)
```

これははるかに明確ですが (そしてより高速になりそうです) 、合成による手法の利点と明確さをすべて失います。

ですから可能であれば、通常は合成によるアプローチを選んでください！

## 完成したコード

ここに、これまでの全サンプルの完全なコードを記載します。

```fsharp
// ==============================================
// タイプ
// ==============================================

type Stack = float listのStackContents

// ==============================================
// スタックのプリミティブ
// ==============================================

/// スタックに値をプッシュする
let push x (StackContents contents) =
    StackContents (x::contents)

/// スタックから値をポップして返す
/// そして新しいスタックをタプルとして返す
let pop (StackContents contents) =
    match contents with
    | top::rest ->
        let newStack = StackContents rest
        (top,newStack)
    | [] ->
        failwith "Stack underflow"

// ==============================================
// オペレータコア
// ==============================================

// 上の2つの要素を取り出す
// それらに対して二項演算を行う
// その結果をプッシュする
let binary mathFn stack =
    let y,stack' = pop stack
    let x,stack'' = pop stack'
    let z = mathFn x y
    push z stack''

// 一番上の要素をポップ
// その要素に対して単項演算を行う
// その結果をプッシュ
let unary f stack =
    let x,stack' = pop stack
    push (f x) stack'

// ==============================================
// その他のコア
// ==============================================

/// スタックの先頭の値をポップして表示する
let SHOW stack =
    let x,_ = pop stack
    printfn "The answer is %f" x
    stack // 同じスタックを使い続ける

/// スタックの一番上の値を複製する
let DUP stack =
    let x,s = pop stack
    push x (push x s)

/// 上の2つの値を入れ替える
let SWAP stack =
    let x,s = pop stack
    let y,s' = pop s
    push y (push x s')

/// スタックの一番上の値を落とす
let DROP stack =
    let _,s = pop stack //スタックの一番上の値をポップする
    s                   //残りの部分を返す

// ==============================================
// プリミティブに基づく単語
// ==============================================

// 定数
// -------------------------------
let EMPTY = StackContents []
let START = EMPTY


// 数値
// -------------------------------
let ONE = push 1.0
let TWO = push 2.0
let THREE = push 3.0
let FOUR = push 4.0
let FIVE = push 5.0

// 算術関数
// -------------------------------
let ADD = binary (+)
let SUB = binary (-)
let MUL = binary (*)
let DIV = binary (/)

let NEG = unary (fun x -> -x)


// ==============================================
// 合成に基づいた単語
// ==============================================

let SQUARE =
    DUP >> MUL

let CUBE =
    DUP >> DUP >> MUL >> MUL

let SUM_NUMBERS_UPTO =
    DUP      // n, n      スタック上の2アイテム
    >> ONE   // n, n, 1   スタック上の3つのアイテム
    >> ADD   // n, (n+1)  スタック上の2つのアイテム
    >> MUL   // n(n+1)    スタック上のアイテム
    >> TWO   // n(n+1), 2 スタックの2項目
    >> DIV   // n(n+1)/2  スタック上の1つのアイテム
```

## まとめ

これは単純なスタックベースの計算機です。私たちは、いくつかの基本的な演算 (`push`, `pop`, `binary`, `unary`) から始めて、実装が簡単で使いやすいドメイン固有の言語を構築する方法を見てきました。

ご想像のとおり、この例は第四世代言語に深く準拠しています。私は、無料の本 ["Thinking Forth"]
(http://thinking-fort.sourceforge.net/) を強くお勧めします。これは、4GLだけでなく、関数型プログラミングにも同様に適用可能な (*非*オブジェクト指向な！) 問題の分割手法について書かれています。

この記事のアイデアは、 [Ashley Feniello](http://blogs.msdn.com/b/ashleyf/archive/2011/04/21/programming-is-pointless.aspx) さんの素晴らしいブログから得ました。F#でスタックベースの言語をエミュレートしたいなら、そこから始めましょう。どうぞお楽しみください！
