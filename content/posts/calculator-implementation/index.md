---
layout: post
title: "Calculator Walkthrough: Part 2"
description: "Testing the design with a trial implementation"
date: 2014-10-14
categories: ["Worked Examples"]
seriesId: "Annotated walkthroughs"
seriesOrder: 2

---

今回の記事では、引き続き以下のようなシンプルなポケット電卓アプリを開発してみます。

![電卓画像](../calculator-design/calculator_1.png)

[前の記事](/posts/calculator-design/)では、型だけを使ってデザインの最初のドラフトを完成させました（UML図はありません！）。

次はその設計を使って実際に実装してみる番です。

この時点で実際にコーディングを行うことで、現実を確認することができます。ドメインモデルが実際に意味を持ち、抽象的すぎないことを確認します。
そしてもちろん、要件やドメインモデルについての質問が増えることもよくあります。


## First implementation

それでは、メインの電卓機能を実装してみて、どうなるか見てみましょう。

まず、各種類の入力にマッチし、それに応じて処理するスケルトンをすぐに作ることができます。

```fsharp
let createCalculate (services:CalculatorServices) :Calculate =
    fun (input,state) ->
        match input with
        | Digit d ->
            let newState = // do something
            newState //return
        | Op op ->
            let newState = // do something
            newState //return
        | Action Clear ->
            let newState = // do something
            newState //return
        | Action Equals ->
            let newState = // do something
            newState //return
```

このスケルトンでは、入力の種類ごとにケースを用意して、適切に処理していることがわかります。
すべてのケースで、新しいステートが返されることに注意してください。

このようなスタイルの関数の書き方は、奇妙に見えるかもしれません。もう少し詳しく見てみましょう。

まず、`createCalculate`は、電卓の関数そのものではなく、別の関数を*返す*関数であることがわかります。
返された関数は、`Calculate`型の値です。これが最後の`:Calculate`の意味です。

上の部分だけを見てみましょう。

```fsharp
let createCalculate (services:CalculatorServices) :Calculate =
    fun (input,state) ->
        match input with
            // code
```

関数を返しているので、ラムダを使って書くことにしました。それは`fun (input,state) -> `のためです。

しかし，次のように内部関数を使って書くこともできました。

```fsharp
let createCalculate (services:CalculatorServices) :Calculate =
    let innerCalculate (input,state) =
        match input with
            // code
    innerCalculate // return the inner function
```

どちらのアプローチも基本的には同じです* -- お好みでどうぞ。

{{<footnote "*">}}
多少の性能差はあるかもしれませんが。
{{</footnote>}}

## サービスのディペンデンシー・インジェクション

しかし、`createCalculate`は単に関数を返すだけではなく、`services`というパラメータを持っています。
このパラメータは、サービスの「依存性の注入」を行うために使用されます。

つまり、サービスは `createCalculate` 自身でのみ使用され、返される `Calculate` 型の関数では表示されません。

アプリケーションのすべてのコンポーネントを組み立てる「メイン」または「ブートストラップ」のコードは、以下のようになります。

```fsharp
// サービスの作成
let services = CalculatorServices.createServices()

// サービスを "factory "メソッドに注入する
let calculate = CalculatorImplementation.createCalculate services

// 返される "calculate "関数は、Calculate型です。
// そして、以下のようにUIに渡すことができます。

// UIの作成と実行
let form = new CalculatorUI.CalculatorForm(calculate)
form.Show()
```

## 実装：数字の扱い

それでは、計算機能の各部分の実装を始めましょう。まずは数字を扱うロジックから始めましょう。

メインの関数をすっきりさせるために、すべての処理をヘルパー関数の`updateDisplayFromDigit`に渡しましょう。

```fsharp
let createCalculate (services:CalculatorServices) :Calculate =
    fun (input,state) ->
        match input with
        | Digit d ->
            let newState = updateDisplayFromDigit services d state
            newState //return
```

`updateDisplayFromDigit`の結果から`newState`の値を作成し、それを別のステップで返していることに注意してください。

