---
layout: post
title: "Match expressions"
description: "The workhorse of F#"
date: 2012-06-28
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 9
categories: [Patterns,Folds]
---

F#にはパターンマッチングがいたるところにあります。`let`式で値を束縛するとき、関数パラメータ内で値を束縛するとき、そして`match..with`構文を使って分岐するときなどです。

["Why use F#?"シリーズの記事](/posts/conciseness-pattern-matching) では、値を式に束縛する方法について簡単に説明しましたが、 [型を調べる](/posts/overview-of-types-in-fsharp) では何度も取り上げます。

今回の記事で、`match..with`構文、およびこの構文を使った制御フローについて説明します。

## match式とは?

`match..with`という表現はすでに何度も見てきました。これには次のような形があります:

    match [something] with
    | パターン1 -> 式1
    | パターン2 -> 式2
    | パターン3 -> 式3

目を細めてみると、ラムダ式を並べたような形になっています。

    match [something] with
    | ラムダ式-1
    | ラムダ式-2
    | ラムダ式-3

各ラムダ式にはパラメータが1つだけあります:

    param -> 式

つまり、'match..with'はラムダ式のセットから1つを選択していると考えることができます。でもどうやって選んでいるでしょう?

ここでパターンが登場します。"match with"の値がラムダ式のパラメータと一致するかどうかに基づいて選択されます。
パラメータと入力値が一致した最初のラムダが"選ばれます"！

たとえば、パラメータがワイルドカード`_`である場合は常に一致し、先頭にある場合は常に優先されます。

    _ -> expression

### 順番が重要！

次の例を見てみましょう。

```fsharp
let x =
    match 1 with
    | 1 -> "a"
    | 2 -> "b"
    | _ -> "z"
```

マッチするラムダ式は3つあり、この順番であることがわかります:

    fun 1 -> "a"
    fun 2 -> "b"
    fun _ -> "z"

つまり、`1`のパターンが最初に試され、次に`2`のパターン、最後に`_`のパターンが試されるのです。

一方、ワイルドカードを先頭に置くように順序を変更すると、ワイルドカードが最初に試され、常にすぐに勝利します:

```fsharp
let x =
    match 1 with
    | _ -> "z"
    | 1 -> "a"
    | 2 -> "b"
```

この場合、F#コンパイラは、他のルールは決してマッチしないことを親切に警告してくれます。

これが、"`switch`"や"`case`"と"`match..with`"の大きな違いのひとつです。`match...with`では、**順番が重要**です。

## match式の書式設定

F#はインデントに敏感なので、この多くの可動部分がある式はどのようにフォーマットすればよいのか、疑問に思うかもしれません。

