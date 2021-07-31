---
layout: post
title: "Control flow expressions"
description: "And how to avoid using them"
date: 2012-05-20
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 7
---

この記事では、制御フロー式を見ていきます:

* if-then-else
* for x in collection (これはC#のforeachと同じです)
* for x = start to end
* while-do

これらの制御フロー式は、皆さんにとって非常になじみのあるものです。しかし、これらは関数的というよりも、むしろ"命令的"なものです。

ですから、特に関数型思考を学んでいるときは、可能な限り使わないことを強くお勧めします。これを松葉杖として使うと、命令的思考から抜け出すのが非常に難しくなります。

これを支援するために、各章では、より慣用的構文を使用して、それらを使用しない方法の例から始めます。使用する必要がある場合は、いくつかの"落とし穴"に注意する必要があります。

## If-then-else

### if-then-else関数の使用を避ける方法

`if-then-else`を避ける最善の方法は、代わりに"match"を使うことです。ブール値を照合できます。これは、従来のthen/else分岐に似ています。しかし、はるかに優れています、以下の最後の実装に示されているように、等価性テストを避け、オブジェクト自体に一致させるからです。

```fsharp
//不適切
let f x =
    if x = 1
    then "a"
    else "b"

// あまり良くありません
let f x =
    match x = 1 with
    | true -> "a"
    | false -> "b"

// ベスト
let f x =
    match x with
    | 1 -> "a"
    | _ -> "b"
```

直接照合が優れている理由の1つは、等価性テストでは、しばしば有用な情報が破棄され再取得が必要になる点にあります。

次のシナリオでは、リストを出力するため先頭の要素を取得します。当然、空のリストの場合はこの操作を行わないように注意する必要があります。

最初の実装は空であるかどうかをテストし、次に*2つ目の*演算を行って最初の要素を取得します。その次の実装に示すように、1ステップで要素をマッチングして抽出する方が、はるかに優れた方法です。

```fsharp
// 不適切
let f list=
    if List.isEmpty list
    printfn "is empty"
    else printfn "first element is %s" (List.head list)

// より適切
let f list=
    match list with
    | [] -> printfn "is empty"
    | X::_ -> printfn "first element is %s" x
```

2番目の実装は理解しやすいだけでなく、より効率的です。

ブーリアンテストが複雑な場合でも、"`when`"句 ("guards"と呼ばれます) を追加することで、matchで行うことができます。以下の1番目と2番目の実装を比較して、違いを確認してください。

```fsharp
//不適切
let f list =
    if List.isEmpty list
        printfn "is empty"
        elif (List.head list) > 0
            then printfn "first element is > 0"
            else printfn "first element is <= 0"

// より適切
let f list =
    match list with
    | [] -> printfn "is empty"
    | x::_ when x > 0 -> printfn "first element is > 0"
    | x::_ -> printfn "first element is <= 0"
```

繰り返しますが、2番目の実装は理解しやすく、効率的です。

もし、if-then-elseやブーリアンの照合を使っていたら、コードのリファクタリングを検討してみてはいかがでしょうか。

### if-then-else の使い方

if-then-elseを使用する必要がある場合は、構文が見慣れたものであっても、注意しなければならない落とし穴があることに注意してください。"`if-then-else`"は*文*ではなく*式*であり、F#の他の式と同様に、特定の型の値を返す必要があります。

戻り値の型がstringである2つの例を次に示します:

```fsharp
let v = if true then "a" else "b" // value : string
let f x = if x then "a" else "b" // function : bool->string
```

しかし，結果として，どちらのブランチも同じ型を返さなければなりません  もしそうでなければ，式全体として一貫した型を返すことができず，コンパイラが文句を言います。

ここでは，各分岐で異なる型を返す例を示します。

```fsharp
let v = if true then "a" else 2
  // error FS0001: if' 式のすべてのブランチは同じ型である必要があります。
  //               この式に必要な型は 'string' ですが、ここでは型 'int' になっています。
```

"else"節は必須ではありませんが、省略した場合、"else"節はunitを返すものとみなされます。つまり、"then"節もunitを返す必要があります。この間違いをすると、コンパイラが文句を言います。

```fsharp
let v = if true then "a"
  // error FS0001: 'if' 式に 'else' ブランチがありません。'then' ブランチは型 'string' です。
  //               'if' はステートメントではなく式であるため、同じ型の値を返す 'else' ブランチを追加してください。
```

もし"then "節がunitを返せば，コンパイラは喜ぶでしょう。

```fsharp
let v2 = if true then printfn "a"   // printfnがunitを返すのでOK
```

Note: 分岐で早く戻る方法はありません。戻り値は式全体です。つまり、if-then-else式は、C#if-then-else文よりもC#三項if演算子 (`<if expr>?<then expr>:<else expr>`) に近い関係にあります。

###ワンライナーのためのif-then-else

if-then-elseが本当に役に立つのは、他の関数に渡すためのシンプルなワンライナーを作る場合です。

```fsharp
let posNeg x = if x > 0 then "+" elif x < 0 then "-" else "0"
[-5..5] |> List.map posNeg
```

### 関数を返す

if-then-else式は、関数の値を含めてあらゆる値を返せることを忘れてはいけません。たとえば、以下のようになります:

```fsharp
let greetings =
    if (System.DateTime.Now.Hour < 12)
    then (fun name -> "good morning, " + name)
    else (fun name -> "good day, " + name)

//テスト
greetings "Alice"
```

もちろん、どちらの関数も同じ型、つまり同じ関数シグネチャを持っていなければなりません。

## ループ ##

### ループを使わずに済ませるには ###

ループを回避する最善の方法は、代わりにビルトインのリストやシーケンス関数を使用することです。明示的なループを使用しなくても、ほとんどの操作を実行できます。また、多くの場合、副次的な利点として、変更可能な値を回避することもできます。最初にいくつかの例を示します。詳細については、リスト操作とシーケンス操作に特化した今後のシリーズを参照してください。

例:何かを10回出力する:

```fsharp
// 悪い
for i = 1 to 10 do
   printf "%i" i

// ずっと良い
[1..10] |> List.iter (printf "%i")
```

例 リストの和をとる:

```fsharp
// 悪い
let sum list =
    let mutable total = 0    // あーあ -- ミュータブルな値
    for e in list do
        total <- total + e   // ミュータブルな値を更新する
    total                    // 合計を返す

// ずっと良い
let sum list = List.reduce (+) list

//テスト
sum [1..10]
```

例:乱数の生成と出力:

```fsharp
// 悪い
let printRandomNumbersUntilMatched matchValue maxValue =
  let mutable continueLooping = true  // もうひとつの mutable 値
  let randomNumberGenerator = new System.Random()
  while continueLooping do
    // 1からmaxValueの間の乱数を生成します。
    let rand = randomNumberGenerator.Next(maxValue)
    printf "%d " rand
    if rand = matchValue then
       printfn "\nFound a %d!" matchValue
       continueLooping <- false

// もっと良い方法
let printRandomNumbersUntilMatched matchValue maxValue =
  let randomNumberGenerator = new System.Random()
  let sequenceGenerator _ = randomNumberGenerator.Next(maxValue)
  let isNotMatch = (<>) matchValue

  //乱数のシーケンスの作成と処理
  Seq.initInfinite sequenceGenerator
    |> Seq.takeWhile isNotMatch
    |> Seq.iter (printf "%d ")

  // done
  printfn "\nFound a %d!" matchValue

//テスト
printRandomNumbersUntilMatched 10 20
```

if-then-elseの場合と同様です; ループやmutableを使用している場合は、それらを避けるためにコードのリファクタリングを検討してください。

### ループの3つのタイプ

ループを使用する場合、C# と同様に 3 種類のループ表現があります。

* `for-in-do`.  これは、`for x in enumerable do something`という形をしています。これはC#の`foreach`ループと同じで、F#で最もよく見られる形式です。
* `for-to-do`.  これは `for x = start to finish do something` という形をしています。これはC#の標準的な`for (i=start; i<end; i++)`ループと同じです。
* `while-do`. これは、`while test do something`という形をしています。これはC#の`while`ループと同じです。 なお、F#には`do-while`に相当するものはありません。

使い方は簡単なので、これ以上の詳細は説明しません。困ったときは[MSDNドキュメント](http://msdn.microsoft.com/en-us/library/dd233227.aspx)を確認してみてください。

### ループの使い方

if-then-else式と同様に、ループ式は見慣れたものに見えますが、いくつかの落とし穴があります。

* すべてのループ式は常に式全体をunitとして返すため、ループ内から値を返す方法はありません。
* すべての"do"束縛と同様に、ループ内の式もunitを返さなければなりません。
* "break"と"continue"に相当するものはありません (どちらにしても通常はシーケンスを使用した方が良いでしょう) 。

次に、unit制約の例を示します。ループ内の式はintではなくunitでなければならないので、コンパイラは文句を言います。

```fsharp
let f =
  for i in [1..10] do
    i + i // 警告: この式の結果の型は 'int' で、暗黙的に無視されます。

// バージョン2
let f =
  for i in [1..10] do
    i + i |> ignore   // 修正
```

### ワンライナーのforループ

ループが実際に使用される場所の1つは、リストおよびシーケンスのジェネレータです。

```fsharp
let myList = [for x in 0..100 do if x*x < 100 then yield x ]
```

## まとめ

この記事の冒頭で述べたことを繰り返します: 関数的な思考を学ぶ際は、命令的な制御フローを使わないようにしましょう。
そのルールを裏付ける例外を理解できたら;ワンライナーは使用してかまいません。