同じことを、以下のように、明示的な`newState`値を使わずに、1つのステップで行うこともできました。

```fsharp
let createCalculate (services:CalculatorServices) :Calculate =.
    fun (input,state) ->
        入力にマッチ
        | ディジット d ->
            updateDisplayFromDigit services d state
```

どちらの方法も自動的に最善ではありません。私は文脈に応じてどちらかを選択します。

単純なケースでは、余計な行は必要ないので避けますが、明示的な戻り値がある方が読みやすい場合もあります。
値の名前を見れば、戻り値のタイプがわかりますし、必要に応じてデバッガで確認することもできます。

さて、`updateDisplayFromDigit`を実装してみましょう。これはとても簡単です。

* まず、サービスで `updateDisplayFromDigit` を使用して、実際にディスプレイを更新します。
* 次に、新しいディスプレイから新しいステートを作成し、それを返します。

```fsharp
let updateDisplayFromDigit services digit state =
    let newDisplay = services.updateDisplayFromDigit (digit,state.display)
    let newState = {state with display=newDisplay}
    newState //return
```

## 実装：ClearとEqualsの処理

演算の実装に移る前に、よりシンプルな `Clear` と `Equals` の処理について見てみましょう。

`Clear`では、提供されている`initState`サービスを使って、状態を初期化するだけです。

`Equals`では、保留中の数学演算があるかどうかをチェックします。もしあれば、それを実行してディスプレイを更新し、そうでなければ何もしません。
このロジックは、`updateDisplayFromPendingOp`というヘルパー関数に記述します。

現在、`createCalculate`は次のようになっています。

```fsharp
let createCalculate (services:CalculatorServices) :Calculate =.
    fun (input,state) ->
        match input with
        | Digit d -> // as above
        | Op op -> // to do
        | Action Clear ->
            let newState = services.initState()
            newState //return
        | Action Equals ->
            let newState = updateDisplayFromPendingOp services state
            newState //return
```

さて、`updateDisplayFromPendingOp`です。数分考えて、以下のようなアルゴリズムでディスプレイを更新することにしました。

* まず、ペンディング中のオペがあるかどうかをチェックします。そうでなければ、何もしません。
* 次に、ディスプレイから現在の番号を取得しようとします。もしできなければ、何もしない。
* 次に、保留中の番号とディスプレイからの現在の番号でオペを実行します。もしエラーが出たら、何もしない。
* 最後に、ディスプレイを結果で更新し、新しい状態を返します。
* 新しい状態では、処理が完了しているため、pending opも`None`に設定されています。

このロジックを命令型のコードにすると、以下のようになります。

```fsharp
// updateDisplayFromPendingOpの最初のバージョン
// * 非常に命令的で醜い
let updateDisplayFromPendingOp services state =
    if state.pendingOp.IsSome then
        let op,pendingNumber = state.pendingOp.Value
        let currentNumberOpt = services.getDisplayNumber state.display
        if currentNumberOpt.IsSome then
            let currentNumber = currentNumberOpt.Value
            let result = services.doMathOperation (op,pendingNumber,currentNumber)
            match result with
            | Success resultNumber ->
                let newDisplay = services.setDisplayNumber resultNumber
                let newState = {display=newDisplay; pendingOp=None}
                newState //return
            | Failure error ->
                state // original state is untouched
        else
            state // original state is untouched
    else
        state // original state is untouched
```

ええっ！？家では試さないでくださいね。

このコードはアルゴリズムに正確に従っていますが、非常に醜く、またエラーが発生しやすいものです（オプションに `.Value` を使用するのはコードの臭いがします）。

プラス面としては、「サービス」を多用したことで、実際の実装の詳細から切り離されています。

では、どうすればより機能的に書き換えることができるでしょうか？


{{< linktarget "bind" >}}

## bindにぶつかる

