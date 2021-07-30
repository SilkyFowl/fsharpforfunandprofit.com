---
layout: post
title: "Expressions vs. statements"
description: "Why expressions are safer and make better building blocks"
date: 2012-05-16
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 2
---

プログラミング言語の用語では、"式"とは、単に独立した実行単位であり、何も返さない"文"とは対照的に、新しい値を作成するためにコンパイラによって合成および解釈される、値や機能の組み合わせです。これを理解するための1つの考え方として、「式の目的は (何らかの副作用が発生する可能性のある) 値を生成することであり、文の目的はその副作用を受け持つことにある」 というものがあります。

C#、および大部分の命令型言語は、式と文とを明確に区別し、それぞれがどこで使用できるかについてのルールを持っています。しかし明らかなように、真に純粋な関数型言語は文をまったくサポートできません。なぜなら、真に純粋な関数型言語には副作用がないからです。

F#は完全に純粋ではありませんが、同じ原則に従っています。F#で全ては式であり、値や関数だけでなく、制御フロー (if-then-elseやループなど) やパターンマッチングなども同様です。

文ではなく式を使用することには、ちょっとした利点があります。第1に、文とは異なり、小さな式を結合 (つまり "合成") して大きな式にすることができます。すべてが式であるならば、あらゆるものが合成可能です。

第2に、文が連続するということは、常に特定の評価の順序が存在することを意味します。つまり、文は前の文を参照しなければ解釈できません。しかし、純粋な式の場合、部分式には実行順序や依存関係はありません。

したがって、式`a+b`において、’`a`'と'`b`'の両者が純粋であれば、’`a`'の部分を単独に分離し、解釈し、試験し、そして評価することができます。'`b`'の部分も同様です。
この式の "分離可能性"は、関数型プログラミングのもう1つの利点です。

{{<alertinfo>}}
注: F#インタラクティブウィンドウも、すべてが式であるという前提のもとで成り立っています。C#でインタラクティブウィンドウを利用するのはずっと難しいでしょう。
{{</alertinfo>}}

## 式はより安全でよりコンパクトです ##

式を一貫して使用することで、より安全で簡潔なコードを作成できます。これがどういう意味か見てみましょう。

まず、文ベースのアプローチを見てみましょう。文は値を返さないため、文の内部で代入するための一時変数を使用する必要があります。F#ではない、Cライクな言語 (OK, C#) を用いた例を以下に示します:

```csharp
public void IfThenElseStatement(bool aBool)
{
   int result;     // 使用される前のresultの値は？
   if (aBool)
   {
      result = 42; // 'else'の場合の結果は何ですか？
   }
   Console.WriteLine("result={0}", result);
}
```

"if-then"は文なので、`result`変数は文の*外側*に定義される必要がありますが、代入は文の*内側*で行う必要があります、これはいくつかの問題を引き起こします。

* `result`変数は、文の外部で設定する必要があります。初期値はどのように設定するべきでしょうか?
* `if`文で`result`への代入を忘れた場合はどうなるのでしょうか?"if"文の目的は、ただ副作用 (変数への代入) を受け持つことにあります。これは文は潜在的にバグを含んでいることを意味します。なぜなら、ある分岐内で代入を忘れるというのは珍しくないからです。また、この代入はあくまでも副作用なので、コンパイラは何の警告も出しませんでした。`result`変数はスコープ内ですでに定義されているため、それが無効であることに気づかずに簡単に使用できてしまいました。
* `else'の場合、`result`変数の値はどうなりますか?この場合、値を指定していません。忘れたのでしょうか?これは潜在的なバグではないですでしょうか?
* 最後に、処理を実行するために副作用に依存しているということは、文自体には含まれていない変数に依存しているということで、この文は別のコンテキスト (たとえば、リファクタリングまたは並列化のために抽出されたもの) では簡単に利用できないという点です。

Note:上記のコードはC#ではコンパイルされません。というのは、このように未割り当てのローカル変数を使用するとコンパイラからエラーが出るからです。しかし、`result`を使用する前に*何らかの*既定値を定義しなければならないという問題は依然として解決されていません。

比較のために，同じコードを式指向のスタイルで書き直してみましょう．

```csharp
public void IfThenElseExpression(bool aBool)
{
    int result = aBool ? 42 : 0;
    Console.WriteLine("result={0}", result);
}
```

式指向バージョンの場合、これ以前の問題のいずれも該当しません！

* `result`変数は、代入時に宣言されます。式の"外側"に変数を設定する必要はなく、初期値を設定する必要もありません。
* `else`は明示的に処理されます。いずれかの分岐で代入し忘れる可能性はありません。
* そして、`result`に代入を忘れることはありません。そうすると変数は存在しようがないからです！

F#では、2つの例を次のように記述します:

```fsharp
let IfThenElseStatement aBool =
   let mutable result = 0       // mutable キーワードが必要です。
   if (aBool) then result <- 42
   printfn "result=%i" result
