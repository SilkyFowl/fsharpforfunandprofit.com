---
layout: post
title: "Worked example: Roman numerals"
description: "More pattern matching in practice"
date: 2012-06-30
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 12
categories: [Patterns, Worked Examples]
---

[前回](/posts/pattern-matching-command-line)では、コマンドラインの解析を見ました。今回は、ローマ数字を使ったパターンマッチングの例を見てみましょう。

前回と同様に、入力を内部モデルに変換する段階と、内部モデルから出力に変換する段階を分けて、「純粋な」内部モデルにしてみます。

![](../pattern-matching-command-line/function_transform2.png)

## 要件

まず、要件から説明します。

	1) "MMMXCLXXIV "のような文字列を文字列として受け取り、それを整数に変換する。
	変換の仕方は I=1、V=5、X=10、L=50、C=100、D=500、M=1000である。

	下位の文字が上位の文字の前に来ると、それに応じて上位の文字の値が減るので、次のようになります。
	IV=4、IX=9、XC=90といった具合です。

	2) さらに、その文字列が有効な数字であるかどうかを検証する。例えば、以下のようになります。"IIVVMM "は有効なローマ数字ではありません。


## 最初のバージョン

前回と同様に、まず内部モデルを作成し、次に内部モデルへの入力をどのように解析するかを見ていきます。

ここでは、モデルの最初の試みを行います。ここでは、`RomanNumeral`を`RomanDigits`のリストとして扱います。

```fsharp
type RomanDigit = int
type RomanNumeral = RomanDigit list
```

いや、ここでストップ! RomanDigit`はただの数字ではなく、限られたセットの中から選ばれなければなりません。

また、`RomanNumeral`は単に数字のリストの[type alias](/posts/type-abbreviations)であってはいけません。独自の特別なタイプであるほうがいいでしょう。
これは[single case union type](/posts/discriminated-unions)を作ることで実現できます。

もっと良いバージョンです:

```fsharp
type RomanDigit = I | V | X | L | C | D | M
type RomanNumeral = RomanNumeral of RomanDigit list
```

### 出力。数字をint型に変換する

それでは、ローマ数字をint型に変換する出力ロジックを行ってみましょう。

数字の変換は簡単です。

```fsharp
/// 1つのローマ数字を整数に変換する
let digitToInt =
    function
    | I -> 1
    | V -> 5
    | X -> 10
    | L -> 50
    | C -> 100
    | D -> 500
    | M -> 1000

// tests
I  |> digitToInt
V  |> digitToInt
M  |> digitToInt
```

ここでは `match..with` 式の代わりに `function` キーワードを使っていることに注意してください。

数字のリストを変換するには、再び再帰的なループを使います。
特殊なケースとして，次の桁を先読みして，それが現在の桁よりも大きければ，その差を利用する必要があります。

```fsharp
let rec digitsToInt =
    function

    // 空は0
    | [] -> 0

    // 小さい方が大きい方の前に来る場合の特殊なケース
    // 両方の桁を変換し、その差を和に加える
	// 例 "IV "と "CM"
    | smaller::larger::ns when smaller < larger ->
        (digitToInt larger - digitToInt smaller)  + digitsToInt ns

    // そうでない場合は、数字を変換して合計に加える
    | digit::ns ->
        digitToInt digit + digitsToInt ns

// tests
[I;I;I] |> digitsToInt
[I;V] |> digitsToInt
[V;I] |> digitsToInt
[I;X] |> digitsToInt
[M;C;M;L;X;X;I;X] |> digitsToInt // 1979
[M;C;M;X;L;I;V] |> digitsToInt // 1944
```

*less than"の演算は定義する必要がないことに注意してください。型は宣言順に自動的にソートされています。

最後に，`RomanNumeral`の中身をリストに展開して，`digitsToInt`を呼び出すことで，`RomanNumeral`の型自体を変換することができます。

```fsharp
/// RomanNumeralを整数に変換します。
let toInt (RomanNumeral digits) = digitsToInt digits

// テスト
let x = RomanNumeral [I;I;I;I]
x |> toInt