このトリックは、「何かが存在すれば、その値に作用する」というパターンが、[ここ](/posts/computation-expressions-continuations/)や[ここ](/rop/computation-expressions-continuations/)
と [ここ](/rop/)で説明した`bind`パターンです。

bindパターンを効果的に使うためには、コードをいくつもの小さな塊に分けるのがよいでしょう。

まず，`if state.pendingOp.IsSome then do something`というコードは，`Option.bind`で置き換えることができます．

```fsharp
let updateDisplayFromPendingOp services state =
    let result =
        state.pendingOp
        |> Option.bind ???
```

しかし、この関数は状態を返さなければならないことを覚えておいてください。
バインドの全体的な結果が `None` であれば、新しい状態を作成したことにはならないので、渡された元の状態を返さなければなりません。

これを実現するには，組み込みの`defaultArg`関数を使います。この関数は，オプションに適用すると，オプションがあればその値を，`None`であれば第2パラメータを返します。

```fsharp
let updateDisplayFromPendingOp services state =
    let result =
        state.pendingOp
        |> Option.bind ???
    defaultArg result state
```

また、次のように結果を直接`defaultArg`にパイプすることで、少し整理することができます。

```fsharp
let updateDisplayFromPendingOp services state =
    State.pendingOp
    |> Option.bind ??
    |> defaultArg <| state
```

確かに`state`のリバースパイプは不思議な感じがしますが、これは後天的なものです。

では、次のステップに進みましょう。さて、`bind`のパラメータはどうでしょうか？ これが呼ばれると、pendingOpが存在することがわかるので、これらのパラメータを持つラムダを次のように書くことができます。

```fsharp
let result =
    state.pendingOp
    |> Option.bind (fun (op,pendingNumber) ->
        let currentNumberOpt = services.getDisplayNumber state.display
        // code
        )
```

あるいは，次のように，ローカルなヘルパー関数を作成して，それをbindに接続することもできます．

```fsharp
let executeOp (op,pendingNumber) =
    let currentNumberOpt = services.getDisplayNumber state.display
    /// etc

let result =
    state.pendingOp
    |> Option.bind executeOp
```

私自身は、ロジックが複雑な場合は、バインドの連鎖をシンプルにできる2番目のアプローチを好みます。
つまり，私は自分のコードを次のようにするようにしています。

```fsharp
let doSomething input = return an output option
let doSomethingElse input = return an output option
let doAThirdThing input = return an output option

state.pendingOp
|> Option.bind doSomething
|> Option.bind doSomethingElse
|> Option.bind doAThirdThing
```

この方法では、各ヘルパー関数の入力は非オプションですが、出力は常に*オプション*でなければならないことに注意してください。

## bindの実際の使い方

保留中の演算子を手に入れたら、次は足し算をするために現在の数字をディスプレイから取得します（あるいは何でも）。

たくさんのロジックを用意するのではなく、ヘルパー関数 (`getCurrentNumber`) はシンプルにしておきましょう。

* 入力はペア(op,pendingNumber)
* 出力は、currentNumberが`Some`であればトリプル(op,pendingNumber,currentNumber)、そうでなければ`None`となります。

つまり，`getCurrentNumber`のシグネチャは，`pair -> triple option`となり，`Option.bind`関数で使用可能であることが確認できます．

ペアをトリプルに変換するには？これは`Option.map`を使って，currentNumberオプションをトリプルオプションに変換すればよいのです．
currentNumberが`Some`であれば，マップの出力は`Some triple`となります．
一方，currentNumberが`None`であれば，マップの出力も`None`となります。

```fsharp
let getCurrentNumber (op,pendingNumber) =
    let currentNumberOpt = services.getDisplayNumber state.display
    currentNumberOpt
    |> Option.map (fun currentNumber -> (op,pendingNumber,currentNumber))

let result =
    state.pendingOp
    |> Option.bind getCurrentNumber
    |> Option.bind ???
```

パイプを使って，`getCurrentNumber`をもう少しイディオム的に書き換えることができます．

