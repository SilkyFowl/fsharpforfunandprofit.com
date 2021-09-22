---
layout: post
title: "Calculator Walkthrough: Part 4"
description: "Designing using a state machine"
date: 2014-10-16
categories: ["Worked Examples"]
seriesId: "Annotated walkthroughs"
seriesOrder: 4
---

この一連の投稿では、シンプルなポケット電卓アプリを開発してきました。

[最初の投稿](/posts/calculator-design/)では、タイプファースト開発を用いてデザインのファーストドラフトを完成させました。
そして、[2つ目の投稿](/posts/calculator-implementation/)では、初期の実装を作成しました。

[前の投稿](/posts/calculator-complete-v1/)では、ユーザーインターフェイスを含む残りのコードを作成し、それを使おうとしました。

しかし、最終的には使い物になりませんでした。 問題はコードがバグっていたことではなく、事前に要件を十分に考えていなかったことでした。
問題はコードがバグっていたことではなく、コーディングを始める前に要件を十分に考えていなかったことです。

そうですか。フレッド・ブルックスの有名な言葉がある。「捨てるつもりでやれ、どうせ捨てるんだから」[(ちょっと単純化してますが)](http://www.davewsmith.com/blog/2010/brook-revisits-plan-to-throw-one-away)。

良いニュースは、以前の悪い実装から学び、デザインをより良くするための計画があることです。

## 悪いデザインを見直す

設計と実装（[this gist](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_v1_patched-fsx)を参照）を見ると、いくつかの点が目立ちます。

まず、`UpdateDisplayFromDigit`などのイベント処理タイプは、*context*、つまり電卓の現在の状態を考慮していませんでした。
パッチとして追加した `allowAppend` フラグは、コンテキストを考慮するための一つの方法でしたが、ひどく嫌な感じがします。

次に、特定の入力（`Zero`と`DecimalSeparator`）に対するちょっとした特殊なケースのコードがありましたが、これは次のコードスニペットからわかります。

```fsharp
let appendCh=
    match digit with
    | Zero ->
        // only allow one 0 at start of display
        if display="0" then "" else "0"
    | One -> "1"
    | // snip
    | DecimalSeparator ->
        if display="" then
            // handle empty display with special case
            "0" + config.decimalSeparator
        else if display.Contains(config.decimalSeparator) then
            // don't allow two decimal separators
            ""
        else
            config.decimalSeparator
```

これを見ると、これらの入力は、実装では隠さずに、*デザイン自体で*異なるものとして扱われるべきだと思います。
-- 結局のところ、私たちは設計をできるだけドキュメントとしても機能させたいのです。

## 設計ツールとしての有限状態機械の使用

その場しのぎのアプローチが失敗した場合、代わりに何をすべきでしょうか？

私は [finite state machines](https://en.wikipedia.org/wiki/Finite-state_machine)を使うことを支持しています。
(「FSM」 -- [True FSM](https://en.wikipedia.org/wiki/Flying_Spaghetti_Monster) と混同してはいけません)
プログラムがステートマシンとしてモデル化されていることには驚かされます。

ステートマシンを使うメリットは何でしょうか？ [別の投稿](/posts/designing-with-types-representing-states/)で述べたことを繰り返します。

**それぞれの状態には、異なる動作が許容されます。**
言い換えれば、ステートマシンは、文脈と、その文脈でどのような選択肢があるかを考えることを強いるものです。

今回のケースでは、`Add`が処理された後にコンテキストが変わったことを忘れていたため、桁の累積に関するルールも変わってしまいました。

**すべての状態が明示的に文書化されています**。
暗黙の了解となっている重要な状態があっても、それが文書化されていないことはよくあることです。

例えば、私はゼロと10進数のセパレータを処理する特別なコードを作成しました。現在は実装の中に埋もれていますが、本来は設計の一部であるべきなのです。

**これは、起こりうるすべての可能性を考えることを強いる設計ツールです。**
一般的なエラーの原因は、特定のエッジケースが処理されていないことですが、ステートマシンでは、すべてのケースを考えなければなりません。

今回のケースでは、最も明白なバグに加えて、適切に処理されていないエッジケースがまだあります。
例えば、ある演算処理の直後に、別の演算処理を行う場合です。ではどうすればいいのでしょうか？


## How to implement simple finite state machine in F# ##

皆さんは、言語パーサーや正規表現で使われるような複雑なFSMをご存知でしょう。
この種のステートマシンは、ルールセットや文法から生成され、非常に複雑です。

しかし、ここでお話しするステートマシンは、もっともっと単純なものです。
せいぜいいくつかのケースがあり、遷移の数も少ないので、複雑なジェネレータを使う必要はありません。

ここではその例を紹介します。
![ステートマシン](./state_machine_1.png)

では、このような単純なステートマシンをF#で実装するには、どのような方法があるのでしょうか。

FSMの設計と実装は、それ自体が複雑なテーマで、以下のような用語があります。
独自の用語（[NFAs and DFAs](https://en.wikipedia.org/wiki/Powerset_construction)、[Moore vs. Mealy](https://stackoverflow.com/questions/11067994/difference-between-mealy-and-moore)など）があります。
そして、その周りには[全ビジネス](http://www.stateworks.com/)が構築されています。

F#では、テーブルドリブン、相互に再帰的な関数、エージェント、OOスタイルのサブクラスなど、様々なアプローチが考えられます。

しかし、私の好みのアプローチは（アドホックな手動実装のために）ユニオン型とパターンマッチを多用することです。

まず、すべての状態を表すユニオン型を作ります。
例えば、"A"、"B"、"C "という3つの状態があるとすると、型は次のようになります。

```fsharp
type State =
    | AState
    | BState
    | CState
```

多くの場合、各ステートには、そのステートに関連するデータを格納する必要があります。
そのため、そのデータを保持するための型も作成する必要があります。

```fsharp
type State =
    | AState of AStateData
    | BState of BStateData
    | CState
and AStateData =
    {something:int}
and BStateData =
    {somethingElse:int}
```

次に、起こりうるすべてのイベントを別のユニオン型で定義します。イベントにデータが付随している場合は、それも追加します。

```fsharp
type InputEvent =
    | XEvent
    | YEvent of YEventData
    | ZEvent
and YEventData =
    {eventData:string}
```

最後に、現在の状態と入力イベントが与えられたときに、新しい状態を返す「トランジション」関数を作ります。

```fsharp
let transition (currentState,inputEvent) =
    match currentState,inputEvent with
    | AState, XEvent -> // 新しい状態
    | AState,YEvent -> // 新しい状態
    | AState, ZEvent -> // 新しい状態
    | BState, XEvent -> // 新しい状態
    | BState, YEvent -> // 新しい状態
    | CState, XEvent -> // 新しい状態
    | CState, ZEvent -> // 新しい状態
```

F#のようなパターンマッチングを持つ言語でのこのアプローチで気に入っているのは
F#のようなパターン・マッチングを備えた言語でこのアプローチが好きなのは、**状態とイベントの特定の組み合わせを処理し忘れると、コンパイラの警告が出る**ことです。これは素晴らしいことですね。

確かに、多くの状態と入力イベントを持つシステムでは、すべての可能な組み合わせを明示的に処理することを期待するのは無理があるかもしれません。
しかし、私の経験では、多くの厄介なバグは、処理すべきではないイベントを処理した場合に発生します。まさに、オリジナルのデザインが、すべきではないのに桁を蓄積したようにです。

あらゆる可能性のある組み合わせを考慮しなければならないというのは、設計上有益なことなのです。

さて、状態やイベントの数が少ない場合でも、可能な組み合わせの数はすぐに大きくなってしまいます。
実際に管理しやすくするために、私は通常、次のような一連のヘルパー関数を作成しています。

```fsharp
let aStateHandler stateData inputEvent =
    match inputEvent with
    | XEvent -> // 新しい状態
    | YEvent _ -> // 新しい状態
    | ZEvent -> // 新しい状態

let bStateHandler stateData inputEvent =
    match inputEvent with
    | XEvent -> // new state
    | YEvent _ -> // new state
    | ZEvent -> // new state

let cStateHandler inputEvent =
    match inputEvent with
    | XEvent -> // new state
    | YEvent _ -> // new state
    | ZEvent -> // new state

let transition (currentState,inputEvent) =
    match currentState with
    | AState stateData ->
        // new state
        aStateHandler stateData inputEvent
    | BState stateData ->
        // new state
        bStateHandler stateData inputEvent
    | CState ->
        // new state
        cStateHandler inputEvent
```

では、この方法で、上の状態図の実装を試みてみましょう。

```fsharp
let aStateHandler stateData inputEvent =
    match inputEvent with
    | XEvent ->
        // transition to B state
        BState {somethingElse=stateData.something}
    | YEvent _ ->
        // stay in A state
        AState stateData
    | ZEvent ->
        // transition to C state
        CState

let bStateHandler stateData inputEvent =
    match inputEvent with
    | XEvent ->
        // stay in B state
        BState stateData
    | YEvent _ ->
        // transition to C state
        CState

let cStateHandler inputEvent =
    match inputEvent with
    | XEvent ->
        // stay in C state
        CState
    | ZEvent ->
        // transition to B state
        BState {somethingElse=42}

let transition (currentState,inputEvent) =
    match currentState with
    | AState stateData ->
        aStateHandler stateData inputEvent
    | BState stateData ->
        bStateHandler stateData inputEvent
    | CState ->
        cStateHandler inputEvent
```

これをコンパイルしようとすると、すぐにいくつかの警告が表示されます。

* (near bStateHandler) `Incomplete pattern matches on this expression. 例えば、値 'ZEvent' はパターンでカバーされていないケースを示している可能性があります。`
* (near cStateHandler) `Incomplete pattern matches on this expression. たとえば、値 'YEvent (_)' は、パターンでカバーされないケースを示している可能性があります。`

これは本当に役に立ちます。つまり、いくつかのエッジケースを見逃していたということで、これらのイベントを処理するためにコードを変更する必要があります。

ところで、ワイルドカードマッチ（アンダースコア）を使ってコードを修正することは、絶対にやめてください。これでは目的が果たせません。
イベントを無視したいのであれば、明示的に行ってください。

修正したコードは以下の通りで、警告なしにコンパイルできます。

```fsharp
let bStateHandler stateData inputEvent =
    match inputEvent with
    | XEvent
    | ZEvent ->
        // stay in B state
        BState stateData
    | YEvent _ ->
        // transition to C state
        CState

let cStateHandler inputEvent =
    match inputEvent with
    | XEvent
    | YEvent _ ->
        // stay in C state
        CState
    | ZEvent ->
        // transition to B state
        BState {somethingElse=42}
```

*この例のコードは[this gist](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-statemachine-fsx)で見ることができます。*.

## 電卓のステートマシンを設計する

それでは、電卓のステートマシンをスケッチしてみましょう。最初の試みは以下の通りです。

![電卓のステートマシン v1](./calculator_states_1.png)

各状態はボックスになっていて、遷移のきっかけとなるイベント (数字や数学の演算、`Equals`など) は赤で表示されています。

`1` `Add` `2` `Equals` のような一連のイベントをたどっていくと、最終的には一番下の「結果を表示」という状態になることがわかります。

しかし、ゼロと小数のセパレータの処理をデザインレベルにまで高めたいと思ったことを覚えていますか？

そこで、これらの入力に対して特別なイベントを作成し、その後の小数点以下の区切り文字を無視する新しい状態「accumulate with decimal」を作成しましょう。

これがバージョン2です。

![Calculator State Machine v1](./calculator_states_2.png)

## ステートマシンの最終調整

> "良い芸術家はコピーする。偉大な芸術家は盗む。"
> -- Pablo Picasso ([but not really](http://quoteinvestigator.com/2013/03/06/artists-steal/))

この時点で、電卓のモデルにステートマシンを使うことを考えたのは、きっと私だけではないだろうと思いました。
もしかしたら、ちょっと調べて、他の人のデザインを～～盗むことができるかもしれない？

案の定、"calculator state machine "でググると、[this one](http://cnx.org/contents/9bac155d-509e-46a6-b48b-30731ed08ce6@2/Finite_State_Machines_and_the_)を含むいろいろな種類の結果が出てきました。
これには詳細な仕様と状態遷移図が載っています。

この図を見て、もう少し考えてみると、次のようなことがわかりました。

* クリア」状態と「ゼロ」状態は同じである。保留中の演算がある場合とない場合がある。
* 演算と`Equals`は、保留中の計算でディスプレイを更新するという点でよく似ています。
  唯一の違いは、保留中の演算が状態に追加されるかどうかです。
* エラーメッセージのケースは、絶対に別のステートである必要があります。これは`Clear`以外のすべての入力を無視します。

以上のことを踏まえて、バージョン 3 の状態遷移図を見てみましょう。

![Calculator State Machine v1](./calculator_states_3.png)

重要な遷移だけを表示しています。すべての遷移を表示すると、あまりにも膨大な量になってしまいます。
しかし、これで詳細な要件の検討を開始するのに十分な情報が得られます。

ご覧のように、5つの状態があります。

* ゼロステート
* アキュムレータステート
* アキュムレータデシマルステート
* 計算済み状態
* エラー状態

そして、6つの可能な入力があります。

* ゼロ
* ゼロ以外の桁
* デシマルセパレーター
* MathOp
* イコール
* Clear

それぞれの状態と、保存する必要のあるデータがあれば、それを記録してみましょう。

{{<rawtable>}}
<table class="table table-condensed table-striped">

<tr>
<th>State</th>
<th>Data associated with state</th>
<th>Special behavior?</th>
</tr>

<tr>
<td>ZeroState</td>
<td>(optional) pending op</td>
<td>Ignores all Zero input</td>
</tr>

<tr>
<td>AccumulatorState</td>
<td>buffer and (optional) pending op</td>
<td>Accumulates digits in buffer</td>
</tr>

<tr>
<td>AccumulatorDecimalState</td>
<td>buffer and (optional) pending op</td>
<td>Accumulates digits in buffer, but ignores decimal separators</td>
</tr>

<tr>
<td>ComputedState</td>
<td>Calculated number and (optional) pending op</td>
<td></td>
</tr>

<tr>
<td>ErrorState</td>
<td>Error message</td>
<td>Ignores all input other than Clear</td>
</tr>

</table>
{{</rawtable>}}


## 各状態とイベントの組み合わせを記録する

次に、それぞれの状態とイベントの組み合わせで何が起こるかを考えます。
上のサンプルコードのように、グループ分けをして、一度に一つの状態のイベントだけを処理すればよいようにします。

まずは、`ZeroState`の状態から始めましょう。各入力タイプのトランジションは以下のとおりです。

{{<rawtable>}}
<table class="table table-condensed table-striped">

<tr>
<th>Input</th>
<th>Action</th>
<th>New State</th>
</tr>

<tr>
<td>Zero</td>
<td>(ignore)</td>
<td>ZeroState</td>
</tr>

<tr>
<td>NonZeroDigit</td>
<td>Start a new accumulator with the digit.</td>
<td>AccumulatorState</td>
</tr>

<tr>
<td>DecimalSeparator</td>
<td>Start a new accumulator with "0."</td>
<td>AccumulatorDecimalState</td>
</tr>

<tr>
<td>MathOp</td>
<td>Go to Computed or ErrorState state.
   <br>If there is a pending op, update the display based on the result of the calculation (or error).
   <br>Also, if calculation was successful, push a new pending op, built from the event, using a current number of "0".
   </td>
<td>ComputedState</td>
</tr>

<tr>
<td>Equals</td>
<td>As with MathOp, but without any pending op</td>
<td>ComputedState</td>
</tr>

<tr>
<td>Clear</td>
<td>(ignore)</td>
<td>ZeroState</td>
</tr>

</table>
{{</rawtable>}}

このプロセスを `AccumulatorState` 状態で繰り返すことができます。ここでは、各タイプの入力に対するトランジションを示します。

{{<rawtable>}}
<table class="table table-condensed table-striped">

<tr>
<th>Input</th>
<th>Action</th>
<th>New State</th>
</tr>

<tr>
<td>Zero</td>
<td>Append "0" to the buffer.</td>
<td>AccumulatorState</td>
</tr>

<tr>
<td>NonZeroDigit</td>
<td>Append the digit to the buffer.</td>
<td>AccumulatorState</td>
</tr>

<tr>
<td>DecimalSeparator</td>
<td>Append the separator to the buffer, and transition to new state.</td>
<td>AccumulatorDecimalState</td>
</tr>

<tr>
<td>MathOp</td>
<td>Go to Computed or ErrorState state.
   <br>If there is a pending op, update the display based on the result of the calculation (or error).
   <br>Also, if calculation was successful, push a new pending op, built from the event, using a current number based on whatever is in the accumulator.
   </td>

<td>ComputedState</td>
</tr>

<tr>
<td>Equals</td>
<td>As with MathOp, but without any pending op</td>
<td>ComputedState</td>
</tr>

<tr>
<td>Clear</td>
<td>Go to Zero state. Clear any pending op.</td>
<td>ZeroState</td>
</tr>

</table>
{{</rawtable>}}

AccumulatorDecimalState`状態のイベント処理は、`DecimalSeparator`が無視されること以外は同じです。

ComputedState`状態はどうでしょう。入力の種類ごとに、以下のような遷移があります。

{{<rawtable>}}
<table class="table table-condensed table-striped">

<tr>
<th>Input</th>
<th>Action</th>
<th>New State</th>
</tr>

<tr>
<td>Zero</td>
<td>Go to ZeroState state, but preserve any pending op</td>
<td>ZeroState</td>
</tr>

<tr>
<td>NonZeroDigit</td>
<td>Start a new accumulator, preserving any pending op</td>
<td>AccumulatorState</td>
</tr>

<tr>
<td>DecimalSeparator</td>
<td>Start a new decimal accumulator, preserving any pending op</td>
<td>AccumulatorDecimalState</td>
</tr>

<tr>
<td>MathOp</td>
<td>Stay in Computed state. Replace any pending op with a new one built from the input event</td>
<td>ComputedState</td>
</tr>

<tr>
<td>Equals</td>
<td>Stay in Computed state. Clear any pending op</td>
<td>ComputedState</td>
</tr>

<tr>
<td>Clear</td>
<td>Go to Zero state. Clear any pending op.</td>
<td>ZeroState</td>
</tr>

</table>
{{</rawtable>}}

最後に、`ErrorState`状態はとても簡単です。 :

{{<rawtable>}}
<table class="table table-condensed table-striped">

<tr>
<th>Input</th>
<th>Action</th>
<th>New State</th>
</tr>

<tr>
<td>Zero, NonZeroDigit, DecimalSeparator<br>MathOp, Equals</td>
<td>(ignore)</td>
<td>ErrorState</td>
</tr>

<tr>
<td>Clear</td>
<td>Go to Zero state. Clear any pending op.</td>
<td>ZeroState</td>
</tr>

</table>
{{</rawtable>}}

## ステートをF#コードに変換する

ここまでの作業が終わったので、型への変換は簡単です。

主な型は以下の通りです。

```fsharp
type Calculate = CalculatorInput * CalculatorState -> CalculatorState
// five states
and CalculatorState =
    | ZeroState of ZeroStateData
    | AccumulatorState of AccumulatorStateData
    | AccumulatorWithDecimalState of AccumulatorStateData
    | ComputedState of ComputedStateData
    | ErrorState of ErrorStateData
// six inputs
and CalculatorInput =
    | Zero
    | Digit of NonZeroDigit
    | DecimalSeparator
    | MathOp of CalculatorMathOp
    | Equals
    | Clear
// data associated with each state
and ZeroStateData =
    PendingOp option
and AccumulatorStateData =
    {digits:DigitAccumulator; pendingOp:PendingOp option}
and ComputedStateData =
    {displayNumber:Number; pendingOp:PendingOp option}
and ErrorStateData =
    MathOperationError
```

これらの型を最初のデザイン（下記）と比較すると、`Zero`と`DecimalSeparator`には何か特別なものがあることがわかりました。
これらは、入力タイプの一級市民に昇格したのです。

```fsharp
// from the old design
type CalculatorInput =
    | Digit of CalculatorDigit
    | Op of CalculatorMathOp
    | Action of CalculatorAction

// from the new design
type CalculatorInput =
    | Zero
    | Digit of NonZeroDigit
    | DecimalSeparator
    | MathOp of CalculatorMathOp
    | Equals
    | Clear
```

また、旧デザインでは、すべてのコンテキストのデータを格納する単一のステートタイプ（以下）を持っていましたが、新デザインでは、各コンテキストごとにステートが*明示的に異なります。
`ZeroStateData`, `AccumulatorStateData`, `ComputedStateData`, `ErrorStateData`という型がこれを明らかにしています。

```fsharp
// from the old design
type CalculatorState = {
    display: CalculatorDisplay
    pendingOp: (CalculatorMathOp * Number) option
    }

// from the new design
type CalculatorState =
    | ZeroState of ZeroStateData
    | AccumulatorState of AccumulatorStateData
    | AccumulatorWithDecimalState of AccumulatorStateData
    | ComputedState of ComputedStateData
    | ErrorState of ErrorStateData
```

新しいデザインの基本がわかったところで、それによって参照される他の型を定義する必要があります。

```fsharp
and DigitAccumulator = string
and PendingOp = (CalculatorMathOp * Number)
and Number = float
and NonZeroDigit=
    | One | Two | Three | Four
    | Five | Six | Seven | Eight | Nine
and CalculatorMathOp =
    | Add | Subtract | Multiply | Divide
and MathOperationResult =
    | Success of Number
    | Failure of MathOperationError
and MathOperationError =
    | DivideByZero
```

そして最後に、サービスを定義します。

```fsharp
// 電卓自身が使用するサービス
type AccumulateNonZeroDigit = NonZeroDigit * DigitAccumulator -> DigitAccumulator
type AccumulateZero = DigitAccumulator -> DigitAccumulator
type AccumulateSeparator = DigitAccumulator -> DigitAccumulator
type DoMathOperation = CalculatorMathOp * Number * Number -> MathOperationResult
type GetNumberFromAccumulator = AccumulatorStateData -> Number

// UIやテストで使用されるサービス
type GetDisplayFromState = CalculatorState -> string
type GetPendingOpFromState = CalculatorState -> string

type CalculatorServices = {
    accumulateNonZeroDigit :AccumulateNonZeroDigit
    accumulateZero :AccumulateZero
    accumulateSeparator :AccumulateSeparator
    doMathOperation :DoMathOperation
    getNumberFromAccumulator :GetNumberFromAccumulator
    getDisplayFromState :GetDisplayFromState
    getPendingOpFromState :GetPendingOpFromState
    }
```

なお、ステートはもっと複雑なので、ステートから表示テキストを抽出するヘルパー関数 `getDisplayFromState` を追加しました。
このヘルパー関数は、UIや他のクライアント（テストなど）が表示するテキストを取得する必要がある場合に使用します。

また、`getPendingOpFromState`を追加して、保留中の状態をUIでも表示できるようにしました。

## ステートベースの実装の作成

先ほどのパターンを使って、ステートベースの実装を作ってみましょう。

*(完全なコードは[this gist](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_v2-fsx)にあります。)*。

まずは、状態遷移を行うメインの関数から始めましょう。

```fsharp
let createCalculate (services:CalculatorServices) :Calculate =.
    // 部分的に適用されたサービスでいくつかのローカル関数を作成する
    let handleZeroState = handleZeroState services
    let handleAccumulator = handleAccumulatorState services
    let handleAccumulatorWithDecimal = handleAccumulatorWithDecimalState services
    let handleComputed = handleComputedState services
    let handleError = handleErrorState

    fun (input,state) ->
        match state with
        | ZeroState stateData ->
            handleZeroState stateData input
        | AccumulatorState stateData ->
            handleAccumulator stateData input
        | AccumulatorWithDecimalState stateData ->
            handleAccumulatorWithDecimal stateData input
        | ComputedState stateData ->
            handleComputed stateData input
        | ErrorState stateData ->
            handleError stateData input
```

ご覧のように、各ステートごとに1つずつ、いくつかのハンドラに責任を渡していますが、これについては後述します。

しかし、その前に、新しいステートマシンベースのデザインを、以前に行った（バグだらけの！）デザインと比較することが有益ではないかと思いました。

以下は、以前のコードです。

```fsharp
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

この2つの実装を比較してみると、イベントからステートに重点が移っていることがわかります。
これは、2つの実装でメインのパターンマッチングがどのように行われているかを比較することでわかります。

* オリジナルのバージョンでは、入力に重点が置かれ、状態は二の次でした。
* 新しいバージョンでは、状態に焦点が当てられ、入力は二の次です。

文脈を無視して「状態」よりも「入力」に焦点を当てたことが、旧バージョンのデザインが悪かった理由です。

上で述べたことの繰り返しになりますが、多くの厄介なバグは、処理すべきでない時にイベントを処理することによって引き起こされます（オリジナルのデザインで見られたように）。
新しいデザインでは、最初から状態と文脈を明確に強調しているので、ずっと自信を持っています。

実は、このような問題に気づいているのは私だけではありません。
多くの人が、古典的な「イベント駆動型プログラミング」(https://en.wikipedia.org/wiki/Event-driven_programming)には欠陥があると考えています。
そして、より「ステートドリブンなアプローチ」を推奨しています（例えば、[こちら](http://www.barrgroup.com/Embedded-Systems/How-To/State-Machines-Event-Driven-Systems)や[こちら](http://seabites.wordpress.com/2011/12/08/your-ui-is-a-statechart/)）。
私がここで行ったように。

## ハンドラの作成

各状態遷移の要件はすでに文書化されているので、コードを書くのは簡単です。
まずは、`ZeroState`ハンドラのコードから。

```fsharp
let handleZeroState services pendingOp input =
    // create a new accumulatorStateData object that is used when transitioning to other states
    let accumulatorStateData = {digits=""; pendingOp=pendingOp}
    match input with
    | Zero ->
        ZeroState pendingOp // stay in ZeroState
    | Digit digit ->
        accumulatorStateData
        |> accumulateNonZeroDigit services digit
        |> AccumulatorState  // transition to AccumulatorState
    | DecimalSeparator ->
        accumulatorStateData
        |> accumulateSeparator services
        |> AccumulatorWithDecimalState  // transition to AccumulatorWithDecimalState
    | MathOp op ->
        let nextOp = Some op
        let newState = getComputationState services accumulatorStateData nextOp
        newState  // transition to ComputedState or ErrorState
    | Equals ->
        let nextOp = None
        let newState = getComputationState services accumulatorStateData nextOp
        newState  // transition to ComputedState or ErrorState
    | Clear ->
        ZeroState None // transition to ZeroState and throw away any pending ops
```

繰り返しになりますが、実際の作業は `accumulateNonZeroDigit` や `getComputationState` といったヘルパー関数で行われます。これらについては後ほど説明します。

以下は、`AccumulatorState`ハンドラのコードです。

```fsharp
let handleAccumulatorState services stateData input =
    match input with
    | Zero ->
        stateData
        |> accumulateZero services
        |> AccumulatorState  // stay in AccumulatorState
    | Digit digit ->
        stateData
        |> accumulateNonZeroDigit services digit
        |> AccumulatorState  // stay in AccumulatorState
    | DecimalSeparator ->
        stateData
        |> accumulateSeparator services
        |> AccumulatorWithDecimalState  // transition to AccumulatorWithDecimalState
    | MathOp op ->
        let nextOp = Some op
        let newState = getComputationState services stateData nextOp
        newState  // transition to ComputedState or ErrorState
    | Equals ->
        let nextOp = None
        let newState = getComputationState services stateData nextOp
        newState  // transition to ComputedState or ErrorState
    | Clear ->
        ZeroState None // transition to ZeroState and throw away any pending op
```

以下は、`ComputedState`ハンドラのコードです。

```fsharp
let handleComputedState services stateData input =
    let emptyAccumulatorStateData = {digits=""; pendingOp=stateData.pendingOp}
    match input with
    | Zero ->
        ZeroState stateData.pendingOp  // transition to ZeroState with any pending op
    | Digit digit ->
        emptyAccumulatorStateData
        |> accumulateNonZeroDigit services digit
        |> AccumulatorState  // transition to AccumulatorState
    | DecimalSeparator ->
        emptyAccumulatorStateData
        |> accumulateSeparator services
        |> AccumulatorWithDecimalState  // transition to AccumulatorWithDecimalState
    | MathOp op ->
        // replace the pending op, if any
        let nextOp = Some op
        replacePendingOp stateData nextOp
    | Equals ->
        // replace the pending op, if any
        let nextOp = None
        replacePendingOp stateData nextOp
    | Clear ->
        ZeroState None // transition to ZeroState and throw away any pending op
```

## ヘルパー関数

最後に，ヘルパー関数を見てみましょう．

アキュムレータヘルパーは些細なもので，適切なサービスを呼び出し，その結果を `AccumulatorData` レコードにまとめるだけです．

```fsharp
let accumulateNonZeroDigit services digit accumulatorData =
    let digits = accumulatorData.digits
    let newDigits = services.accumulateNonZeroDigit (digit,digits)
    let newAccumulatorData = {accumulatorData with digits=newDigits}
    newAccumulatorData // return
```

getComputationState`ヘルパーはもっと複雑で、このコードベース全体で最も複雑な関数だと思います。

以前に実装した `updateDisplayFromPendingOp` と非常によく似ています。
しかし、いくつかの変更点があります。

* ステートベースのアプローチにより、`services.getNumberFromAccumulator`コードは絶対に失敗しません。これは、人生をよりシンプルにするものです。
* `match result with Success/Failure` コードは、2つの*可能な状態を返すようになりました。それは、「ComputedState」または「ErrorState」です。
* これが `computeStateWithNoPendingOp` の役割です。

```fsharp
let getComputationState services accumulatorStateData nextOp =.

    // 与えられたdisplayNumberから新しいComputedStateを作成するヘルパー
    // そしてnextOpパラメータ
    let getNewState displayNumber =
        let newPendingOp =
            nextOp |> Option.map (fun op -> op,displayNumber )
        {displayNumber=displayNumber; pendingOp = newPendingOp }
        |> ComputedState

    let currentNumber =
        services.getNumberFromAccumulator accumulatorStateData

    // If there is no pending op, create a new ComputedState using the currentNumber
    let computeStateWithNoPendingOp =
        getNewState currentNumber

    maybe {
        let! (op,previousNumber) = accumulatorStateData.pendingOp
        let result = services.doMathOperation(op,previousNumber,currentNumber)
        let newState =
            match result with
            | Success resultNumber ->
                // If there was a pending op, create a new ComputedState using the result
                getNewState resultNumber
            | Failure error ->
                error |> ErrorState
        return newState
        } |> ifNone computeStateWithNoPendingOp

```

最後に、以前の実装には全くなかった新しいコードをご紹介します。

2つのmath opが連続して発生した場合、どうすればよいでしょうか？保留中の演算があれば、それを新しい演算に置き換えるだけです。

```fsharp
let replacePendingOp (computedStateData:ComputedStateData) nextOp =
    let newPending = maybe {
        let! existing,displayNumber  = computedStateData.pendingOp
        let! next = nextOp
        return next,displayNumber
        }
    {computedStateData with pendingOp=newPending}
    |> ComputedState
```

## 計算機の完成

アプリケーションを完成させるためには、先ほどと同じように、サービスとUIを実装するだけです。

偶然にも、以前のコードのほとんどすべてを再利用することができます。実際に変更したのは
入力イベントの構成方法で、これはボタンハンドラの作成方法に影響します。

ステートマシン版の電卓のコードは[こちら](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_v2-fsx)から入手できます。

新しいコードを試してみると、すぐに動作し、より堅牢になっていることがわかると思います。ステートマシン・ドリブンな設計の勝利です。

## 練習問題

このデザインが気に入って、似たようなものを作ってみたいと思ったら、以下のような練習ができます。

* まず、他の操作を追加してみましょう。`1/x`や`sqrt`のような単項演算を実装するためには、何を変更しなければならないでしょうか？
* いくつかの電卓には戻るボタンがあります。これを実装するには何が必要ですか？幸いなことに、すべてのデータ構造は不変なので、簡単にできるはずです。
* ほとんどの電卓は、保存と呼び出しが可能な1スロットのメモリを持っています。これを実装するには何を変えなければならないでしょうか?
* ディスプレイに10文字までしか表示できないというロジックは、デザインからはまだ隠されています。これを表示するにはどうしたらいいでしょうか？


## まとめ

この小さな実験が役に立ったことを願っています。私は確かに何かを学びました、すなわち
そして、最初からステートベースのアプローチを使用することを検討してください -- 長い目で見れば、時間の節約になるかもしれません