let x = RomanNumeral [M;C;M;L;X;X;I;X]
x |> toInt
```

これで出力は大丈夫です。

### 入力: 文字列をローマ数字に変換する

それでは、文字列を内部モデルに変換する入力ロジックを行ってみましょう。

まず、1文字の変換を行ってみます。簡単そうですね。

```fsharp
let charToRomanDigit =
    function
    | 'I' -> I
    | 'V' -> V
    | 'X' -> X
    | 'L' -> L
    | 'C' -> C
    | 'D' -> D
    | 'M' -> M
```

コンパイラはこれを嫌います。他の文字が出てきたらどうなるでしょう？

これは、[網羅的パターンマッチング](/posts/correctness-exhaustive-pattern-matching)によって、不足している要件について考えざるを得なくなるという素晴らしい例です。

では、悪い入力に対してはどうすればいいのか。エラーメッセージを表示するのはどうでしょうか？

もう一度、他の文字を処理するケースを追加してみましょう。

```fsharp
let charToRomanDigit =
    function
    | 'I' -> I
    | 'V' -> V
    | 'X' -> X
    | 'L' -> L
    | 'C' -> C
    | 'D' -> D
    | 'M' -> M
	| ch -> eprintf "%c is not a valid character" ch
```

コンパイラもこれも嫌がります！通常の場合は有効な `RomanDigit`を返しますが、エラーの場合は`unit`を返します。[以前の投稿](/posts/pattern-matching) で見たように、全ての分岐は*必ず*同じ型を返さなければなりません*。

どう解決すればいいのでしょうか?例外をスローすることもできますが、それは少々やりすぎです。よく考えてみると、`charToRomanDigit`は有効な`RomanDigit`を*常に*返すことはできません。
できることもあるし、できないこともある。つまり、option 型のようなものを利用する必要があります。

しかし、更に深く考えてみると、呼び出し側には不良文字が何であるかを知らせた方が良いかもしれません。そこで、どちらの場合にも対応できるように、独自のoption型のちょっとしたバリエーションを作る必要があるようです。

こちらが修正版です:

```fsharp
type ParsedChar =
    | Digit of RomanDigit
    | BadChar of char

let charToRomanDigit =
    function
    | 'I' -> Digit I
    | 'V' -> Digit V
    | 'X' -> Digit X
    | 'L' -> Digit L
    | 'C' -> Digit C
    | 'D' -> Digit D
    | 'M' -> Digit M
    | ch -> BadChar ch
```

Note: エラーメッセージが消えました。不正な文字が返される場合、呼び出し元は`BadChar`の場合に独自のメッセージを出力することもできます。

次に、関数シグネチャをチェックして、それが期待どおりであることを確認します:

```fsharp
charToRomanDigit : char -> ParsedChar
```

いい感じですね。

では、どうやって文字列を変換すればよいのでしょうか?まず、文字列をchar配列に変換し、それをリストに変換してから、最後に`charToRomanDigit`を使って変換します。

```fsharp
let toRomanDigitList s =
    s.ToCharArray() // error FS0072
    |> List.ofArray
    |> List.map charToRomanDigit
```

ところが、コンパイラが"FS0072:このプログラムの場所の前方にある情報に基づく不確定の型のオブジェクトに対する参照です"と再び文句を言います。

これは通常、関数ではなくメソッドを使用する場合に発生します。どのようなオブジェクトでも`.ToCharArray()`を使用するため、型推論ではどの型を意味するのかを判断できません。

この場合の解決策は、パラメータに明示的な型アノテーションを使用することです。 -- これは今回が初めてです!

```fsharp
let toRomanDigitList (s:string) =
    s.ToCharArray()
    |> List.ofArray
    |> List.map charToRomanDigit
```

しかし、シグネチャーを見てください。

```fsharp
toRomanDigitList : string -> ParsedChar list
```

まだ`RomanDigits`ではなく、面倒な`ParsedChar`が入っています。どのように処理を進めましょうか?そうです、もう一度責任を転嫁して、他の誰かに処理させましょう!

この場合の"責任転嫁"は、実際には優れた設計原則です。この関数はクライアントが何をしたいのかを知りません--エラーを無視したい人もいれば、ただちに終了させたい人もいるでしょう。だから、情報を返して、彼らに決めさせてください。

この場合、クライアントは`RomanNumber`型を作成するトップレベルの関数です。これが最初の挑戦です:

```fsharp
//文字列をRomanNumeralに変換する
let toRomanNumeral s =
    toRomanDigitList s
    |> RomanNumeral