```fsharp
let getCurrentNumber (op,pendingNumber) =
    state.display
    |> services.getDisplayNumber
    |> Option.map (fun currentNumber -> (op,pendingNumber,currentNumber))
```

有効な値を持つトリプルができたので、数学演算のヘルパー関数を書くのに必要なものが揃いました。

* 入力としてトリプルを受け取ります（`getCurrentNumber`の出力です）。
* 演算処理を行います。
* そして、成功/失敗の結果をパターンマッチし、該当する場合は新しい状態を出力します。

```fsharp
let doMathOp (op,pendingNumber,currentNumber) =
    let result = services.doMathOperation (op,pendingNumber,currentNumber)
    match result with
    | Success resultNumber ->
        let newDisplay = services.setDisplayNumber resultNumber
        let newState = {display=newDisplay; pendingOp=None}
        Some newState //return something
    | Failure error ->
        None // failed
```

先ほどのifを入れ子にしたバージョンとは異なり、このバージョンでは成功時には`Some`を、失敗時には`None`を返すことに注意してください。

## エラーの表示

`Failure`ケースのコードを書いていて、あることに気づきました。
失敗した場合、それを *全く* 表示しておらず、ただ表示を放置しているだけです。エラーを表示すべきではないでしょうか？

おい、見落とされていた要件が見つかったぞ！」と。だから私は、設計したものをできるだけ早く実装したいのです。
すべてのケースに対応する実際のコードを書くと、必ずと言っていいほど「この場合はどうなるんだろう」という瞬間が訪れます。

では、この新しい要求をどうやって実装するのでしょうか？

そのためには、`MathOperationError`を受け取り、`CalculatorDisplay`を生成する新しい「サービス」が必要になります。

```fsharp
type SetDisplayError = MathOperationError -> CalculatorDisplay
```

そして，これを`CalculatorServices`構造体にも追加する必要があります．

```fsharp
type CalculatorServices = {
    // as before
    setDisplayNumber: SetDisplayNumber
    setDisplayError: SetDisplayError
    initState: InitState
    }
```

`doMathOp` は新しいサービスを使用するように変更できるようになりました。`Success` と `Failure`の両方のケースで、新しい表示が行われるようになり、それが新しいステートでラップされるようになりました。

```fsharp
let doMathOp (op,pendingNumber,currentNumber) = ...
    let result = services.doMathOperation (op,pendingNumber,currentNumber)
    let newDisplay =
        match result with
        | Success resultNumber ->
            services.setDisplayNumber resultNumber
        | Failure error ->
            services.setDisplayError error
    let newState = {display=newDisplay;pendingOp=None}
    Some newState //return something
```

結果に`Some`を残すことで、結果のパイプラインで`Option.bind`を残すことができるようになります*。
{{<footnote "*">}}。
代わりに `Some` を返さずに，結果パイプラインで `Option.map` を使うこともできます。
{{</footnote>}}となります。

以上をまとめると、`updateDisplayFromPendingOp`の最終バージョンが完成しました。
なお、defaultArgをパイピングに適したものにするために、`ifNone`ヘルパーも追加しています。

```fsharp
// defaultArgをパイピングに適したものにするヘルパー
let ifNone defaultValue input =
    // パラメータを逆にしてみましょう
    defaultArg input defaultValue

// 3番目のバージョンのupdateDisplayFromPendingOp
// * 失敗した場合にディスプレイにエラーを表示するように更新
// * 厄介な defaultArg 構文を置き換える
let updateDisplayFromPendingOp services state = !
    // CurrentNumberを抽出するヘルパー
    let getCurrentNumber (op,pendingNumber) =
        state.display
        |> services.getDisplayNumber
        |> Option.map (fun currentNumber -> (op,pendingNumber,currentNumber))

    // 計算を行うヘルパー op
    let doMathOp (op,pendingNumber,currentNumber) =
        let result = services.doMathOperation (op,pendingNumber,currentNumber)
        let newDisplay =
            match result with
            | Success resultNumber ->
                services.setDisplayNumber resultNumber
            | Failure error ->
                services.setDisplayError error
        let newState = {display=newDisplay;pendingOp=None}
        Some newState //return something

    // すべてのヘルパーを接続する
    state.pendingOp
    |> Option.bind getCurrentNumber
    |> Option.bind doMathOp
    |> ifNone state // 何か失敗したら元の状態に戻す
```

