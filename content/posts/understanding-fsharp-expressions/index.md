---
layout: post
title: "Overview of F# expressions"
description: "Control flows, lets, dos, and more"
date: 2012-05-17
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 3
---

この記事では、F#で使用できるさまざまな種類の式と、それらを使用するための一般的なヒントを見ていきます。

## 本当にすべてが式なのか?

“すべてが式である”と言うが、実際どうなのか疑問に思うかもしれません。

まずは、おなじみの基本的な式の例を見てみましょう。

```fsharp
1                            // リテラル
[1;2;3]                      // リスト式
-2                           // 接頭辞演算子
2 + 2                        // 接頭辞演算子
"string".Length              // ドットルックアップ
printf "hello"               // 関数の適用
```

特に問題はありません。これらは明らかに式です。

しかし、より複雑なものがあり、それらも*やはり*式です。つまり、これらはそれぞれ、別の目的に利用できる値を返します。

```fsharp
fun () -> 1                  //ラムダ式

match 1 with                 //match式
    | 1 -> "a"
    | _ -> "b"

if true then "a" else "b"    // if-then-else

for i in [1..10]             //forループ
  do printf "%i" i

try                          //例外処理
  let result = 1 / 0
  printfn "%i" result
with
   | e ->
     printfn "%s" e.Message


let n=1 in n+2               // let式
```

他の言語では、これらは文であるかもしれませんが、F#では実際に値を返します、これらの結果を値に束縛することで確認できます:

```fsharp
let x1 = fun () -> 1

let x2 = match 1 with
         | 1 -> "a"
         | _ -> "b"

let x3 = if true then "a" else "b"

let x4 = for i in [1..10]
          do printf "%i" i

let x5 = try
            let result = 1 / 0
            printfn "%i" result
         with
            | e ->
                printfn "%s" e.Message


let x6 = let n=1 in n+2
```

## どんな種類の式があるの？

F#には様々な種類の式があり、現在約50種類あります。 リテラル、演算子、関数の適用、"dotting into"など、ほとんどのものは些細で明白です。

より重要で高レベルなものは、次のように分類できます:

* ラムダ式
* 次のような"制御フロー"式
  * マッチ式 (`match..with`構文を使用)
  * if-then-else, ループなどの命令制御フローに関連する式
  * 例外に関連する式
* "let"と"use"式
* `async{...}`のようなコンピュテーション式
* キャスト、インターフェイスなど、オブジェクト指向のコードに関連する式

ラムダについては、すでに ["thinking functionally"](/series/thinking-function.html) シリーズで説明しましたが、前述のように、コンピュテーション式とオブジェクト指向の式については今後のシリーズで扱います。

このシリーズの今後の記事では、 "制御フロー"式と"let"式に焦点を当てます。

### "制御フロー"式

命令型言語では、if-then-else、for-in-do、match-withなどの制御フロー式は、通常、副作用を持つ文として実装されます。F#では、これらはすべて単なる式として実装されます。

事実、 "制御フロー"を関数型言語で考えるのは無意味ですし、その概念は実際には存在しません。プログラムを、評価されるものと評価されないものがある部分式を含む巨大な式と考える方がよいでしょう。この考え方を頭で理解することができれば、関数型思考をはじめることができます。

これらの異なるタイプの制御フロー式については、今後いくつかの投稿を予定しています。

* [マッチ式](/posts/match-expression)
* [制御フロー: if-then-elseとforループ](/posts/control-flow-expression)
* [例外](/posts/exceptions)

### 式としての"let"束縛

`let x=something`とは?これまでの例で、次のことを確認しました:

```fsharp
let x6 = let n=1 in n+2
```

どうすれば"`let`"が式になるのでしょうか?その背景については、次の ["let","use",および"do"](/posts/let-use-do) の記事で説明します。

## 式を使う際の主なコツ

ここでは、重要な式の型について詳しく説明する前に、式の使用に関する一般的なコツをいくつか紹介します。

### 1行に複数の式を記述する

通常、各式は改行されます。ただし必要に応じて、セミコロンを使用して1行に複数の式を記述することもできます。これは、リスト要素とレコード要素の区切り文字以外で、F#でセミコロンが使用される数少ない例の1つです。

```fsharp
let f x = // 1行に1つの式
      printfn "x=%i" x
      x + 1

let f x = printfn "x=%i" x; x + 1 // すべての式を"; "を使って同じ行に書く
```

もちろん、直前の式に対してunit値が要求されるという原則は適用されます:

```fsharp
let x = 1;2              // エラーです。"1; "は単位式でなければならない
let x = ignore 1;2       // OK
let x = printf "hello";2 // OK
```

### 式の評価順序を理解する

F#では、式は"内側から外側へ"評価されます--つまり、部分式が完成したように"見えた"時点で評価されます。

次のコードを見て、何が起こるか推測してから、そのコードを評価してみてください。

```fsharp
// if-then-elseのクローンを作る。
let test b t f = if b then t else f

// 2つの異なる選択肢でこれを呼び出す
test true (printfn "true") (printfn "false")
```

なんとtest関数は実際には"else"分岐を評価しないにもかかわらず、"true"と"false"の両方が出力されました。どうしてでしょうか?その理由は`(printfn "false")`式は、test関数がこの式をどのように使うかにかかわらず、即時評価されるためです。

これを"eager"(先行評価)と呼びます。分かりやすいという利点もありますが、場合によっては非効率になることもあります。

別の評価方法は"lazy"(遅延評価)と呼ばれ、式は必要なときにのみ評価されます。Haskellはこのアプローチに従っているので、先ほどのコード例を行うと"true"のみが出力されます。

F#には、式を即時評価*しない*ためのテクニックが数多くあります。最も簡単な方法は、必要に応じて評価される関数にラップすることです。

```fsharp
// 単純な値ではなく関数を受け取るif-then-elseのクローンを作成します。
let test b t f = if b then t() else f()

// 2つの異なる関数で呼び出す
test true (fun () -> printfn "true") (fun () -> printfn "false")
```

この場合の欠点は、 "true"関数が誤って2回評価される可能性があることです。

そこで、式が即時評価されないようにしたい場合は、`Lazy<>`というラッパーを使用することをお勧めします。

```fsharp
// 制限のないif-then-elseのクローンを作る...
let test b t f = if b then t else f

// ...ただし、遅延値で呼び出す
let f = test true (lazy (printfn "true")) (lazy (printfn "false"))
```

最終的な結果の値`f`も遅延値であり、最終的に結果を得る準備ができるまで評価せずに渡すことができます。

```fsharp
f.Force()     // Force()を使って、遅延値を強制的に評価します。
```

もし、結果が必要なく、`Force()`を呼ばなければ、ラップされた値が評価されることはありません。

遅延については、パフォーマンスに関する次のシリーズでもっと詳しく説明します。
