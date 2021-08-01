---
layout: post
title: "Choosing between collection functions"
description: "A guide for the perplexed"
date: 2015-08-14
categories: []
image: "/posts/list-module-functions/cyoa_list_module.jpg"
---

新しい言語を学ぶことには、言語自体を学ぶこと以上の意味があります。生産性を高めるためには、標準ライブラリーの大部分を記憶し、
残りの大部分を認識できるようになる必要があります。例えば、C#を知っていれば言語としてのJavaはすぐに理解できますが、
Javaのクラスライブラリを使いこなせるようになるまでは、本当の意味で使いこなすことは難しいでしょう。

同様に、F#を効果的に使えるようになるには、コレクションを扱う全てのF#関数を熟知する必要があります。

C#では知る必要があるのはほんの少しのLINQメソッド{{<sup 1>}}(`Select`,`Where`など) だけです。
しかしF#では現在、Listモジュールにおよそ100の関数が存在します (SeqモジュールやArrayモジュールにも同様の数があります) 。ずいぶん多いですね！
{{<footnote "1">}}
ええ、まだありますが、ほんの少しですみます。F#では、全てを知ることが重要です。
{{</footnote>}}

もしC#からF#に来たのであれば、リスト関数の数の多さに圧倒されているかもしれません。

そこで、ご所望の関数をご案内するために、この記事を書きました。
お楽しみとして"Choose Your Own Adventure"スタイルでやってみました!

![](./cyoa_list_module.jpg)

## どんなコレクションが欲しいのか？

まず、標準的なコレクションの種類についての情報を表にまとめました。F#には5つの "ネイティブ "なコレクションがあります:`list`,`seq`,`array`,`map`,`set`,です。
また、`ResizeArray`や`IDictionary`もよく使われます。

{{<rawtable>}}
<table class=\"table table-condensed table-striped\">
<tr>
<th></th>
<th>Immutable？</th>
<th>ノート</th>
</tr>
<tr>
<th>list</th>
<td>Yes</td>
<td>
    <b>長所</b>
    <ul>
    <li>パターンマッチングが可能。</li>
    <li>再帰的に複雑な反復が可能。</li>
    <li>前方反復が速い。前置詞が速い。</li>
    </ul>
    <b>欠点</b>
    <ul>
    <li>インデックス付きのアクセスやその他のアクセススタイルは遅い。</li>
    </ul>
</td>
</tr>
<tr>
<th>seq</th>
<td>Yes</td>
<td>
    <p> <code>IEnumerable</code>の別名。</p>
    <b>長所は</b>
    <ul>
    <li>遅延評価</li>
    <li>メモリ効率が良い（一度に1つの要素しか読み込まれない</li>
    <li>無限のシーケンスを表現できる。</li>
    <li>IEnumerableを使用する.NETライブラリとの相互運用。</li>
    </ul>
    <b>短所</b>
    <ul>
    <li>パターンマッチングができない。</li>
    <li>フォワードのみのイテレーション。</li>
    <li>インデックス付きアクセスなどのアクセススタイルは遅い。</li>
    </ul>
</td>
</tr>
<tr>
<th>array</th>
<td>No</td>
<td>
    <p>BCL<code>Array</code>と同じです。</p>
    <b>長所は</b>
    <ul>
    <li>高速なランダムアクセス</li>
    <li>Memory efficient and cache locality, especially with struct.</li>
    <li>Arrayを使用する.NETライブラリとのインターロップ。</li>
    <li>2D、3D、4D配列のサポート</li>
    </ul>
    <b>短所</b>
    <ul>
    <li>限定的なパターンマッチング。</li>
    <li> <a href=\"https://en.wikipedia.org/wiki/Persistent_data_structure\">永続的</a>ではない。</li>
    </ul>
</td>
</tr>
<tr>
<th>map</th>
<td>Yes</td>
<td>不変の辞書。<code>IComparable</code>を実装したキーが必要。</td>
</tr>
<tr>
<th>set</th>
<td>Yes</td>
<td>不変のセット。<code>IComparable</code>を実装するために要素が必要</td>
</tr>
<tr>
<th>ResizeArray</th>
<td>No</td>
<td>BCL<code>リスト</code>の別名。長所と短所は配列と似ていますが、サイズ変更が可能です。</td>
</tr>
<tr>
<th>IDictionary</th>
<td>Yes</td>
<td>
    <p>要素に<code>IComparable</code>の実装を必要としない代替ディクショナリとして,
    BCL<a href=\"https://msdn.microsoft.com/en-us/library/s4ys34ea.aspx\">IDictionary</a> を使用することができます。
    F#でそのコンストラクタは<a href=\"https://msdn.microsoft.com/en-us/library/ee353774.aspx\"><code>dict</code></a>です。</p>
    <p> <code>Add</code>などの突然変異メソッドは存在しますが、呼び出されるとランタイム・エラーが発生することに注意してください。</p>
</td>
</tr>
</table>
{{</rawtable>}}



これらはF#で主に使用されるコレクション型であり、通常の場合にはこれで十分です。

しかし、それ以外の種類のコレクションが必要な場合も、選択肢は数多くあります。

