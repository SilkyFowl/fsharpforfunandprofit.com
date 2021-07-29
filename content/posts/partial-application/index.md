---
layout: post
title: "Partial application"
description: "Baking-in some of the parameters of a function"
date: 2012-05-06
nav: thinking-functionally
seriesId: "Thinking functionally"
seriesOrder: 6
categories: [Currying, Partial Application]
---


前の記事では、複数パラメータ関数をより小さな1つのパラメータ関数に分割する方法について説明しました。これは数学的に正しい方法ですが、それだけではありません--これは**部分適用**と呼ばれる非常に強力なテクニックにもつながります。これは関数型プログラミングで非常に広く使われているスタイルで、理解することが重要です。

部分適用の考え方は、関数の最初のN個のパラメータを固定すると、残りのパラメータを引数とする関数が得られるというものです。カリー化の話から、これは当たり前に起こることであると分かるでしょう。

これを示す簡単な例をいくつか示します。

```fsharp {src=#demo1}
// addの部分適用による「加算器」の作成
let add42 = (+) 42    // 部分適用
add42 1
add42 3

// 各要素にadd42関数を適用して
// 新しいリストを作成する
[1;2;3] |> List.map add42

// 「less than」 の部分適用による 「テスター」 の作成
let twoIsLessThan = (<) 2   //  部分適用
twoIsLessThan 1
twoIsLessThan 3

// twoIsLessThan関数で各要素をフィルタリング
[1;2;3] |> List.filter twoIsLessThan

// printfnを部分適用し 「プリンタ」 を作成する
let printer = printfn "printing param=%i"

// 各要素をループしてprinter関数を呼び出す
[1;2;3] |> List.iter printer
```

いずれの場合も、部分適用された関数を作成し、複数のコンテキストで再利用できます。

もちろん、関数パラメータを固定することも、部分適用で簡単に行えます。次に例を示します。

```fsharp {src=#listDemo}
// List.mapを使用した例
let add1 = (+) 1
let add1ToEach = List.map add1   // 「add1」 関数を修正

// test
add1ToEach [1;2;3;4]

// List.filterを使用した例
let filterEvens =
  List.filter (fun i -> i%2 = 0) // フィルタ関数を修正

// test
filterEvens [1;2;3;4]
```

次のより複雑な例は、同じ方法で透過的な 「プラグイン」 動作を実現する方法を示しています。

* 2つの数値を加算する関数を作成し、さらに2つの数値とその結果をログに出力するロギング関数を引数とします。
* ロギング関数には (string) 「name」 と (generic) 「value」 の2つのパラメータがあるため、シグネチャ`string->'a->unit`があります。
* 次に、コンソールロガーや色付きロガーなど、ロギング機能のさまざまな実装を作成します。
* 最後に、main関数で部分適用することで、特定のロガーを組み込んだ新しい関数を作成します。

```fsharp {src=#logger}
// プラグイン可能なロギング関数をサポートする加算器を作成
let adderWithPluggableLogger logger x y =
  logger "x" x
  logger "y" y
  let result = x + y
  logger "x+y"  result
  result

// コンソールに出力するロギング関数を作成
let consoleLogger argName argValue =
  printfn "%s=%A" argName argValue

//コンソールロガーが部分適用された加算器を作成
let addWithConsoleLogger = adderWithPluggableLogger consoleLogger
addWithConsoleLogger 1 2
addWithConsoleLogger 42 99

// 赤文字を使用するロギング関数を作成
let redLogger argName argValue =
  let message = sprintf "%s=%A" argName argValue
  System.Console.ForegroundColor <- System.ConsoleColor.Red
  System.Console.WriteLine("{0}",message)
  System.Console.ResetColor()

// 部分適用されたポップアップロガーを使用する加算器を作成
let addWithRedLogger = adderWithPluggableLogger redLogger
addWithRedLogger 1 2
addWithRedLogger 42 99
```

ロガーを組み込んだこれらの関数は、他の関数と同様に使用できます。例えば、単純な「`add42`」 関数で行ったように、42を追加して部分適用された関数を作成し、それをリスト関数に渡すことができます。

```fsharp {src=#add42WithConsoleLogger}
// 42を組み込んだ別の加算器を作成
let add42WithConsoleLogger = addWithConsoleLogger 42
[1;2;3] |> List.map add42WithConsoleLogger
[1;2;3] |> List.map add42               //ロガーなしとの比較
```

これらの部分適用された関数は非常に有用なツールです。柔軟な (しかし複雑な) ライブラリ関数を作りつつ、再使用可能な既定値を簡単に作ることができるので、呼出し側が常にその複雑さにさらされる必要がなくなります。

## 部分適用のための関数設計 ##

パラメータの順序によって、部分適用の使いやすさは大きく異なります。たとえば、`List.map`や`List.filter`など、`List`ライブラリ内のほとんどの関数は、次のような似た形式を持っています。

	List-function [function parameter(s)] [list]

リストは常に最後の引数です。完全な形式の例を次に示します。

```fsharp {src=#listWithoutPa}
List.map    (fun i -> i+1) [0;1;2;3]
List.filter (fun i -> i>1) [0;1;2;3]
List.sortBy (fun i -> -i ) [0;1;2;3]
```

部分適用を使った同じ例:

