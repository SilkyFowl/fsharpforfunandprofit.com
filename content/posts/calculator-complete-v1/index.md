---
layout: post
title: "Calculator Walkthrough: Part 3"
description: "Adding the services and user interface, and dealing with disaster"
date: 2014-10-15
categories: ["Worked Examples"]
seriesId: "Annotated walkthroughs"
seriesOrder: 3
---

今回の投稿では、シンプルなポケット電卓アプリの開発を続けます。

[最初の投稿](/posts/calculator-design/)では、型だけを使って(UML図は使わない！)デザインの最初のドラフトを完成させました。
[前の記事](/posts/calculator-implementation/)では、設計を実行し、欠けている要件を明らかにする初期実装を作成しました。

次は、残りのコンポーネントを構築し、完全なアプリケーションに組み立てる番です。

## サービスの作成

私たちには実装があります。しかし、この実装はいくつかのサービスに依存しており、そのサービスをまだ作成していません。

しかし実際には、この部分はとても簡単でわかりやすいです。ドメインで定義された型は、制約を強いるものです。
そのため、コードを書く方法は本当に1つしかありません。

すべてのコードを一度に表示します（以下）。後にコメントを追加します。

```fsharp
// ================================================
// Implementation of CalculatorConfiguration
// ================================================
module CalculatorConfiguration =

    // A record to store configuration options
    // (e.g. loaded from a file or environment)
    type Configuration = {
        decimalSeparator : string
        divideByZeroMsg : string
        maxDisplayLength: int
        }

    let loadConfig() = {
        decimalSeparator =
            System.Globalization.CultureInfo.CurrentCulture.NumberFormat.CurrencyDecimalSeparator
        divideByZeroMsg = "ERR-DIV0"
        maxDisplayLength = 10
        }

// ================================================
// Implementation of CalculatorServices
// ================================================
module CalculatorServices =
    open CalculatorDomain
    open CalculatorConfiguration

    let updateDisplayFromDigit (config:Configuration) :UpdateDisplayFromDigit =
        fun (digit, display) ->

        // determine what character should be appended to the display
        let appendCh=
            match digit with
            | Zero ->
                // only allow one 0 at start of display
                if display="0" then "" else "0"
            | One -> "1"
            | Two -> "2"
            | Three-> "3"
            | Four -> "4"
            | Five -> "5"
            | Six-> "6"
            | Seven-> "7"
            | Eight-> "8"
            | Nine-> "9"
            | DecimalSeparator ->
                if display="" then
                    // handle empty display with special case
                    "0" + config.decimalSeparator
                else if display.Contains(config.decimalSeparator) then
                    // don't allow two decimal separators
                    ""
                else
                    config.decimalSeparator

        // ignore new input if there are too many digits
        if (display.Length > config.maxDisplayLength) then
            display // ignore new input
        else
            // append the new char
            display + appendCh

    let getDisplayNumber :GetDisplayNumber = fun display ->
        match System.Double.TryParse display with
        | true, d -> Some d
        | false, _ -> None

    let setDisplayNumber :SetDisplayNumber = fun f ->
        sprintf "%g" f

    let setDisplayError divideByZeroMsg :SetDisplayError = fun f ->
        match f with
        | DivideByZero -> divideByZeroMsg

    let doMathOperation  :DoMathOperation = fun (op,f1,f2) ->
        match op with
        | Add -> Success (f1 + f2)
        | Subtract -> Success (f1 - f2)
        | Multiply -> Success (f1 * f2)
        | Divide ->
            try
                Success (f1 / f2)
            with
            | :? System.DivideByZeroException ->
                Failure DivideByZero

    let initState :InitState = fun () ->
        {
        display=""
        pendingOp = None
        }

    let createServices (config:Configuration) = {
        updateDisplayFromDigit = updateDisplayFromDigit config
        doMathOperation = doMathOperation
        getDisplayNumber = getDisplayNumber
        setDisplayNumber = setDisplayNumber
        setDisplayError = setDisplayError (config.divideByZeroMsg)
        initState = initState
        }
```