## bindの代わりに "may "計算式を使う

これまでは、bindを直接使っていました。 これにより、カスケード接続された `if/else` を取り除くことができました。

しかし、F#では、[computation expression](/posts/computation-expressions-intro/)を作成することで、別の方法で複雑さを隠すことができます。

今回はオプションを扱っているので、オプションをきれいに扱える「maybe」計算式を作ることができます。
(他の型を扱う場合は、型ごとに異なる計算式を作成する必要があります）。)

以下にその定義を示します -- たった4行です!
```fsharp
type MaybeBuilder() =
    member this.Bind(x, f) = Option.bind f x
    member this.Return(x) = Some x

let maybe = new MaybeBuilder()
```

この計算式が使えるようになると，bindの代わりに`maybe`が使えるようになり，コードは以下のようになります。

```fsharp
let doSomething input = return an output option
let doSomethingElse input = return an output option
let doAThirdThing input = return an output option

let finalResult = maybe {
    let! result1 = doSomething
    let! result2 = doSomethingElse result1
    let! result3 = doAThirdThing result2
    return result3
    }
```

私たちの場合、`updateDisplayFromPendingOp`の別のバージョンを書くことができます。

```fsharp
// updateDisplayFromPendingOpの4つ目のバージョン
// * "may" 計算式を使うように変更しました
let updateDisplayFromPendingOp services state = !

    // 計算式を実行するヘルパー
    let doMathOp (op,pendingNumber,currentNumber) =
        let result = services.doMathOperation (op,pendingNumber,currentNumber)
        let newDisplay =
            match result with
            | Success resultNumber ->
                services.setDisplayNumber resultNumber
            | Failure error ->
                services.setDisplayError error
        {display=newDisplay;pendingOp=None}

    // 2つのオプションを取得して組み合わせる
    let newState = maybe {
        let! (op,pendingNumber) = state.pendingOp
        let! currentNumber = services.getDisplayNumber state.display
        return doMathOp (op,pendingNumber,currentNumber)
        }
    newState |> ifNone state
```

この実装では、`services.getDisplayNumber` を直接呼び出すことができるので、`getCurrentNumber` ヘルパーはもう必要ないことに注意してください。

さて、これらのバリエーションのうち、どのバージョンがいいのでしょうか？

それは場合によります。

* [ROP](/rop/)アプローチのように、非常に強い「パイプライン」感がある場合は、明示的な `bind` を使用することを好みます。
* 一方で、さまざまな場所からオプションを引き出し、それらをさまざまな方法で組み合わせたい場合は、`maybe`という計算式がそれを容易にしてくれます。

ということで、今回は `maybe` を使った最後の実装にしてみます。

## 実装：数学演算の処理

これで、数学演算のケースの実装を行う準備が整いました。

まず、保留中の演算があれば、`Equals`の場合と同様に、その結果がディスプレイに表示されます。
しかし、*それに加えて*、新しい保留中の操作をステートにプッシュする必要があります。

算術演算のケースでは、状態の変換が*2回*行われ、`createCalculate`は次のようになります。

```fsharp
let createCalculate (services:CalculatorServices) :Calculate =
    fun (input,state) ->
        match input with
        | Digit d -> // as above
        | Op op ->
            let newState1 = updateDisplayFromPendingOp services state
            let newState2 = addPendingMathOp services op newState1
            newState2 //return
```

上記で`updateDisplayFromPendingOp`を定義しました。
そのため、操作をステートに反映させるためのヘルパー関数として、`addPendingMathOp`が必要になるだけです。

`addPendingMathOp` のアルゴリズムは以下の通りです。

