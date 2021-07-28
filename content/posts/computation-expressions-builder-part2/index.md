---
layout: post
title: "Implementing a builder: Combine"
description: "How to return multiple values at once"
date: 2013-01-26
nav: thinking-functionally
seriesId: "Computation Expressions"
seriesOrder: 7
---

この記事では、`Combine`メソッドを使って、計算式から複数の値を返すことを見ていきます。

## The story so far...

これまでのところ、式ビルダークラスは以下のようになっています。

```fsharp
type TraceBuilder() =
    member this.Bind(m, f) =
        match m with
        | None ->
            printfn "Binding with None. Exiting."
        | Some a ->
            printfn "Binding with Some(%A). Continuing" a
        Option.bind f m

    member this.Return(x) =
        printfn "Returning a unwrapped %A as an option" x
        Some x

    member this.ReturnFrom(m) =
        printfn "Returning an option (%A) directly" m
        m

    member this.Zero() =
        printfn "Zero"
        None

    member this.Yield(x) =
        printfn "Yield an unwrapped %A as an option" x
        Some x

    member this.YieldFrom(m) =
        printfn "Yield an option (%A) directly" m
        m

// make an instance of the workflow
let trace = new TraceBuilder()
```

このクラスは、ここまでは問題なく動作しています。しかし、これから問題が発生します...

## 2つの「yield」に関する問題

前回は、`yield`を使って、`return`と同じように値を返す方法を見ました。

通常、`yield`はもちろん一度だけではなく、列挙などの処理の異なる段階で値を返すために複数回使用します。では、試してみましょう。

```fsharp
trace {
    yield 1
    yield 2
    } |> printfn "Result for yield then yield: %A"
```

しかし、困ったことに、エラーメッセージが表示されます。

```text
This control construct may only be used if the computation expression builder defines a 'Combine' method.
```

また、`yield`の代わりに`return`を使うと、同じエラーが出ます。

```fsharp
trace {
    return 1
    return 2
    } |> printfn "Result for return then return: %A"
```

また、この問題は他の文脈でも発生します。 例えば、次のように、何かをしてからリターンしたい場合です。

```fsharp
trace {
    if true then printfn "hello"
    return 1
    } |> printfn "Result for if then return: %A"
```

同じように「Combine」メソッドがないというエラーメッセージが表示されます。

## 問題点の把握

では、何が起こっているのでしょうか？

理解するためには、計算式の舞台裏に戻ってみましょう。`return`と`yield`は実際には以下のような一連の連続処理の最後のステップであることを見てきました。

```fsharp
Bind(1,fun x ->
   Bind(2,fun y ->
     Bind(x + y,fun z ->
        Return(z)  // or Yield
```

`return`（または`yield`）は、インデントを「リセットする」と考えてもいいでしょう。つまり、`return/yield`した後、再び`return/yield`すると、次のようなコードが生成されるのです。

```fsharp
Bind(1,fun x ->
   Bind(2,fun y ->
     Bind(x + y,fun z ->
        Yield(z)
// start a new expression
Bind(3,fun w ->
   Bind(4,fun u ->
     Bind(w + u,fun v ->
        Yield(v)
```

しかし，実際にはこれは次のように単純化できます。

```fsharp
let value1 = some expression
let value2 = some other expression
```

つまり、計算式の中に*2つの*値が入っていることになります。では，この2つの値をどのように組み合わせれば，計算式全体として1つの結果が得られるのでしょうか？

これは非常に重要なポイントです。**RETURNやYEARは、計算式から早期にリターンを得ることは*できません*** 最後の中括弧に至るまで、計算式全体が評価され、結果として*単一の*値が得られるのは*常に*です。 繰り返しになりますが 計算式のすべての部分は常に評価されます。-- つまり、短絡的な処理は行われません。 もし短絡して早く返したいのであれば、そのためのコードを自分で書かなければなりません（その方法は後ほど説明します）。

さて、差し迫った質問に戻ります。2つの式から2つの値が得られていますが、これらの複数の値をどのようにして1つにまとめるのでしょうか？

## "Combine "の紹介

その答えは、`Combine`メソッドを使うことです、これは、2つの*ラップ*された値を受け取り、それらを組み合わせて別のラップされた値を作ります。