```

コンパイラは満足していません--`RomanNumber`コンストラクタには`RomanDigits`のリストが必要ですが、`toRomanDigitList`にあるのは`ParsedChars`のリストです。

ここでようやくエラー処理の方針を確定する*必要があります*。不正な文字を無視し、エラーが発生した場合は出力するようにしましょう。これには、`List.choose`関数を使用します。
`List.map`に似ていますが、それに加えてフィルタが組み込まれています。有効な要素 (`Some something`) が返されますが、`None`の要素は除外されます。

ここでのchoose関数は次のようにします:

* 有効な桁数の場合は`Some digit`
* 無効な`BadChars`については、エラーメッセージを表示して`None`を返します。

これを行うと、`List.choose`の出力は、`RomanNumber`コンストラクタへの入力とまったく同じように、`RomanDigits`のリストになります。

これがすべてをまとめたものです:

```fsharp
/// 文字列をRomanNumeralに変換する
/// 例えば、"IVIV "は有効です。
let toRomanNumeral s =
    toRomanDigitList s
    |> List.choose (
        function
        | Digit digit ->
            Some digit
        | BadChar ch ->
            eprintfn "%c is not a valid character" ch
            None
        )
    |> RomanNumeral
```

テストしてみましょう。

```fsharp
// 良いケースをテストする

"IIII"  |> toRomanNumeral
"IV"  |> toRomanNumeral
"VI"  |> toRomanNumeral
"IX"  |> toRomanNumeral
"MCMLXXIX"  |> toRomanNumeral
"MCMXLIV" |> toRomanNumeral
"" |> toRomanNumeral

// エラーケース
"MC?I" |> toRomanNumeral
"abc" |> toRomanNumeral
```

OK、ここまではすべて順調です。それでは、入力規則に移りましょう。

### 入力規則

入力規則は要件に記載されていなかったので、ローマ数字についてわかっていることに基づいて、もっともらしい推測をしてみましょう。

* 1桁に5つ続けて入力することはできません
* 一部の数字は4文字まで連続して使用できます。I、X、C、およびMです。その他の数字 (V、L、D) は単独でのみ使用できます。
* 下位の桁の中には上位の桁の前に来るものがありますが、それは単一で表示される場合に限られます。例:「IX」は問題ありませんが、「IIIX」は無効です。
* ただし、これは2つの数字のペアにのみ適用されます。連続した3つの昇順の数字は無効です。例:「IX」は問題ありませんが、「IXC」は無効です。
* 数字1桁 (桁区切りなし) は常に許可されます。

これらの要件は、次のようなパターンマッチング関数に変換できます:

```fsharp
let runsAllowed =
    function
    | I | X | C | M -> true
    | V | L | D -> false

let noRunsAllowed  = runsAllowed >> not

// 妥当性のチェック
let rec isValidDigitList digitList =
    match digitList with

    // 空のリストは有効
    | [] -> true

    // 5つ以上の何かがある場合は無効
    // 例  XXXXX
    | d1::d2::d3::d4::d5::_
        when d1=d2 && d1=d3 && d1=d4 && d1=d5 ->
            false

    // 2桁以上のランナブルでない数字は無効
    // 例  VV
    | d1::d2::_
        when d1=d2 && noRunsAllowed d1 ->
            false

    // 中間の2,3,4のランは、次の桁が大きい場合は無効です。
    // 例  IIIX
    | d1::d2::d3::d4::higher::ds
        when d1=d2 && d1=d3 && d1=d4
        && runsAllowed d1 // マッチングの順番があるのであまり必要ない
        && higher > d1 ->
            false

    | d1::d2::d3::higher::ds
        when d1=d2 && d1=d3
        && runsAllowed d1
        && higher > d1 ->
            false

    | d1::d2::higher::ds
        when d1=d2
        && runsAllowed d1
        && higher > d1 ->
            false

    // 昇順の数字が3つ並んでいると無効
    // 例  IVX
    | d1::d2::d3::_  when d1<d2 && d2<= d3 ->
        false

    // ランのない1桁の数字は常に許される
    | _::ds ->
        // リストの残りの部分をチェックする
        isValidDigitList ds

```

*再度、「等しい」と「より小さい」は定義する必要がなかったことに注意してください*。

そして、検証をテストしてみましょう。

```fsharp
// バリデーションのテスト
let validList = [
    [I;I;I;I]
    [I;V]
    [I;X]
    [I;X;V]
    [V;X]
    [X;I;V]
    [X;I;X]
    [X;X;I;I]
    ]

