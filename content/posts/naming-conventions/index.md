---
layout: post
title: "Parameter and value naming conventions"
description: "a, f, x and friends"
date: 2012-05-19
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 6
---

C#のような命令型言語からF#に来たのであれば、あなたが慣れ親しんでいるよりも短くてわかりにくい名前がたくさん見つかるかもしれません。

C#とJavaのベストプラクティスは、長い記述識別子を持つことです。関数型言語では、関数名自体を記述的にすることができますが、関数内のローカル識別子は非常に短くなる傾向があり、パイプと合成は、最小限の行数ですべてを取得するために頻繁に使用されます。

例えば、以下に示すのは、ローカルな値に対して非常に記述的な名前を持つ素数ふるいの大まかな実装です。

```fsharp
let primesUpTo n =
    // 再帰的な中間関数を作成する
    let rec sieve listOfNumbers  =
        match listOfNumbers with
        | [] -> []
        | primeP::sievedNumbersBiggerThanP->
            let sievedNumbersNotDivisibleByP =
                sievedNumbersBiggerThanP
                |> List.filter (fun i-> i % primeP > 0)
            //再帰部分
            let newPrimes = sieve sievedNumbersNotDivisibleByP
            primeP :: newPrimes
    // ふるいを使う
    let listOfNumbers = [2..n]
    sieve listOfNumbers     // return

//test
primesUpTo 100
```

以下に、簡潔で慣用名を持ち、よりコンパクトなコードを使った同じ実装を示します:

```fsharp
let primesUpTo n =
   let rec sieve l  =
      match l with
      | [] -> []
      | p::xs ->
            p :: sieve [for x in xs do if (x % p) > 0 then yield x]
   [2..n] |> sieve
```

もちろん、このような暗号化的な名前が常に優れているわけではありませんが、関数が数行に抑えられ、使用される演算が標準である場合、これはかなり一般的な慣用句です。

一般的な命名規則は次のとおりです。

* "a","b","c"などは型です。
* "f","g","h"などは関数です。
* "x","y","z"などは、関数の引数です。
* リストは"s"という添字で示されるので、"xs"は"x"のリスト、"fs"は関数のリスト、などとなります。リストの先頭 (最初の要素) と末尾 (残りの要素) を意味する"`x::xs`"は非常にありふれた表現です。
* "_"は、値を無視する場合に使用します。つまり"`x::_`"はリストの残りの部分を無視することを意味し、"`let f _ = something`"は`f`の引数を無視することを意味します。

短い名前を使用するもう1つの理由は、多くの場合、意味のある名前に割り当てることができないことです。たとえば、パイプ演算子の定義は次のとおりです。

```fsharp
let (|>) x f = f x
```

`f`と`x`がどのようなものになるかはわかりませんが、`f`は任意の関数であり、`x`は任意の値です。これを明示的にしても、コードが理解しやすくなるわけではありません。

```fsharp
let (|>) aValue aFunction = aFunction aValue // 良くなりましたか？
```

### このサイトで使用されているスタイル

このサイトでは両方のスタイルを使います。入門シリーズでは、新しい概念が大半を占める場合には、中間値と長い名前を持つ、非常に説明的なスタイルを使用します。しかし、上級者向けのシリーズでは、スタイルはより簡潔になります。