```

`mutable`キーワードはF#ではコードの臭いとみなされ、特定の特別な場合を除いて推奨されません。勉強している間は絶対に避けるべきです！

式ベースの場合、mutable変数は取り除かれ、再割り当ては行われません。

```fsharp
let IfThenElseExpression aBool =
   let result = if aBool then 42 else 0
                // elseケースを指定しなければならないことに注意
   printfn "result=%i" result
```

いったん`if`文を式に変換したら、リファクタリングして、エラーを発生させずに部分式全体を別のコンテキストへ移動するのは簡単なことです。

C#でリファクタリングされたバージョンを以下に示します:

```csharp
public int StandaloneSubexpression(bool aBool)
{
    return aBool ? 42 : 0;
}

public void IfThenElseExpressionRefactored(bool aBool)
{
    int result = StandaloneSubexpression(aBool);
    Console.WriteLine("result={0}", result);
}
```

また、F#では:

```fsharp
let StandaloneSubexpression aBool =
   if aBool then 42 else 0

let IfThenElseExpressionRefactored aBool =
   let result = StandaloneSubexpression aBool
   printfn "result=%i" result
```



### ループにおける文と式の比較 ###

C#に戻って、ループ文を用いた文と式との比較例を次に示します。

```csharp
public void LoopStatement()
{
    int i;    //iが使われる前の値は？
    int length;
    var array = new int[] { 1, 2, 3 };
    int sum;  //配列が空の場合、sumの値はどうなりますか？

    length = array.Length;   //lengthに代入するのを忘れた場合は？
    for (i = 0; i < length; i++)
    {
        sum += array[i];
    }

    Console.WriteLine("sum={0}", sum);
}
```

ここでは、インデックス変数をループの外で宣言するという、古いスタイルの"for"文を使っています。ループのインデックス"`i`"や最大値"`length`"は、ループの外で使えるのか、代入されなかったらどうなるのかなど、先に述べた多くの問題があります。また、これらが割り当てられていない場合はどうなるのでしょうか？

より現代的なforループでは、ループ変数の宣言と割り当てをforループ内で行い、"`sum`"変数の初期化を要求することで、これらの問題に対処しています:

```csharp
public void LoopStatementBetter()
{
    var array = new int[] { 1, 2, 3 };
    int sum = 0;        // 初期化が必要です。

    for (var i = 0; i < array.Length; i++)
    {
        sum += array[i];
    }

    Console.WriteLine("sum={0}", sum);
}
```

このより現代的なバージョンは、ローカル変数の宣言とその最初の代入を組み合わせるという一般的な原則に従っています。

しかしもちろん、`for`ループの代わりに`foreach`ループを使うことで、改善を続けることができます:

```csharp
public void LoopStatementForEach()
{
    var array = new int[] { 1, 2, 3 };
    int sum = 0; // 初期化が必要です。

    foreach (var i in array)
    {
        sum += i;
    }

    Console.WriteLine("sum={0}", sum);
}
```

毎回、コードを凝縮しているだけでなく、エラーの可能性も低くなっています。

しかし、この原則を突き詰めると、完全に式ベースのアプローチになります。LINQを使用した場合の方法をご紹介します:

```csharp
public void LoopExpression()
{
    var array = new int[] { 1, 2, 3 };

    var sum = array.Aggregate(0, (sumSoFar, i) => sumSoFar + i);

    Console.WriteLine("sum={0}", sum);
}
```

LINQの組み込み関数"sum"を使用することもできましたが、文に埋め込まれたsumロジックが、どのようにラムダに変換されて式の一部として使用されるかを示すために、`Aggregate`を使用しました。

次の記事では、F#のさまざまな式について見ていきます。