* .NETのコレクション型を使用できます。 [従来の変更可能型](https://msdn.microsoft.com/en-us/library/system.collections.generic)
  [System.Collections.Immutable名前空間](https://msdn.microsoft.com/en-us/library/system.collections.immutable.aspx) のような新しい名前空間もあります。
* または、F#のコレクションライブラリのいずれかを使用することもできます。
  * [**FSharpx.Collections**](https://fsprojects.github.io/FSharpx.Collections/), FSharpxシリーズの一部です。
  * [**ExtCore**](https://github.com/jack-pappas/ExtCore/tree/master/ExtCore) 。FSharpのMap型とSet型の (ほぼ) 代替品です。特定の状況でパフォーマンスを向上させるCore (HashMapなど) 。また、特定のタスクのコーディングを支援する独自機能を備えているものもあります (LazyListやLruCacheなど) 。
  * [**Funq**](https://github.com/GregRos/Funq): .NET用の高性能で不変のデータ構造。
  * [**Persistent**](https://persistent.codeplex.com/documentation):効率的な永続的 (不変) データ構造。

## ドキュメントについて

特に明記されていない限り、F#v4の`list`、`seq`、および`array`ですべての関数を使用できます。`Map`と`Set`モジュールにおいても一部使用できますが、本稿では`map`と`set`については触れません。

関数シグネチャには、標準のコレクション型として`list`を使用します。`seq`と`array`バージョンのシグネチャも似たようなものになります。

これらの関数の多くはまだMSDNに文書化されていないので、最新のコメントがあるGitHubのソースコードに直接リンクします。
リンクの関数名をクリックしてください。

## 可用性に関する注意事項

これらの関数を使用できるかどうかは、使用するF#のバージョンによって異なります。

* F#バージョン3 (Visual Studio 2013) では, List, Arrays, Sequenceにある程度の不整合がありました。
* F#バージョン4 (Visual Studio 2015) では,これが解消され, 3つのコレクション型すべてに対して,ほぼすべての関数が利用可能になりました。

F#v3とF#v4の間で何が変更されたのか知りたい場合は、 [この表](http://blogs.msdn.com/cfs-filesystemfile.ashx/__key/communityserver-blogs-components-weblogfiles/00-00-01-39-71-metablogapi/3125.collectionAPI_5F00_254EA354.png)を参照してください。
( [ここ](http://blogs.msdn.com/b/fsharpteam/archive/2014/11/12/announcing-a-preview-of-f-4-0-and-the-visual-f-tools-in-vs-2015.aspx)を参照。)
グラフにはF#v4の新しいAPI (緑) 、既存のAPI (青) 、意図的に残した空白 (白) が表示されています。

以下で説明する関数のいくつかはこの表にはありません--これらは新しい関数です!古いバージョンのF#を使っているなら、
GitHub上のコードを使って自分で再実装するだけです。

免責事項をきちんと理解したら、冒険を始めることができます!


{{< linktarget "toc" >}}

----

## Table of Contents

* [1. どんなコレクションを持っていますか？](#1)
* [2. 新しいコレクションを作る](#2)
* [3. 空や1要素のコレクションを作る](#3)
* [4. サイズがわかっている新しいコレクションの作成](#4)
* [5. 各要素が同じ値を持つ既知のサイズの新しいコレクションの作成](#5)
* [6. 各要素が異なる値を持つ、サイズが既知の新しいコレクションの作成](#6)
* [7. 新しい無限コレクションの作成](#7)
* [8. 不定形の新しいコレクションを作る](#8)
* [9. 1つのリストを扱う](#9)
* [10. ある位置の要素を取得する](#10)
* [11. 検索による要素の取得](#11)
* [12. コレクションから要素のサブセットを取得する](#12)
* [13. パーティショニング、チャンキング、グルーピング](#13)
* [14. コレクションの集計・要約](#14)
* [15. 要素の順序を変更する](#15)
* [16. コレクションの要素をテストする](#16)
* [17. 各要素を別のものに変換する](#17)
* [18. 各要素の反復処理](#18)
* [19. イテレーションで状態を通す](#19)
* [20. 各要素のインデックスを扱う](#20)
* [21. コレクション全体を別のコレクションタイプに変換する](#21)
* [22. コレクション全体の動作を変更する](#22)
* [23. 2つのコレクションを扱う](#23)
* [24. 3つのコレクションでの作業](#24)
* [25. 3つ以上のコレクションでの作業](#25)
* [26. コレクションの結合と解除](#26)
* [27. その他の配列のみの関数](#27)
* [28. 使い捨ての配列を使う](#28)


{{< linktarget "1" >}}

----

## 1. どんな種類のコレクションを持っていますか？

どのようなコレクションを持っていますか？

* コレクションを持っておらず、作成したい場合は、[セクション2](#2)に進んでください。
* もし、すでにコレクションを持っていて、それを使いたい場合は、[セクション9](#9)に進んでください。
* 扱いたいコレクションが2つある場合は、[セクション23](#23)に進んでください。
* 扱いたいコレクションが3つある場合は、[セクション24](#24)に進んでください。
* 扱いたいコレクションが3つ以上ある場合は、[セクション25](#25)に進んでください。
* コレクションを結合・解除したい場合は、[セクション26](#26)へ。

{{< linktarget "2" >}}

----

## 2. 新しいコレクションの作成

新しいコレクションを作成したいと思います。どのように作成するのでしょうか？

* 新しいコレクションが空であるか、1つの要素を持つ場合は、[セクション3](#3)に進みます。
* 新しいコレクションが既知のサイズである場合、[セクション4](#4)に進みます。
* 新しいコレクションが無限になる可能性がある場合は、[セクション7](#7)へ。
* コレクションの大きさがわからない場合は、[セクション8](#8)へ。

{{< linktarget "3" >}}

----

## 3. 新しい空または1要素のコレクションの作成

空または1要素のコレクションを新規に作成する場合は、以下の関数を使用します。

* [`empty : 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L142).
  与えられた型の空のリストを返します。
* [`singleton : value:'T -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L635).
  1つの要素だけを含むリストを返します。

コレクションのサイズがあらかじめわかっている場合は、一般的に別の関数を使った方が効率的です。以下の[セクション4](#4)を参照してください。

### 使用例

```fsharp
let list0 = List.empty
// list0 = []

let list1 = List.singleton "hello"
// list1 = ["hello"]
```


{{< linktarget "4" >}}

----

## 4. サイズが既知の新しいコレクションの作成

* コレクションの全ての要素が同じ値を持つ場合は、[セクション5](#5)へ。
* コレクションの要素が異なる可能性がある場合には、[セクション6](#6)に進みます。


{{< linktarget "5" >}}

----

## 5. 各要素が同じ値を持つ、サイズが既知の新しいコレクションの作成

各要素が同じ値を持つ、サイズが既知の新しいコレクションを作成したい場合は、`replicate`を使用します。

* [`replicate : count:int -> initial:'T -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L602).
  与えられた初期値を複製することで、コレクションを作成します。
* (Array only) [`create : count:int -> value:'T -> 'T[]`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/array.fsi#L125).
  すべての要素が与えられた初期値である配列を作成します。
* (Array only) [`zeroCreate : count:int -> 'T[]`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/array.fsi#L467).
  初期状態としてデフォルトの値が入力されている配列を作成します。

`Array.create` は基本的に `replicate` と同じですが (実装は微妙に異なります!)、`replicate` は F# v4 で `Array` に対してのみ実装されました。

### 使用例

```fsharp
let repl = List.replicate 3 "hello"
// val repl : string list = ["hello"; "hello"; "hello"]

let arrCreate = Array.create 3 "hello"
// val arrCreate : string [] = [|"hello"; "hello"; "hello"|]

let intArr0 : int[] = Array.zeroCreate 3
// val intArr0 : int [] = [|0; 0; 0|]

let stringArr0 : string[] = Array.zeroCreate 3
// val stringArr0 : string [] = [|null; null; null|]
```

なお，`zeroCreate`では，ターゲットの型がコンパイラに知られている必要があります。


{{< linktarget "6" >}}

----

## 6. 各要素が異なる値を持つ既知のサイズの新しいコレクションの作成

各要素の値が異なる可能性がある既知のサイズのコレクションを新規作成する場合は、次の3つの方法のいずれかを選択できます:

* [`init:length:int->initializer: (int->'T) ->'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L347)。
  各インデックスが引数として与えられるジェネレータ関数を呼び出してコレクションを作成します。
* listとarrayでは、` [1;2;3] ` (lists) や` [|1;2;3|] ` (arrays) のようなリテラルも使用できます。
* list、array、seqには、`for..in..do..yield`を使用できます。

### 使用例

```fsharp
// リストイニシャライザの使用
let listInit1 = List.init 5 (fun i-> i*i)
// val listInit1 : int list = [0; 1; 4; 9; 16]

// リスト内包を使用
let listInit2 = [for i in [1..5] do yield i*i]
// val listInit2 : int list = [1; 4; 9; 16; 25]

// リテラル
let listInit3 = [1; 4; 9; 16; 25]
// val listInit3 : int list = [1; 4; 9; 16; 25]

let arrayInit3 = [|1; 4; 9; 16; 25|]
// val arrayInit3 : int [] = [|1; 4; 9; 16; 25|]
```

リテラル構文では，インクリメントも可能です:

```fsharp
// リテラルで＋2の増分
let listOdd= [1..2..10]
// val listOdd : int list = [1; 3; 5; 7; 9]
```

内包構文は，複数回の`yield`が可能なので，さらに柔軟性があります．

```fsharp
// リスト内包を使う
let listFunny = [
    for i in [2..3] do
        yield i
        yield i*i
        yield i*i*i
        ]
// val listFunny : int list = [2; 4; 8; 3; 9; 27]
```

また，クイック＆ダーティなインラインフィルタとして使うこともできます．

```fsharp
let primesUpTo n =
   let rec sieve l  =
      match l with
      | [] -> []
      | p::xs ->
            p :: sieve [for x in xs do if (x % p) > 0 then yield x]
   [2..n] |> sieve

primesUpTo 20
// [2; 3; 5; 7; 11; 13; 17; 19]
```

他にも2つのトリックがあります。

* `yield!`を使うと、単一の値ではなくリストを返すことができます。
* 再帰を使うこともできます。

以下は，この2つのトリックを使って，10までの数字を2つずつ数える例です。

```fsharp
let rec listCounter n = [
    if n <= 10 then
        yield n
        yield! listCounter (n+2)
    ]

listCounter 3
// val it : int list = [3; 5; 7; 9]
listCounter 4
// val it : int list = [4; 6; 8; 10]
```

{{< linktarget "7" >}}

----

## 7. 新しい無限のコレクションの作成

無限リストを作りたい場合は、リストや配列ではなくseqを使う必要があります。

* [`initInfinite : initializer:(int -> 'T) -> seq<'T>`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/seq.fsi#L599)
  新しいシーケンスを生成し、イテレートされると、与えられた関数を呼び出して連続した要素を返します。
* 再帰ループでseq内包表記を使用して、無限シーケンスを生成することもできます。

### 使用例

```fsharp
// ジェネレータ版
let seqOfSquares = Seq.initInfinite (fun i -> i*i)
let firstTenSquares = seqOfSquares |> Seq.take 10

firstTenSquares |> List.ofSeq // [0; 1; 4; 9; 16; 25; 36; 49; 64; 81]

// 再帰バージョン
let seqOfSquares_v2 =
    let rec loop n = seq {
        yield n * n
        yield! loop (n+1)
        }
    loop 1
let firstTenSquares_v2 = seqOfSquares_v2 |> Seq.take 10
```

{{< linktarget "8" >}}

----

## 8. 不定形サイズの新しいコレクションの作成

事前にコレクションのサイズが分からないこともあります。この場合、停止信号を受け取るまで要素を追加し続ける関数が必要です。
ここでは`unfold`があなたの友達です。"停止の合図"はあなたが`None` (停止) を返すか`Some` (続行) を返すかです。

* [`unfold : generator:('State -> ('T * 'State) option) -> state:'State -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L846) 。
  与えられた演算によって生成された要素のコレクションを返します。

### 使用例

この例では、空行が入るまでループでコンソールから読み取りを行います:

```fsharp
let getInputFromConsole lineNo =
    let text = System.Console.ReadLine()
    if System.String.IsNullOrEmpty(text) then
        None
    else
        // return value and new threaded state
        // "text" will be in the generated sequence
        Some (text,lineNo+1)

let listUnfold = List.unfold getInputFromConsole 1
```

`unfold`では、ジェネレーターによって状態がスレッド化される必要があります。無視してもかまいません (上記の`ReadLine`の例のように) し、
これを使用して、それまでに行ったことを記録することもできます。たとえば、`unfold`を使用してFibonacci級数のジェネレータを作成できます:

```fsharp
let fibonacciUnfolder max (f1,f2)  =
    if f1 > max then
        None
    else
        // return value and new threaded state
        let fNext = f1 + f2
        let newState = (f2,fNext)
        // f1 will be in the generated sequence
        Some (f1,newState)

let fibonacci max = List.unfold (fibonacciUnfolder max) (1,1)
fibonacci 100
// int list = [1; 1; 2; 3; 5; 8; 13; 21; 34; 55; 89]
```

{{< linktarget "9" >}}

----

## 9. 一つのリストを扱う場合

1つのリストで作業する場合

* 既知の位置にある要素を取得したい場合は、[セクション10](#10)へ。
* 検索して1つの要素を取得したい場合は、[セクション11](#11)へ。
* コレクションのサブセットを取得したい場合は、[セクション12](#12)へ
* コレクションをより小さなコレクションに分割、チャンク、またはグループ化したい場合は、[セクション13](#13)へ。
* コレクションを単一の値に集約、または要約したい場合は、[セクション14](#14)へ。
* 要素の順序を変更したい場合は、[セクション15](#15)へ。
* コレクション内の要素をテストしたい場合は、[セクション16](#16)へ
* 各要素を別のものに変換したい場合は、[セクション17](#17)へ。
* 各要素を反復処理したい場合は、[セクション18](#18)に進んでください。
* 反復処理の中でステートをスレッドさせたい場合は、[セクション19](#19)へ。
* 反復処理やマッピング中に各要素のインデックスを知る必要がある場合は、[セクション20](#20)へ。
* コレクション全体を別のコレクションタイプに変換したい場合は、[セクション21](#21)へどうぞ。
* コレクション全体の動作を変更したい場合は、[セクション22](#22)に進んでください。
* コレクションをその場で変異させたい場合は、[セクション27](#27)へ。
* IDisposableで遅延コレクションを使用したい場合は、[セクション28](#28)へ。

{{< linktarget "10" >}}

----

## 10. 既知の位置にある要素の取得

以下の関数は、コレクション内の要素を位置ごとに取得します。

* [`head : list:'T list -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L333).
  コレクションの最初の要素を返します。
* [`last : list:'T list -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L398).
  コレクションの最後の要素を返します。
* [`item : index:int -> list:'T list -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L520).
  コレクションへのインデックスアクセス。最初の要素のインデックスは 0 です。
  注意: リストやシーケンスに `nth` や `item` を使うことは避けてください。これらはランダムアクセス用に設計されていないので、一般的には遅くなります。
* [`nth : list:'T list -> index:int -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L520)。
  `item` の古いバージョンです。NOTE: v4では非推奨です -- 代わりに `item` を使用してください。
* (Array only) [`get : array:'T[] -> index:int -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/array.fsi#L220).
  また、`item`の別バージョンです。
* [`exactlyOne : list:'T list -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L165).
  要素数1のコレクションから要素を返します。

しかし、コレクションが空だったらどうでしょうか? その場合、`head`と`last`は例外(ArgumentException)を発生して失敗します。

また、コレクションの中にインデックスが見つからない場合は？また、別の例外が発生します（リストの場合は ArgumentException、配列の場合は IndexOutOfRangeException）。

したがって、これらの関数は避け、以下のような `tryXXX` に相当するものを使用することをお勧めします。

* [`tryHead : list:'T list -> 'T option`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L775).
  コレクションの最初の要素、またはコレクションが空の場合は None を返します。
* [`tryLast : list:'T list -> 'T option`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L411).
  コレクションの最後の要素を返すか、コレクションが空の場合は None を返します。
* [`tryItem : index:int -> list:'T list -> 'T option`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L827).
  コレクションにインデックスアクセス、インデックスが有効でない場合は None を返します。

### 使用例

```fsharp
let head = [1;2;3] |> List.head
// val head : int = 1

let badHead : int = [] |> List.head
// System.ArgumentException: 入力リストは空でした。

let goodHeadOpt =
    [1;2;3] |> List.tryHead
// val goodHeadOpt : int option = Some 1

let badHeadOpt : int option =
    [] |> List.tryHead
// val badHeadOpt : int option = None

let goodItemOpt =
    [1;2;3] |> List.tryItem 2
// val goodItemOpt : int option = Some 3

let badItemOpt =
    [1;2;3] |> List.tryItem 99
// val badItemOpt : int option = None
```

前述のように、リストでは `item`関数は使用しないでください。たとえばもしあなたが命令型の出身である場合、
リストの各項目を処理したいなら、次のようなループを書くかもしれません:

```fsharp
// こんなことしないで!
let helloBad =
    let list = ["a";"b";"c"]
    let listSize = List.length list
    [ for i in [0..listSize-1] do
        let element = list |> List.item i
        yield "hello " + element
    ]
// val helloBad : string list = ["hello a"; "hello b"; "hello c"]
```

これはやめておきましょう。代わりに `map` のようなものを使ってください。これは、より簡潔で効率的です。

```fsharp
let helloGood =
    let list = ["a";"b";"c"]
    list |> List.map (fun element -> "hello " + element)
// val helloGood : string list = ["hello a"; "hello b"; "hello c"]
```

{{< linktarget "11" >}}

----

## 11.	検索による要素の取得

`find` や `findIndex` を使って、要素やそのインデックスを検索することができます。

* [`find : predicate:('T -> bool) -> list:'T list -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L201).
  与えられた関数がtrueを返す最初の要素を返します。
* [`findIndex : predicate:('T -> bool) -> list:'T list -> int`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L222).
  与えられた関数がtrueを返すような最初の要素のインデックスを返します。

また、逆方向に検索することもできます。

* [`findBack : predicate:('T -> bool) -> list:'T list -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L211).
  与えられた関数がtrueを返す最後の要素を返します。
* [`findIndexBack : predicate:('T -> bool) -> list:'T list -> int`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L233).
  与えられた関数がtrueを返す最後の要素のインデックスを返します。

しかし、もしその項目が見つからなかったらどうでしょうか？その場合，これらは例外（`KeyNotFoundException`）を伴って失敗します．

したがって、`head`や`item`と同様に、これらの関数は一般的には避け、以下のような`tryXXX`に相当するものを使用することをお勧めします。

* [`tryFind : predicate:('T -> bool) -> list:'T list -> 'T option`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L800).
  与えられた関数がtrueを返す最初の要素を返すか、そのような要素が存在しない場合は None を返します。
* [`tryFindBack : predicate:('T -> bool) -> list:'T list -> 'T option`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L809).
  与えられた関数がtrueを返す最後の要素、またはそのような要素が存在しない場合は None を返します。
* [`tryFindIndex : predicate:('T -> bool) -> list:'T list -> int option`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L819).
  与えられた関数がtrueを返す最初の要素のインデックスを返し、そのような要素が存在しない場合は None を返します。
* [`tryFindIndexBack : predicate:('T -> bool) -> list:'T list -> int option`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L837).
  与えられた関数がtrueを返した最後の要素のインデックスを返し、そのような要素が存在しない場合は None を返します。

もしも `map` を `find` の前に行っているのであれば、しばしば `pick` (あるいは `tryPick`) を使って、2つのステップを1つにまとめることができます。以下に使用例を示します。

* [`pick : chooser:('T -> 'U option) -> list:'T list -> 'U`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L561).
  与えられたchooser関数を要素に適用し、Someを返す最初の結果を返します。
* [`tryPick : chooser:('T -> 'U option) -> list:'T list -> 'U option`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L791).
  与えられたchooser関数を要素に適用し、Someを返す最初の結果を返し、そのような要素が存在しない場合はNoneを返します。


### 使用例

```fsharp
let listOfTuples = [ (1, "a"); (2, "b"); (3, "b"); (4, "a"); ]

listOfTuples |> List.find ( fun (x,y) -> y = "b")
// (2, "b")

listOfTuples |> List.findBack ( fun (x,y) -> y = "b")
// (3, "b")

listOfTuples |> List.findIndex ( fun (x,y) -> y = "b")
// 1

listOfTuples |> List.findIndexBack ( fun (x,y) -> y = "b")
// 2

listOfTuples |> List.find ( fun (x,y) -> y = "c")
// KeyNotFoundException
```

`pick`では、boolを返すのではなく、optionを返します。

```fsharp
listOfTuples |> List.pick ( fun (x,y) -> if y = "b" then Some (x,y) else None)
// (2, "b")
```


{{< linktarget "pick-vs-find" >}}

### Pick vs. Find

この"pick"関数は不要と思われるかもしれませんが、optionを返す関数を扱うときに便利です。

例えば、文字列を解析して、その文字列が有効なintであれば`Some int`を、そうでなければ`None`を返す関数`tryInt`があるとします。

```fsharp
// string -> int option
let tryInt str =
    match System.Int32.TryParse(str) with
    | true, i -> Some i
    | false, _ -> None
```

さて、リストの中から最初の有効なintを見つけたいとします。ざっくりとした方法としては

* `tryInt` を使ってリストをマッピングします。
* `find` を使って最初の `Some` を見つけます。
* `Option.get` を使ってoption内の値を取得する。

コードは以下のようになります．

```fsharp
let firstValidNumber =
    ["a";"2";"three"]
    // 入力のマッピング
    |> List.map tryInt
    // 最初のSomeを見つける
    |> List.find (fun opt -> opt.IsSome)
    // optionからデータを取得
    |> Option.get
// val firstValidNumber : int = 2
```

しかし、`pick`はこれらのステップをすべて一度に行います。そのため、コードはもっとシンプルになります。

```fsharp
let firstValidNumber =
    ["a";"2";"three"]
    |> List.pick tryInt
```

もし、`pick`と同じように多くの要素を返したい場合は、`choose`の使用を検討してください（[セクション12](#12)参照）。

{{< linktarget "12" >}}

----

## 12. コレクションから要素のサブセットを取得する

前のセクションでは、1つの要素の取得について説明しました。複数の要素を取得するにはどうすればよいでしょうか？ それは幸運なことです。たくさんの関数の中から選ぶことができます。

前方から要素を抽出するには、以下のようなものを使います。

* [`take: count:int -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L746).
  コレクションの最初のN個の要素を返します。
* [`takeWhile: predicate:('T -> bool) -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L756).
  指定された述語がtrueを返すまで元のコレクションの要素をすべて含むコレクションを返します (これ以降の要素は返しません) 。
* [`truncate: count:int -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L782).
  新しいコレクションに最大でN個の要素を返します。

後方から要素を取り出すには、以下のいずれかを使用します。

* [`skip: count:int -> list: 'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L644).
  最初のN個の要素を削除した後のコレクションを返します。
* [`skipWhile: predicate:('T -> bool) -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L652).
  与えられた述語がtrueを返す間、コレクションの要素をバイパスして、コレクションの残りの要素を返します。
* [`tail: list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L730).
  最初の要素を削除した後のコレクションを返します。

要素の他のサブセットを抽出するには、これらのいずれかを使用します。

* [`filter: predicate:('T -> bool) -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L241).
  与えられた関数がtrueを返したコレクションの要素だけを含む新しいコレクションを返します。
* [`except: itemsToExclude:seq<'T> -> list:'T list -> 'T list when 'T : equality`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L155).
  入力コレクションの中でitemsToExcludeシーケンスの中に含まれていない要素を持つ新しいコレクションを返します。このとき、値の比較には汎用ハッシュと等価比較を使用します。
* [`choose: chooser:('T -> 'U option) -> list:'T list -> 'U list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L55).
  与えられた関数をコレクションの各要素に適用します。関数がSomeを返す要素で構成されるコレクションを返します。
* [`where: predicate:('T -> bool) -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L866).
  与えられたpredicateがtrueを返すコレクションの要素だけを含む新しいコレクションを返します。
  NOTE: "where "は "filter "の同義語です。
* (配列のみ) `sub : 'T [] -> int -> int -> 'T []`.
  Sub : 'T [] -> int -> 'T []`. 開始インデックスと長さで指定された指定のサブレンジを含む配列を作成します。
* スライス構文を使用することもできます。`myArray.[2..5]`. 例は以下を参照してください。

リストを個別の要素にするには、以下のいずれかを使用します。

* [`distinct: list:'T list -> 'T list when 'T : equality`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L107).
  重複を除外したコレクションを返します。これは、一般的な照合と等価比較の結果に基づくものです。
* [`distinctBy: projection:('T -> 'Key) -> list:'T list -> 'T list when 'Key : equality`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L118).
  指定されたキー生成関数から返されたキーの一般的な照合と等価比較に従い、重複を除外したコレクションを返します。

### 使用例

前方から要素を取り出す。

```fsharp
[1..10] |> List.take 3
// [1; 2; 3]

[1..10] |> List.takeWhile (fun i -> i < 3)
// [1; 2]

[1..10] |> List.truncate 4
// [1; 2; 3; 4]

[1..2] |> List.take 3
// System.InvalidOperationException: 入力シーケンスの要素数が不足しています。

[1..2] |> List.takeWhile (fun i -> i < 3)
// [1; 2]

[1..2] |> List.truncate 4
// [1; 2] // エラーはありません!
```

後ろから要素を取る

```fsharp
[1..10] |> List.skip 3
// [4; 5; 6; 7; 8; 9; 10]

[1..10] |> List.skipWhile (fun i -> i < 3)
// [3; 4; 5; 6; 7; 8; 9; 10]

[1..10] |> List.tail
// [2; 3; 4; 5; 6; 7; 8; 9; 10]

[1..2] |> List.skip 3
// System.ArgumentException: The index is outside the legal range.

[1..2] |> List.skipWhile (fun i -> i < 3)
// []

[1] |> List.tail |> List.tail
// System.ArgumentException: 入力リストは空でした。
```

要素の他のサブセットを抽出するには

```fsharp
[1..10] |> List.filter (fun i -> i%2 = 0) // さらに
// [2; 4; 6; 8; 10]

[1..10] |> List.where (fun i -> i%2 = 0) // 偶数
// [2; 4; 6; 8; 10]

[1..10] |> List.except [3;4;5]
// [1; 2; 6; 7; 8; 9; 10]
```

スライスを抽出するには

```fsharp
Array.sub [|1..10|] 3 5
// [|4; 5; 6; 7; 8|]

[1..10].[3..5]
// [4; 5; 6]

[1..10].[3..]
// [4; 5; 6; 7; 8; 9; 10]

[1..10].[..5]
// [1; 2; 3; 4; 5; 6]
```

リストのスライスは、ランダムアクセスではないので、遅いことに注意してください。しかし，配列のスライスは高速です。

別々の要素を取り出すには

```fsharp
[1;1;1;2;3;3] |> List.distinct
// [1; 2; 3]

[ (1,"a"); (1,"b"); (1,"c"); (2,"d")] |> List.distinctBy fst
// [(1, "a"); (2, "d")]
```


{{< linktarget "choose-vs-fliter" >}}

### Choose vs. Filter

`pick`と同様に、`choose`関数は扱いにくいかもしれませんが、オプションを返す関数を扱うときには便利です。

実際に、`choose`と`filter`の関係は、 [`pick`と`find`](#pick-vs-find) と同様です。この場合、シグナルはブール値のフィルタではなく、`Some`と`None`のどちらかです。

前述のように、文字列を解析し、その文字列が有効なintであれば`Some int`を返し、そうでなければ`None`を返す関数`tryInt`があるとします。

```fsharp
// string -> int option
let tryInt str =
    match System.Int32.TryParse(str) with
    | true, i -> Some i
    | false, _ -> None
```

次に、リスト内のすべての有効なintを検索するとします。大まかな方法は次のようになります。

* `tryInt`を使用してリストをマップする
* `Some`であるもののみを含めるフィルタ
* `Option.get`を使用して各オプション内から値を取得する

コードは次のようになります。

```fsharp
let allValidNumbers =
    ["a";"2";"three"; "4"]
    //入力をマップします
    |> List.map tryInt
    / /“Some”だけを含める
    |> List.filter (fun opt -> opt.IsSome)
    // 各optionからデータを取得する
    |> List.mapオプション.get
// val allValidNumbers : int list = [2; 4]
```

でも`choose`はこれらのステップをすべて一度に行うのです！そのため、コードは非常にシンプルになります。

```fsharp
let allValidNumbers =
    ["a";"2";"three"; "4"]
    |> List.choose tryInt
```

optionsのリストが既にある場合は、`id`を`choose`に渡すことで、1ステップで"Some"をフィルタリングして返すことができます。

```fsharp
let reduceOptions =
    [None; Some 1; None; Some 2]
    |> List.choose id
// val reduceOptions : int list = [1; 2]
```

最初の要素を`choose`と同じ方法で返したい場合は、`pick`を使用することを検討してください ( [section 11] (#11) を参照) 。

`choose`と同様の動作をしたいが、他のラッパー型 ( Success/Failure resultなど) を使用したい場合は、 [a discussion here](/posts/elevated-world-5/) を参照してください。

{{< linktarget "13" >}}

----

##13 パーティション、チャンク、グループ化

コレクションを分ける方法はたくさんあります。相違点については、使用例を参照してください。

* [`chunkBySize:chunkSize:int->list:'T list->'T list list list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L63).
  入力されたコレクションを最大`chunkSize`個のチャンクに分割します。
* [`groupBy:projection:('T ->'Key) ->list:'T list-> ('Key*'T list) list when'Key:equality`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L325).
  コレクションの各要素にキー生成関数を適用し、一意のキーのリストを生成します。各キーには、このキーに対応するすべての要素のリストが含まれます。
* [`pairwise:list:'T list-> ('T*'T) list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L541).
  入力コレクション内の各要素とその前の要素のコレクションを返します。ただし、1番目の要素は、2番目の要素の前の要素として返されます。
* (Seqを除く) [`partition:predicate: ('T->bool) ->list:'T list-> ('T list*'T list) `](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L551).
  コレクションを、与えられたpredicateの戻り値がtrueおよびfalseであるかに応じて2つのコレクションに分割します。
* (Seqを除く) [`splitAt:index:int->list:'T list-> ('T list*'T list)`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L688).
  指定されたインデックスの位置でコレクションを2つに分割します。
* [`splitInto:count:int->list:'T list->'T list list list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L137).
  入力コレクションを最大count個のチャンクに分割します。
* [`window : windowSize:int -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L875).
  入力コレクションから取得された要素を含むスライドウィンドウのリストを返します。各ウィンドウは新しいコレクションとして返されます。"ペアごとの"ウィンドウとは異なり、ウィンドウはコレクションです。
  タプルではありません。

### 使用例

```fsharp
[1..10] |> List.chunkBySize 3
// [[1; 2; 3]; [4; 5; 6]; [7; 8; 9]; [10]]
// 最後のチャンクには1つの要素があることに注意

[1..10] |> List.splitInto 3
// [[1; 2; 3; 4]; [5; 6; 7]; [8; 9; 10]]
// 最初のチャンクには4つの要素があることに注意してください。

['a'..'i'] |> List.splitAt 3
// (['a'; 'b'; 'c'], ['d'; 'e'; 'f'; 'g'; 'h'; 'i'])

['a'..'e'] |> List.pairwise
// [('a', 'b'); ('b', 'c'); ('c', 'd'); ('d', 'e')]

['a'..'e'] |> List.windowed 3
// [['a'; 'b'; 'c']; ['b'; 'c'; 'd']; ['c'; 'd'; 'e']]

let isEven i = (i%2 = 0)
[1..10] |> List.partition isEven
// ([2; 4; 6; 8; 10], [1; 3; 5; 7; 9])

let firstLetter (str:string) = str.[0]
["apple"; "alice"; "bob"; "carrot"] |> List.groupBy firstLetter
// [('a', ["apple"; "alice"]); ('b', ["bob"]); ('c', ["carrot"])]
```

`splitAt`と`pairwise`以外のすべての関数は、エッジケースも適切に処理します:

```fsharp
[1] |> List.chunkBySize 3
// [[1]]

[1] |> リスト.splitInto 3
// [[1]]

['a'; 'b'] |> List.splitAt 3
// InvalidOperationException: 入力シーケンスの要素数が不足しています。

['a'] |> List.pairwise
// InvalidOperationException: 入力シーケンスの要素数が不足しています。

['a'] |> List.windowed 3
// []

[1] |> List.partition isEven
// ([], [1])

[] |> List.groupBy firstLetter
//  []
```


{{< linktarget "14" >}}

----

## 14.	コレクションの集約または要約

コレクションの要素を集約する最も一般的な方法は `reduce` を使うことです。

* [`reduce : reduction:('T -> 'T -> 'T) -> list:'T list -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L584).
  コレクションの各要素に関数を適用します、この計算の途中結果はアキュレータ引数が保持します。
* [`reduceBack : reduction:('T -> 'T -> 'T) -> list:'T list -> 'T`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L595).
  コレクションの各要素に関数を末尾から適用します、この関数は途中の計算結果をアキュレータ引数に保持します。

また、頻繁に使用される集約のための特定のバージョンの `reduce` もあります。

* [`max : list:'T list -> 'T when 'T : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L482).
  Operators.maxを使用して比較し、コレクションの全要素のうち最大の要素を返します。
* [`maxBy : projection:('T -> 'U) -> list:'T list -> 'T when 'U : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L492).
  与えられた関数の結果に対してOperators.maxを使用して比較し、コレクションのすべての要素のうち最大のものを返します。
* [`min : list:'T list -> 'T when 'T : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L501).
  Operators.minを使用して比較し、コレクションのすべての要素のうち、最も小さい要素を返します。
* [`minBy : projection:('T -> 'U) -> list:'T list -> 'T when 'U : comparison `](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L511).
  与えられた関数の結果に対してOperators.minを使用して比較し、コレクションの全要素のうち最小の要素を返します。
* [`sum : list:'T list -> 'T when 'T has static members (+) and Zero`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L711).
  コレクションに含まれる要素の和を返します。
* [`sumBy : projection:('T -> 'U) -> list:'T list -> 'U when 'U has static members (+) and Zero`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L720)。
  コレクションの各要素に関数を適用して得られた結果の合計を返します。
* [`average : list:'T list -> 'T when 'T has static members (+) and Zero and DivideByInt`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L30).
  コレクション内の要素の平均値を返します。
  intsのリストは平均化できないことに注意してください -- floatまたはdecimalsにキャストする必要があります。
* [`averageBy : projection:('T -> 'U) -> list:'T list -> 'U when 'U has static members (+) and Zero and DivideByInt`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L43).
  コレクションの各要素に関数を適用して得られた結果の平均を返します。

最後に，いくつかのカウント関数を紹介します。

* [`length: list:'T list -> int`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L404).
  コレクションの長さを返します。
* [`countBy : projection:('T -> 'Key) -> list:'T list -> ('Key * int) list when 'Key : equality`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L129)。
  各要素にキー生成関数を適用し、一意なキーと元のコレクション内での出現数のタプルを返します。

### 使用例

`reduce`は初期状態を持たない`fold`の派生形です--`fold`の詳細については、 [セクション19](#19) を参照してください。
これを理解するために、各要素の間に演算子を挿入するという方法があります。

```fsharp
["a";"b";"c"] |> List.reduce (+)
// "abc"
```

は，以下と同じです．

```fsharp
"a" + "b" + "c"
```

別の例を示します。

```fsharp
[2;3;4] |> List.reduce (*)
// と同じです．
2 * 3 * 4
// 結果は24
```

要素を組み合わせる方法の中には、組み合わせる順番に依存するものがあり、そのため"reduce"には2つのバリエーションがあります。

* `reduce` はリストを順に進んでいきます。
* `reduceBack` は意外と知られていませんが、リストを後方に移動します。

この違いを見てみましょう。まずは`reduce`です:

```fsharp
[1;2;3;4] |> List.reduce (fun state x -> (state)*10 + x)

// built up from                // state at each step
1                               // 1
(1)*10 + 2                      // 12
((1)*10 + 2)*10 + 3             // 123
(((1)*10 + 2)*10 + 3)*10 + 4    // 1234

// 最終結果は1234
```

*同じ*組み合わせに`reduceBack`を使用すると、異なる結果が得られます！次のようになります:

```fsharp
[1;2;3;4] |> List.reduceBack (fun x state -> x + 10*(state))

// built up from                // state at each step
4                               // 4
3 + 10*(4)                      // 43
2 + 10*(3 + 10*(4))             // 432
1 + 10*(2 + 10*(3 + 10*(4)))    // 4321

// 最終結果は4321
```

関連する関数 `fold` と `foldBack` についての詳しい説明は [セクション19](#19) を参照してください。

他の集約関数はもっと簡単です。

```fsharp
type Suit = Club | Diamond | Spade | Heart
type Rank = Two | Three | King | Ace
let cards = [ (Club,King); (Diamond,Ace); (Spade,Two); (Heart,Three); ]

cards |> List.max        // (Heart, Three)
cards |> List.maxBy snd  // (Diamond, Ace)
cards |> List.min        // (Club, King)
cards |> List.minBy snd  // (Spade, Two)

[1..10] |> List.sum
// 55

[ (1, "a"); (2, "b") ] |> List.sumBy fst
// 3

[1..10] |> List.average
// 型 'int' は演算子 'DivideByInt' をサポートしていません

[1..10] |> List.averageBy float
// 5.5

[ (1, "a"); (2, "b") ] |> List.averageBy (fst >> float)
// 1.5

[1..10] |> List.length
// 10

[ ("a", "A"); ("b", "B"); ("a", "C") ] |> List.countBy fst
// [("a", 2); ("b", 1)]

[ ("a", "A"); ("b", "B"); ("a", "C") ] |> List.countBy snd
// [("A", 1); ("B", 1); ("C", 1)]
```

ほとんどの集約関数は空リストが嫌いです！安全のために`fold`関数のいずれかを使用することを検討してもよいでしょう-- [セクション19](#19) を参照してください。

```fsharp
let emptyListOfInts : int list = []

emptyListOfInts |> List.reduce (+)
// ArgumentException: 入力リストが空でした。

emptyListOfInts|> List.max
// ArgumentException: 入力シーケンスが空でした。

emptyListOfInts |> List.min
// ArgumentException: 入力シーケンスが空でした。

emptyListOfInts |> List.sum
// 0

emptyListOfInts|> List.averageBy float
// ArgumentException: 入力シーケンスが空でした。

let emptyListOfTuples : (int*int) list = []
emptyListOfTuples|> List.countBy fst
// (int * int) list = []
```

{{< linktarget "15" >}}

----

## 15.	要素の順序の変更

反転、ソート、パーミットを使って、要素の順序を変えることができます。以下のものはすべて *新しい* コレクションを返します。

* [`rev: list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L608).
  要素を逆順に並べた新しいコレクションを返します。
* [`sort: list:'T list -> 'T list when 'T : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L678).
  Operators.compareを使って、与えられたコレクションをソートします。
* [`sortDescending: list:'T list -> 'T list when 'T : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L705).
  Operators.compareを使って、与えられたコレクションを降順にソートします。
* [`sortBy: projection:('T -> 'Key) -> list:'T list -> 'T list when 'Key : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L670).
  与えられたコレクションを、与えられたプロジェクションによって得られたキーを使ってソートします。キーの比較には Operators.compare を使用します。
* [`sortByDescending: projection:('T -> 'Key) -> list:'T list -> 'T list when 'Key : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L697).
  与えられたコレクションを、与えられたプロジェクションによって得られたキーを使って、降順にソートします。キーの比較には Operators.compare を使用します。
* [`sortWith: comparer:('T -> 'T -> int) -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L661).
  与えられた比較関数を使って、与えられたコレクションをソートします。
* [`permute : indexMap:(int -> int) -> list:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L570)となります。
  指定された順列にしたがってすべての要素を順列化したコレクションを返します。

また、配列のみの関数で、その場でソートするものもあります。

* (配列のみ) [`sortInPlace: array:'T[] -> unit when 'T : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/array.fsi#L874).
  配列をその場で変更することにより、配列の要素をソートします。要素の比較には Operators.compare を使用します。
* (Array only) [`sortInPlaceBy: projection:('T -> 'Key) -> array:'T[] -> unit when 'Key : comparison`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/array.fsi#L858).
  キーに与えられたプロジェクションを使用して、配列をその場で変異させることにより、配列の要素をソートします。キーの比較には Operators.compare を使用します。
* (Array only) [`sortInPlaceWith: comparer:('T -> 'T -> int) -> array:'T[] -> unit`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/array.fsi#L867).
  与えられた比較関数を順序として、配列をその場で変異させることにより、配列の要素をソートします。

### 使用例

```fsharp
[1..5] |> List.rev
// [5; 4; 3; 2; 1]

[2;4;1;3;5] |> List.sort
// [1; 2; 3; 4; 5]

[2;4;1;3;5] |> List.sortDescending
// [5; 4; 3; 2; 1]

[ ("b","2"); ("a","3"); ("c","1") ]  |> List.sortBy fst
// [("a", "3"); ("b", "2"); ("c", "1")]

[ ("b","2"); ("a","3"); ("c","1") ]  |> List.sortBy snd
// [("c", "1"); ("b", "2"); ("a", "3")]

// コンパラーの例
let tupleComparer tuple1 tuple2  =
    if tuple1 < tuple2 then
        -1
    elif tuple1 > tuple2 then
        1
    else
        0

[ ("b", "2"); ("a", "3"); ("c", "1") ] |> List.sortWith tupleComparer
// [("a", "3"); ("b", "2"); ("c", "1")]

[1..10] |> List.permute (fun i -> (i + 3) % 10)
// [8; 9; 10; 1; 2; 3; 4; 5; 6; 7]

[1..10] |> List.permute (fun i -> 9 - i)
// [10; 9; 8; 7; 6; 5; 4; 3; 2; 1]
```

{{< linktarget "16" >}}

----

## 16.	コレクションの要素のテスト

これらの関数群は全てtrueかfalseを返します。

* [`contains: value:'T -> source:'T list -> bool when 'T : equality`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L97).
  コレクションが指定された要素を含んでいるかどうかをテストします。
* [`exists: predicate:('T -> bool) -> list:'T list -> bool`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L176).
  コレクションの任意の要素が、指定された述語を満たすかどうかをテストします。
* [`forall: predicate:('T -> bool) -> list:'T list -> bool`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L299).
  コレクションの全ての要素が与えられた述語を満たすかどうかをテストします。
* [`isEmpty: list:'T list -> bool`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L353).
  コレクションに要素が含まれていない場合はtrue、そうでない場合はfalseを返します。

### 使用例

```fsharp
[1..10] |> List.contains 5
// true

[1..10] |> List.contains 42
// false

[1..10] |> List.exists (fun i -> i > 3 && i < 5)
// true

[1..10] |> List.exists (fun i -> i > 5 && i < 3)
// false

[1..10] |> List.forall (fun i -> i > 0)
// true

[1..10] |> List.forall (fun i -> i > 5)
// false

[1..10] |> List.isEmpty
// false
```

{{< linktarget "17" >}}

----

## 17.	各要素を異なるものに変換する

私は関数型プログラミングを"変換指向プログラミング"と考えることもありますが、このアプローチの最も基本的な要素の1つが`map` (LINQでは"`Select`") です。
実際、私は全シリーズを [ここ](/posts/elevated-world/) に割いています。

* [`map:mapping: ('T->'U) ->list:'T list->'U list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L419)
  指定された関数をコレクションの各要素に適用して、結果を要素とする新しいコレクションを作成します。

場合によっては、各要素がリストにマップされ、それらのリストを平坦化したいこともあります。その場合は、`collect` (LINQでは`SelectMany`) を使用します。

* [`collect:mapping: ('T->'U list) ->list:'T list->'U list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L70)
  リストの各要素に対して、指定された関数を適用します。すべての結果を連結し、結合されたリストを返します。

その他の変換関数は次のとおりです。

* [section 12](#12) の`choose`は、mapとoptionフィルタを組み合わせたものです。
* (シーケンスのみ) [`cast:source:IEnumerable->seq<'T>`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/seq.fsi#L599)
  緩やかに型指定された`System.Collections`のシーケンスを型付きシーケンスとしてラッピングします。

### 使用例

リストとマッピング関数を受け取り、新しい変換済リストを戻す関数として、`map`を従来の方法で使用する例を次に示します:

```fsharp
let add1 x = x + 1

// リストを変換する関数としてのmap
[1..5] |> List.map add1
// [2; 3; 4; 5; 6]

// マップされるリストには何でも入ります！
let times2 x = x * 2
[ add1; times2] |> List.map (fun f -> f 5)
// [6; 10]
```

`map`は*関数変換*と見なすこともできます。要素から要素への関数をリストからリストへの関数に変換します。

```fsharp
let add1ToEachElement = List.map add1
// "add1ToEachElement"は、intsからintsではなく、リストからリストに変換します。
// val add1ToEachElement : (int list -> int list)

// ここで使用
[1..5] |> add1ToEachElement
// [2; 3; 4; 5; 6]
```

`collect` はリストをフラットにする働きがあります。すでにリストがある場合は、`id`と一緒に`collect`を使ってリストを平坦化することができます。

```fsharp
[2..5] |> List.collect (fun x -> [x; x*x; x*x*x] )
// [2; 4; 8; 3; 9; 27; 4; 16; 64; 5; 25; 125]

// コレクトで"id"を使用
let list1 = [1..3]
let list2 = [4...6]
[list1; list2] |> List.collect id
// [1; 2; 3; 4; 5; 6]
```

### Seq.cast

最後に、ジェネリックではなく特殊なコレクションクラスを持つBCLの古い部分を扱うときには、`Seq.cast`が便利です。

例えば、Regexライブラリにはこの問題があり、`MatchCollection`が`IEnumerable<T>`ではないため、以下のコードはコンパイルされません。
Note: F#v5の場合はコンパイル可能

```fsharp
open System.Text.RegularExpressions

let matches =
    let pattern = "\d\d\d"
    let matchCollection = Regex.Matches("123 456 789",pattern)
    matchCollection
    |> Seq.map (fun m -> m.Value) // ERROR
    // ERROR: 型'MatchCollection'は型'seq<'a>'と互換性がありません。
    |> Seq.toList
```

修正方法は、`MatchCollection`を`Seq<Match>`にキャストすることで、コードがうまく動作するようになります。

```fsharp
let matches =
    let pattern = "\d\d\d"
    let matchCollection = Regex.Matches("123 456 789",pattern)
    matchCollection
    |> Seq.cast<Match>
    |> Seq.map (fun m -> m.Value)
    |> Seq.toList
// output = ["123"; "456"; "789"]
```

{{< linktarget "18" >}}

----

## 18. 各要素の反復

通常、コレクションを処理する場合、`map`を使用して各要素を新しい値に変換します。しかし時として、「単位関数」有用な値を*生成しない*関数を使用してすべての要素を処理する必要があります。

* [`iter:action: ('T->unit) ->list:'T list->unit`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L367)
  指定された関数をコレクションの各要素に適用します。
* または、for-loopを使用することもできます。forループ内の式は*必ず*`unit`を返します。

### 使用例

unit関数の最も典型的な例は、コンソールへの出力、データベースの更新、キューにメッセージを書き込むといった副作用に関するものです。
以下の例では、unit関数として`printfn`を使用します:

```fsharp
[1..3] |> List.iter (fun i -> printfn "i is %i" i)
(*
i is 1
i is 2
i is 3
*)

// or using partial application
[1..3] |> List.iter (printfn "i is %i")

// or using a for loop
for i = 1 to 3 do
    printfn "i is %i" i

// or using a for-in loop
for i in [1..3] do
    printfn "i is %i" i
```

前述のように、`iter`やfor-loopの中の式はunitを返さなければなりません。 以下の例では，要素に1を加えようとすると，コンパイラエラーが発生します．

```fsharp
[1..3] |> List.iter (fun i -> i + 1)
//                               ~~~
// ERROR error FS0001: 型 'unit' は型 'int' と一致しません

// for-loop式は*必ず*unitを返さなければならない
for i in [1..3] do
     i + 1 // ERROR
     // この式の結果の型は 'int' で、暗黙的に無視されます。
     // 'ignore' を使用してこの値を明示的に破棄してください...
```

もし、これがコードの論理的なバグではないと確信していて、このエラーを除去したい場合は、結果をパイプで`ignore`に送ることができます。

```fsharp
[1..3] |> List.iter (fun i -> i + 1 |> ignore)

for i in [1..3] do
     i + 1 |> ignore
```

{{< linktarget "19" >}}

----

## 19.	イテレーションによるステートのスレッド化

`fold`関数はコレクションの中で最も基本的で強力な関数です。他のすべての関数 (`unfold`のようなジェネレータを除く) は、この関数で記述できます。次の例を参照してください:

* [`fold<'T,'State> : folder:('State -> 'T -> 'State) -> state:'State -> list:'T list -> 'State`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L254).
  コレクションの各要素に関数を適用し、その関数が計算によってアキュムレータ引数をスレッド化します。
* [`foldBack<'T,'State> : folder:('T -> 'State -> 'State) -> list:'T list -> state:'State -> 'State`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L276).
  コレクションの各要素に関数を適用します。最後から始めて、計算を介してアキュムレータ引数をスレッド化します。
  警告:無限リストでの`Seq.foldBack`の使用は注意してください!ランタイムはあなたを笑って、その後静かになります。

`fold`関数は"fold left"と呼ばれることが多く、`foldBack`は"fold right"と呼ばれることが多いです。

`scan`関数は`fold`に似ていますが、中間結果を返すため、反復処理のトレースや監視に使用できます。

* [`scan<'T,'State>  : folder:('State -> 'T -> 'State) -> state:'State -> list:'T list -> 'State list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L619).
  `fold`と似ていますが、中間結果と最終結果の両方を返します。
* [`scanBack<'T,'State> : folder:('T -> 'State -> 'State) -> list:'T list -> state:'State -> 'State list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L627).
  `foldBack`と似ていますが、中間結果と最終結果の両方を返します。

双子のように`scan`は"scan left"、`scanBack'は"scan right"と呼ばれます。

最後に`mapFolder`は`map`と`fold`を1つの驚異的な力へと合成されます。`map`と`fold`を別々に使用するよりも複雑ですが、より効率的です。

* [`mapFold<'T,'State,'Result> : mapping:('State -> 'T -> 'Result * 'State) -> state:'State -> list:'T list -> 'Result list * 'State`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L447).
  mapとfoldの合成です。与えられた関数を入力コレクションの各要素に適用した結果を要素とする新しいコレクションを構築します。この関数は、計算結果を累積するためにも使用されます。
* [`mapFoldBack<'T,'State,'Result> : mapping:('T -> 'State -> 'Result * 'State) -> list:'T list -> state:'State -> 'Result list * 'State`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L456).
  mapとfoldBackを組み合わせたものです。与えられた関数を入力コレクションの各要素に適用した結果を要素とする新しいコレクションを構築します。この関数は、計算結果を累積するためにも使用されます。

### `fold`の例

`fold`は`reduce`に似ていますが、初期状態のパラメータが追加される点が異なります:

```fsharp
["a"; "b"; "c"] |> List.fold (+) "hello: "
// "hello: abc"
// "hello: " + "a" + "b" + "c"

[1;2;3] |> List.fold (+) 10
// 16
// 10 + 1 + 2 + 3
```

`reduce`と同様に、`fold`と`foldBack`は全く異なる答えを出すことができます:

```fsharp
[1;2;3;4] |> List.fold (fun state x -> (state)*10 + x) 0
                                // state at each step
1                               // 1
(1)*10 + 2                      // 12
((1)*10 + 2)*10 + 3             // 123
(((1)*10 + 2)*10 + 3)*10 + 4    // 1234
// 最終結果は1234
```

そして、これが`foldBack`バージョンです:

```fsharp
List.foldBack (fun x state -> x + 10*(state)) [1;2;3;4] 0
                                // state at each step
4                               // 4
3 + 10*(4)                      // 43
2 + 10*(3 + 10*(4))             // 432
1 + 10*(2 + 10*(3 + 10*(4)))    // 4321
// 最終結果は4321
```

ただし、`foldBack` は `fold` とはパラメータの順番が異なります。リストは最後から2番目、初期状態は最後になりますので、パイピングの利便性は低くなります。

### 再帰と反復

`fold` と `foldBack` を混同しがちです。私は、`fold` は *反復* で、`foldBack` は *再帰* だと考えると良いと思います。

例えば、リストの和を計算したいとしましょう。反復的な方法としては、for-loopを使用します。
ミュータブルなアキュムレータを用意して，それを反復しながら更新していきます．

```fsharp
let iterativeSum list =
    let mutable total = 0
    for e in list do
        total <- total + e
    total // sumを返す
```

一方、再帰的なアプローチでは次のようになります。
リストにheadとtailがある場合、まずtail（より小さいリスト）の合計を計算し、次にheadをそれに加えます。

毎回、tailはどんどん小さくなり、空になった時点で終了します。

```fsharp
let rec recursiveSum list =
    match list with
    | [] ->
        0
    | head::tail ->
        head + (recursiveSum tail)
```

どちらのアプローチが良いのでしょうか?

集計の場合、反復方法 (`fold`) が最も理解しやすいことがよくあります。
しかし、新しいリストを作成する場合などは、再帰的な方法 (`foldBack`) の方が理解しやすいでしょう。

たとえば、各要素を対応する文字列に変換する関数を一から作成しようとした場合、
こんな風に書くかもしれません:

```fsharp
let rec mapToString list =
    match list with
    | [] ->
        []
    | head::tail ->
        head.ToString() :: (mapToString tail)

[1..3] |> mapToString
// ["1"; "2"; "3"]
```

`foldBack`を使用すると、同じロジック”をそのまま”転送できます。

* 空のリストに対するアクション = `[]`
* 空でないlistに対するアクション = `head.ToString() :: state`

結果の関数は次のとおりです:

```fsharp
let foldToString list =
    let folder head state =
        head.ToString() :: state
    List.foldBack folder list []

[1..3] |> foldToString
// ["1"; "2"; "3"]
```

一方、`fold`の大きな利点は、"インライン"に使うのが簡単なことです、なぜならば、パイプとの相性が良いからです。

幸運なことに、最後のリストを反転させれば、`foldBack`のように`fold`を (少なくともリスト構築のために) 使うことができます。

```fsharp
// "foldToString"のインライン版
[1..3]
|> List.fold (fun state head -> head.ToString() :: state) []
|> List.rev
// ["1"; "2"; "3"]
```

### `fold`を使って他の関数を実装する

先に述べたように、`fold`はリストを操作するためのコアな関数で、他のほとんどの関数をエミュレートすることができます。
カスタムの実装ほど効率的ではないかもしれませんが。

例えば、`fold`を使って実装した`map`を紹介します:

```fsharp
/// 関数 "f"を全ての要素にマッピングする
let myMap f list =.
    // ヘルパー関数
    let folder state head =
        f head :: state

    // メインフロー
    list
    |> List.fold folder []
    |> List.rev

[1..3] |> myMap (fun x -> x + 2)
// [3; 4; 5]
```

また、`fold`を使って実装した`filter`は以下の通りです:

```fsharp
/// "pred"がtrueである要素の新しいリストを返す
let myFilter pred list =.
    // ヘルパー関数
    let folder state head =
        if pred head then
            head :: state
        else
            state

    // メインフロー
    list
    |> List.fold folder []
    |> List.rev

let isOdd n = (n%2=1)
[1..5] |> myFilter isOdd
// [1; 3; 5]
```

もちろん、他の関数も同様の方法でエミュレートできます。

### `scan`の例

先ほど、`fold`の中間ステップの例を紹介しました。

```fsharp
[1;2;3;4] |> List.fold (fun state x -> (state)*10 + x) 0
                                // 各ステップでの状態
1                               // 1
(1)*10 + 2                      // 12
((1)*10 + 2)*10 + 3             // 123
(((1)*10 + 2)*10 + 3)*10 + 4    // 1234
// 最終結果は1234
```

この例では、中間状態を手動で計算する必要がありました。

もし、`scan`を使っていたら、その中間状態は無料で手に入っていたでしょうね。

```fsharp
[1;2;3;4] |> List.scan (fun state x -> (state)*10 + x) 0
// 左から積み上げていく ===> [0; 1; 12; 123; 1234]
```

`scanBack`も同じように動作しますが、もちろん逆方向です。

```fsharp
List.scanBack (fun x state -> (state)*10 + x) [1;2;3;4] 0
// [4321; 432; 43; 4; 0] <=== 右から積み上げていく
```

`foldBack`と同様に、右方向にスキャンする場合と左方向にスキャンする場合とでは、パラメータの順序が逆になります。

### `scan`で文字列を切り詰める

ここでは、`scan`が役立つ例を紹介します。例えば、ニュースサイトを運営していて、見出しが50文字に収まるようにする必要があるとします。

文字列を50で切り詰めることもできますが、それでは見た目が悪いです。その代わりに、単語の境界で切り捨てを終了させたいとします。

ここでは、`scan`を使ってそれを行う一つの方法を紹介します。

* 見出しを単語に分割します。
* `scan`を使用して単語を連結し、それぞれに追加の単語を追加したフラグメントのリストを生成します。
* 50文字以下の最長のフラグメントを取得します。

```fsharp
// まず、テキストを単語に分割します。
let text = "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor."
let words = text.Split(' ')
// [|"Lorem"; "ipsum"; "dolor"; "sit"; ... ]

// 一連のフラグメントを蓄積する
let fragments = words |> Seq.scan (fun frag word -> frag + " " + word) ""
(*
" Lorem"
" Lorem ipsum"
" Lorem ipsum dolor"
" Lorem ipsum dolor sit"
" Lorem ipsum dolor sit amet,"
etc
*)

// 50以下の最長のフラグメントを取得
let longestFragUnder50 =
    fragments
    |> Seq.takeWhile (fun s -> s.Length <= 50)
    |> Seq.last

// 最初の空白を切り捨てる
let longestFragUnder50Trimmed =
    longestFragUnder50 |> (fun s -> s.[1..])

// 結果はこうなります。
// "Lorem ipsum dolor sit amet, consectetur"
```

Array.scan "ではなく、"Seq.scan "を使っていることに注意してください。これは、遅延スキャンを行い、必要のないフラグメントを作らなくて済むようにするためです。

最後に、完全なロジックをユーティリティー関数として示します:

```fsharp
// 関数としての全体像
let truncText max (text:string) =
    if text.Length <= max then
        text
    else
        text.Split(' ')
        |> Seq.scan (fun frag word -> frag + " " + word) ""
        |> Seq.takeWhile (fun s -> s.Length <= max-3)
        |> Seq.last
        |> (fun s -> s.[1..] + "...")

"a small headline" |> truncText 50
// "a small headline"

text |> truncText 50
// "Lorem ipsum dolor sit amet, consectetur..."
```

もちろん、これよりも効率的な実装があることは承知していますが、この小さな例が `scan` の力を示してくれれば幸いです。

### `mapFold` の例

`mapFold` 関数は、マップと折り返しを一度に行うことができるので、便利な場合があります。

ここでは、`mapFold`を使って、足し算と和算を一度に行う例を紹介します。

```fsharp
let add1 x = x + 1

// マップを使ったadd1
[1..5] |> List.map (add1)
// 結果 => [2; 3; 4; 5; 6]

// foldを使ったsum
[1..5] |> List.fold (fun state x -> state + x) 0
// 結果 => 15

// mapFoldを使ったマップとサム
[1..5] |> List.mapFold (fun state x -> add1 x, (state + x)) 0
// 結果 => ([2; 3; 4; 5; 6], 15)
```


{{< linktarget "20" >}}

----

## 20. 各要素のインデックスを扱う

反復処理を行う際に、要素のインデックスが必要になることがよくあります。ミュータブルカウンタを使うこともできますが、ここはひとつ、ライブラリに任せてみませんか？

* [`mapi: mapping:(int -> 'T -> 'U) -> list:'T list -> 'U list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L465).
  `map`と同じですが、整数のインデックスも関数に渡されます。`map` の詳細については [セクション 17](#17) を参照してください。
* [`iteri: action:(int -> 'T -> unit) -> list:'T list -> unit`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L382).
  `iter` と同様ですが、整数のインデックスも関数に渡されます。`iter` の詳細については [セクション 18](#18) を参照してください。
* [`indexed: list:'T list -> (int * 'T) list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L340).
  入力リストの対応する要素を要素とし、各要素の(0からの)インデックスをペアにした新しいリストを返します。


### 使用例

```fsharp
['a'..'c'] |> List.mapi (fun index ch -> sprintf "the %ith element is '%c'" index ch)
// ["the 0th element is 'a'"; "the 1th element is 'b'"; "the 2th element is 'c'"]

// 部分適用で
['a'..'c'] |> List.mapi (sprintf "the %ith element is '%c'")
// ["the 0th element is 'a'"; "the 1th element is 'b'"; "the 2th element is 'c'"]

['a'..'c'] |> List.iteri (printfn "the %ith element is '%c'")
(*
the 0th element is 'a'
the 1th element is 'b'
the 2th element is 'c'
*)
```

`indexed` は，インデックスを持つタプルを生成します -- `mapi` の特定の使い方のショートカットです．

```fsharp
['a'..'c'] |> List.mapi (fun index ch -> (index, ch) )
// [(0, 'a'); (1, 'b'); (2, 'c')]

// "indexed "は上記を短くしたもの
['a'..'c'] |> List.indexed
// [(0, 'a'); (1, 'b'); (2, 'c')]
```


{{< linktarget "21" >}}

----

## 21. コレクション全体を異なるコレクションタイプに変換する

ある種類のコレクションから別の種類のコレクションに変換する必要があることがよくあります。これらの関数はこれを行います。

`ofXXX`関数は、`XXX`からモジュールタイプへの変換に使用されます。例えば、`List.ofArray`は配列をリストに変換します。

* (Arrayを除く) [`ofArray : array:'T[] -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L526).
  与えられた配列から新しいコレクションを構築します。
* (Seqを除く) [`ofSeq: source:seq<'T> -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L532).
  与えられた列挙可能なオブジェクトから新しいコレクションを構築します。
* (Listを除く) [`ofList: source:'T list -> seq<'T>`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/seq.fsi#L864).
  与えられたリストから新しいコレクションを構築します。

`toXXX`はモジュールタイプからタイプ`XXX`への変換に使われます。例えば、`List.toArray` はリストを配列に変換します。

* (Arrayを除く) [`toArray: list:'T list -> 'T[]`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L762).
  与えられたコレクションから配列を構築します。
* (Seqを除く) [`toSeq: list:'T list -> seq<'T>`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L768).
  与えられたコレクションをシーケンスとして表示します。
* (Listを除く) [`toList: source:seq<'T> -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/seq.fsi#L1189).
  与えられたコレクションからリストを構築します。

### 使用例

```fsharp
[1..5] |> List.toArray      // [|1; 2; 3; 4; 5|]
[1..5] |> Array.ofList      // [|1; 2; 3; 4; 5|]
// など
```

### Disposablesでのシーケンスの使用

これらの変換関数の重要な使い方として、遅延列挙（`seq`）を `list` のような完全に評価されるコレクションに変換することがあります。これは特に
ファイルハンドルやデータベース接続のような使い捨てのリソースがある場合に重要です．シーケンスをリストに変換しないと
要素へのアクセスでエラーが発生する可能性があります。 詳しくは[セクション28](#28)をご覧ください。



{{< linktarget "22" >}}

----

## 22. コレクション全体の振る舞いを変える

コレクション全体の動作を変更する特別な関数がいくつかあります（Seqのみ）。

* (Seqのみ) [`cache: source:seq<'T> -> seq<'T>`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/seq.fsi#L98).
  入力シーケンスをキャッシュしたものに対応するシーケンスを返します。この結果のシーケンスは、入力シーケンスと同じ要素を持ちます。この結果
  は複数回、列挙することができます。入力シーケンスは最大で1回、必要な範囲内でのみ列挙されます。
* (Seqのみ) [`readonly : source:seq<'T> -> seq<'T>`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/seq.fsi#L919).
  与えられたシーケンスオブジェクトにデリゲートする新しいシーケンスオブジェクトを構築します。これにより、元のシーケンスが再発見されたり、タイプキャストによって変異したりすることがないようになります。
* (Seqのみ) [`delay : generator:(unit -> seq<'T>) -> seq<'T>`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/seq.fsi#L221).
  与えられたシーケンスの遅延指定から構築されたシーケンスを返します。

### `cache` の例

以下に `cache` の使用例を示します。

```fsharp
let uncachedSeq = seq {
    for i = 1 to 3 do
        printfn "Calculating %i" i
        yield i
    }

// 2回反復する
uncachedSeq |> Seq.iter ignore
uncachedSeq |> Seq.iter ignore
```

シーケンスを2回反復処理した結果は、期待通りの結果となりました。

```text
Calculating 1
Calculating 2
Calculating 3
Calculating 1
Calculating 2
Calculating 3
```

しかし、このシーケンスをキャッシュすると...

```fsharp
let cachedSeq = uncachedSeq |> Seq.cache

// 2回イテレートする
cachedSeq |> Seq.iter ignore
cachedSeq |> Seq.iter ignore
```

...すると、各要素は1回だけ出力されます．

```text
Calculating 1
Calculating 2
Calculating 3
```

### `readonly` の例

以下に、シーケンスの基本的な型を隠すために `readonly` を使用した例を示します。

```fsharp
// シーケンスの基礎となる型を表示する
let printUnderlyingType (s:seq<_>) =
    let typeName = s.GetType().Name
    printfn "%s" typeName

[|1;2;3|] |> printUnderlyingType
// Int32[]

[|1;2;3|] |> Seq.readonly |> printUnderlyingType
// mkSeq@589 // 一時的な型
```

### `delay` の例

ここでは、`delay`の例を紹介します。

```fsharp
let makeNumbers max =
    [ for i = 1 to max do
        printfn "Evaluating %d." i
        yield i ]

let eagerList =
    printfn "Started creating eagerList"
    let list = makeNumbers 5
    printfn "Finished creating eagerList"
    list

let delayedSeq =
    printfn "Started creating delayedSeq"
    let list = Seq.delay (fun () -> makeNumbers 5 |> Seq.ofList)
    printfn "Finished creating delayedSeq"
    list
```

上のコードを実行してみると、`eagerList`を作成するだけで、すべての"Evaluating"メッセージが表示されることがわかります。しかし、`delayedSeq`を作成しても、リストのイテレーションは行われません。

```text
Started creating eagerList
Evaluating 1.
Evaluating 2.
Evaluating 3.
Evaluating 4.
Evaluating 5.
Finished creating eagerList

Started creating delayedSeq
Finished creating delayedSeq
```

シーケンスが反復されたときに初めてリストの作成が行われます。

```fsharp
eagerList |> Seq.take 3 // 既にリストが作成されている
delayedSeq |> Seq.take 3 // リスト作成のトリガーとなる
```

delayを使う代わりに、次のように`seq`の中にリストを埋め込むこともできます。

```fsharp
let embeddedList = seq {
    printfn "Started creating embeddedList"
    yield! makeNumbers 5
    printfn "Finished creating embeddedList"
    }
```

`delayedSeq`と同様に、`makeNumbers`関数は、シーケンスが反復されるまで呼ばれません。

{{< linktarget "23" >}}

----

## 23. 2つのリストを扱う

2つのリストがある場合、map や fold のような一般的な関数のほとんどが類似しています。

* [`map2: mapping:('T1 -> 'T2 -> 'U) -> list1:'T1 list -> list2:'T2 list -> 'U list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L428).
  2つのコレクションの各要素のペアに指定された関数を適用し、その結果を要素とする新しいコレクションを作成します。
* [`mapi2: mapping:(int -> 'T1 -> 'T2 -> 'U) -> list1:'T1 list -> list2:'T2 list -> 'U list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L473)となります。
    `mapi`に似ていますが、同じ長さの2つのリストの各要素のペアをマッピングします。
* [`iter2: action:('T1 -> 'T2 -> unit) -> list1:'T1 list -> list2:'T2 list -> unit`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L375).
  指定された関数を2つのコレクションに同時に適用します。コレクションのサイズは一致している必要があります。
* [`iteri2: action:(int -> 'T1 -> 'T2 -> unit) -> list1:'T1 list -> list2:'T2 list -> unit`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L391)となります。
  `iteri`と同様に、同じ長さの2つのリストの各要素のペアをマッピングします。
* [`forall2: predicate:('T1 -> 'T2 -> bool) -> list1:'T1 list -> list2:'T2 list -> bool`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L314).
  この述語は、2つのコレクションの長さのうち小さい方の長さまで、2つのコレクションの要素にマッチするように適用されます。適用した結果がいずれもfalseを返した場合、全体の結果はfalseとなり、そうでなければtrueとなります。
* [`exists2: predicate:('T1 -> 'T2 -> bool) -> list1:'T1 list -> list2:'T2 list -> bool`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L191).
  predicateが適用される範囲は、コレクションの2つのうち、小さい方のサイズまでとなります。いずれかの処理でfalseが返された場合、全体の結果はfalse、そうでない場合はtrueになります。
* [`fold2<'T1,'T2,'State> : folder:('State -> 'T1 -> 'T2 -> 'State) -> state:'State -> list1:'T1 list -> list2:'T2 list -> 'State`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L266).
  2つのコレクションの対応する要素に関数を適用します、アキュムレータ引数に計算の途中結果が保持されます。
* [`foldBack2<'T1,'T2,'State> : folder:('T1 -> 'T2 -> 'State -> 'State) -> list1:'T1 list -> list2:'T2 list -> state:'State -> 'State`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L288).
  2つのコレクションの対応する要素に関数を末尾から適用します、アキュムレータ引数に計算の途中結果が保持されます。
* [`compareWith: comparer:('T -> 'T -> int) -> list1:'T list -> list2:'T list -> int`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L84).
  2つのコレクションについて、与えられた比較関数を使って各要素を比較します。返り値が0でない最初の結果を返します。 
  コレクションの終わりに達した場合はリスト長を比較して、最初のコレクションが短ければ -1 を、2番目のコレクションが短ければ 1 を返します。
* [セクション26: コレクションの結合と解除](#26)の `append`, `concat`, `zip` も参照してください。

### 使用例

これらの関数は簡単に使うことができます。

```fsharp
let intList1 = [2;3;4]
let intList2 = [5;6;7]

List.map2 (fun i1 i2 -> i1 + i2) intList1 intList2
// [7; 9; 11]

// TIP ||>演算子を使用して、タプルを2つの引数としてパイプ処理します
(intList1,intList2) ||> List.map2 (fun i1 i2 -> i1 + i2)
// [7; 9; 11]

(intList1,intList2) ||> List.mapi2 (fun index i1 i2 -> index,i1 + i2)
 // [(0, 7); (1, 9); (2, 11)]

(intList1,intList2) ||> List.iter2 (printf "i1=%i i2=%i; ")
// i1=2 i2=5; i1=3 i2=6; i1=4 i2=7;

(intList1,intList2) ||> List.iteri2 (printf "index=%i i1=%i i2=%i; ")
// index=0 i1=2 i2=5; index=1 i1=3 i2=6; index=2 i1=4 i2=7;

(intList1,intList2) ||> List.forall2 (fun i1 i2 -> i1 < i2)
// true

(intList1,intList2) ||> List.exists2 (fun i1 i2 -> i1+10 > i2)
// true

(intList1,intList2) ||> List.fold2 (fun state i1 i2 -> (10*state) + i1 + i2) 0
// 801 = 234 + 567

List.foldBack2 (fun i1 i2 state -> i1 + i2 + (10*state)) intList1 intList2 0
// 1197 = 432 + 765

(intList1,intList2) ||> List.compareWith (fun i1 i2 -> i1.CompareTo(i2))
// -1

(intList1,intList2) ||> List.append
// [2; 3; 4; 5; 6; 7]

[intList1;intList2] |> List.concat
// [2; 3; 4; 5; 6; 7]

(intList1,intList2) ||> List.zip
// [(2, 5); (3, 6); (4, 7)]
```

### ここにはない関数が必要ですか？

`fold2`や`foldBack2`を使えば、独自の関数を簡単に作ることができます。例えば、いくつかの`filter2`関数は以下のように定義できます。

```fsharp
/// ペアの各要素に関数を適用する
/// どちらかの結果が通る場合、そのペアを結果に含める
let filterOr2 filterPredicate list1 list2 =
    let pass e = filterPredicate e
    let folder e1 e2 state =
        if (pass e1) || (pass e2) then
            (e1,e2)::state
        else
            state
    List.foldBack2 folder list1 list2 ([])

/// ペアの各要素に関数を適用する
/// 両方の結果が合格した場合のみ、そのペアを結果に含める
let filterAnd2 filterPredicate list1 list2 =
    let pass e = filterPredicate e
    let folder e1 e2 state =
        if (pass e1) && (pass e2) then
            (e1,e2)::state
        else
            state
    List.foldBack2 folder list1 list2 []

// テスト
let startsWithA (s:string) = (s.[0] = 'A')
let strList1 = ["A1"; "A3"]
let strList2 = ["A2"; "B1"]

(strList1, strList2) ||> filterOr2 startsWithA
// [("A1", "A2"); ("A3", "B1")]
(strList1, strList2) ||> filterAnd2 startsWithA
// [("A1", "A2")]
```

セクション25](#25)も参照してください。

{{< linktarget "24" >}}

----

## 24. 3つのリストを扱う

3つのリストがある場合、利用できる組み込み関数は1つだけです。しかし、[セクション25](#25)では、独自の3つのリストの関数を構築する方法の例を紹介しています。

* [`map3: mapping:('T1 -> 'T2 -> 'T3 -> 'U) -> list1:'T1 list -> list2:'T2 list -> list3:'T3 list -> 'U list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L438).
  与えられた関数を3つのコレクションの対応する要素に同時に適用した結果を要素とする新しいコレクションを構築します。
* [セクション 26: コレクションの結合と解除](#26)の `append`, `concat`, `zip3` も参照してください。

{{< linktarget "25" >}}

----

## 25. 3つ以上のリストを扱う場合

3つ以上のリストを扱う場合、組み込まれた関数はありません。

もしこのようなことが頻繁に起こらないのであれば、`zip2`や`zip3`を連続して使ってリストを1つのタプルに畳み込み、そのタプルを`map`で処理すればよいでしょう。

あるいは、applicativesを使って、自分の関数を"zip lists"の世界に"lift"することもできます。

```fsharp
let (<*>) fList xList =
    List.map2 (fun f x -> f x) fList xList

let (<!>) = List.map

let addFourParams x y z w =
    x + y + z + w

// "addFourParams"をListの世界に持ち込んで、パラメータとしてintではなくリストを渡す
addFourParams <!> [1;2;3] <*> [1;2;3] <*> [1;2;3] <*> [1;2;3]
// 結果 = [4; 8; 12]
```

これが魔法のように思えるなら、このコードが何をしているのか、[this series](/posts/elevated-world/#lift)で説明しています。


{{< linktarget "26" >}}

----

## 26. コレクションの結合と結合解除

最後に、コレクションを結合したり結合解除したりする関数がいくつかあります。

* [`append: list1:'T list -> list2:'T list -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L21).
  最初のコレクションの要素と、2番目のコレクションの要素を含む新しいコレクションを返します。
* `@` はリストに対する `append` の 中置演算子バージョンです。
* [`concat: lists:seq<'T list> -> 'T list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L90).
  与えられた関数をコレクションの対応する要素に同時に適用した結果を要素とする新しいコレクションを構築します。
* [`zip: list1:'T1 list -> list2:'T2 list -> ('T1 * 'T2) list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L882).
  2つのコレクションを組み合わせて、2つの要素を持つタプルのリストにします。2つのコレクションの長さは同じでなければなりません。
* [`zip3: list1:'T1 list -> list2:'T2 list -> list3:'T3 list -> ('T1 * 'T2 * 'T3) list`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L890).
  3つのコレクションを組み合わせて、3つの要素を持つタプルのリストを作ります。3つのコレクションは同じ長さでなければなりません。
* (Seq を除く) [`unzip: list:('T1 * 'T2) list -> ('T1 list * 'T2 list)`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L852).
  2つの要素を持つタプルのコレクションを 2 つのコレクションに分割します。
* (Seq を除く) [`unzip3: list:('T1 * 'T2 * 'T3) list -> ('T1 list * 'T2 list * 'T3 list)`](https://github.com/fsharp/fsharp/blob/4331dca3648598223204eed6bfad2b41096eec8a/src/fsharp/FSharp.Core/list.fsi#L858).
  3つの要素を持つタプルのコレクションを3つのコレクションに分割します。


### 使用例

これらの関数は、簡単に使うことができます。

```fsharp
List.append [1;2;3] [4;5;6]
// [1; 2; 3; 4; 5; 6]

[1;2;3] @ [4;5;6]
// [1; 2; 3; 4; 5; 6]

List.concat [ [1]; [2;3]; [4;5;6] ]
// [1; 2; 3; 4; 5; 6]

List.zip [1;2] [10;20]
// [(1, 10); (2, 20)]

List.zip3 [1;2] [10;20] [100;200]
// [(1, 10, 100); (2, 20, 200)]

List.unzip [(1, 10); (2, 20)]
// ([1; 2], [10; 20])

List.unzip3 [(1, 10, 100); (2, 20, 200)]
// ([1; 2], [10; 20], [100; 200])
```

なお、`zip`関数は、長さが同じであることを要求します。

```fsharp
List.zip [1;2] [10]
// ArgumentException: リストの長さが異なります。
```

{{< linktarget "27" >}}

----

## 27. その他の配列専用の関数

配列は可変型なので、リストやシーケンスには適用できない関数がいくつかあります。

* [セクション15](#15)の "sort in place" 関数を参照してください。
* `Array.blit: source:'T[] -> sourceIndex:int -> target:'T[] -> targetIndex:int -> count:int -> unit`.
   1つ目の配列から要素の範囲を読み取って、2つ目の配列に書き込みます。
* `Array.copy: array:'T[] -> 'T[]`.
   与えられた配列の要素を含む新しい配列を構築します。
* `Array.fill: target:'T[] -> targetIndex:int -> count:int -> value:'T -> unit`.
   配列の要素の範囲を、与えられた値で埋めます。
* `Array.set: array:'T[] -> index:int -> value:'T -> unit`.
   配列の要素を設定します。
* これらに加えて、他のすべての[BCL配列関数](https://msdn.microsoft.com/en-us/library/system.array.aspx)も利用できます。

例は挙げません。[MSDNドキュメント](https://msdn.microsoft.com/en-us/library/ee370273.aspx)をご覧ください。

{{< linktarget "28" >}}

----

## 28. 使い捨てのシーケンスの使用

`List.ofSeq`のような変換関数の重要な用途の一つは、遅延列挙（`seq`）を`list`のような完全に評価されたコレクションに変換することです。
これは，ファイルハンドルやデータベース接続のような使い捨てのリソースがある場合に特に重要です．リソースが利用可能な状態でシーケンスをリストに変換しないと、
リソースが破棄された後に要素にアクセスする際にエラーが発生する可能性があります。

ここでは、データベースとUIをエミュレートするヘルパー関数から始めましょう。

```fsharp
// 使い捨てのデータベース接続
let DbConnection() =
    printfn "Opening connection"
    { new System.IDisposable with
        member this.Dispose() =
            printfn "Disposing connection" }

// データベースからいくつかのレコードを読み込む
let readNCustomersFromDb dbConnection n =
    let makeCustomer i =
        sprintf "Customer %i" i

    seq {
        for i = 1 to n do
            let customer = makeCustomer i
            printfn "Loading %s from db" customer
            yield customer
        }

// いくつかのレコードを画面上に表示する
let showCustomersinUI customers =
    customers |> Seq.iter (printfn "Showing %s in UI")
```

素朴な実装では、接続が閉じられた後にシーケンスが評価されます。

```fsharp
let readCustomersFromDb() =
    use dbConnection = DbConnection()
    let results = readNCustomersFromDb dbConnection 2
    results

let customers = readCustomersFromDb()
customers |> showCustomersinUI
```

出力は以下の通りです。接続が閉じられてから、シーケンスが評価されているのがわかります。

```text
Opening connection
Disposing connection
Loading Customer 1 from db  // エラー！接続が閉じられました
Showing Customer 1 in UI
Loading Customer 2 from db
Showing Customer 2 in UI
```

より良い実装では、接続が開いている間にシーケンスをリストに変換し、シーケンスをすぐに評価するようにします。

```fsharp
let readCustomersFromDb() =
    use dbConnection = DbConnection()
    let results = readNCustomersFromDb dbConnection 2
    results |> List.ofSeq
    // 接続が開いている間にリストに変換

let customers = readCustomersFromDb()
customers |> showCustomersinUI
```

結果はずっと良くなりました。すべてのレコードは、接続が破棄される前に読み込まれます。

```text
Opening connection
Loading Customer 1 from db
Loading Customer 2 from db
Disposing connection
Showing Customer 1 in UI
Showing Customer 2 in UI
```

3つ目の方法は、disposableをシーケンス自体に埋め込むことです。

```fsharp
let readCustomersFromDb() =
    seq {
        // シーケンスの中にdisposableを入れる
        use dbConnection = DbConnection()
        yield! readNCustomersFromDb dbConnection 2
        }

let customers = readCustomersFromDb()
customers |> showCustomersinUI
```

出力を見ると、今度は接続が開かれている間にUIの表示も行われています。

```text
Opening connection
Loading Customer 1 from db
Showing Customer 1 in UI
Loading Customer 2 from db
Showing Customer 2 in UI
Disposing connection
```

これは、状況によって、悪いこと（接続が開いている時間が長くなる）や良いこと（メモリ使用量が少なくなる）があります。

{{< linktarget "29" >}}

----

## 29. 冒険の終わり

あなたは最後までやり遂げました−−よくできました!とはいえ、大した冒険ではありませんでしたね。ドラゴンも出ませんでした。ただ、お役に立てば幸いです。

