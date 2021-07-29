---
layout: post
title: "Function signatures"
description: "A function signature can give you some idea of what it does"
date: 2012-05-09
nav: thinking-functionally
seriesId: "Thinking functionally"
seriesOrder: 9
categories: [Functions]
---

わかりにくいかもしれませんが、f#には実際には2つの構文があります。1つは通常の(値)式用で、もう1つは型定義用です。たとえば。

```fsharp
[1;2;3]      // a normal expression
int list     // a type expression

Some 1       // a normal expression
int option   // a type expression

(1,"a")      // a normal expression
int * string // a type expression
```

型式には、通常の式で使用される構文とは*異なる*特別な構文があります。このような例は、インタラクティブ・セッションを使用するときに多く見てきました。それぞれの式の型とその評価が出力されているからです。

ご存知のように、f#は型推論を用いて型を導き出しますので、明示的にコードの中で型指定する必要はあまりありません。特別に数の場合はそうです。しかし、f#を効果的に動作させるためには、型構文を理解し、独自の型を構築し、型エラーをデバッグし、そして関数シグネチャを理解する必要があります。この記事では、関数シグネチャの使用方法に焦点を当てます。

以下に、型の構文を使用した関数シグネチャの例を示します。

```fsharp
// expression syntax          // type syntax
let add1 x = x + 1            // int -> int
let add x y = x + y           // int -> int -> int
let print x = printf "%A" x   // 'a -> unit
System.Console.ReadLine       // unit -> string
List.sum                      // 'a list -> 'a
List.filter                   // ('a -> bool) -> 'a list -> 'a list
List.map                      // ('a -> 'b) -> 'a list -> 'b list
```

## シグネチャによる関数の理解 ##

関数のシグネチャを調べるだけで、その関数が何をするのかある程度把握できます。いくつかの例を見て、順番に分析してみましょう。

```fsharp
// function signature 1
int -> int -> int
```

この関数は、2つの `int` パラメータを受け取り、別のパラメータを返すので、おそらく、加算、減算、乗算、指数などの何らかの数学的な関数であると考えられます。

```fsharp
// function signature 2
int -> unit
```

この関数は`int`を受け取り、`unit`を返します。これは、この関数が副次的に何か重要なことを行っていることを意味します。有用な戻り値がないので、おそらくその副作用は、ロギング、ファイルやデータベースへの書き込みなど、IOへの書き込みに関係することだと思います。

```fsharp
// function signature 3
unit -> string
```

この関数は入力を受け取りませんが、`string`を返します。つまり、この関数は何もないところから文字列を生成しているのです！明示的な入力がないので、この関数はおそらく読み込み (例えばファイルから) や生成(例えばランダムな文字列)に関係するものでしょう。

```fsharp
// function signature 4
int -> (unit -> string)
```

この関数は、`int`を入力として受け取り、呼び出されるとstringを返す関数を戻り値として返します。繰り返しますが、この関数はおそらく読み取りや生成に関係しています。入力は、何らかの方法で返される関数を初期化する可能性があります。たとえば、入力はファイルハンドルで、返される関数は`readline()`のようなものかもしれません。または、ランダムな文字列生成のためのシードを入力してるとも考えられます。正確には言えませんが、ある程度の推測はできます。

```fsharp
// function signature 5
'a list -> 'a
```

この関数は、ある型のリストを受け取りますが、その型の1つだけを返します。つまり、この関数はリストの要素を結合したり、選択したりしていることを意味します。このシグネチャを持つ関数の例としては，`List.sum`，`List.max`，`List.head`などがあります。

```fsharp
// function signature 6
('a -> bool) -> 'a list -> 'a list
```

この関数には2つのパラメータがあります。1つは何らかの値からbool (述部) への写像である関数で、もう1つはリストです。戻り値は同じ型のリストです。述部は、値がある種の基準を満たすかどうかを判断するために使用されるので、関数は述部が真かどうかに基づいてリストから要素を選択し、元のリストのサブセットを返しているように見えます。このシグネチャを持つ一般的な関数は、`List.filter`です。

```fsharp
// function signature 7
('a -> 'b) -> 'a list -> 'b list
```

この関数には2つのパラメータがあります。1つは型`'a`から型`'b`への写像で、もう1つは型`'a`のリストです。戻り値は異なる型`'b`のリストです。この関数は、最初のパラメータとして渡された関数を使用して、リスト内の各 「`a`」 を 「`b`」 へマップし、新しい 「`b`」 のリストを返すというのが妥当な推測です。そして、実際にこのシグネチャを持つ典型的な関数は`List.map`です。

### 関数シグネチャを用いたライブラリメソッドの検索 ###

関数シグネチャは、ライブラリ関数を探す際に重要です。F#ライブラリには何百もの関数が含まれており、最初のうちは圧倒されます。オブジェクト指向言語とは異なり、単純にオブジェクトに 「ドットイン」 して、あらゆる適切なメソッドを見つけるということはできません。しかしながら、探している関数のシグネチャがわかっていれば、候補のリストを素早く絞り込むことができます。

たとえば、2つのリストがあり、それらを1つに結合する関数を探しているとします。この関数のシグネチャは何ですか?この関数は2つのリストパラメータを取り、3番目の同じ型のパラメータを返すので、次のようなシグネチャになります:

```fsharp
'a list -> 'a list -> 'a list
```

次に、 [MSDN documentation for the F#List module](http://msdn.microsoft.com/en-us/library/ee353738) にアクセスして、関数のリストを下にスクロールし、一致するものを探します。このシグネチャを持つ関数は1つだけです。

```fsharp
append : 'T list -> 'T list -> 'T list
```

これこそまさに私たちが欲しいものです！

## 独自の型に関数シグネチャを定義する ##

必要な関数シグネチャに一致する独自型を作成することもできます。これを行うには、「type」 キーワードを使用し、シグニチャを記述するのと同じ方法で型を定義します。

```fsharp
type Adder = int -> int
type AdderGenerator = int -> Adder
```

これらの型を用いて、関数の値やパラメータを制約することができます。

たとえば、次の2番目の定義は、型制約のために失敗します。(3番目の定義のように) 型制約を削除した場合は問題ありません。

```fsharp
let a:AdderGenerator = fun x -> (fun y -> x + y)
let b:AdderGenerator = fun (x:float) -> (fun y -> x + y)
let c                = fun (x:float) -> (fun y -> x + y)
```

## 関数シグネチャの理解度を試す ##

関数シグネチャについてどの程度理解していますか?これらの各シグネチャを持つ簡単な関数を作成できるかどうかを確認してください。明示的な型注釈を使用しないでください！

```fsharp
val testA = int -> int
val testB = int -> int -> int
val testC = int -> (int -> int)
val testD = (int -> int) -> int
val testE = int -> int -> int -> int
val testF = (int -> int) -> (int -> int)
val testG = int -> (int -> int) -> int
val testH = (int -> int -> int) -> int
```
