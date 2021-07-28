---
layout: page
title: "Troubleshooting F#"
description: "Why won't my code compile?"
nav: troubleshooting-fsharp
hasComments: 1
date: 2020-01-01
---

諺にもあるように、「コンパイルできれば正しい」のですが、コードをコンパイルしようとするだけで非常にイライラすることがあります。 そこでこのページでは、F#コードのトラブルシューティングをお手伝いします。

まず、トラブルシューティングに関する一般的なアドバイスと、初心者が陥りやすいエラーをいくつか紹介します。その後、一般的なエラーメッセージをそれぞれ詳しく説明し、どのように発生するか、どのように修正するかの例を示します。

[(エラー番号へのジャンプ)](#NumericErrors)

## トラブルシューティングの一般的なガイドライン ##

最も重要なことは、F#がどのように動作するか、特に関数や型システムに関わる中核的な概念を正確に理解するために時間と労力をかけることです。 本格的なコーディングを始める前に、["thinking functionally"](/series/thinking-functionally.html)と["understanding F# types"](/series/understanding-fsharp-types.html)を読み直し、例題で遊んで、考え方に慣れてください。関数や型がどのように機能するかを理解していなければ、コンパイラのエラーも意味をなさないでしょう。

C#のような命令型言語から来た人は、間違ったコードを見つけて修正するためにデバッガに頼るという悪い習慣が身についているかもしれません。  F#では、コンパイラが多くの点で非常に厳しいので、おそらくそこまではできないでしょう。 もちろん、コンパイラーを「デバッグ」して、その処理を段階的に進めるツールもありません。 コンパイラのエラーをデバッグするための最良のツールはあなたの脳であり、F#はそれを使うことを強要しているのです。

とはいえ、初心者が陥りやすいエラーがいくつかありますので、それらを簡単に説明します。

### 関数を呼び出すときに括弧を使わない ### ##F#

F#では、関数のパラメータには空白が標準的な区切り文字となっています。括弧を使う必要はほとんどありませんが、特に、関数を呼び出すときには括弧を使わないようにしましょう。

```fsharp
let add x y = x + y
let result = add (1 2)  //wrong
    // error FS0003: This value is not a function and cannot be applied
let result = add 1 2    //correct
```

### 複数のパラメータを持つタプルを混ぜてはいけません ### 。

コンマがあれば、それはタプルです。そして、タプルは1つのオブジェクトであり、2つのオブジェクトではありません。そのため、間違ったタイプのパラメータを渡したり、パラメータが少なすぎたりするとエラーになります。

```fsharp
addTwoParams (1,2)  // trying to pass a single tuple rather than two args
   // error FS0001: This expression was expected to have type
   //               int but here has type 'a * 'b
```

コンパイラは `(1,2)` を一般的なタプルとして扱い、それを "`addTwoParams`" に渡そうとします。すると、`addTwoParams`の最初のパラメータはint型で、タプルを渡そうとしていると文句を言います。

もし、*1*個のタプルを期待する関数に*2*個の引数を渡そうとすると、別の不明瞭なエラーが発生します。

```fsharp
addTuple 1 2   // trying to pass two args rather than one tuple
  // error FS0003: This value is not a function and cannot be applied
```

### 引数の数が少なすぎたり多すぎたりすると注意が必要です##。

F#コンパイラは、関数に渡す引数の数が少なすぎても文句を言いませんが(実際、「部分適用」は重要な機能です)、何が起こっているのかを理解していないと、後で奇妙な「型の不一致」のエラーがしばしば発生します。

同様に、引数が多すぎる場合のエラーも、もっとわかりやすいエラーではなく、典型的には「この値は関数ではありません」となります。

"printf"関数ファミリーは、この点において非常に厳格です。引数の数は正確でなければなりません。

これは非常に重要なトピックです。部分適用がどのように機能するかを理解することは非常に重要です。より詳しい説明は ["thinking functionally"](/series/thinking-functionally.html)シリーズをご覧ください。

### リストのセパレータにセミコロンを使用する##。

リストやレコードなど、F# が明示的なセパレータ文字を必要とする数少ない場所では、セミコロンを使用します。 カンマは決して使用しません。(壊れたレコードのように「カンマはタプルのためのものである」と唱えましょう)。

```fsharp
let list1 = [1,2,3]    // wrong! This is a ONE-element list containing
                       // a three-element tuple
let list1 = [1;2;3]    // correct

type Customer = {Name:string, Address: string}  // wrong
type Customer = {Name:string; Address: string}  // correct
```

### not に ! を、not-equal に != を使用してはいけません##。

感嘆符の記号は「NOT」演算子ではありません。これは可変型参照のdeferencing演算子です。誤って使用すると、以下のようなエラーが発生します。

```fsharp
let y = true
let z = !y
// => error FS0001: This expression was expected to have
//    type 'a ref but here has type bool
```

正しい構文は、"not "キーワードを使うことです。C言語の構文ではなく，SQLやVBの構文を考えてみてください．

```fsharp
let y = true
let z = not y       //correct
```

また，"not equal "には"<>"を使います。これもSQLやVBの構文と同じです。

```fsharp
let z = 1 <> 2      //correct
```

### 代入には = を使わない ###

ミュータブルな値を使っている場合、代入操作は「`<-`」と書きます。 等号を使った場合は、エラーにならず、予想外の結果が出るだけかもしれません。

```fsharp
let mutable x = 1
x = x + 1          // returns false. x is not equal to x+1
x <- x + 1         // assigns x+1 to x
```

### 隠れたタブ文字に気をつけよう ###

インデントのルールは非常にわかりやすいので、コツをつかむのは簡単です。しかし、タブを使うことはできず、スペースのみを使うことができます。

```fsharp
let add x y =
{tab}x + y
// => error FS1161: TABs are not allowed in F# code
```

エディタでタブをスペースに変換するように設定してください。また、他の場所からコードを貼り付けている場合は注意してください。また、他の場所からコードを貼り付けている場合にも注意が必要です。 もし、コードに問題がある場合には、空白を削除して再度追加してみてください。

### 単純な値と関数の値を間違えない ###

関数ポインタやデリゲートを作成する際には、既に評価された単純な値を誤って作成しないように注意してください。

パラメータレスで再利用可能な関数を作りたい場合は、ユニットパラメータを明示的に渡すか、ラムダとして定義する必要があります。

```fsharp
let reader = new System.IO.StringReader("hello")
let nextLineFn   =  reader.ReadLine()  //wrong
let nextLineFn() =  reader.ReadLine()  //correct
let nextLineFn   =  fun() -> reader.ReadLine()  //correct

let r = new System.Random()
let randomFn   =  r.Next()  //wrong
let randomFn() =  r.Next()  //correct
let randomFn   =  fun () -> r.Next()  //correct
```

パラメータレス関数については、シリーズ["thinking functionally"](/series/thinking-functionally.html)を参照してください。

### "not enough information" エラーのトラブルシューティングのヒント ###

F# コンパイラは現在、ワンパスの左から右へのコンパイラであるため、プログラムの 後方にある型情報がまだ解析されていない場合、コンパイラはその型情報を利用できません。

このため、["FS0072: Lookup on object of indeterminate type"](#FS0072)や["FS0041: A unique overload for could not be determined"](#FS0041)など、多くのエラーが発生してしまいます。これらの具体的なケースに対する修正案を以下に説明しますが、型の欠落や十分な情報がないことをコンパイラーが訴えている場合に役立つ一般的な指針がいくつかあります。これらのガイドラインは

* 使用する前に定義しておく（ファイルを正しい順序でコンパイルすることも含む）。
* 「既知の型」を持つものを「未知の型」を持つものよりも先に配置する。特に、パイプや類似の連鎖関数を、型付きのオブジェクトが最初に来るように並べ替えることができるかもしれません。
* 必要に応じてアノテーションを行う。よくあるトリックは、すべてがうまくいくまでアノテーションを追加し、必要最小限になるまで1つずつアノテーションを削除していくことです。

できる限りアノテーションをつけないようにしてください。見た目が悪いだけでなく、コードがもろくなってしまいます。明示的な依存関係がなければ、型を変更するのはずっと簡単です。

----

# F# コンパイラのエラー{#NumericErrors}について

----

以下は、文書化する価値があると思われる主なエラーのリストです。ここでは、初心者にはわかりにくいと思われるエラーだけを取り上げ、説明可能なエラーは記録していません。

今後もこのリストを追加していきますので、追加のご提案をお待ちしております。

* [FS0001: The type 'X' does not match the type 'Y'](#FS0001)
* [FS0003: この値は関数ではないため、適用できません](#FS0003)
* [FS0008: この実行時強制または型テストには不確定な型が含まれています](#FS0008)
* [FS0010: Unexpected identifier in binding](#FS0010a)
* [FS0010: 不完全な構造化構成](#FS0010b)
* [FS0013: The static coercion from type X to Y involves an indeterminate type](#FS0013)
* [FS0020: この式の型は'unit'であるべきです](#FS0020)
* [FS0030: 値の制限](#FS0030)
* [FS0035: This construct is deprecated](#FS0035)
* [FS0039: The field, constructor or member X is not defined](#FS0039)
* [FS0041: unique overload for could not be determined](#FS0041)
* [FS0049: 大文字の変数識別子は、一般的にパターンで使用すべきではありません](#FS0049)
* [FS0072: 不確定な型のオブジェクトのルックアップ](#FS0072)
* [FS0588: let に続くブロックが未完成です](#FS0588)


## FS0001: 型'X'は型'Y'と一致しません{#FS0001}。

このエラーは、おそらく最も一般的なエラーでしょう。さまざまな状況で発生する可能性があるため、最も一般的な問題をグループ化し、例と修正方法を示しました。エラーメッセージには注意を払ってください。

{{<rawtable>}}
<table class="table table-striped table-bordered table-condensed">
<thead>
  <tr>
	<th> エラーメッセージ</th>
	<th>考えられる原因</th>
  </tr>
</thead>
<tbody>
  <tr>
	<td>「float」という型が「int」という型と一致しない</td>
	<td><a href="#FS0001A">A. floatとintを混在させることはできません</a></td>
  </tr>
  <tr>
  <td>'int'型は'DivideByInt'という名前の演算子をサポートしていません</td>
	<td><a href="#FS0001A">A. floatとintを混在させることはできません。</a></td>
  </tr>
  <tr>
  <td>型'X'は、どの型とも互換性がありません</td>
	<td><a href="#FS0001B">B. 間違った数値型を使用しています。</a></td>
  </tr>
  <tr>
  <td>このタイプ(関数型)はタイプ(単純型)と一致しません。注意：関数型には、<code>'a -&gt; 'b</code>のように、矢印が付いています。</td>
	<td><a href="#FS0001C">C. 関数に渡す引数の数が多すぎる。</a></td>
  </tr>
  <tr>
  <td>この式は、(関数型)を持つことが期待されていましたが、ここでは(単純型)を持ちます</td>
	<td><a href="#FS0001C">C. 関数に渡す引数の数が多すぎる</a></td>
  </tr>
  <tr>
  <td>この式は(N部の関数)を持つことが期待されましたが、ここでは(N-1部の関数)を持ちます</td>
	<td><a href="#FS0001C">C. 関数に渡す引数の数が多すぎる</a></td>
  </tr>
  <tr>
  <td>この式は(単純型)を持つことが期待されましたが、ここでは(関数型)を持ちます</td>
	<td><a href="#FS0001D">D. 関数に渡す引数の数が少なすぎる</a></td>
  </tr>
  <tr>
  <td>この式は、(型)を持つことが期待されていましたが、ここでは(他の型)を持っています</td>
	<td><a href="#FS0001E">E. ストレートなタイプミスマッチです。</a><br>
	<a href="#FS0001F">F. ブランチやマッチで一貫性のないリターンがある場合</a><br>
	<a href="#FS0001G">G. 関数の中に埋もれた型推論効果に注意。</a><br>
	</td>
  </tr>
  <tr>
	<td>タイプミスマッチ。(単純な型)を期待しているのに、(タプル型)が与えられた。注意：タプル型には、<code>'a * 'b</code>のように星マークがついています。</td>
	<td><a href="#FS0001H">H. スペースやセミコロンの代わりにコンマを使用しましたか？</a></td>
  </tr>
  <tr>
	<td>タイプミスマッチ。(タプルタイプ)を期待しているのに、(異なるタプルタイプ)が与えられた。</td>
  <td><a href="#FS0001I">I. タプルは同じタイプでなければ比較できません。</a></td>
  </tr>
  <tr>
  <td>この式は<code>'a ref</code>という型を持つことが期待されましたが、ここではXという型を持ちます</td>
	<td><a href="#FS0001J">J. 「not」演算子に「！」を使ってはいけません</a></td>
  </tr>
  <tr>
	<td>タイプ(型)がタイプ(他の型)と一致しない</td>
	<td><a href="#FS0001K">K. 演算子の優先順位（特に関数やパイプ）</a></td>
  </tr>
  <td>この式は型(モナディック型)を持つことが期待されていましたが、ここでは型 <code>'b * 'c</code>を持ちます。</td>
	<td><a href="#FS0001L">L. let! 計算式のエラー</a></td>
  </tr>
</tbody>
</table>
{{</rawtable>}}

### A. intsとfloatsを混在させることはできない{#FS0001A}。

C#や多くの命令型言語とは異なり、intsとfloatsを式の中で混在させることはできません。これを試みるとタイプエラーが発生します。

```fsharp
1 + 2.0  //wrong
   // => error FS0001: The type 'float' does not match the type 'int'
```

この問題を解決するには，まずint型を`float`型にキャストする必要があります。

```fsharp
float 1 + 2.0  //correct
```

この問題は、ライブラリ関数やその他の場所でも発生します。例えば、intのリストに対して「`average`」を行うことはできません。

```fsharp
[1..10] |> List.average   // wrong
   // => error FS0001: The type 'int' does not support any
   //    operators named 'DivideByInt'
```

以下のように，それぞれのintをまずfloatにキャストする必要があります。

```fsharp
[1..10] |> List.map float |> List.average  //correct
[1..10] |> List.averageBy float  //correct (uses averageBy)
```

### B. 誤った数値型の使用 {#FS0001B}。

数値型のキャストに失敗すると、「互換性がありません」というエラーが発生します。

```fsharp
printfn "hello %i" 1.0  // should be a int not a float
  // error FS0001: The type 'float' is not compatible
  //               with any of the types byte,int16,int32...
```

1つの可能な修正方法は、適切にキャストすることです。

```fsharp
printfn "hello %i" (int 1.0)
```


### C. 関数に渡す引数の数が多すぎる {#FS0001C}.

```fsharp
let add x y = x + y
let result = add 1 2 3
// ==> error FS0001: The type ''a -> 'b' does not match the type 'int'
```

手がかりはこのエラーの中にあります。

修正方法は，引数の1つを削除することです!

同様のエラーは，`printf`に渡す引数の数が多すぎる場合に発生します．

```fsharp
printfn "hello" 42
// ==> error FS0001: This expression was expected to have type 'a -> 'b
//                   but here has type unit

printfn "hello %i" 42 43
// ==> Error FS0001: Type mismatch. Expecting a 'a -> 'b -> 'c
//                   but given a 'a -> unit

printfn "hello %i %i" 42 43 44
// ==> Error FS0001: Type mismatch. Expecting a  'a -> 'b -> 'c -> 'd
//                   but given a 'a -> 'b -> unit
```

### D. 関数に渡す引数の数が少なすぎる場合 {#FS0001D}。

関数に渡す引数の数が少ないと、部分的に適用されてしまいます。後で使うときには、単純な型ではないのでエラーになります。

```fsharp
let reader = new System.IO.StringReader("hello");

let line = reader.ReadLine        //wrong but compiler doesn't complain
printfn "The line is %s" line     //compiler error here!
// ==> error FS0001: This expression was expected to have type string
//                   but here has type unit -> string
```

これは、上記の `ReadLine` のように、ユニットパラメータを要求するいくつかの .NET ライブラリ関数で特によく見られます。

この問題を解決するには、正しい数のパラメータを渡す必要があります。結果の値の型をチェックして，それが本当に単純な型であることを確認してください。 `ReadLine`のケースでは、`()`の引数を渡すことで解決します。

```fsharp
let line = reader.ReadLine()      //correct
printfn "The line is %s" line     //no compiler error
```


### E. ストレートな型のミスマッチ {#FS0001E}。

最も単純なケースは、型が間違っているか、プリントフォーマット文字列で間違った型を使用していることです。

```fsharp
printfn "hello %s" 1.0
// => error FS0001: This expression was expected to have type string
//                  but here has type float
```

### F. ブランチやマッチでの一貫性のない戻り値の型 {#FS0001F}.

よくある間違いは、分岐式やマッチ式がある場合、すべての分岐が同じ型を返さなければならないというものです。 そうでない場合は、型エラーが発生します。

```fsharp
let f x =
  if x > 1 then "hello"
  else 42
// => error FS0001: This expression was expected to have type string
//                  but here has type int
```

```fsharp
let g x =
  match x with
  | 1 -> "hello"
  | _ -> 42
// error FS0001: This expression was expected to have type
//               string but here has type int
```

明らかに，ストレートな修正方法は，各ブランチが同じ型を返すようにすることです．

```fsharp
let f x =
  if x > 1 then "hello"
  else "42"

let g x =
  match x with
  | 1 -> "hello"
  | _ -> "42"
```

"else"ブランチがない場合、unitを返すと仮定されるので、"true"ブランチもunitを返さなければならないことを覚えておいてください。

```fsharp
let f x =
  if x > 1 then "hello"
// error FS0001: This expression was expected to have type
//               unit but here has type string
```

両方のブランチが同じ型を返すことができない場合、両方の型を含むことができる新しいユニオン型を作成する必要があるかもしれません。

```fsharp
type StringOrInt = | S of string | I of int  // new union type
let f x =
  if x > 1 then S "hello"
  else I 42
```


### G. 関数に埋もれた型推論の効果に注意{#FS0001G}。

関数が予期せぬ型推論を引き起こし、コードに波及することがあります。例えば、以下の例では、無害な print format string が、誤って `doSomething` に文字列を期待させています。

```fsharp
let doSomething x =
   // do something
   printfn "x is %s" x
   // do something more

doSomething 1
// => error FS0001: This expression was expected to have type string
//    but here has type int
```

修正方法は、関数のシグネチャをチェックして、罪を犯した人を見つけるまで掘り下げていくことです。 また、可能な限り汎用的な型を使用し、可能な限り型アノテーションを避けるようにしてください。

### H. スペースやセミコロンの代わりにコンマを使っていませんか？{#FS0001H}。

F#に慣れていないと、関数の引数を区切るのに、スペースではなくコンマを誤って使ってしまうことがあります。

```fsharp
// define a two parameter function
let add x y = x + 1

add(x,y)   // FS0001: This expression was expected to have
           // type int but here has type  'a * 'b
```

修正方法は、カンマを使わないことです。

```fsharp
add x y    // OK
```

カンマが使われるのは、.NETライブラリ関数を呼び出すときです。
これらの関数はすべてタプルを引数として取るので、コンマの形式は正しいです。実際には，これらの呼び出しはC#からの呼び出しと同じように見えます。

```fsharp
// correct
System.String.Compare("a","b")

// incorrect
System.String.Compare "a" "b"
```



### I. タプルの比較やパターンマッチを行うには、同じタイプである必要があります {#FS0001I}。

異なるタイプのタプルは比較できません。タイプ `int * int` のタプルとタイプ `int * string` のタプルを比較しようとすると、エラーが発生します。

```fsharp
let  t1 = (0, 1)
let  t2 = (0, "hello")
t1 = t2
// => error FS0001: Type mismatch. Expecting a int * int
//    but given a int * string
//    The type 'int' does not match the type 'string'
```

また、長さも同じでなければなりません。

```fsharp
let  t1 = (0, 1)
let  t2 = (0, 1, "hello")
t1 = t2
// => error FS0001: Type mismatch. Expecting a int * int
//    but given a int * int * string
//    The tuples have differing lengths of 2 and 3
```

バインディング時にタプルをパターンマッチさせると，同じ問題が発生します．

```fsharp
let x,y = 1,2,3
// => error FS0001: Type mismatch. Expecting a 'a * 'b
//                  but given a 'a * 'b * 'c
//                  The tuples have differing lengths of 2 and 3

let f (x,y) = x + y
let z = (1,"hello")
let result = f z
// => error FS0001: Type mismatch. Expecting a int * int
//                  but given a int * string
//                  The type 'int' does not match the type 'string'
```



### J. ! を "not" 演算子として使用しないでください {#FS0001J}。

not "演算子として`!`を使うと、"ref "という単語に言及した型エラーが発生します。

```fsharp
let y = true
let z = !y     //wrong
// => error FS0001: This expression was expected to have
//    type 'a ref but here has type bool
```

修正方法は，代わりに "not "キーワードを使うことです．

```fsharp
let y = true
let z = not y   //correct
```


### K. 演算子の優先順位 (特に関数とパイプ) {#FS0001K}。

演算子の優先順位を間違えてしまうと、型エラーが発生することがあります。 一般に、関数の適用は他の演算子に比べて優先順位が高いので、以下のようなケースではエラーになります。

```fsharp
String.length "hello" + "world"
   // => error FS0001:  The type 'string' does not match the type 'int'

// what is really happening
(String.length "hello") + "world"
```

修正方法は、括弧を使うことです。

```fsharp
String.length ("hello" + "world")  // corrected
```

逆に，パイプ演算子は他の演算子に比べて優先順位が低い．

```fsharp
let result = 42 + [1..10] |> List.sum
 // => => error FS0001:  The type ''a list' does not match the type 'int'

// what is really happening
let result = (42 + [1..10]) |> List.sum
```

繰り返しになりますが，修正方法は括弧を使うことです。

```fsharp
let result = 42 + ([1..10] |> List.sum)
```


### L. let! コンピュテーション式(モナド)のエラー {#FS0001L}.

以下に簡単なコンピュテーション式を示します。

```fsharp
type Wrapper<'a> = Wrapped of 'a

type wrapBuilder() =
    member this.Bind (wrapper:Wrapper<'a>) (func:'a->Wrapper<'b>) =
        match wrapper with
        | Wrapped(innerThing) -> func innerThing

    member this.Return innerThing =
        Wrapped(innerThing)

let wrap = new wrapBuilder()
```

しかし、これを使おうとすると、エラーが発生します。

```fsharp
wrap {
    let! x1 = Wrapped(1)   // <== error here
    let! y1 = Wrapped(2)
    let z1 = x + y
    return z
    }
// error FS0001: This expression was expected to have type Wrapper<'a>
//               but here has type 'b * 'c
```

その理由は、「`Bind`」は、2つのパラメータではなく、タプル `(wrapper,func)` を期待しているからです。 (F#のドキュメントでbindのシグネチャを確認してください)。

修正方法は，bind関数がタプルを（1つの）パラメータとして受け取るように変更することです．

```fsharp
type wrapBuilder() =
    member this.Bind (wrapper:Wrapper<'a>, func:'a->Wrapper<'b>) =
        match wrapper with
        | Wrapped(innerThing) -> func innerThing
```

## FS0003: この値は関数ではないので、適用できません {#FS0003}。

このエラーは通常、関数に引数を多く渡したときに発生します。

```fsharp
let add1 x = x + 1
let x = add1 2 3
// ==>   error FS0003: This value is not a function and cannot be applied
```

演算子のオーバーロードを行った場合にも発生しますが、その演算子をプレフィックスやインフィックスとして使用することはできません。

```fsharp
let (!!) x y = x + y
(!!) 1 2              // ok
1 !! 2                // failed !! cannot be used as an infix operator
// error FS0003: This value is not a function and cannot be applied
```

## FS0008: FS0008: この実行時強制または型テストには不確定な型が含まれています {#FS0008}。

これは "`:?`" 演算子を使って型にマッチさせようとしたときによく見られます。

```fsharp
let detectType v =
    match v with
        | :? int -> printfn "this is an int"
        | _ -> printfn "something else"
// error FS0008: This runtime coercion or type test from type 'a to int
// involves an indeterminate type based on information prior to this program point.
// Runtime type tests are not allowed on some types. Further type annotations are needed.
```

このメッセージは、「ランタイムの型テストが一部の型では許可されていない」という問題点を伝えています。

答えは、値を強制的に参照型にする "ボックス化 "することで、型チェックができるようになります。

```fsharp
let detectTypeBoxed v =
    match box v with      // used "box v"
        | :? int -> printfn "this is an int"
        | _ -> printfn "something else"

//test
detectTypeBoxed 1
detectTypeBoxed 3.14
```


## FS0010: バインディング内の予期せぬ識別子 {#FS0010a}。

典型的な原因は、ブロック内の式を整列させるための「オフサイド」ルールを破ったことです。

```fsharp
//3456789
let f =
  let x=1     // offside line is at column 3
   x+1        // oops! don't start at column 4
              // error FS0010: Unexpected identifier in binding
```

修正方法は、コードを正しく整列させることです

アラインメントに起因する別の問題については、[FS0588: Block following this 'let' is unfinished](#FS0588)もご覧ください。

## FS0010: 不完全な構造化された構造 {#FS0010b}。

クラスのコンストラクタから括弧が抜けている場合によく発生します。

```fsharp
type Something() =
   let field = ()

let x1 = new Something     // Error FS0010
let x2 = new Something()   // OK!
```

演算子を括弧で囲むのを忘れた場合にも発生します。

```fsharp
// define new operator
let (|+) a = -a

|+ 1    // error FS0010:
        // Unexpected infix operator

(|+) 1  // with parentheses -- OK!
```

infix演算子の片側が欠けている場合にも発生します。

```fsharp
|| true  // error FS0010: Unexpected symbol '||'
false || true  // OK
```

名前空間の定義をF#インタラクティブに送ろうとした場合にも発生します。インタラクティブコンソールでは、名前空間を使用できません。

```fsharp
namespace Customer  // FS0010: Incomplete structured construct

// declare a type
type Person= {First:string; Last:string}
```


## FS0013: X型からY型への静的な強制には、不確定な型が含まれています {#FS0013}。

これは一般的に暗黙の了解によって引き起こされます。

## FS0020 この式の型は'unit'でなければなりません {#FS0020}。

このエラーは以下の2つの状況でよく見られます。

* ブロック内の最後の式ではない式
* 誤った代入演算子の使用

### FS0020 ブロック内の最後の式ではない式である場合 ###

ブロック内の最後の式のみが値を返すことができます。他の式はユニットを返さなければなりません。そのため、一般的には、最後の関数ではない場所に関数がある場合に発生します。

```fsharp
let something =
  2+2               // => FS0020: This expression should have type 'unit'
  "hello"
```

簡単な方法は `ignore` を使うことです。 しかし、なぜ関数を使って、その答えを捨ててしまうのかを考えてみてください。

```fsharp
let something =
  2+2 |> ignore     // ok
  "hello"
```

これは，C#を書いているつもりで，誤ってセミコロンで式を区切ってしまった場合にも起こります。

```fsharp
// wrong
let result = 2+2; "hello";

// fixed
let result = 2+2 |> ignore; "hello";
```

### FS0020 代入時 ###

このエラーの別のバリエーションは、プロパティへの代入時に発生します。

    この式の型は'unit'であるべきですが、型は'Y'です。

このエラーでは、可変値の代入演算子「`<-`」と、等比演算子「`=`」を混同している可能性があります。

```fsharp
// '=' versus '<-'
let add() =
    let mutable x = 1
    x = x + 1          // warning FS0020
    printfn "%d" x
```

修正方法は、適切な代入演算子を使うことです。

```fsharp
// fixed
let add() =
    let mutable x = 1
    x <- x + 1
    printfn "%d" x
```


## FS0030: 値の制限{#FS0030}。

これは、F#が可能な限り自動的にジェネリック型に一般化することに関連しています。

例えば，以下のような場合です．

```fsharp
let id x = x
let compose f g x = g (f x)
let opt = None
```

F#の型推論は、ジェネリック型を巧妙に把握します。

```fsharp
val id : 'a -> 'a
val compose : ('a -> 'b) -> ('b -> 'c) -> 'a -> 'c
val opt : 'a option
```

しかし，いくつかのケースでは，F#コンパイラはコードが曖昧であると感じ，型を正しく推測しているように見えても，より具体的に記述する必要があると考えます。

```fsharp
let idMap = List.map id             // error FS0030
let blankConcat = String.concat ""  // error FS0030
```

ほとんどの場合、この現象は、部分的に適用された関数を定義しようとしたことが原因です。そして、ほとんどの場合、最も簡単な修正方法は、不足しているパラメータを明示的に追加することです。

```fsharp
let idMap list = List.map id list             // OK
let blankConcat list = String.concat "" list  // OK
```

詳細はMSDNの["自動汎化"]の記事(http://msdn.microsoft.com/en-us/library/dd233183%28v=VS.100%29.aspx)を参照してください。

## FS0035: This construct is deprecated {#FS0035}.

F# の構文はここ数年で整理されてきているので、古い F# の本や Web ページの例を使っていると、この問題に遭遇するかもしれません。 正しい構文については、MSDNのドキュメントを参照してください。

```fsharp
let x = 10
let rnd1 = System.Random x         // Good
let rnd2 = new System.Random(x)    // Good
let rnd3 = new System.Random x     // error FS0035
```

## FS0039: The field, constructor or member X is not defined {#FS0039}.

このエラーは4つの状況でよく見られます。

* 明らかに何かが本当に定義されていない場合です。また、タイプミスや大文字小文字の不一致がないかどうかも確認してください。
* インターフェイス
* 再帰
* 拡張メソッド

### FS0039 インターフェイスについて ###

F#では、すべてのインターフェイスは「暗黙的」ではなく「明示的」な実装です。(この違いについては、C#のドキュメント["explicit interface implementation"](http://msdn.microsoft.com/en-us/library/aa288461%28v=vs.71%29.aspx)をお読みください)。

重要な点は、インターフェイスのメンバーが明示的に実装されている場合、通常のクラスのインスタンスからはアクセスできず、インターフェイスのインスタンスからしかアクセスできないということで、`:>`演算子を使ってインターフェイスの型にキャストする必要があります。

以下は，インターフェイスを実装したクラスの例です．

```fsharp
type MyResource() =
   interface System.IDisposable with
       member this.Dispose() = printfn "disposed"
```

これではうまくいきません。

```fsharp
let x = new MyResource()
x.Dispose()  // error FS0039: The field, constructor
             // or member 'Dispose' is not defined
```

修正方法は、以下のようにオブジェクトをインターフェイスにキャストすることです。

```fsharp
// fixed by casting to System.IDisposable
(x :> System.IDisposable).Dispose()   // OK

let y =  new MyResource() :> System.IDisposable
y.Dispose()   // OK
```


### FS0039 with recursion ###

標準的なフィボナッチの実装を紹介します。

```fsharp
let fib i =
   match i with
   | 1 -> 1
   | 2 -> 1
   | n -> fib(n-1) + fib(n-2)
```

残念ながら、これはコンパイルできません。

    エラー FS0039: The value or constructor 'fib' is not defined

これは、コンパイラがボディ内の'fib'を見たときに、まだコンパイルが終わっていないために、その関数について知らないからです。

これを解決するには、"`rec`"キーワードを使います。

```fsharp
let rec fib i =
   match i with
   | 1 -> 1
   | 2 -> 1
   | n -> fib(n-1) + fib(n-2)
```

これは、「`let`」関数にのみ適用されることに注意してください。メンバ関数では、スコープのルールが若干異なるため、これは必要ありません。

```fsharp
type FibHelper() =
    member this.fib i =
       match i with
       | 1 -> 1
       | 2 -> 1
       | n -> fib(n-1) + fib(n-2)
```

### FS0039 拡張メソッドについて ###

拡張メソッドを定義した場合、そのモジュールがスコープ内にないと使用できません。

ここでは、簡単な拡張機能を紹介します。

```fsharp
module IntExtensions =
    type System.Int32 with
        member this.IsEven = this % 2 = 0
```

この拡張機能を使おうとすると、FS0039エラーが発生します。

```fsharp
let i = 2
let result = i.IsEven
    // FS0039: The field, constructor or
    // member 'IsEven' is not defined
```

この問題を解決するには、`IntExtensions`モジュールを開くだけです。

```fsharp
open IntExtensions // bring module into scope
let i = 2
let result = i.IsEven  // fixed!
```

## FS0041: A unique overload for could not be determined {#FS0041}.

この現象は、複数のオーバーロードを持つ.NETライブラリ関数を呼び出した場合に発生します。

```fsharp
let streamReader filename = new System.IO.StreamReader(filename) // FS0041
```

この問題を解決する方法はいくつかあります。1つの方法は、明示的な型アノテーションを使用することです。

```fsharp
let streamReader filename = new System.IO.StreamReader(filename:string) // OK
```

型のアノテーションを使わずに、名前付きのパラメータを使うこともできます。

```fsharp
let streamReader filename = new System.IO.StreamReader(path=filename) // OK
```

あるいは、型アノテーションを必要とせずに、型推論を助ける中間オブジェクトを作成することもできます。

```fsharp
let streamReader filename =
    let fileInfo = System.IO.FileInfo(filename)
    new System.IO.StreamReader(fileInfo.FullName) // OK
```

## FS0049: Uppercase variable identifiers should not generally be used in patterns {#FS0049}.

パターンマッチングの際には、タグのみで構成される純粋なF#のユニオン型と、.NETのEnum型の微妙な違いに注意してください。

純粋なF#のユニオン型:

```fsharp
type ColorUnion = Red | Yellow
let redUnion = Red

match redUnion with
| Red -> printfn "red"     // no problem
| _ -> printfn "something else"
```

しかし、.NETのenumでは、完全に修飾しなければなりません:

```fsharp
type ColorEnum = Green=0 | Blue=1      // enum
let blueEnum = ColorEnum.Blue

match blueEnum with
| Blue -> printfn "blue"     // warning FS0049
| _ -> printfn "something else"
```

修正版です:

```fsharp
match blueEnum with
| ColorEnum.Blue -> printfn "blue"
| _ -> printfn "something else"
```

## FS0072: 型が不明なオブジェクトの検索{#FS0072}について

これは、型が不明なオブジェクトに "ドットイン"したときに発生します。

次のような例を考えてみましょう。

```fsharp
let stringLength x = x.Length // Error FS0072
```

コンパイラは "x "の型を知らないので、"`Length`"が有効なメソッドであるかどうかもわかりません。

これを修正する方法はいくつかあります。最も粗野な方法は、明示的な型アノテーションを行うことです。

```fsharp
let stringLength (x:string) = x.Length  // OK
```

しかし、いくつかのケースでは、コードを適切に再配置することで解決できます。たとえば、下の例はうまくいくように見えます。`List.map`関数が文字列のリストに適用されていることは人間には明らかですが、なぜ`x.Length`がエラーになるのでしょうか？

```fsharp
List.map (fun x -> x.Length) ["hello"; "world"] // Error FS0072
```

その理由は、F#コンパイラは現在ワンパスコンパイラなので、プログラムの後半に存在する型情報がまだ解析されていない場合には、それを使用できないからです。

はい，いつでも明示的にアノテーションすることができます．

```fsharp
List.map (fun x:string -> x.Length) ["hello"; "world"] // OK
```

しかし、別のもっとエレガントな方法では、既知の型が最初に来るように並べ替えて、コンパイラが次の節に移る前にそれらを消化できるようにすることで、しばしば問題を解決できます。

```fsharp
["hello"; "world"] |> List.map (fun x -> x.Length)   // OK
```

明示的な型のアノテーションは避けるのは良いことなので、実現可能であれば、この方法がベストです。

## FS0588: この'let'に続くブロックは未完成です {#FS0588}.

ブロック内の式をアウトデントすることで、"オフサイドルール "を破っていることが原因です。

```fsharp
//3456789
let f =
  let x=1    // offside line is at column 3
 x+1         // offside! You are ahead of the ball!
             // error FS0588: Block following this
             // 'let' is unfinished
```

修正方法は、コードを正しく整列させることです。

アラインメントに起因する別の問題については、[FS0010: Unexpected identifier in binding](#FS0010a)も参照してください。