いくつかのコメントです。

* 小数点以下の区切り文字など、サービスをパラメータ化するためのプロパティを格納するコンフィギュレーションレコードを作成しました。
* コンフィグレーションレコードは、`createServices`関数に渡され、必要なサービスにコンフィグレーションを渡します。
* すべての関数は、`UpdateDisplayFromDigit`や`DoMathOperation`など、デザインで定義された型の1つを返すという同じアプローチを使用しています。
* 割り算の例外をトラップしたり、2つ以上の小数点セパレータが付加されないようにするなど、いくつかのトリッキーなエッジケースがあります。


## ユーザーインターフェースの作成

ユーザーインターフェイスには、WPFやウェブベースのアプローチではなく、WinFormsを使用します。これはシンプルで、WindowsだけでなくMono/Xamarinでも動作するはずです。
また、他のUIフレームワークにも簡単に移植できるはずです。

UI開発ではよくあることですが、私は他のどの部分よりもこの作業に時間を費やしました。
辛い反復作業は省き、最終バージョンに直接進むことにしました。

コードは約200行なので、すべては示しませんが（[gist](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_v1-fsx)で見ることができます）、いくつかのハイライトを紹介します。

```fsharp
module CalculatorUI =

    open CalculatorDomain

    type CalculatorForm(initState:InitState, calculate:Calculate) as this =
        inherit Form()

        // コンストラクタ前の初期化
        let mutable state = initState()
        let mutable setDisplayedText =
            fun text -> () // 何もしない
```

`CalculatorForm`は、例によって`Form`のサブクラスです。

コンストラクタには2つのパラメータがあります。
1つは`initState`という空の状態を作る関数で、もう1つは`calculate`という入力に基づいて状態を変化させる関数です。
つまり、ここでは標準的なコンストラクタベースの依存性注入を使用しているのです。

ミュータブルなフィールドが2つあります（驚）。

1つは状態そのものです。明らかに、ボタンが押されるたびに修正されます。

2つ目は、`setDisplayedText`という関数です。これはいったい何なのでしょうか？

ステートが変更された後、テキストを表示するコントロール（Label）を更新する必要があります。

これを行う標準的な方法は、次のようにラベルコントロールをフォーム内のフィールドにすることです。

```fsharp
type CalculatorForm(initState:InitState, calculate:Calculate) as this =
    inherit Form()

    let displayControl :Label = null
```

そして、フォームが初期化されたときに、実際のコントロールの値に設定します。

```fsharp
member this.CreateDisplayLabel() =
    let display = new Label(Text="",Size=displaySize,Location=getPos(0,0))
    display.TextAlign <- ContentAlignment.MiddleRight
    display.BackColor <- Color.White
    this.Controls.Add(display)

    // traditional style - set the field when the form has been initialized
    displayControl <- display
```

しかし、この方法では、ラベルコントロールが初期化される前に誤ってアクセスしようとすると、NREが発生するという問題があります。
また、誰でもどこからでもアクセスできる「グローバル」なフィールドを用意するよりも、望ましい動作に焦点を当てたいと思います。

関数を使用することで、(a)実際のコントロールへのアクセスをカプセル化し、(b)null参照の可能性を回避します。

mutable関数は、最初は安全なデフォルトの実装（`fun text -> ()`）で始まります。
そして，ラベルコントロールが作成されると，*新しい*実装に変更されます．

```fsharp
member this.CreateDisplayLabel() =
    let display = new Label(Text="",Size=displaySize,Location=getPos(0,0))
    this.Controls.Add(display)

    // update the function that sets the text
    setDisplayedText <-
        (fun text -> display.Text <- text)
```


## ボタンの作成

ボタンはグリッドに配置されているので、グリッド上の論理的な(row,col)から物理的な位置を取得するヘルパー関数 `getPos(row,col)` を作成しています。