今回の例では、特に `int options` を扱っているので、単純な実装としては、数字を足し合わせることが思い浮かびます。各パラメータはもちろん`option`（ラップされた型）なので、それらを分解して4つの可能なケースを処理する必要があります。

```fsharp
type TraceBuilder() =
    // other members as before

    member this.Combine (a,b) =
        match a,b with
        | Some a', Some b' ->
            printfn "combining %A and %A" a' b'
            Some (a' + b')
        | Some a', None ->
            printfn "combining %A with None" a'
            Some a'
        | None, Some b' ->
            printfn "combining None with %A" b'
            Some b'
        | None, None ->
            printfn "combining None with None"
            None

// make a new instance
let trace = new TraceBuilder()
```

テストコードを再度実行します。

```fsharp
trace {
    yield 1
    yield 2
    } |> printfn "Result for yield then yield: %A"
```

しかし、今度は違うエラーメッセージが表示されます。

```text
This control construct may only be used if the computation expression builder defines a 'Delay' method
```

`Delay`メソッドは、計算式の評価を必要なときまで遅らせることができるフックです。これについてはすぐに詳しく説明しますが、とりあえずデフォルトの実装を作ってみましょう。

```fsharp
type TraceBuilder() =
    // other members as before

    member this.Delay(f) =
        printfn "Delay"
        f()

// make a new instance
let trace = new TraceBuilder()
```

テストコードを再度実行します。

```fsharp
trace {
    yield 1
    yield 2
    } |> printfn "Result for yield then yield: %A"
```

そして最後に、コードを完成させます。

```text
Delay
Yield an unwrapped 1 as an option
Delay
Yield an unwrapped 2 as an option
combining 1 and 2
Result for yield then yield: Some 3
```

ワークフロー全体の結果は、すべてのyieldの合計である`Some 3`となります。

もし、ワークフローに「失敗」があった場合（例：`None`）、2つ目のyieldは発生せず、全体の結果は`Some 1`となります。

```fsharp
trace {
    yield 1
    let! x = None
    yield 2
    } |> printfn "Result for yield then None: %A"
```

2つではなく、3つの`yield`を持つことができます。

```fsharp
trace {
    yield 1
    yield 2
    yield 3
    } |> printfn "Result for yield x 3: %A"
```

結果は期待通り、`Some 6`となります。

さらに、`yield`と`return`を一緒に混ぜてみることもできます。構文が違うだけで，全体的な効果は同じです。

```fsharp
trace {
    yield 1
    return 2
    } |> printfn "Result for yield then return: %A"

trace {
    return 1
    return 2
    } |> printfn "Result for return then return: %A"
```

## Combineを使ったシーケンス生成

しかし、`yield`の目的は、数字の足し算ではありません。おそらく、`StringBuilder`のように、連結された文字列を作るために同じようなアイデアを使うかもしれません。

また、`Combine`を理解したことで、前回のワークフローであるListBuilderを必要なメソッドで拡張することができます。

* `Combine`メソッドは、単なるリストの連結です。
* `Delay` メソッドは、今のところデフォルトの実装を使うことができます。

これがクラスの全貌です。

```fsharp
type ListBuilder() =
    member this.Bind(m, f) =
        m |> List.collect f

    member this.Zero() =
        printfn "Zero"
        []

    member this.Yield(x) =
        printfn "Yield an unwrapped %A as a list" x
        [x]

    member this.YieldFrom(m) =
        printfn "Yield a list (%A) directly" m
        m

    member this.For(m,f) =
        printfn "For %A" m
        this.Bind(m,f)

    member this.Combine (a,b) =
        printfn "combining %A and %A" a b
        List.concat [a;b]

    member this.Delay(f) =
        printfn "Delay"
        f()

// make an instance of the workflow
let listbuilder = new ListBuilder()
```

そして、実際に使用してみます。

```fsharp
listbuilder {
    yield 1
    yield 2
    } |> printfn "Result for yield then yield: %A"

listbuilder {
    yield 1
    yield! [2;3]
    } |> printfn "Result for yield then yield! : %A"
```

さらに、`for`ループと`yield`を使った複雑な例を示します。

```fsharp
listbuilder {
    for i in ["red";"blue"] do
        yield i
        for j in ["hat";"tie"] do
            yield! [i + " " + j;"-"]
    } |> printfn "Result for for..in..do : %A"
```