let testValid = validList |> List.map isValidDigitList

let invalidList = [
    // どの桁でも5つ並べることはできない
    [I;I;I;I]
    // V,L,Dの2桁が連続してはならない
    [V;V]
    [L;L]
    [D;D]
    // 中間に2,3,4のランがあっても、次の桁が高ければ無効です。
    [I;I;V]
    [X;X;X;M]
    [C;C;C;C;D]
    // 昇順の数字が3つ並んでいる場合は無効
    [I;V;X]
    [X;L;D]
    ]
let testInvalid = invalidList |> List.map isValidDigitList
```

最後に、トップレベルの関数を追加して、`RomanNumeral`型自体の有効性をテストします。
```fsharp
// トップレベルの有効性のチェック
let isValid (RomanNumeral digitList) =
    isValidDigitList digitList


// 良いケースのテスト
"IIII"  |> toRomanNumeral |> isValid
"IV"  |> toRomanNumeral |> isValid
"" |> toRomanNumeral |> isValid

// エラーケース
"IIXX" |> toRomanNumeral |> isValid
"VV" |> toRomanNumeral |> isValid

// グランドフィナーレ
[ "IIII"; "XIV"; "MMDXC";
"IIXX"; "VV"; ]
|> List.map toRomanNumeral
|> List.iter (function
    | n when isValid n ->
        printfn "%A is valid and its integer value is %i" n (toInt n)
    | n ->
        printfn "%A is not valid" n
    )
```

## 最初のバージョンのコード全体

ここでは，すべてのコードを1つのモジュールにまとめています。

```fsharp
module RomanNumeralsV1 =

    // ==========================================
    // タイプ
    // ==========================================

    type RomanDigit = I | V | X | L | C | D | M
    type RomanNumeral = RomanNumeral of RomanDigit list

    // ==========================================
    // 出力ロジック
    // ==========================================

    /// 単一のRomanDigitを整数に変換する
    let digitToInt =
        function
        | I -> 1
        | V -> 5
        | X -> 10
        | L -> 50
        | C -> 100
        | D -> 500
        | M -> 1000

    /// 数字のリストを整数に変換する
    let rec digitsToInt =
        function

        // 空は0
        | [] -> 0

        // 小さい方が大きい方の前に来る場合の特殊なケース
        // 両方の桁を変換し、その差を和に加える
        // 例 "IV "と "CM"
        | smaller::larger::ns when smaller < larger ->
            (digitToInt larger - digitToInt smaller)  + digitsToInt ns

        // そうでない場合は、数字を変換して合計に加える
        | digit::ns ->
            digitToInt digit + digitsToInt ns

    /// RomanNumeralを整数に変換します。
    let toInt (RomanNumeral digits) = digitsToInt digits

    // ==========================================
    // 入力ロジック
    // ==========================================

    type ParsedChar =
        | Digit of RomanDigit
        | BadChar of char

    let charToRomanDigit =
        function
        | 'I' -> Digit I
        | 'V' -> Digit V
        | 'X' -> Digit X
        | 'L' -> Digit L
        | 'C' -> Digit C
        | 'D' -> Digit D
        | 'M' -> Digit M
        | ch -> BadChar ch

    let toRomanDigitList (s:string) =
        s.ToCharArray()
        |> List.ofArray
        |> List.map charToRomanDigit

    /// 文字列をRomanNumeralに変換する
    /// 例："IVIV "は有効です。
    let toRomanNumeral s =
        toRomanDigitList s
        |> List.choose (
            function
            | Digit digit ->
                Some digit
            | BadChar ch ->
                eprintfn "%c is not a valid character" ch
                None
            )
        |> RomanNumeral

    // ==========================================
    // 検証ロジック
    // ==========================================

    let runsAllowed =
        function
        | I | X | C | M -> true
        | V | L | D -> false

    let noRunsAllowed  = runsAllowed >> not

    // check for validity
    let rec isValidDigitList digitList =
        match digitList with

        // 空のリストは有効
        | [] -> true

        // A run of 5 or more anything is invalid
        // Example:  XXXXX
        | d1::d2::d3::d4::d5::_
            when d1=d2 && d1=d3 && d1=d4 && d1=d5 ->
                false

        // 2 or more non-runnable digits is invalid
        // Example:  VV
        | d1::d2::_
            when d1=d2 && noRunsAllowed d1 ->
                false

        // runs of 2,3,4 in the middle are invalid if next digit is higher
        // Example:  IIIX
        | d1::d2::d3::d4::higher::ds
            when d1=d2 && d1=d3 && d1=d4
            && runsAllowed d1 // not really needed because of the order of matching
            && higher > d1 ->
                false

        | d1::d2::d3::higher::ds
            when d1=d2 && d1=d3
            && runsAllowed d1
            && higher > d1 ->
                false

        | d1::d2::higher::ds
            when d1=d2
            && runsAllowed d1
            && higher > d1 ->
                false

        // three ascending numbers in a row is invalid
        // Example:  IVX
        | d1::d2::d3::_  when d1<d2 && d2<= d3 ->
            false

        // ランのない1桁の数字は常に許される
        | _::ds ->
            // リストの残りの部分をチェックする
            isValidDigitList ds

    // トップレベルの有効性のチェック
    let isValid (RomanNumeral digitList) =
        isValidDigitList digitList