以下はボタンの作成例です。

```fsharp
member this.CreateButtons() =
    let sevenButton = new Button(Text="7",Size=buttonSize,Location=getPos(1,0),BackColor=DigitButtonColor)
    sevenButton |> addDigitButton Seven

    let eightButton = new Button(Text="8",Size=buttonSize,Location=getPos(1,1),BackColor=DigitButtonColor)
    eightButton |> addDigitButton Eight

    let nineButton = new Button(Text="9",Size=buttonSize,Location=getPos(1,2),BackColor=DigitButtonColor)
    nineButton |> addDigitButton Nine

    let clearButton = new Button(Text="C",Size=buttonSize,Location=getPos(1,3),BackColor=DangerButtonColor)
    clearButton |> addActionButton Clear

    let addButton = new Button(Text="+",Size=doubleHeightSize,Location=getPos(1,4),BackColor=OpButtonColor)
    addButton |> addOpButton Add
```

また、すべての桁のボタンは、すべての数学のOPボタンと同様に同じ動作をするので、一般的な方法でイベントハンドラを設定するヘルパーを作成しました。

```fsharp
let addDigitButton digit (button:Button) =
    button.Click.AddHandler(EventHandler(fun _ _ -> handleDigit digit))
    this.Controls.Add(button)

let addOpButton op (button:Button) =
    button.Click.AddHandler(EventHandler(fun _ _ -> handleOp op))
    this.Controls.Add(button)
```

また、キーボードのサポートも追加しました。

```fsharp
member this.KeyPressHandler(e:KeyPressEventArgs) =
    match e.KeyChar with
    | '0' -> handleDigit Zero
    | '1' -> handleDigit One
    | '2' -> handleDigit Two
    | '.' | ',' -> handleDigit DecimalSeparator
    | '+' -> handleOp Add
    //など
```

ボタンのクリックやキーボードの押下は、最終的にキー関数である`handleInput`にルーティングされ、計算が行われます。

```fsharp
let handleInput input =
     let newState = calculate(input,state)
     state <- newState
     setDisplayedText state.display

let handleDigit digit =
     Digit digit |> handleInput

let handleOp op =
     Op op |> handleInput
```

ご覧の通り、`handleInput`の実装は些細なものです。
インジェクションされた計算関数を呼び出し、 mutable stateに結果を設定し、ディスプレイを更新しています。

これで、完全な計算機が完成しました。

この[gist](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_v1-fsx)からコードを取得して、F#スクリプトとして実行してみましょう。

## Disaster strikes!

簡単なテストから始めましょう。`1` `Add` `2` `Equals` と入力してみてください。何を期待しているのでしょうか？

あなたのことは知りませんが、私は電卓のディスプレイに「12」と表示されることは期待していません。

何が起こっているのでしょうか？ちょっとした実験をしてみると、私は本当に重要なことを忘れていました。
`Add` や `Equals`の操作が行われた場合、それ以降の数字は現在のバッファに追加されるのではなく、新しいバッファが開始されます。
いやいや、これはとんでもないバグですよ。