そして、その結果は

```text
["red"; "red hat"; "-"; "red tie"; "-"; "blue"; "blue hat"; "-"; "blue tie"; "-"]
```

`for...in...do`と`yield`を組み合わせることで、組み込みの`seq`式の構文からそれほど離れていないことがわかります（もちろん、`seq`が遅延していることを除いて）。

舞台裏で何が起こっているのかがはっきりするまで、これを少し遊んでみることを強くお勧めします。
上の例からわかるように、`yield`をクリエイティブな方法で使うことで、単純なものだけでなく、あらゆる種類の不規則なリストを生成することができます。

*注：`While`について気になる方は、次回の投稿で`Delay`を見てからにしましょう*。

## "combine "の処理順序

`Combine`メソッドには2つのパラメータしかありません。 では、2つ以上の値を組み合わせる場合はどうなるのでしょうか？例えば、4つの値を組み合わせる場合を考えてみましょう。

```fsharp
listbuilder {
    yield 1
    yield 2
    yield 3
    yield 4
    } |> printfn "Result for yield x 4: %A"
```

出力を見てみると、期待通りに値がペアで結合されているのがわかります。

```text
combining [3] and [4]
combining [2] and [3; 4]
combining [1] and [2; 3; 4]
Result for yield x 4: [1; 2; 3; 4]
```

微妙なところですが、重要なポイントは、最後の値から「逆」に組み合わせていることです。 まず "3 "が "4 "と結合され、その結果が "2 "と結合される、という具合です。

![Combine](./combine.png)

## シーケンス以外の文字列の結合

先ほどの問題の2つ目の例では、シーケンスはなく、2つの独立した式が並んでいるだけでした。

```fsharp
trace {
    if true then printfn "hello"  //expression 1
    return 1                      //expression 2
    } |> printfn "Result for combine: %A"
```

これらの式をどのように組み合わせるべきでしょうか。

ワークフローがサポートする概念に応じて、いくつかの一般的な方法があります。

### "成功 "や "失敗 "を伴うワークフローへのコンバインの実装

ワークフローに「成功」や「失敗」の概念がある場合、標準的な方法は以下の通りです。

* 最初の式が「成功」した場合は、その値を使用します。
* それ以外の場合は、2番目の式の値を使用します。

この場合、一般的には `Zero` の "failure" の値を使用します。

この方法は、一連の「or else」式を連鎖させて、最初に成功したものが全体の結果となるような場合に便利です。

```text
if (do first expression)
or else (do second expression)
or else (do third expression)
```

例えば、`maybe`というワークフローでは、以下のように、`Some`であれば第一の式を返し、そうでなければ第二の式を返すというのが一般的です。

```fsharp
type TraceBuilder() =
    // other members as before

    member this.Zero() =
        printfn "Zero"
        None  // failure

    member this.Combine (a,b) =
        printfn "Combining %A with %A" a b
        match a with
        | Some _ -> a  // a succeeds -- use it
        | None -> b    // a fails -- use b instead

// make a new instance
let trace = new TraceBuilder()
```

**Example: 構文解析**の例

この実装を使って、パーシングの例を試してみましょう。

```fsharp
type IntOrBool = I of int | B of bool

let parseInt s =
    match System.Int32.TryParse(s) with
    | true,i -> Some (I i)
    | false,_ -> None

let parseBool s =
    match System.Boolean.TryParse(s) with
    | true,i -> Some (B i)
    | false,_ -> None

trace {
    return! parseBool "42"  // fails
    return! parseInt "42"
    } |> printfn "Result for parsing: %A"
```

次のような結果が得られます。

```text
Some (I 42)
```

最初の `return!` 式は `None` で、無視されているのがわかります。つまり、全体の結果は2番目の式である`Some (I 42)`となります。

**Example: 辞書検索の例

この例では，同じキーを複数の辞書で検索し，値が見つかったら返すということをやってみましょう。

```fsharp
let map1 = [ ("1","One"); ("2","Two") ] |> Map.ofList
let map2 = [ ("A","Alice"); ("B","Bob") ] |> Map.ofList

trace {
    return! map1.TryFind "A"
    return! map2.TryFind "A"
    } |> printfn "Result for map lookup: %A"
```