```

## 第二弾

このコードは動作していますが、気になる点があります。検証ロジックが非常に複雑に見えます。ローマ人はこのようなことを考える必要はなかったのではないでしょうか？

また、"VIV "のように、バリデーションに失敗するはずのものが合格してしまう例も考えられます。

```fsharp
"VIV" |> toRomanNumeral |> isValid
```

バリデーションのルールを厳しくすることもできますが、別の方法を試してみましょう。複雑なロジックは、多くの場合、ドメインを正しく理解していないことの証です。

つまり、内部モデルを変更して、すべてをシンプルにすることはできないだろうか。

文字と数字を対応させるのをやめて、ローマ人が考えたようなドメインを作ってみたらどうだろう。 このモデルでは、"I"、"II"、"III"、"IV "などがそれぞれ別の数字になります。

実際にやってみて、どうなるか見てみましょう。

これがドメインの新しいタイプです。これで、ありとあらゆる桁に対応する桁型ができました。RomanNumeral "の型はそのままです。

```fsharp
type RomanDigit =
    | I | II | III | IIII
    | IV | V
    | IX | X | XX | XXX | XXXX
    | XL | L
    | XC | C | CC | CCC | CCCC
    | CD | D
    | CM | M | MM | MMM | MMMM
type RomanNumeral = RomanNumeral of RomanDigit list
```

### 出力：セカンドバージョン

次に、1つの`RomanDigit`を整数に変換する方法は、先ほどと同じですが、ケースが増えています。

```fsharp
/// 単一のRomanDigitを整数に変換する
let digitToInt =
    function
    | I -> 1 | II -> 2 | III -> 3 | IIII -> 4
    | IV -> 4 | V -> 5
    | IX -> 9 | X -> 10 | XX -> 20 | XXX -> 30 | XXXX -> 40
    | XL -> 40 | L -> 50
    | XC -> 90 | C -> 100 | CC -> 200 | CCC -> 300 | CCCC -> 400
    | CD -> 400 | D -> 500
    | CM -> 900 | M -> 1000 | MM -> 2000 | MMM -> 3000 | MMMM -> 4000

// テスト
I  |> digitToInt
III  |> digitToInt
V  |> digitToInt
CM  |> digitToInt
```

桁の合計を計算することは、今や些細なことです。特別なケースは必要ありません。

```fsharp
/// 数字のリストを整数に変換します。
let digitsToInt list =
    list |> List.sumBy digitToInt

// テスト
[IIII] |> digitsToInt
[IV] |> digitsToInt
[V;I] |> digitsToInt
[IX] |> digitsToInt
[M;CM;L;X;X;IX] |> digitsToInt // 1979
[M;CM;XL;IV] |> digitsToInt // 1944
```

最後に、トップレベルの関数は同じです。

```fsharp
///ローマ数字を整数に変換します
let toInt (RomanNumeral digits) = digitsToInt digits

// test
let x = RomanNumeral [M;CM;LX;X;IX]
x |> toInt
```

### 入力：第二弾

入力の解析については、`ParsedChar`型のままにします。しかし、今回は一度に1,2,3,4文字をマッチさせる必要があります。
つまり、最初のバージョンのように1文字だけを取り出すことはできず、メインループの中でマッチさせる必要があります。つまり、このループは再帰的でなければならないということです。

また、IIIIを4つの独立した`I`桁ではなく、1つの`IIII`桁に変換したいので、最も長くマッチするものを先頭に置きます。

```fsharp
type ParsedChar =
    | Digit of RomanDigit
    | BadChar of char