[post on F#syntax](/posts/fsharp-syntax) では、アラインメントの仕組みの概要を説明していますが、`match..with`式については、いくつかの特別なルールがあります。

**ガイドライン1: `| expression`のアライメントは、`match`の直下でなければならない**。

このガイドラインは単純明快です。

```fsharp
let f x = match x with
            // 揃える
            | 1 -> "パターン1"
            // 揃える
            | 2 -> "パターン2"
            // 揃える
            | _ -> "エブリシング"
```


**ガイドライン2: `match...with` は改行してください**。

`match..with`は同一行でも改行でも構いませんが、改行することで、名前の長さに関係なく、インデントの一貫性を保つことができます。

```fsharp
                                              // 醜いアライメント!
let myVeryLongNameForAFunction myParameter = match myParameter with
                                              | 1 -> "something"
                                              | _ -> "anything"

// ずっといい
let myVeryLongNameForAFunction myParameter =
    match myParameter with
    | 1 -> "something"
    | _ -> "anything"
```

**ガイドライン3: 矢印 `->` の後の式は、改行してください**。

この場合も、結果の式を矢印と同じ行に置くことができますが、新しい行を使用するとインデントの一貫性が保たれ、
パターンマッチを結果式と区別するのに役立ちます。

```fsharp
let f x =
    match x with
    | "a very long pattern that breaks up the flow" -> "something"
    | _ -> "anything"

let f x =
    match x with
    | "a very long pattern that breaks up the flow" ->
        "something"
    | _ ->
        "anything"
```

もちろん，すべてのパターンが非常にコンパクトである場合には，常識的な例外を作ることができます．

```fsharp
let f list =
    match list with
    | [] -> "something"
    | x::xs -> "something else"
```


## match..withは式である

`match.with`が実際には "制御フロー"構文ではないことを認識することが重要です。"制御"は分岐を"流れる"のではなく、他の式と同様に、ある時点で評価される式です。実際の結果は同じかもしれませんが、重要なのは概念的な違いです。

式であることの帰結として、すべての分岐は*同じ*型に評価*されなくてはならない*ということがあります--if-then-else式とforループではすでに同じ挙動を見てきました。

```fsharp
let x =
    match 1 with
    | 1 -> 42
    | 2 -> true  // エラー 間違った型
    | _ -> "hello" // エラー 間違った型
```

式内で型を混在させてマッチングすることはできません。

{{< book_page_pdf >}}

### match式はどこでも使用できる

match式は正規の式であるため、式を使用できる場所であればどこでも使えます。

たとえば、次のようなネストされた一致式があります:

```fsharp
// 入れ子のmatch..withsでもOK
let f aValue =
    match aValue with
    | x ->
        match x with
        | _ -> "something"
```

そして，これはラムダに埋め込まれたmatch式です:

```fsharp
[2..10]
|> List.map (fun i ->
        match i with
        | 2 | 3 | 5 | 7 -> sprintf "%i is prime" i
        | _ -> sprintf "%i is not prime" i
        )
```



## 網羅的マッチング

式であることのもう1つの利点は、必ず一致する*何らかの*ブランチが存在する必要があることです。式全体が*何か*に評価される必要があります!

つまり、"網羅的マッチング"という価値ある概念は、F#の"すべてが式である"という性質に基づくものです。文指向言語には、このような要求はありません。

不完全なマッチングの例を次に示します:

```fsharp
let x =
    マッチ 42 with
    | 1 -> "a"
    | 2 -> "b"
```

コンパイラは分岐が欠けていると思うと警告を出します。
そして、この警告を意図的に無視すると、どのパターンにもマッチしなかったときに、厄介なランタイムエラー(`MatchFailureException`)が発生します。

### 網羅的マッチングは完璧ではない

可能性のあるすべてのマッチがリストアップされているかどうかをチェックするアルゴリズムは 優れていますが、常に完璧というわけではありません。場合によっては、可能性のあるすべてのケースにマッチしていることがわかっているにもかかわらず、マッチしていないと警告されることがあります。
このような場合、コンパイラを満足させるために余分なケースを追加する必要があるかもしれません。

### ワイルドカードマッチの使用 (および回避)

常にすべてのケースにマッチすることを保証する方法として、ワイルドカードパラメータを最後にマッチさせる方法があります。

```fsharp
レット x =
    42にマッチ
    | 1 -> "a"
    | 2 -> "b"
    | _ -> "z"
```

このパターンはよく見かけますし、ここでの例でもよく使っています。これは、switch文の中に全てを拾う`default`があるのと同じことです。

しかし、包括的なパターンマッチの利点を最大限に享受したいのであれば、ワイルドカードを使用*しない*ことをお勧めします。
可能であれば、すべてのケースを明示的にマッチさせるようにしてください。これは特に、
判別共用体のケースを照合する場合に当てはまります:

```fsharp
type Choices = A | B | C
let x =
    match A with
    | A -> "a"
    | B -> "b"
    | C -> "c"
    //デフォルトマッチなし
```

このように常に明示的にすることで、共用体に新しいケースを追加することで発生するエラーを捕捉できます。一致するワイルドカードがあると、何もわかりません。

*すべての*ケースを明確にできない場合は、可能な限り境界条件を文書化し、ワイルドカードのケースで実行時エラーをアサートした方がよいでしょう。

```fsharp
let x =
    match -1 with
    | 1 -> "a"
    | 2 -> "b"
    | i when i >= 0 && i<=100 -> "ok"
    // 最後のケースは常にマッチする
    | x -> failwithf "%i is out of range" x
```


## パターンの種類

パターンのマッチングにはさまざまな方法がありますが、それについては次に説明します。

各種パターンの詳細については、[MSDNドキュメント](http://msdn.microsoft.com/en-us/library/dd547125%28v=vs.110%29)を参照してください。

### 値への束縛

 基本パターンは、マッチングの中で値に束縛することです:

```fsharp
let y=
    match (1,0) with
    //指定した値に束縛する
    | (1, x) ->printfn "x=%A"x
```

*ところで、私は意図的にこのパターン (およびこの記事の他のパターン) は不完全なままにしています。練習として、ワイルドカードを使用せずに完全なものにしてみましょう。*

*束縛される*値は、各パターンで異なる*必要がある*ことに注意してください。こんなことはできません:

```fsharp
let elementsAreEqual aTuple =
    match aTuple with
    | (x,x) ->
        printfn "both parts are the same"
    | (_,_) ->
        printfn "both parts are different"
```

この場合は、次のようにします:

```fsharp
let elementsAreEqual aTuple =
    match aTuple with
    | (x,y) ->
        if (x=y) then printfn "both parts are the same"
        else printfn "both parts are different"
```

この2番目の方法は、"ガード" (`when`句) を使って書き換えることもできます。ガードについては後ほど説明します。

### AND と OR

ORとANDを使用して、1行に複数のパターンを組み合わせることができます:

```fsharp
let y =
    match (1,0) with
    // OR -- 1行に複数の場合と同じ
    | (2,x) | (3,x) | (4,x) -> printfn "x=%A" x

    // AND -- 一度に両方のパターンにマッチしなければならない
	// "&"は1つしか使わないことに注意
    | (2,x) & (_,1) -> printfn "x=%A" x
```

ORは，多数のユニオンケースをマッチさせる場合に特によく使われます。

```fsharp
type Choices = A | B | C | D
let x =
    match A with
    | A | B | C -> "a or b or c"
    | D -> "d"
```


### リストのマッチング

リストは `[x;y;z]` という形式で明示的にマッチさせることもできますし、 `head::tail` という "cons" 形式でマッチさせることもできます:

```fsharp
let y =
    match [1;2;3] with
    // 明示的な位置への束縛
    // 角括弧を使う!
    | [1;x;y] -> printfn "x=%A y=%A" x y

    // ヘッド::テイルへの束縛。
    // 角括弧は使わない!
    | 1::tail -> printfn "tail=%A" tail

    // 空のリスト
    | [] -> printfn "empty"
```

同様の構文で、配列を正確に `[|x;y;z|]` と一致させることができます。

シーケンス (別名`IEnumerables`) は "遅延評価"であり、一度に1つの要素だけにアクセスするように設計されているため、この方法で直接マッチすることは*できない*ことを理解することが重要です。
これに対し、リストと配列はマッチさせることができます。

これらのパターンの中で最も一般的なものは"cons"パターンで、リストの要素をループするために再帰と組み合わせてよく使われます。

再帰を使用したリストのループの例を次に示します:

```fsharp
// リストをループして値を表示する
let rec loopAndPrint aList =
    match aList with
    // 空のリストは終了を意味します。
    | [] ->
        printfn "empty"

    // head::tailに束縛します。
    | x::xs ->
        printfn "element=%A," x
        // 残りのリストについても同じことを繰り返します。
        // リストの残りの部分で再び行う
        loopAndPrint xs

//テスト
loopAndPrint [1..5]

// ------------------------
// リストをループして値を合計する
let rec loopAndSum aList sumSoFar =
    match aList with
    // 空のリストは終了を意味します。
    | [] ->
        sumSoFar

    // head::tailに束縛します。
    | x::xs ->
        let newSumSoFar = sumSoFar + x
        // リストの残りの部分と新しい合計を使って
        // 再帰処理します。
        loopAndSum xs newSumSoFar

//テスト
loopAndSum [1..5] 0
```

2番目の例は、特殊"アキュムレータ"パラメータ (ここでは`sumSoFar`) を使用して、次のループへ状態を渡す方法を示しています。これはよくあるパターンです。

### タプル、レコード、判別共用体のマッチング

パターンマッチングは、すべての組み込みF#型で使用できます。詳細は、 [series on types](/posts/overview-of-types-in-fsharp) を参照してください。

```fsharp
// -----------------------
// Tuple pattern matching
let aTuple = (1,2)
match aTuple with
| (1,_) -> printfn "first part is 1"
| (_,2) -> printfn "second part is 2"


// -----------------------
// Record pattern matching
type Person = {First:string; Last:string}
let person = {First="john"; Last="doe"}
match person with
| {First="john"}  -> printfn "Matched John"
| _  -> printfn "Not John"

// -----------------------
// Union pattern matching
type IntOrBool= I of int | B of bool
let intOrBool = I 42
match intOrBool with
| I i  -> printfn "Int=%i" i
| B b  -> printfn "Bool=%b" b
```


### "as"句で全体と一部をマッチングする

場合によっては、全体*と*個々の要素を一致させたいこともあります。そのために`as`句を使用できます。

```fsharp
let y =
    match (1,0) with
    // 3つの値に束縛
    | (x,y) as t ->
        printfn "x=%A and y=%A" x y
        printfn "The whole tuple is %A" t
```


### 派生型のマッチング

派生型をマッチングするには、`:?`演算子を使用すると、簡単なポリモーフィズムがを実現できます。

```fsharp
let x = new Object()
let y =
    match x with
    | :? System.Int32 ->
        printfn "matched an int"
    | :? System.DateTime ->
        printfn "matched a datetime"
    | _ ->
        printfn "another type"
```

これは、上位クラス (この場合はObject) の派生型を検出する場合にのみ機能します。式全体の型は、入力として親クラスを持ちます。

場合によっては、値を"box"する必要があります。

```fsharp
let detectType v =
    match v with
        | :? int -> printfn "this is an int"
        | _ -> printfn "something else"
//エラーFS 0008:型'aからintへのこの実行時強制または型テスト
//このプログラムポイントより前の情報に基づいた不確定タイプが含まれています。
//ランタイムタイプテストは一部のタイプでは許可されていません。さらに型アノテーションが必要です。
```

"ランタイム型テストが許可されていない型もあります。"とメッセージが表示されます。
その答えは、値を参照型へ強制的に”box”することです。こうすると、以下のように型テストできます。

```fsharp
let detectTypeBoxed v =
    match box v with      // "box v"を使う
        | :? int -> printfn "this is an int"
        | _ -> printfn "something else"

//テスト
detectTypeBoxed 1
detectTypeBoxed 3.14
```

個人的には、型の一致やディスパッチは、オブジェクト指向プログラミングの場合と同様、コードの臭いだと思います。
必要な場合もありますが、安易に使用されている場合、設計が不十分であることを示します。

優れたオブジェクト指向設計では、 [polymorphism to replace subtype tests](http://sourcemaking.com/refactoring/replace-conditional-with-polymorphism),を [double dispatch] (http://www.c2.com/cgi/wiki?DoubleDispatchExample) などの手法と共に使用するのが正しい方法です。F#でこのようなオブジェクト指向をするなら、おそらく同じテクニックを使うべきでしょう。

## 複数値のマッチング

これまで見てきたパターンはすべて、*1つ*の値に対してパターンマッチングを行っています。では、複数の値に対してはどうすればよいのでしょうか？

簡単に言うと、できません。マッチは単一の値に対してのみ許されます。

しかし、ちょっと待ってください。2つの値をその場でタプルにして、それにマッチさせることはできるでしょうか？はい、できます。

```fsharp
let matchOnTwoParameters x y =
    match (x,y) with
    | (1,y) ->
        printfn "x=1 and y=%A" y
    | (x,1) ->
        printfn "x=%A and y=1" x
```

実際，このトリックは，複数の値をマッチさせたいときにはいつでも使えます．

```fsharp
let matchOnTwoTuples x y =
    match (x,y) with
    | (1,_),(1,_) -> "both start with 1"
    | (_,2),(_,2) -> "both end with 2"
    | _ -> "something else"

// テスト
matchOnTwoTuples (1,3) (1,2)
matchOnTwoTuples (3,2) (1,2)
```

## ガード、または"when"節

次の例のように，パターンマッチだけでは不十分な場合があります．

```fsharp
let elementsAreEqual aTuple =
    match aTuple with
    | (x,y) ->
        if (x=y) then printfn "both parts are the same"
        else printfn "both parts are different"
```

パターンマッチングはパターンのみに基づいて行われるため、関数やその他の条件テストは使用できません。

しかし、パターンマッチの一部として等価性テストをする方法は*あります* -- -関数矢印の左側に`when`句を追加します。
この句は"ガード"と呼ばれます。

これは、ガードを使用して記述された同じロジックです:

```fsharp
let elementsAreEqual aTuple =
    match aTuple with
    | (x,y) when x=y ->
        printfn "both parts are the same"
    | _ ->
        printfn "both parts are different"
```

これは、マッチングが完了した後に検査するのではなくパターン自体に統合されているため、より適切です。

ガードは、純粋なパターンでは使用できないあらゆる種類の用途に使用できます。

* 境界値の比較
* オブジェクトプロパティの検証
* 正規表現など、他の種類の照合の実行
* 関数由来の述語

これらの例をいくつか見てみましょう:

```fsharp
// --------------------------------
// when節での値の比較
let makeOrdered aTuple =
    match aTuple with
    // xがyより大きければ入れ替える
    | (x,y) when x > y -> (y,x)

    // そうでなければそのまま
    | _ -> aTuple

//テスト
makeOrdered (1,2)
makeOrdered (2,1)

// --------------------------------
// when節でのプロパティのテスト
let isAM aDate =
    match aDate:System.DateTime with
    | x when x.Hour <= 12->
        printfn "AM"

    // そうでなければ、そのままにする
    | _ ->
        printfn "PM"

//テスト
isAM System.DateTime.Now

// --------------------------------
// 正規表現を使ったパターンマッチング
open System.Text.RegularExpressions

let classifyString aString =
    match aString with
    | x when Regex.Match(x,@".+@.+").Success->
        printfn "%s is an email" aString

    // そうでなければ、そのままにする
    | _ ->
        printfn "%s is something else" aString


//テスト
classifyString "alice@example.com"
classifyString "google.com"

// --------------------------------
// 任意の条件式を用いたパターンマッチング
let fizzBuzz x =
    match x with
    | i when i % 15 = 0 ->
        printfn "fizzbuzz"
    | i when i % 3 = 0 ->
        printfn "fizz"
    | i when i % 5 = 0 ->
        printfn "buzz"
    | i  ->
        printfn "%i" i

//テスト
[1..30] |> List.iter fizzBuzz
```

### ガードの代わりにアクティブパターンを使う

ガードは単発のマッチには最適です。しかし、何度も使うガードがある場合は、代わりにアクティブパターンを使うことを検討してください。

例えば、上のメールの例は次のように書き換えられます。

```fsharp
open System.Text.RegularExpressions

// 電子メールアドレスにマッチするアクティブパターンを作成する
let (|EmailAddress|_|) input =
   let m = Regex.Match(input,@".+@.+")
   if (m.Success) then Some input else None

// アクティブなパターンをマッチで使用
let classifyString aString =
    match aString with
    | EmailAddress x ->
        printfn "%s is an email" x

    // そうでなければ、放っておく
    | _ ->
        printfn "%s is something else" aString

//テスト
classifyString "alice@example.com"
classifyString "google.com"
```

アクティブパターンの他の例は、[過去の投稿](/posts/convenience-active-patterns)で見ることができます。

## "function" 句

これまでの例では、次のようなものが多く見られました。

```fsharp
let f aValue =
    match aValue with
    | _ -> "something"
```

関数の定義という特殊なケースでは，`function`句を使うことで，これを劇的に単純化することができます．

```fsharp
let f =
    function
    | _ -> "something"
```

ご覧のように、`aValue`というパラメータは、`match...with`とともに完全に消えています。

この句は、標準的なラムダの `fun` 句とは *違う* もので、むしろ `fun` と `match..with` を組み合わせたものです。

`function`句は、入れ子のマッチングのように、関数定義やラムダが使用できる場所ならどこでも使えます:

```fsharp
// match...withを使う
let f aValue =
    aValueにマッチするのは
    | x ->
        xと一致
        | _ -> "何か"

// function句を使用
let f =
    function
    | x ->
        function
        | _ -> "something"
```

または、高次の関数に渡されるラムダで:

```fsharp
// match.withを使う
[2..10] |> List.map (fun i ->
        match i with
        | 2 | 3 | 5 | 7 -> sprintf "%i is prime" i
        | _ -> sprintf "%i is not prime" i
        )

// function句を使用
[2..10] |> List.map (function
        | 2 | 3 | 5 | 7 -> sprintf "prime"
        | _ -> sprintf "not prime"
        )
```

`match..with`に比べて`function`のちょっとした欠点は、元の入力値を見ることができず、パターンの中の値の束縛に頼らなければならないことです。

## try..withによる例外処理

[前回の記事](/posts/exceptions)では、`try..with`式で例外をキャッチする方法を見てみました。

```fsharp
try
    failwith "fail"
with
    | Failure msg -> "caught: " + msg
    | :? System.InvalidOperationException as ex -> "unexpected"
```

`try..with`式は、`match..with`と同じようにパターンマッチを実装しています。

上の例では、カスタムパターンでのマッチングを行っています。

* `| Failure msg` は、アクティブなパターン（のように見えるもの）に対するマッチングの例です。
* `| :? System.IvalidOperationException as ex` は派生型にマッチする例です（`as` も使用しています）。

`try..with`式は完全なパターンマッチングを実装しているため、必要に応じてガードも使用し、追加の条件ロジックを追加することができます。

```fsharp
let debugMode = false
try
    failwith "fail"
with
    | Failure msg when debugMode  ->
        reraise()
    | Failure msg when not debugMode ->
        printfn "silently logged in production: %s" msg
```


## match式を関数でラップする

match式は非常に便利ですが、注意して使用しないとコードが複雑になる可能性があります。

最大の問題は、マッチ式の構文があまり良くないことです。要するに、`match.with`式を連鎖させ、単純な式を複雑な式へと構築することは困難です。

これを解決する最善の方法は、`match..with`式を関数にラップすることです。

簡単な例を示します。`match x with 42`は`isAnswerToEverything`関数でラップされています。

```fsharp
let times6 x = x * 6

let isAnswerToEverything x =.
    match x with
    | 42 -> (x,true)
    | _ -> (x,false)

// この関数は、連鎖や合成に使用できます。
[1..10] |> List.map (times6 >> isAnswerToEverything)
```

### 特定のマッチングに代わるライブラリ関数

ほとんどの組み込みF#型にはこうした関数が既に用意されています。

たとえば、再帰を使用してリストをループするのではなく、`List`モジュール内の関数を使用するようにしてください、必要な操作のほとんどはこれで実行できます。

具体的には、前に記述した関数です:

```fsharp
let rec loopAndSum aList sumSoFar =
    match aList with
    | [] ->
        sumSoFar
    | x::xs ->
        let newSumSoFar = sumSoFar + x
        loopAndSum xs newSumSoFar
```

`List`モジュールを用いて少なくとも3通りの方法で書き直すことができます!

```fsharp
// 最も単純な方法
let loopAndSum1 aList = List.sum aList
[1..10] |> loopAndSum1

// reduceは非常に強力です
let loopAndSum2 aList = List.reduce (+) aList
[1..10] |> loopAndSum2

// foldは最も強力である
let loopAndSum3 aList = List.fold (fun sum i -> sum+i) 0 aList
[1..10] |> loopAndSum3
```

同様に，Option型（[this post](/posts/the-option-type)で詳しく説明しています）には，多くの便利な関数を持つ`Option`モジュールが関連付けられています。

例えば，`Some`と`None`のマッチを行う関数は、`Option.map`で置き換えることができます．

```fsharp
// これを明示的に実装する必要はない
let addOneIfValid optionalInt =
    match optionalInt with
    | Some i -> Some (i + 1)
    | None -> None

Some 42 |> addOneIfValid

// 内蔵の関数を使う方がはるかに簡単です。
let addOneIfValid2 optionalInt =
    optionalInt |> Option.map (fun i->i+1)

Some 42 |> addOneIfValid2
```

{{< linktarget "folds" >}}

### マッチングロジックを隠すための"fold"関数の作成

最後に、頻繁に一致させる必要がある独自の型を作成する場合は、
適切にラッピングされた、汎用的な "fold"関数を作成することを
お勧めします。

たとえば、温度を定義する型があるとします。

```fsharp
type TemperatureType  = F of float | C of float
```

こういった場合、おそらく何度もマッチングすることになるので、そのための汎用関数を作成しましょう。

```fsharp
module Temperature =
    let fold fahrenheitFunction celsiusFunction aTemp =
        match aTemp with
        | F f -> fahrenheitFunction f
        | C c -> celsiusFunction c
```

すべての`fold`関数は、以下のような共通パターンになります:

* 判別共用体 (またはパターンマッチの句) の各ケースに1つの関数があります。
* 実際に照合する値は末尾になります。(なぜ? ["部分適用のための関数設計"](/posts/partial-application) の記事を参照してください)

fold関数ができたので、これをさまざまな状況で利用できます。

まずは発熱検査から始めましょう。華氏で熱を測る機能と、摂氏で熱を測る機能が必要です。

そして、fold関数を使用して両方を合成します。

```fsharp
let fFever tempF =
    if tempF > 100.0 then "Fever!" else "OK"

let cFever tempC =
    if tempC > 38.0 then "Fever!" else "OK"

// 畳み込みを使った組み合わせ
let isFever aTemp = Temperature.fold fFever cFever aTemp
```

そして、これでテストができます。

```fsharp
let normalTemp = C 37.0
let result1 = isFever normalTemp

let highTemp = F 103.1
let result2 = isFever highTemp
```

全く別の用途として、温度変換ユーティリティを作成しましょう。

ここでも、各ケースの関数を作成し、それらを合成します。

```fsharp
let fConversion tempF =
    let convertedValue = (tempF - 32.0) / 1.8
    TemperatureType.C convertedValue    //型でラッピング

let cConversion tempC =
    let convertedValue = (tempC * 1.8) + 32.0
    TemperatureType.F convertedValue    //型でラッピング

// foldを使った束縛
let convert aTemp = Temperature.fold fConversion cConversion aTemp
```

Note: 変換関数は変換された値を新たに`TemperatureType`にラップするので、`convert`関数のシグネチャは次のようになります。

```fsharp
val convert : TemperatureType -> TemperatureType
```

これで，テストができるようになりました。

```fsharp
let c20 = C 20.0
let resultInF = convert c20

let f75 = F 75.0
let resultInC = convert f75
```

convertを2回連続して呼び出すこともでき、最初に設定したのと同じ温度を得ることができます。

```fsharp
let resultInC = C 20.0 |> convert |> convert
```


foldについては、再帰、および再帰型に関する今後のシリーズでさらに詳しく説明します。