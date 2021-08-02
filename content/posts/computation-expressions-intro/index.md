---
layout: post
title: "Computation expressions: Introduction"
description: "Unwrapping the enigma..."
date: 2013-01-20
nav: thinking-functionally
seriesId: "Computation Expressions"
seriesOrder: 1
---

ご要望にお応えして、コンピュテーション式の謎、コンピュテーション式とは何か、そして実際にどのように役立つのかについてお話しします（なお、[禁断のm-word](/about/#banned)は使わないようにします）。

このシリーズでは、コンピュテーション式とは何か、コンピュテーション式の作り方、コンピュテーション式を使ったよくあるパターンなどを学びます。その過程で、コンティニュエーション、バインド関数、ラッパー型などについても見ていきます。

## 背景 ##

コンピュテーション式は、難解で理解しにくいという評判があるようです。

一方で、コンピュテーション式を使うのはとても簡単です。F#のコードをたくさん書いたことがある人なら、`seq{...}`や`async{...}`といった標準的なものを使ったことがあるでしょう。

しかし、このようなものを新しく作るにはどうしたらいいのでしょうか？舞台裏ではどのように動いているのでしょうか？

残念ながら、多くの説明は事態をさらに混乱させているようです。 何か精神的な橋のようなものを渡らなければならないようです。
向こう側に行けば当たり前のことでも、こちら側にいる人にとっては不可解なことなのです。

[MSDN公式ドキュメント](http://msdn.microsoft.com/en-us/library/dd233182.aspx)を参考にすると、そこには明確に書かれていますが、初心者には全く役に立たないものです。

例えば、コンピュテーション式の中に次のようなコードがある場合、次のように書かれています。

```fsharp
{| let! pattern = expr in cexpr |}
```

これは、単に、このメソッド呼び出しのシンタクティックシュガーです．

```fsharp
builder.Bind(expr, (fun pattern -> {| cexpr |}))
```

しかし、これは正確には何を意味するのでしょうか？

この連載が終わる頃には、上のドキュメントが明らかになっていることを期待しています。 信じられませんか？読んでみてください。

## コンピュテーション式の実際 ##

コンピュテーション式の仕組みを説明する前に、コンピュテーション式を使う前と後で同じコードを表示する、いくつかの些細な例を見てみましょう。

まずは、簡単な例から見てみましょう。 あるコードがあって、各ステップのログを取りたいとします。そこで、小さなロギング関数を定義し、値が生成されるたびにそれを呼び出すことにします。

```fsharp
let log p = printfn "expression is %A" p

let loggedWorkflow =
    let x = 42
    log x
    let y = 43
    log y
    let z = x + y
    log z
    //return
    z
```

これを実行すると、次のような出力が表示されます。

```text
expression is 42
expression is 43
expression is 85
```

十分にシンプルです。

しかし、毎回すべてのlog文を明示的に書かなければならないのは面倒です。隠す方法はないのでしょうか?

おかしな質問ですね...コンピュテーション式ならできます。これとまったく同じことができます。

最初に、`LoggingBuilder`という新しい型を定義します:

```fsharp
type LoggingBuilder() =
    let log p = printfn "expression is %A" p

    member this.Bind(x, f) =
        log x
        f x

    member this.Return(x) =
        x
```

*謎の`Bind`と`Return`がまだ何のためにあるのかは心配しないでください--すぐに説明します。*

次に、型のインスタンス`logger`を作成します。

```fsharp
let logger = new LoggingBuilder()
```

この`logger`の値を使って、元のロギングの例を次のように書き換えることができます。

```fsharp
let loggedWorkflow =
    logger
        {
        let! x = 42
        let! y = 43
        let! z = x + y
        return z
        }
```

これを実行すると、まったく同じ出力が得られますが、`logger{...}`ワークフローを使用することで、反復的なコードを隠すことができたことがわかります。

### 安全な分割 ###

ここで、昔からよく使われている方法を見てみましょう。

連続した数字を次々と割り算したいが、その中に0があるかもしれないとします。どうすればいいのでしょうか？例外を発生させるのは醜いですね。 しかし、`option`型にはぴったりのようです。

まず、割り算をして `int option` を返すヘルパー関数を作る必要があります。
問題がなければ `Some` が得られ、除算が失敗すれば `None` が得られます。

そして、分割を連鎖させることができます。各分割の後に、失敗したかどうかをテストし、成功した場合にのみ処理を続ける必要があります。

ここでは、まずヘルパー関数、そしてメインのワークフローを紹介します。

```fsharp
let divideBy bottom top =
    if bottom = 0
    then None
    else Some(top/bottom)
```

パラメータリストの中で、除算器を最初に置いていることに注意してください。これは、`12 |> divideBy 3`のような式を書けるようにするためで、これにより連鎖が容易になります。

それでは実際に使ってみましょう。最初の数字を3回に分けようとするワークフローを紹介します。

```fsharp
let divideByWorkflow init x y z =
    let a = init |> divideBy x
    match a with
    | None -> None  // give up
    | Some a' ->    // keep going
        let b = a' |> divideBy y
        match b with
        | None -> None  // give up
        | Some b' ->    // keep going
            let c = b' |> divideBy z
            match c with
            | None -> None  // give up
            | Some c' ->    // keep going
                //return
                Some c'
```

使ってみましょう。

```fsharp
let good = divideByWorkflow 12 3 2 1
let bad = divideByWorkflow 12 3 0 1
```

`bad`ワークフローは3番目のステップで失敗し、全体に対して`None`を返します。

*ワークフロー全体*も`int option`を返さなければならないことに注意してください。`int`を単に返すことはできません、というのは、悪い場合にはどのように評価されるのでしょうか?
そして、ワークフローの "内部"で使用したタイプ、つまりoption型が、最終的な結果として出力された型である必要があることが分かるでしょう。この点を覚えておいてください。後でまた出てきます。

とにかく、この継続的な評価と分岐は本当に見苦しいです!コンピュテーション式に変換するとうまくいくのでしょうか?

もう一度、新しいタイプ (`MaybeBuilder`) を定義し、そのタイプのインスタンス (`maybe`) を作成します。

```fsharp
type MaybeBuilder() =

    member this.Bind(x, f) =
        match x with
        | None -> None
        | Some a -> f a

    member this.Return(x) =
        Some x

let maybe = new MaybeBuilder()
```

ここでは、`divideByBuilder`ではなく、`MaybeBuilder`と呼んでいます。これは、コンピュテーション式を使ってこのようにオプション型を扱うという問題はよくあることで、`maybe`はこのようなものの標準的な名前だからです。

では、`maybe`のワークフローを定義したので、それを使うように元のコードを書き換えてみましょう。

```fsharp
let divideByWorkflow init x y z =
    maybe
        {
        let! a = init |> divideBy x
        let! b = a |> divideBy y
        let! c = b |> divideBy z
        return c
        }
```

かなり、すっきりしましたね。この `maybe` 式によって、分岐のロジックが完全に隠されました!

これをテストしてみると、以前と同じ結果が得られました。

```fsharp
let good = divideByWorkflow 12 3 2 1
レット バッド = ディバイドバイワークフロー 12 3 0 1
```


### "or else"テストの連鎖

先ほどの「divide by」の例では、各ステップが成功した場合にのみ続行したいと考えていました。

しかし、その逆の場合もあります。制御の流れは、一連の「or else」テストに依存していることがあります。1つのことを試して、それが成功すれば終わりです。そうでなければ、別のことを試し、それが失敗すれば、3つ目のことを試す、という具合です。

簡単な例を見てみましょう。3つの辞書があり、あるキーに対応する値を探したいとします。それぞれの検索は成功するか失敗するかわからないので、一連の検索を連鎖的に行う必要があります。

```fsharp
let map1 = [ ("1", "One"); ("2", "Two") ] |> Map.ofList
let map2 = [ ("A", "Alice"); ("B", "Bob") ] |> Map.ofList
let map3 = [ ("CA", "California"); ("NY", "New York") ] |> Map.ofList

let multiLookup key =
    match map1.TryFind key with
    | Some result1 -> Some result1   // success
    | None ->   // failure
        match map2.TryFind key with
        | Some result2 -> Some result2 // success
        | None ->   // failure
            match map3.TryFind key with
            | Some result3 -> Some result3  // success
            | None -> None // failure
```

F#ではすべてが式になっているので、アーリーリターンはできず、すべてのテストを単一の式でカスケードする必要があります。

これをどのように使うかを説明します。

```fsharp
multiLookup "A" |> printfn "Result for A is %A"
multiLookup "CA" |> printfn "Result for CA is %A"
multiLookup "X" |> printfn "Result for X is %A"
```

問題なく動作していますが、もっと簡単にできるのでしょうか？

その通りです。ここでは、このようなルックアップを単純化するための "or else"ビルダーを紹介します。

```fsharp
type OrElseBuilder() =
    member this.ReturnFrom(x) = x
    member this.Combine (a,b) =
        match a with
        | Some _ -> a // aは成功する -- それを使う
        | None -> b // aは失敗する -- 代わりにbを使う
    member this.Delay(f) = f()

let orElse = new OrElseBuilder()
```

これを使うためにルックアップのコードを変更する方法は以下の通りです。

```fsharp
let map1 = [ ("1", "One"); ("2", "Two") ] |> Map.ofList
let map2 = [ ("A", "Alice"); ("B", "Bob") ] |> Map.ofList
let map3 = [ ("CA", "California"); ("NY", "New York") ] |> Map.ofList

let multiLookup key = orElse {
    return! map1.TryFind key
    return! map2.TryFind key
    return! map3.TryFind key
    }
```

再度、このコードが期待通りに動作することを確認できます。

```fsharp
multiLookup "A" |> printfn "Result for A is %A"
multiLookup "CA" |> printfn "Result for CA is %A"
multiLookup "X" |> printfn "Result for X is %A"
```

### コールバックによる非同期呼び出し

最後に、コールバックについて説明します。 .NETで非同期処理を行うには、非同期処理が完了したときに呼び出される[AsyncCallback delegate](http://msdn.microsoft.com/en-us/library/ms228972.aspx)を使用するのが標準的な方法です。

このテクニックを使ったWebページのダウンロードの例を示します。

```fsharp
open System.Net
let req1 = HttpWebRequest.Create("http://fsharp.org")
let req2 = HttpWebRequest.Create("http://google.com")
let req3 = HttpWebRequest.Create("http://bing.com")

req1.BeginGetResponse((fun r1 ->
    use resp1 = req1.EndGetResponse(r1)
    printfn "Downloaded %O" resp1.ResponseUri

    req2.BeginGetResponse((fun r2 ->
        use resp2 = req2.EndGetResponse(r2)
        printfn "Downloaded %O" resp2.ResponseUri

        req3.BeginGetResponse((fun r3 ->
            use resp3 = req3.EndGetResponse(r3)
            printfn "Downloaded %O" resp3.ResponseUri

            ),null) |> ignore
        ),null) |> ignore
    ),null) |> ignore
```

`BeginGetResponse`と`EndGetResponse`への多くの呼び出しと、ネストされたラムダの使用は、これを理解するのを非常に困難にします。重要なコード(この場合は、print文だけです) は、コールバックロジックによって隠されます。

実際、コールバックの連鎖を必要とするコードでは、この階層化方式の扱いが常に問題になります。これは ["破滅のピラミッド"](http://raynos.github.com/presentation/shower/controlflow.htm?full#PyramidOfDoom) とさえ呼ばれています (ただし [いずれの解決策もあまりエレガントではありません](https://web.archive.org/web/201706092359/http://adamghill.com/callbacks-considered-a-nose/) ,とIMOは言っています) 。

もちろん、F#にはロジックを単純化しコードをフラットにする`async`コンピュテーション式が組み込まれているため、そのようなコードを記述する必要はありません。

```fsharp
open System.Net
let req1 = HttpWebRequest.Create("http://fsharp.org")
let req2 = HttpWebRequest.Create("http://google.com")
let req3 = HttpWebRequest.Create("http://bing.com")

async {
    use! resp1 = req1.AsyncGetResponse()
    printfn "Downloaded %O" resp1.ResponseUri

    use! resp2 = req2.AsyncGetResponse()
    printfn "Downloaded %O" resp2.ResponseUri

    use! resp3 = req3.AsyncGetResponse()
    printfn "Downloaded %O" resp3.ResponseUri

    } |> Async.RunSynchronously
```

このシリーズの後半では、`async`ワークフローがどのように実装されているかを具体的に見ていきます。

## まとめ ##

以上、コンピュテーション式の非常にシンプルな例を「前」と「後」の両方で見てきました。
これらの例は、コンピュテーション式が有用な問題の種類をよく表しています。

* ロギングの例では、各ステップ間で何らかの副作用を実行したいと考えました。
* 安全分割の例では、エラーをエレガントに処理して、ハッピーパスに集中したいと思いました。
* 複数辞書検索の例では、最初の成功を早期に返したいと考えました。
* 最後に、非同期の例では、コールバックの使用を隠し、"破滅のピラミッド"を避けようとしました。

すべてのケースに共通するのは、コンピュテーション式がそれぞれの式の "裏側で何かをしている"ということです。

よく似ていない例としては、コンピュテーション式をSVNやgitのポストコミット・フックのようなもの、あるいは更新のたびに呼び出されるデータベースのトリガのようなものと考えることができます。
つまりコンピュータ式は、自分のコードを密かに取り込んで*裏側で呼び出せるもの*で、それによって手前にある重要なコードに集中するためのものなのです。

なぜ"コンピュテーション式"と呼ばれるのでしょう?まあ、それはある種の式であることは間違いないです、その点は明らかです。F#チームはもともとこれを"裏側で何かを行うための式"と呼びたかったと思いますが,何らかの理由で少し扱いにくいと判断されたため、より短い"コンピュテーション式"という名称に変更したようです。

また、 "コンピュテーション式"と"ワークフロー"の意味の違いについては、"コンピュテーション式"*は"{..}"と"let!`構文を使用する必要するものとし、*"ワークフロー"*は必要に応じて特定の実装のために使います。すべてのコンピュテーション式実装がワークフローであるわけではありません。たとえば、"asyncワークフロー"や"maybeワークフロー"と言うのは適切ですが、"seqワークフロー"は不適切です。

言い換えると、次のコードの場合、`maybe`は使用しているワークフローであり、`{let!a=....return c}`という特定のコードの塊がコンピュテーション式です。

```fsharp
maybe
    {
    let! a = x |> divideBy y
    let! b = a |> divideBy w
    let! c = b |> divideBy z
    return c
    }
```

今すぐにでも自分のコンピュテーション式を作ってみたいと思うかもしれませんが、その前に少しだけ回り道をして連続式について説明します。それは次の機会に。


*2015-01-11更新:"状態"コンピュテーション式を使った集計例を削除しました。それはあまりにも複雑で、主な概念から逸脱していました。*