let rec toRomanDigitListRec charList =
    match charList with
    // 最も長いパターンを最初にマッチ

    // 4文字のマッチ
    | 'I'::'I'::'I'::'I'::ns ->
        Digit IIII :: (toRomanDigitListRec ns)
    | 'X'::'X'::'X'::'X'::ns ->
        Digit XXXX :: (toRomanDigitListRec ns)
    | 'C'::'C'::'C'::'C'::ns ->
        Digit CCCC :: (toRomanDigitListRec ns)
    | 'M'::'M'::'M'::'M'::ns ->
        Digit MMMM :: (toRomanDigitListRec ns)

    // 3文字マッチ
    | 'I'::'I'::'I'::ns ->
        Digit III :: (toRomanDigitListRec ns)
    | 'X'::'X'::'X'::ns ->
        Digit XXX :: (toRomanDigitListRec ns)
    | 'C'::'C'::'C'::ns ->
        Digit CCC :: (toRomanDigitListRec ns)
    | 'M'::'M'::'M'::ns ->
        Digit MMM :: (toRomanDigitListRec ns)

    // 2文字マッチ
    | 'I'::'I'::ns ->
        Digit II :: (toRomanDigitListRec ns)
    | 'X'::'X'::ns ->
        Digit XX :: (toRomanDigitListRec ns)
    | 'C'::'C'::ns ->
        Digit CC :: (toRomanDigitListRec ns)
    | 'M'::'M'::ns ->
        Digit MM :: (toRomanDigitListRec ns)

    | 'I'::'V'::ns ->
        Digit IV :: (toRomanDigitListRec ns)
    | 'I'::'X'::ns ->
        Digit IX :: (toRomanDigitListRec ns)
    | 'X'::'L'::ns ->
        Digit XL :: (toRomanDigitListRec ns)
    | 'X'::'C'::ns ->
        Digit XC :: (toRomanDigitListRec ns)
    | 'C'::'D'::ns ->
        Digit CD :: (toRomanDigitListRec ns)
    | 'C'::'M'::ns ->
        Digit CM :: (toRomanDigitListRec ns)

    // 1文字マッチ
    | 'I'::ns ->
        Digit I :: (toRomanDigitListRec ns)
    | 'V'::ns ->
        Digit V :: (toRomanDigitListRec ns)
    | 'X'::ns ->
        Digit X :: (toRomanDigitListRec ns)
    | 'L'::ns ->
        Digit L :: (toRomanDigitListRec ns)
    | 'C'::ns ->
        Digit C :: (toRomanDigitListRec ns)
    | 'D'::ns ->
        Digit D :: (toRomanDigitListRec ns)
    | 'M'::ns ->
        Digit M :: (toRomanDigitListRec ns)

    // 不正文字マッチ
    | badChar::ns ->
        BadChar badChar :: (toRomanDigitListRec ns)

    // 0文字のマッチ
    | [] ->
        []

```

さて，最初のバージョンに比べてずいぶん長くなりましたが，それ以外は基本的に同じです。

トップレベルの関数に変更はありません。

```fsharp
let toRomanDigitList (s:string) =
    s.ToCharArray()
    |> List.ofArray
    |> toRomanDigitListRec

/// 文字列をローマ数字に変換する
let toRomanNumeral s =
    toRomanDigitList s
    |> List.choose (
        function
        | Digit digit ->
            Some digit
        | BadChar ch ->
            eprintfn "%c is not a valid character" ch
            None
        )
    |> RomanNumeral

// 良いケースのテスト
"IIII"  |> toRomanNumeral
"IV"  |> toRomanNumeral
"VI"  |> toRomanNumeral
"IX"  |> toRomanNumeral
"MCMLXXIX"  |> toRomanNumeral
"MCMXLIV" |> toRomanNumeral
"" |> toRomanNumeral