次のような結果が得られます。

```text
Result for map lookup: Some "Alice"
```

最初のルックアップは `None` で、無視されているのがわかります。つまり、全体の結果は2番目のルックアップになります。

ご覧のように、この手法は解析や一連の（失敗する可能性のある）操作を評価する際に非常に便利です。

### シーケンシャルステップを持つワークフローへのコンバインの実装

ワークフローに連続したステップの概念がある場合、全体的な結果は最後のステップの値だけであり、前のステップはすべてその副次的な効果のみが評価されることになります。

通常のF#では、このように書かれます。

```text
do some expression
do some other expression
final expression
```

あるいはセミコロンの構文を使って、単に

```text
some expression; some other expression; final expression
```

通常のF#では、（最後の式以外の）各式は単位値で評価されます。

計算式と同等のアプローチは、（最後の式以外の）各式を*ラップ*された単位値として扱い、それを次の式に「渡す」ことで、最後の式に到達するまで続けていきます。

もちろん、これはbindが行っていることと全く同じであり、最も簡単な実装は、`Bind`メソッド自体を再利用することです。また、この方法が機能するためには、`Zero`がラップされた単位値であることが重要です。

```fsharp
type TraceBuilder() =
    // other members as before

    member this.Zero() =
        printfn "Zero"
        this.Return ()  // unit not None

    member this.Combine (a,b) =
        printfn "Combining %A with %A" a b
        this.Bind( a, fun ()-> b )

// make a new instance
let trace = new TraceBuilder()
```

通常のバインドとの違いは、コンティニュエーションがユニットパラメータを持ち、`b`と評価されることです。 これにより、`a` は一般的には `WrapperType<unit>` 型、ここでは `unit option` 型であることが強制されます。

この実装の`Combine`で動作する逐次処理の例を示します。

```fsharp
trace {
    if true then printfn "hello......."
    if false then printfn ".......world"
    return 1
    } |> printfn "Result for sequential combine: %A"
```

以下のようなトレースがあります。通常のF#コードと同じように、式全体の結果は、シーケンスの最後の式の結果であることに注意してください。

```text
hello.......
Zero
Returning a unwrapped <null> as an option
Zero
Returning a unwrapped <null> as an option
Returning a unwrapped 1 as an option
Combining Some null with Some 1
Combining Some null with Some 1
Result for sequential combine: Some 1
```

### データ構造を構築するワークフローへのコンバインの実装

最後に、ワークフローによくあるパターンとして、データ構造を構築するものがあります。この場合、`Combine`は2つのデータ構造を適切な方法でマージする必要があります。
また、`Zero`メソッドは、必要に応じて（可能であれば）、空のデータ構造を作成します。

上の「リストビルダー」の例では、まさにこのようなアプローチをとっています。`Combine`は単なるリスト連結で、`Zero`は空のリストでした。

## "Combine "と "Zero "を混ぜるためのガイドライン

ここでは、オプションタイプに対する `Combine` の2つの異なる実装を見てきました。

* 最初の実装では、オプションを「成功/失敗」の指標として使用し、最初の成功が「勝ち」となります。この場合、`Zero`は`None`として定義されます。
* 2つ目はシーケンシャルなもので、この場合は `Zero` が `Some ()` として定義されました。

どちらのケースもうまくいきましたが、これは運が良かったのでしょうか。それとも、`Combine`と`Zero`を正しく実装するためのガイドラインがあるのでしょうか。

まず、`Combine`はパラメータを入れ替えても同じ結果になるとは限らないことに注意してください。
つまり、`Combine(a,b)` は `Combine(b,a)` と同じである必要はありません。リストビルダがその良い例です。

一方で、`Zero`と`Combine`を結びつける便利なルールがあります。

**ルール：`Combine(a,Zero)`は`Combine(Zero,a)`と同じであるべきであり、`a`だけと同じであるべきである**。

算数で例えると、`Combine`は足し算のようなものだと考えることができます(悪い例えではありません。実際には2つの値を「足し算」しているのですから)。そして、`Zero`は、もちろん、数字のゼロです。ですから、上記のルールは次のように表現できます。

**ルール： `a + 0` は `0 + a` と同じであり、`a` と同じである、ここで、`+`は `Combine` を意味し、`0`は`Zero`を意味である**。

