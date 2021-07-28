---
layout: post
title: "Introducing 'bind'"
description: "Steps towards creating our own 'let!' "
date: 2013-01-22
nav: thinking-functionally
seriesId: "Computation Expressions"
seriesOrder: 3
---

前回の記事では、`let`を舞台裏で継続処理を行うための優れた構文と考えることができることを説明しました。
また、継続パイプラインにフックを追加することができる`pipeInto`関数を紹介しました。

ここで、最初のビルダーメソッドである `Bind` を見てみましょう。これは、このアプローチを公式化したもので、あらゆる計算式の中核となるものです。

### Bindの紹介

[計算式に関するMSDNのページ](http://msdn.microsoft.com/en-us/library/dd233182.aspx)では、`Bind`メソッドのシンタクティックシュガーとして、`let!`式を説明しています。これをもう一度見てみましょう。

ここでは，`let!`式のドキュメントと，実際の例を紹介します。

```fsharp
// documentation
{| let! pattern = expr in cexpr |}

// real example
let! x = 43 in some expression
```

また、`Bind`メソッドのドキュメントと実例を紹介します。

```fsharp
// documentation
builder.Bind(expr, (fun pattern -> {| cexpr |}))

// real example
builder.Bind(43, (fun x -> some expression))
```

これについて、いくつかの興味深い点があります。

* `Bind` は、式(`43`)とラムダの2つのパラメータを取ります。
* ラムダのパラメータ(`x`)は、最初のパラメータとして渡された式にバインドされます。(少なくともこの場合は。詳しくは後述します。)
* `Bind` のパラメータは `let!` の中での順序とは逆になります。

つまり、言い換えると、`let!`の式をいくつも連鎖させると、次のようになります。

```fsharp
let! x = 1
let! y = 2
let! z = x + y
```

コンパイラはこれを次のように`Bind`の呼び出しに変換します。

```fsharp
Bind(1, fun x ->
Bind(2, fun y ->
Bind(x + y, fun z ->
etc
```

これで、私たちが何をしようとしているのか、お分かりいただけると思います。

実際、私たちの `pipeInto` 関数は `Bind` メソッドとまったく同じです。

これは重要な洞察です。*計算式は、自分たちでできることをきれいな構文で表現するための手段にすぎません*。

### 単独のバインド関数

このように「bind」関数を持つことは、実は標準的な関数パターンであり、計算式にはまったく依存しません。

まず、なぜ「bind」と呼ばれているのでしょうか。これまで見てきたように、「bind」関数やメソッドは、ある関数に入力値を与えるものと考えることができます。これは、関数のパラメータに値を「[bind](/posts/function-values-and-simple-values/)する」こととして知られています（すべての関数は[1つのパラメータ](/posts/currying/)しか持たないことを思い出してください）。

このように`bind`を考えてみると、パイピングやコンポジションに似ていることがわかります。

実際、次のような infix 演算にすることもできます。

```fsharp
let (>>=) m f = pipeInto(m,f)
```

*ちなみに、この「>>=」という記号は、bindをインフィックス演算子として記述する標準的な方法です。他のF#コードでこの記号が使われているのを見かけたら、それはおそらくこの記号を表しているのでしょう*。

安全な除算の例に戻ると、次のようにワークフローを1行で書くことができます。

```fsharp
let divideByWorkflow x y w z =
    x |> divideBy y >>= divideBy w >>= divideBy z
```

これが普通の配管や合成とどう違うのか、不思議に思われるかもしれません。すぐにはわかりませんね。

その答えは2つあります。

* まず、`bind`関数は、それぞれの状況に応じてカスタマイズされた動作を *追加* しています。パイプやコンポジションのような汎用的な関数ではありません。

* 第二に、値のパラメータの入力型（上記の `m` ）は、関数のパラメータの出力型（上記の `f` ）とは必ずしも同じではありません。したがって、bindが行うことのひとつは、このミスマッチをエレガントに処理して、関数を連鎖させることです。

次の記事で説明しますが、bindは通常、何らかの「ラッパー」型を使って動作します。値のパラメータは「`WrapperType<TypeA>`」で、「`bind`関数」の関数パラメータのシグネチャは、常に「`TypeA -> WrapperType<TypeB>`」です。

安全な分割のための`bind`の特別なケースでは，ラッパーの型は`Option`です．値のパラメータ（上記の `m` ）の型は `Option<int>` であり，関数のパラメータ（上記の `f` ）のシグネチャは `int -> Option<int>` です．

バインドを別の文脈で使ってみるために，ロギングのワークフローを infix のバインド関数で表現した例を示します．

```fsharp
let (>>=) m f =
    printfn "expression is %A" m
    f m

let loggingWorkflow =
    1 >>= (+) 2 >>= (*) 42 >>= id
```

この場合、ラッパー型はありません。すべてが `int` です。しかし、そうであっても、`bind` には、裏でロギングを行う特別な動作があります。

## Option.bind と "maybe" ワークフローの再訪

F# のライブラリでは、さまざまな場所で `Bind` 関数やメソッドを目にします。これで何のためのものかわかりましたね。

特に便利なのが`Option.bind`で，上で手書きで書いたことをそのまま実行します．

* 入力パラメータが `None` の場合は、継続関数を呼び出しません。
* 入力パラメータが `Some` の場合は、`Some` の内容を渡して、継続関数を呼び出します。

これが私たちの手作りの関数です。

```fsharp
let pipeInto (m,f) =
   match m with
   | None ->
       None
   | Some x ->
       x |> f
```

また，`Option.bind`の実装は以下の通りです．

```fsharp
module Option =
    let bind f m =
       match m with
       | None ->
           None
       | Some x ->
           x |> f
```

これには教訓があります。自分で関数を書くことをあまり急がないでください。再利用可能なライブラリ関数があるかもしれません。

ここでは，"maybe"のワークフローを，`Option.bind`を使うように書き直してみました．

```fsharp
type MaybeBuilder() =
    member this.Bind(m, f) = Option.bind f m
    member this.Return(x) = Some x
```


## これまでの様々なアプローチを振り返る ##

これまで「安全な分割」の例では、4つの異なるアプローチを使ってきました。それらを並べて、もう一度比較してみましょう。

*注：オリジナルの `pipeInto` 関数を `bind` にリネームし、オリジナルのカスタム実装の代わりに `Option.bind` を使用しています*。

最初に，明示的なワークフローを使用したオリジナルのバージョンです．

```fsharp
module DivideByExplicit =

    let divideBy bottom top =
        if bottom = 0
        then None
        else Some(top/bottom)

    let divideByWorkflow x y w z =
        let a = x |> divideBy y
        match a with
        | None -> None  // give up
        | Some a' ->    // keep going
            let b = a' |> divideBy w
            match b with
            | None -> None  // give up
            | Some b' ->    // keep going
                let c = b' |> divideBy z
                match c with
                | None -> None  // give up
                | Some c' ->    // keep going
                    //return
                    Some c'
    // test
    let good = divideByWorkflow 12 3 2 1
    let bad = divideByWorkflow 12 3 0 1
```

次に，独自の "bind"（別名 "pipeInto"）を使用します。

```fsharp
module DivideByWithBindFunction =

    let divideBy bottom top =
        if bottom = 0
        then None
        else Some(top/bottom)

    let bind (m,f) =
        Option.bind f m

    let return' x = Some x

    let divideByWorkflow x y w z =
        bind (x |> divideBy y, fun a ->
        bind (a |> divideBy w, fun b ->
        bind (b |> divideBy z, fun c ->
        return' c
        )))

    // test
    let good = divideByWorkflow 12 3 2 1
    let bad = divideByWorkflow 12 3 0 1
```

次に，計算式を使って

```fsharp
module DivideByWithCompExpr =

    let divideBy bottom top =
        if bottom = 0
        then None
        else Some(top/bottom)

    type MaybeBuilder() =
        member this.Bind(m, f) = Option.bind f m
        member this.Return(x) = Some x

    let maybe = new MaybeBuilder()

    let divideByWorkflow x y w z =
        maybe
            {
            let! a = x |> divideBy y
            let! b = a |> divideBy w
            let! c = b |> divideBy z
            return c
            }

    // test
    let good = divideByWorkflow 12 3 2 1
    let bad = divideByWorkflow 12 3 0 1
```

最後に，bind を infix 演算として使う方法です．

```fsharp
module DivideByWithBindOperator =

    let divideBy bottom top =
        if bottom = 0
        then None
        else Some(top/bottom)

    let (>>=) m f = Option.bind f m

    let divideByWorkflow x y w z =
        x |> divideBy y
        >>= divideBy w
        >>= divideBy z

    // test
    let good = divideByWorkflow 12 3 2 1
    let bad = divideByWorkflow 12 3 0 1
```

Bind関数は非常に強力であることがわかります。次の記事では、`bind` とラッパー型を組み合わせることで、バックグラウンドで余分な情報を渡すエレガントな方法ができることを紹介します。

## Exercise: How well do you understand?

次の記事に移る前に、これまでの内容をすべて理解しているかどうか、自分で試してみませんか？

次の記事に移る前に、ここまでの内容が理解できたかどうか、自分で試してみませんか？

**パート1 - ワークフローの作成**

まず、文字列を解析してint型にする関数を作ります。

```fsharp
let strToInt str = ???
```

そして、以下のように、ワークフローで使えるように、独自の計算式ビルダークラスを作成します。

```fsharp
let stringAddWorkflow x y z =
    yourWorkflow
        {
        let! a = strToInt x
        let! b = strToInt y
        let! c = strToInt z
        return a + b + c
        }

// test
let good = stringAddWorkflow "12" "3" "2"
let bad = stringAddWorkflow "12" "xyz" "2"
```

**パート2 -- バインド関数の作成**

最初の部分が動作するようになったら、さらに2つの関数を追加してアイデアを拡張します。

```fsharp
let strAdd str i = ???
let (>>=) m f = ???
```

そして、これらの関数を使って、次のようなコードが書けるようになります。

```fsharp
let good = strToInt "1" >>= strAdd "2" >>= strAdd "3"
let bad = strToInt "1" >>= strAdd "xyz" >>= strAdd "3"
```


## まとめ ##

今回の記事で取り上げたポイントをまとめてみました。

* 計算式は継続処理のための良いシンタックスを提供し、チェインロジックを隠してくれます。
* `bind` は、あるステップの出力を次のステップの入力にリンクする重要な関数です。
* シンボル `>>=` は、bind を infix 演算子として記述する標準的な方法です。