* ディスプレイから現在の数字を取得しようとします。もしできなければ、何もしません。
* オペレータと現在の数値でステートを更新します。

これが醜いバージョンです。

```fsharp
// addPendingMathOpの最初のバージョン
// * 非常に命令的で醜い
let addPendingMathOp services op state =
    let currentNumberOpt = services.getDisplayNumber state.display
    if currentNumberOpt.IsSome then
        let currentNumber = currentNumberOpt.Value
        let pendingOp = Some (op,currentNumber)
        let newState = {state with pendingOp=pendingOp}
        newState //return
    else
        state // 元の状態はそのまま
```

繰り返しになりますが，`updateDisplayFromPendingOp`で使ったのと全く同じ手法を使って，より関数的にすることができます．

ここでは，`Option.map`と`newStateWithPending`ヘルパー関数を使った，よりイージーなバージョンを紹介します．

```fsharp
// addPendingMathOpの第2バージョン
// * "map "とヘルパー関数を使う
let addPendingMathOp services op state =
    let newStateWithPending currentNumber =
        let pendingOp = Some (op,currentNumber)
        {state with pendingOp=pendingOp}

    state.display
    |> services.getDisplayNumber
    |> Option.map newStateWithPending
    |> ifNone state
```

また、`maybe`を使ったものもあります。

```fsharp
// addPendingMathOpの第3バージョン
// * "maybe "を使う
let addPendingMathOp services op state =
    maybe {
        let! currentNumber =
            state.display |> services.getDisplayNumber
        let pendingOp = Some (op,currentNumber)
        return {state with pendingOp=pendingOp}
        }
    |> ifNone state // 何か失敗したら元の状態に戻す
```

先ほどと同様に、私はおそらく最後の `maybe` を使った実装を選ぶでしょう。しかし、`Option.map`を使った実装も良いでしょう。

## 実装: レビュー

さて，これで実装の部分は終わりです．コードを見てみましょう。

```fsharp
let updateDisplayFromDigit services digit state =
    let newDisplay = services.updateDisplayFromDigit (digit,state.display)
    let newState = {state with display=newDisplay}
    newState //return

let updateDisplayFromPendingOp services state =

    // helper to do the math op
    let doMathOp (op,pendingNumber,currentNumber) =
        let result = services.doMathOperation (op,pendingNumber,currentNumber)
        let newDisplay =
            match result with
            | Success resultNumber ->
                services.setDisplayNumber resultNumber
            | Failure error ->
                services.setDisplayError error
        {display=newDisplay;pendingOp=None}

    // fetch the two options and combine them
    let newState = maybe {
        let! (op,pendingNumber) = state.pendingOp
        let! currentNumber = services.getDisplayNumber state.display
        return doMathOp (op,pendingNumber,currentNumber)
        }
    newState |> ifNone state

let addPendingMathOp services op state =
    maybe {
        let! currentNumber =
            state.display |> services.getDisplayNumber
        let pendingOp = Some (op,currentNumber)
        return {state with pendingOp=pendingOp}
        }
    |> ifNone state // return original state if anything fails

let createCalculate (services:CalculatorServices) :Calculate =
    fun (input,state) ->
        match input with
        | Digit d ->
            let newState = updateDisplayFromDigit services d state
            newState //return
        | Op op ->
            let newState1 = updateDisplayFromPendingOp services state
            let newState2 = addPendingMathOp services op newState1
            newState2 //return
        | Action Clear ->
            let newState = services.initState()
            newState //return
        | Action Equals ->
            let newState = updateDisplayFromPendingOp services state
            newState //return
```

悪くないですね。全体の実装は60行以下のコードです。

## まとめ

私たちは、実装を行うことで、私たちの設計が妥当であることを証明しました -- さらに、見逃していた要件を見つけました。

[次の投稿](/posts/calculator-complete-v1/)では、サービスとユーザーインターフェイスを実装して、完全なアプリケーションを作成します。

*この記事のコードは、GitHubのこの[gist](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_implementation-fsx)にあります。