// エラーケース
"MC?I" |> toRomanNumeral
"abc" |> toRomanNumeral
```

### 入力検証：第二版

最後に、新しいドメインモデルが入力検証にどのような影響を与えるかを見てみましょう。 今回の検証ルールは非常にシンプルになっています。実際には1つしかありません。

* 各桁は前の桁よりも小さくなければならない。

```fsharp
// 妥当性のチェック
let rec isValidDigitList digitList =
    match digitList with

    // 空のリストは有効
    | [] -> true

    // 次の桁が同じかそれ以上の場合はエラー
    | d1::d2::_
        when d1 <= d2  ->
            false

    // 一桁の数字は常に許される
    | _::ds ->
        // リストの残りの部分をチェックする
        isValidDigitList ds

// トップレベルの有効性のチェック
let isValid (RomanNumeral digitList) =
    isValidDigitList digitList

// 良いケースのテスト
"IIII"  |> toRomanNumeral |> isValid
"IV"  |> toRomanNumeral |> isValid
"" |> toRomanNumeral |> isValid

// エラーケース
"IIXX" |> toRomanNumeral |> isValid
"VV" |> toRomanNumeral |> isValid

```

残念ながら、書き換えのきっかけとなった不正なケースはまだ修正されていません。

```fsharp
"VIV" |> toRomanNumeral |> isValid
```

この問題にはそれほど難しくない解決策がありますが、今は放っておいても構わないと思います!

## 第二版のコード全体

第2バージョンの1つのモジュールに含まれるすべてのコードを紹介します。

```fsharp
module RomanNumeralsV2 =

    // ==========================================
    // タイプ
    // ==========================================

    type RomanDigit =
        | I | II | III | IIII
        | IV | V
        | IX | X | XX | XXX | XXXX
        | XL | L
        | XC | C | CC | CCC | CCCC
        | CD | D
        | CM | M | MM | MMM | MMMM
    type RomanNumeral = RomanNumeral of RomanDigit list

    // ==========================================
    // 出力ロジック
    // ==========================================

    /// 単一のRomanDigitを整数に変換する
    let digitToInt =
        function
        | I -> 1 | II -> 2 | III -> 3 | IIII -> 4
        | IV -> 4 | V -> 5
        | IX -> 9 | X -> 10 | XX -> 20 | XXX -> 30 | XXXX -> 40
        | XL -> 40 | L -> 50
        | XC -> 90 | C -> 100 | CC -> 200 | CCC -> 300 | CCCC -> 400
        | CD -> 400 | D -> 500
        | CM -> 900 | M -> 1000 | MM -> 2000 | MMM -> 3000 | MMMM -> 4000

    /// 数字のリストを整数に変換します。
    let digitsToInt list =
        list |> List.sumBy digitToInt

    /// RomanNumeralを整数に変換します。
    let toInt (RomanNumeral digits) = digitsToInt digits

    // ==========================================
    // 入力ロジック
    // ==========================================

    type ParsedChar =
        | Digit of RomanDigit
        | BadChar of char

    let rec toRomanDigitListRec charList =
        match charList with
        // 最も長いパターンを最初にマッチ

        // 4文字のマッチ
        | 'I'::'I'::'I'::'I'::ns ->
            Digit IIII :: (toRomanDigitListRec ns)
        | 'X'::'X'::'X'::'X'::ns ->
            Digit XXXX :: (toRomanDigitListRec ns)
        | 'C'::'C'::'C'::'C'::ns ->
            Digit CCCC :: (toRomanDigitListRec ns)
        | 'M'::'M'::'M'::'M'::ns ->
            Digit MMMM :: (toRomanDigitListRec ns)

        // 3文字マッチ
        | 'I'::'I'::'I'::ns ->
            Digit III :: (toRomanDigitListRec ns)
        | 'X'::'X'::'X'::ns ->
            Digit XXX :: (toRomanDigitListRec ns)
        | 'C'::'C'::'C'::ns ->
            Digit CCC :: (toRomanDigitListRec ns)
        | 'M'::'M'::'M'::ns ->
            Digit MMM :: (toRomanDigitListRec ns)

        // 2文字マッチ
        | 'I'::'I'::ns ->
            Digit II :: (toRomanDigitListRec ns)
        | 'X'::'X'::ns ->
            Digit XX :: (toRomanDigitListRec ns)
        | 'C'::'C'::ns ->
            Digit CC :: (toRomanDigitListRec ns)
        | 'M'::'M'::ns ->
            Digit MM :: (toRomanDigitListRec ns)

        | 'I'::'V'::ns ->
            Digit IV :: (toRomanDigitListRec ns)
        | 'I'::'X'::ns ->
            Digit IX :: (toRomanDigitListRec ns)
        | 'X'::'L'::ns ->
            Digit XL :: (toRomanDigitListRec ns)
        | 'X'::'C'::ns ->
            Digit XC :: (toRomanDigitListRec ns)
        | 'C'::'D'::ns ->
            Digit CD :: (toRomanDigitListRec ns)
        | 'C'::'M'::ns ->
            Digit CM :: (toRomanDigitListRec ns)

        // 1文字マッチ
        | 'I'::ns ->
            Digit I :: (toRomanDigitListRec ns)
        | 'V'::ns ->
            Digit V :: (toRomanDigitListRec ns)
        | 'X'::ns ->
            Digit X :: (toRomanDigitListRec ns)
        | 'L'::ns ->
            Digit L :: (toRomanDigitListRec ns)
        | 'C'::ns ->
            Digit C :: (toRomanDigitListRec ns)
        | 'D'::ns ->
            Digit D :: (toRomanDigitListRec ns)
        | 'M'::ns ->
            Digit M :: (toRomanDigitListRec ns)

        // 不正文字マッチ
        | badChar::ns ->
            BadChar badChar :: (toRomanDigitListRec ns)

        // 0文字のマッチ
        | [] ->
            []

    let toRomanDigitList (s:string) =
        s.ToCharArray()
        |> List.ofArray
        |> toRomanDigitListRec

    /// 文字列をRomanNumeralに変換する
    /// 例："IVIV "は有効です。
    let toRomanNumeral s =
        toRomanDigitList s
        |> List.choose (
            function
            | Digit digit ->
                Some digit
            | BadChar ch ->
                eprintfn "%c is not a valid character" ch
                None
            )
        |> RomanNumeral

    // ==========================================
    // 検証ロジック
    // ==========================================

    // 妥当性のチェック
    let rec isValidDigitList digitList =
        match digitList with

        // 空のリストは有効
        | [] -> true

        // 次の桁が同じかそれ以上の場合はエラー
        | d1::d2::_
            when d1 <= d2  ->
                false

        // 一桁の数字は常に許される
        | _::ds ->
            // リストの残りの部分をチェックする
            isValidDigitList ds

    // トップレベルの有効性のチェック
    let isValid (RomanNumeral digitList) =
        isValidDigitList digitList