どこのバカが「コンパイルすればたぶん動く」と言ったのか、もう一度思い出してみてください。
{{<footnote "*">}実際には、そのバカは私（他の多くの人の中で）です。{{</footnote>}}

では、何が悪かったのでしょうか？

コードはコンパイルされましたが、期待通りに動作しませんでした。それはコードがバグっていたからではなく、*私の設計に欠陥があったからです。

言い換えれば、型優先設計プロセスで型を使用したということは、私が書いたコードが設計の正しい実装であるという高い信頼性を持っているということです。
しかし、要件や設計が間違っていれば、世界中の正しいコードを使っても解決できません。

次の記事で要件を再検討しますが、その間に、問題を修正するパッチを作ることはできないでしょうか？

## Fixing the bug

新しい桁を始めるときと、既存の桁に追加するだけのときの状況を考えてみましょう。
上で述べたように、算術演算や`Equals`では強制的にリセットされます。

そこで、これらの操作が行われたときにフラグを立ててはどうでしょうか。フラグがセットされていれば、新しいディスプレイバッファを開始します。
その後、フラグの設定を解除すれば、以前のように文字が追加されます。

では、どのような変更が必要なのでしょうか？

まず，フラグをどこかに保存する必要があります。もちろん、`CalculatorState`の中に格納します。

```fsharp
type CalculatorState = {
    display: CalculatorDisplay
    pendingOp: (CalculatorMathOp * Number) option
    allowAppend: bool
    }
```

(*現時点では良い解決策に見えるかもしれませんが、このようにフラグを使用することは、本当にデザイン上の問題です。
次の記事では、フラグを使わない[別のアプローチ](/posts/designing-with-types-representing-states/#replace-flags)を使ってみます)*。

## 実装の修正

この変更により、`CalculatorImplementation`のコードをコンパイルすると、新しい状態が作られるたびに壊れてしまうようになりました。

実際、F#を使っていて気に入っているのはこの点です -- レコードに新しいフィールドを追加するようなことは、壊れやすい変更ではなく
誤って見落としてしまうようなものではなく、重大な変更なのです。

コードに以下のような修正を加えます。

* `updateDisplayFromDigit`では、`allowAppend`がtrueに設定された新しいステートを返します。
* `updateDisplayFromPendingOp` と `addPendingMathOp` については、`allowAppend` を false に設定した新しい状態を返します。

## サービスの修正

ほとんどのサービスは問題ありません。現在壊れているのは `initState` だけで、これは起動時に `allowAppend` が true になるように調整するだけです。

```fsharp
let initState :InitState = fun () ->
    {
    display=""
    pendingOp = None
    allowAppend = true
    }
```

## ユーザーインターフェイスの修正

`CalculatorForm` クラスは、何の変更もなく動作を続けています。

しかし、この変更により、`CalculatorForm` が `CalculatorDisplay` 型の内部についてどの程度知っておくべきかという問題が発生します。

この場合、内部構造を変更するたびにフォームが壊れてしまう可能性があります。

それとも、`CalculatorDisplay` は不透明な型にすべきでしょうか？この場合、`CalculatorDisplay` 型からバッファを抽出して、フォームがそれを表示できるようにする別の「サービス」を追加する必要があります。
サービス」を追加する必要があります。

今のところ、変更があればフォームをいじってもいいと思っています。でも、もっと大きなプロジェクトや長期的なプロジェクトでは
しかし、より大きなプロジェクトや長期的なプロジェクトでは、依存性を減らそうとしているので、デザインの脆弱性を減らすために、ドメインタイプをできるだけ不透明にしたいと思います。

## パッチを当てたバージョンのテスト

それでは、パッチを当てたバージョンを試してみましょう (*パッチを当てたバージョンのコードは、この[gist](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_v1_patched-fsx)*から入手できます)。

これで動きますか？

はい、うまくいきました。`1` `Add` `2` `Equals` と入力すると、期待通りに `3` となります。

これで主要なバグが修正されました。ふう。

しかし、この実装で遊び続けていると、他の～～バグ～～文書化されていない機能にも遭遇することになります。

例えば、以下のようなものです。

* `1.0 / 0.0` は `Infinity` を表示します。私たちのゼロ除算エラーはどうなったのでしょうか?
* 変わった順序で演算を入力すると、奇妙な動作をします。例えば、`2 + + -` と入力すると、`8` と表示されます。

つまり、このコードはまだ目的に適合していないということです。


## テスト駆動型開発とは？

この時点で、あなたは自分自身に言い聞かせているかもしれません。「TDDを使っていれば、こんなことにはならなかったのに」と思うかもしれません。

確かに、これだけのコードを書いたのに、2つの数字をきちんと足せるかどうかをチェックするテストすら書いていませんでした。

もし私が最初にテストを書き、それに基づいて設計を進めていたら、きっとこの問題は起こらなかったでしょう。

この例では、おそらくすぐに問題を発見していたでしょう。
TDDのアプローチでは、「1 + 2 = 3」をチェックすることは、私が書いた最初のテストの1つだったでしょう。
しかし一方で、このような明らかな欠陥がある場合、どんなインタラクティブなテストでも問題が明らかになります。

私の考えでは、テスト駆動開発の利点は次のとおりです。

* 実装だけでなく、コードの「設計」を推進することができる。
* リファクタリングの際にコードが正しく保たれることが保証される。

では、実際の問題として、テスト駆動開発は、欠けている要件や微妙なエッジケースを見つけるのに役立つのでしょうか？
必ずしもそうではありません。テスト駆動開発が効果を発揮するのは、そもそも起こりうるすべてのケースを考えられるようになってからです。
その意味で、TDDは想像力の欠如を補うものではありません。

もし良い要件があれば、うまくいけば「違法な状態を表現できないように」型を設計することができます(/posts/designing-with-types-making-illegal-states-unrepresentable/)
そうすれば、テストで正しさを保証する必要がなくなります。

さて、私は自動テストに反対しているわけではありません。実際、私は特定の要件を検証するために、そして特に大規模な統合とテストのために、常に自動テストを使用しています。

例えば、このコードをどのようにテストするかというと、次のようになります。

```fsharp
module CalculatorTests =
    open CalculatorDomain
    open System

    let config = CalculatorConfiguration.loadConfig()
    let services = CalculatorServices.createServices config
    let calculate = CalculatorImplementation.createCalculate services

    let emptyState = services.initState()

    /// Given a sequence of inputs, start with the empty state
    /// and apply each input in turn. The final state is returned
    let processInputs inputs =
        // helper for fold
        let folder state input =
            calculate(input,state)

        inputs
        |> List.fold folder emptyState

    /// Check that the state contains the expected display value
    let assertResult testLabel expected state =
        let actual = state.display
        if (expected <> actual) then
            printfn "Test %s failed: expected=%s actual=%s" testLabel expected actual
        else
            printfn "Test %s passed" testLabel

    let ``when I input 1 + 2, I expect 3``() =
        [Digit One; Op Add; Digit Two; Action Equals]
        |> processInputs
        |> assertResult "1+2=3" "3"

    let ``when I input 1 + 2 + 3, I expect 6``() =
        [Digit One; Op Add; Digit Two; Op Add; Digit Three; Action Equals]
        |> processInputs
        |> assertResult "1+2+3=6" "6"

    // run tests
    do
        ``when I input 1 + 2, I expect 3``()
        ``when I input 1 + 2 + 3, I expect 6``()
```

もちろん、これは[NUnit or similar](/posts/low-risk-ways-to-use-fsharp-at-work-3/)を使って簡単に適応できるでしょう。

## How can I develop a better design?

失敗しました。先ほど言ったように、*実装そのもの*は問題ではありませんでした。タイプファーストの設計プロセスはうまくいったと思います。
本当の問題は、私が急ぎすぎて、要件をよく理解せずに設計に飛び込んでしまったことです。

次回以降、このようなことがないようにするにはどうしたらいいでしょうか？

ひとつの明白な解決策は、適切なTDDアプローチに切り替えることです。
しかし、私は少し頑固になって、タイプファーストの設計を続けられるかどうか試してみるつもりです。

[次の記事](/posts/calculator-complete-v2/)では、その場しのぎの過信をやめまて、
より徹底したプロセスを用いて、設計段階でこの種のエラーを防ぐことにします。

*この記事のコードはGitHubの[this gist (unpatched)](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_v1-fsx)で公開されています。
と[this gist (patched)](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_v1_patched-fsx)で公開されています。*。