```fsharp {src=#listWithPa}
let eachAdd1 = List.map (fun i -> i+1)
eachAdd1 [0;1;2;3]

let excludeOneOrLess = List.filter (fun i -> i>1)
excludeOneOrLess [0;1;2;3]

let sortDesc = List.sortBy (fun i -> -i)
sortDesc [0;1;2;3]
```

もし、ライブラリ関数が異なる順序の引数で書かれていたら、部分適用を使うことがずっと不便になってしたたでしょう。

独自の複数パラメータ関数を作成する場合、最適なパラメータの順序は何か気になるかもしれません。他の設計に関する質問と同様、この質問に対する 「正しい」 回答はありませんが、一般的に受け入れられているガイドラインをいくつか示します。

1. 先に配置:静的になりやすいパラメータ
2. 最後に配置:データ構造またはコレクション (あるいは最も変わりやすい引数)
3. 「減算する」など、よく知られる処理については、想定される順序で配置します。

ガイドライン1は簡単です。部分的適用で「固定」 される可能性が最も高いパラメータが最初になります。先ほどのロガーの例でこれを見ました。

ガイドライン2で、構造またはコレクションを関数から関数へパイプすることが容易になります。これはリスト関数で既に何度も見てきました。

```fsharp {src=#listPipe}
// リスト関数を使用したパイプ処理
let result =
  [1..10]
  |> List.map (fun i -> i+1)
  |> List.filter (fun i -> i>5)
// output => [6; 7; 8; 9; 10; 11]
```

同様に、部分適用されたリスト関数は簡単に合成できます。これは、リストパラメータ自体を簡単に省略できるためです。

```fsharp {src=#listCompose}
let f1 = List.map (fun i -> i+1)
let f2 = List.filter (fun i -> i>5)
let compositeOp = f1 >> f2 // 合成
let result = compositeOp [1..10]
// output => [6; 7; 8; 9; 10; 11]
```

### 部分適用のための基本クラスライブラリのラッパー関数 ###

.NET基本クラスライブラリの機能はF#で簡単にアクセスできますが、実際のところF#のような関数型言語で使うようには設計されていません。例えば、ほとんどの関数は最初にdataパラメータを要求しますが、先ほど説明したようにF#ではdataパラメータは通常最後になります。

しかし、より慣用的なラッパーを作成するのは簡単です。たとえば次のスニペットでは、.NET string関数は、stringターゲットが最初ではなく最後のパラメータになるように書き換えられます。

```fsharp {src=#wrapper}
// .NET string関数のラッパーを作成します。
let replace oldStr newStr (s:string) =
  s.Replace(oldValue=oldStr, newValue=newStr)

let startsWith (lookFor:string) (s:string) =
  s.StartsWith(lookFor)
```

文字列が最後のパラメータになると、期待通りにパイプで使用できます:

```fsharp {src=#wrapperPipes}
let result =
  "hello"
  |> replace "h" "j"
  |> startsWith "j"

["the"; "quick"; "brown"; "fox"]
  |> List.filter (startsWith "f")
```

関数合成を使用する場合:

```fsharp {src=#wrapperCompose}
let compositeOp = replace "h" "j" >> startsWith "j"
let result = compositeOp "hello"
```

### "pipe"関数を理解する ###.

部分適用の仕組みが分かったなら、「pipe」 関数の仕組みも理解できるはずです。

pipe関数は次のように定義されます:

```fsharp {src=#pipeDefinition}
let (|>) x f = f x
```

この関数は、引数を関数の後ではなく前に配置できるようにします。それだけです。

```fsharp {src=#pipe1}
let doSomething x y z = x+y+z
doSomething 1 2 3       // 全てのパラメータが関数の後に配置されてる
```

関数に複数パラメータがある場合は、入力が最終パラメータになるように見えます。しかし実際には、関数は部分適用され、単一のパラメータを持つ関数を返します。

同じ例を、部分適用を用いて書き直しました。

```fsharp {src=#pipe2}
let doSomething x y  =
  let intermediateFn z = x+y+z
  intermediateFn        // intermediateFnを返す

let doSomethingPartial = doSomething 1 2
doSomethingPartial 3     // この時点では関数の後のパラメータは1つだけ
3 |> doSomethingPartial  // 上と同等 - 最終パラメータをパイプで入力
```

既に説明したように、F#ではパイプ演算子は極めて一般的であり、自然な流れを維持するためにいつも使用されます。その他の使用方法を次に示します:

```fsharp {src=#pipe3}
"12" |> int               // 文字列「12」をintに変換
1 |> (+) 2 |> (*) 3       // 算術の連鎖
```

### 逆パイプ関数 ###

パイプの逆関数「<|」が使用されていることがあります。

```fsharp {src=#reversePipe}
let (<|) f x = f x
```

この機能を使っても通常どおりの動作しかしないように見えますが、なぜ存在するのでしょうか?

その理由は、中置形式で二項演算子として使用すると、括弧の必要性が減り、コードが簡潔になるためです。

```fsharp {src=#reversePipeDemo1}
printf "%i" 1+2          // error
printf "%i" (1+2)        // 括弧を使用
printf "%i" <| 1+2       // 逆パイプを使用
```

一度に両方向のパイプを使用して、擬似的に中置記法を使用することもできます。

```fsharp {src=#reversePipeDemo2}
let add x y = x + y
(1+2) add (3+4)          // error
1+2 |> add <| 3+4        // 中値記法
```