```

## 2つのバージョンの比較

どちらのバージョンが好きでしたか?2番目のものは、より多くのケースがあるため、以前より長くなりますが、その一方で、実際のロジックはどの部分でも同じかシンプルで、特殊なケースはありません。
その結果、コードの合計行数は両方のバージョンでほぼ同じになります。

全体的には、特別なケースがないので、私は2番目の実装が好きです。

面白い実験として、同じコードをC#や好みの命令型言語で書いてみてください。

## オブジェクト指向にする

最後に、どのようにしてこのオブジェクト指向にするかを見てみましょう。ヘルパー関数は気にしないので、おそらく3つのメソッドが必要です:

* 静的コンストラクタ
* intに変換するメソッド
* 文字列に変換するメソッド

これがその例です

```fsharp
type RomanNumeral with

    static member FromString s =
        toRomanNumeral s

    member this.ToInt() =
        toInt this

    override this.ToString() =
        sprintf "%A" this
```

*Note: 非推奨のオーバーライドに関するコンパイラ警告は無視できます。*

これをオブジェクト指向の方法で使用してみましょう:

```fsharp
let r = RomanNumeral.FromString "XXIV"
let s = r.ToString()
let i = r.ToInt()
```


## まとめ

この記事では、たくさんのパターンマッチングを見てきました。

繰り返しになりますが、前回の記事と同様に重要なのは、非常に単純なドメインに対して適切に設計された内部モデルを作成するのがいかに簡単かということです。
繰り返しますが、私たちの内部モデルではプリミティブ型を使用していません--ドメインをより適切に表現することは、小さな型をたくさん作成しない理由にはなりません。C#で`ParsedChar`タイプを作成したとしたらどうでしょうか?

また、内部モデルを選択すると、設計の複雑さに大きな違いが生じます。しかし、リファクタリングを行った場合、大抵の場合はコンパイラーが何かを忘れていると警告してくれます。