オプション型に対する最初の `Combine` の実装（"success/failure"）を見てみると、2番目の実装（`Some()`を使った "bind"）と同様に、確かにこのルールに準拠していることがわかります。

一方で、`Combine`の "bind "実装を使って、`Zero`を`None`と定義したままにしていたとしたら、足し算のルールには*従わなかった*でしょうし、何か間違ったことをしたという手がかりになるでしょう。


## bindを使わない "Combine"

他のビルダーメソッドと同様に、必要がなければ実装する必要はありません。 つまり、強いシーケンシャルなワークフローの場合、`Bind`と`Return`を実装することなく、`Combine`、`Zero`、`Yield`を持つビルダークラスを簡単に作ることができます。

以下に，最低限の実装で動作する例を示します．

```fsharp
type TraceBuilder() =

    member this.ReturnFrom(x) = x

    member this.Zero() = Some ()

    member this.Combine (a,b) =
        a |> Option.bind (fun ()-> b )

    member this.Delay(f) = f()

// make an instance of the workflow
let trace = new TraceBuilder()
```

そして、実際に使用してみます。

```fsharp
trace {
    if true then printfn "hello......."
    if false then printfn ".......world"
    return! Some 1
    } |> printfn "Result for minimal combine: %A"
```

同様に、データ構造を重視したワークフローであれば、`Combine`と他のヘルパーを実装するだけでよいでしょう。例えば、リストビルダークラスの最小限の実装を以下に示します。

```fsharp
type ListBuilder() =

    member this.Yield(x) = [x]

    member this.For(m,f) =
        m |> List.collect f

    member this.Combine (a,b) =
        List.concat [a;b]

    member this.Delay(f) = f()

// make an instance of the workflow
let listbuilder = new ListBuilder()
```

また、最小限の実装でも、以下のようなコードを書くことができます。

```fsharp
listbuilder {
    yield 1
    yield 2
    } |> printfn "Result: %A"

listbuilder {
    for i in [1..5] do yield i + 2
    yield 42
    } |> printfn "Result: %A"
```


## スタンドアローンの "bind "関数

前回の記事では、"bind "関数は単独の関数として使われることが多く、通常は演算子 `>>=` が与えられることを紹介しました。

Combine`関数もまた、単独の関数としてよく使われます。bindと違って標準的な記号はなく、combine関数の動作に応じて変化します。

対称的な組み合わせ操作は、しばしば `++` や `<+>` と書かれます。また，前に使った「左バイアスのかかった」組み合わせ（つまり，最初の式が失敗したときに2番目の式だけを実行する）は
また、先ほどオプションで使った「左バイアス」の組み合わせ（つまり、最初の式が失敗した場合に2番目の式を実行する）は、`<++`と書かれることがあります。

以下は，辞書検索の例で使われている，単独の左バイアスのかかったオプションの組み合わせの例です。

```fsharp
module StandaloneCombine =

    let combine a b =
        match a with
        | Some _ -> a  // a succeeds -- use it
        | None -> b    // a fails -- use b instead

    // create an infix version
    let ( <++ ) = combine

    let map1 = [ ("1","One"); ("2","Two") ] |> Map.ofList
    let map2 = [ ("A","Alice"); ("B","Bob") ] |> Map.ofList

    let result =
        (map1.TryFind "A")
        <++ (map1.TryFind "B")
        <++ (map2.TryFind "A")
        <++ (map2.TryFind "B")
        |> printfn "Result of adding options is: %A"
```


## まとめ

この記事で `Combine` について学んだことは？

* 計算式の中で、複数のラップされた値を組み合わせたり、「追加」する必要がある場合には、`Combine`（および `Delay`）を実装する必要があります。
* `Combine` は値をペアにして、最後から最初に向かって結合します。
* すべてのケースで動作する普遍的な `Combine` の実装はありません。ワークフローの特定のニーズに応じてカスタマイズする必要があります。
* `Combine` と `Zero` を関連付ける適切なルールがあります。
* `Combine` は `Bind` を実装する必要はありません。
* `Combine` はスタンドアロンの関数として公開することができます。

次回の記事では、内部式がいつ評価されるかを正確に制御するロジックを追加し、真の短絡評価と遅延評価を紹介します。