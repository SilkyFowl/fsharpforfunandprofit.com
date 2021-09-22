---
layout: post
title: "Calculator Walkthrough: Part 1"
description: "The type-first approach to designing a Calculator"
date: 2014-10-13
categories: ["Worked Examples","DDD"]
seriesId: "Annotated walkthroughs"
seriesOrder: 1

---

よく耳にするコメントに、F#や一般的な関数型プログラミングにおける理論と実践のギャップに対する不満があります。
つまり、理論は知っていても、実際にFPの原則を使ってアプリケーションを設計・実装するにはどうすればいいのかということです。

そこで、私が個人的に、いくつかの小さなアプリケーションを最初から最後までどのように設計・実装していくかをお見せすることが有益ではないかと考えました。

これは、注釈付きの「ライブコーディング」セッションのようなものです。問題を取り上げてコーディングを開始し、各段階での私の思考プロセスを紹介していきます。
私もミスをするので、それにどう対処するか、バックトラックやリファクタリングをどう行うかを見ていただきます。

なお、私はこのコードが量産可能なコードだとは言っていません。これからお見せするコードは、探索的なスケッチのようなものです。
結果として、より重要なコードにはしないような悪いこと(テストをしないなど)をしてしまいます。

このシリーズの最初の記事では、次のようなシンプルなポケット電卓アプリを開発します。

![電卓画像](./calculator_1.png)

## 私の開発アプローチ

私のソフトウェア開発のアプローチは、折衷的で実用的です。さまざまな技術を組み合わせたり、トップダウンとボトムアップのアプローチを交互に行ったりするのが好きです。

一般的には、要件から始めます。私は[要件駆動設計](http://fsharpforfunandprofit.com/posts/roman-numeral-kata/)のファンです。
理想的には、その分野の専門家になることです。

次に、[ドメイン駆動設計](http://fsharpforfunandprofit.com/ddd/)を用いてドメインのモデリングに取り組みます。
単なる静的なデータ(DDD用語では「集約」)ではなく、ドメインのイベント(["イベントストーミング"](http://ziobrando.blogspot.co.uk/2013/11/introducing-event-storming.html))に焦点を当てて、ドメインのモデリングに取り組みます。

モデリングプロセスの一環として、[タイプファースト開発](http://tomasp.net/blog/type-first-development.aspx/)を用いてデザインをスケッチします。
ドメインのデータタイプ(「名詞」)とドメインのアクティビティ(「動詞」)の両方を表す[型の作成](/series/designing-with-types.html)を行います。

ドメインモデルの最初のドラフトを作成した後、私は通常、「ボトムアップ」のアプローチに切り替え、これまでに定義したモデルを実行する小さなプロトタイプをコーディングします。

この時点で実際にコーディングを行うことで、現実を確認することができます。ドメインモデルが実際に意味を持ち、抽象的すぎないことを確認します。
そしてもちろん、要件やドメインモデルについての疑問がさらに深まることもよくあります。
そこで私はステップ1に戻り、いくつかの改良とリファクタリングを行い、満足できるまで繰り返します。

(もし私が大規模なプロジェクトでチームと仕事をしていたら、この時点で[実際のシステムを段階的に構築する](http://www.growing-object-oriented-software.com/)こともできるでしょう)
また、ユーザーインターフェースの開発にも着手することができます(ペーパープロトタイプなど)。これらの活動は、通常、さらに多くの質問や要件の変更を生み出します。
プロセス全体がすべてのレベルで循環しているのです)。

完璧な世界では、これが私のアプローチです。もちろん、実際には完璧な世界ではありません。悪質な管理者に悩まされることもあります。
理想的なプロセスを使用することはほとんどありません。

しかし、この例では、私がボスなので、もし結果が気に入らなければ、自分自身を責めるしかないのです。

## はじめに

では、さっそく始めてみましょう。まず何をすればいいのでしょうか？

通常、私は要件から始めるでしょう。 しかし、電卓の要件を書くのに多くの時間を費やす必要があるでしょうか？

私は怠け者なので、「いいえ」と答えます。その代わりに、電卓がどのように動作するかを知っている自信があるので、ただ飛び込んでみることにします。
(後で分かるように、私は間違っていました。興味深いエッジケースがいくつかあるので、要求事項を書いてみるのもいい練習になったでしょう。)

では、代わりに型優先の設計から始めましょう。

私のデザインでは、すべてのユースケースは、1つの入力と1つの出力を持つ関数です。

この例では、Calculatorのパブリックインターフェースを関数としてモデル化する必要があります。シグネチャーは以下の通りです。

```fsharp
type Calculate = CalculatorInput -> CalculatorOutput
```

簡単でしたね。 では、最初の質問ですが、他にモデル化する必要のあるユースケースはあるでしょうか？
今のところ、ないと思います。まずは、すべての入力を処理する1つのケースから始めましょう。


## 関数の入力と出力の定義

しかし、ここで新たに `CalculatorInput` と `CalculatorOutput` という2つの型を作りましたが、これらは定義されていません。
(これは未定義です(F#のスクリプトファイルにこれを入力すると、赤いスクウィグリーが表示されて気づかせることができます)。)
今のうちに定義しておきましょう。

先に進む前に、この関数の入力と出力の型は純粋でクリーンなものであることを明確にしておきます。
ドメインを設計する際には、文字列、プリミティブデータ型、検証などの面倒な世界を扱いたくありません。

その代わりに、一般的には、信頼できない面倒な世界から、私たちの素敵で純粋なドメインに変換する検証/変換関数があります。
そして、その逆を行う同様の関数が、出てきます。

![ドメインの入力と出力](./domain_input_output.png)


では、まず`CalculatorInput`に取り組んでみましょう。入力の構造はどのようになるでしょうか？

まず、当然のことながら、キー操作や、ユーザーの意図を伝える何らかの方法があるでしょう。
しかし、電卓はステートレスなので、何かの状態も渡す必要があります。この状態には、例えば、これまでに入力された数字などが含まれます。

出力に関しては、もちろん、関数は新しい、更新された状態を出力しなければなりません。

しかし、表示用にフォーマットされた出力を含む構造体など、他に何か必要なものがあるでしょうか？私はそうは思いません。
私たちは表示ロジックから自分たちを分離したいので、UIが状態を表示可能なものに変換するようにします。
表示できるようにします。

エラーについては？[他の記事](/rop/)では、エラー処理について多くの時間を費やしてきました。今回のケースでは必要なのでしょうか？

今回の場合は必要ないと思います。安物の電卓では、エラーがあればすぐに表示されますので、今のところはその方法でいきます。

では、新しいバージョンの関数を示します。

```fsharp
type Calculate = CalculatorInput * CalculatorState -> CalculatorState
```

ここで、`CalculatorInput`はキーストロークなどを意味し、`CalculatorState`は状態を意味しています。

この関数は、入力として[tuple](/posts/tuples/) (`CalculatorInput * CalculatorState`)を使って定義していることに注意してください。
2つの別々のパラメータ(`CalculatorInput -> CalculatorState -> CalculatorState`)ではなく、入力として[tuple](/post/tuples/) (`CalculatorInput * CalculatorState`)を使って定義していることに注目してください。
このようにしたのは、両方のパラメータが常に必要であり、タプルにすることでそれが明確になるからです。例えば、入力を部分的に適用したくはありません。

実際、型優先設計を行う際には、すべての関数に対してこのようにしています。すべての関数は1つの入力と1つの出力を持っています。
これは、後から部分的に適用する可能性がないということではなく、設計段階ではパラメータは1つだけにしたいということです。

また、純粋なドメインの一部ではないもの(設定や接続文字列など)は、この段階では*決して*表示されないことに注意してください。
ただし、実装時には、もちろん設計を実装する関数に追加されます。

## CalculatorState型の定義

それでは、`CalculatorState`を見てみましょう。今必要なのは、表示する情報を保持するものだと思います。

```fsharp
type Calculate = CalculatorInput * CalculatorState -> CalculatorState
and CalculatorState = {
    display: CalculatorDisplay
    }
```

まず、フィールドの値が何に使われるのかを明確にするための資料として、`CalculatorDisplay`という型を定義しました。
そしてもう一つは、実際に表示するものを決めるのを先延ばしにするためです。

では、ディスプレイのタイプは何にしましょうか？float? 文字列ですか？文字のリスト？複数のフィールドを持つレコード？

というのも、上で述べたように、エラーを表示する必要があるかもしれないからです。

```fsharp
type Calculate = CalculatorInput * CalculatorState -> CalculatorState
and CalculatorState = {
    display: CalculatorDisplay
    }
and CalculatorDisplay = string
```

型定義をつなぐのに、`and`を使っていることに注目してください。なぜでしょう？

F#は上から下に向かってコンパイルするので、型を使用する前に型を定義する必要があります。以下のコードはコンパイルされません。

```fsharp
type Calculate = CalculatorInput * CalculatorState -> CalculatorState
type CalculatorState = {
    display: CalculatorDisplay
    }
type CalculatorDisplay = string
```

宣言の順番を変えることで、この問題を解決することができます。
しかし、私は「スケッチ」モードにいるので、常に順序を変えたいとは思いません。
新しい宣言を一番下に追加して、`and`を使ってそれらをつなげます。

しかし、最終的な量産コードでは、デザインが安定してきたら、`and`を使わないように、これらの型を並べ替えることになります。
その理由は、`and`は[型の間のサイクルを隠してしまい](/posts/cyclic-dependencies/)、リファクタリングを妨げてしまうからです。

## CalculatorInput型の定義

`CalculatorInput`型では、電卓のボタンをすべて列挙してみます!

```fsharp
// as above
and CalculatorInput =
    | Zero | One | Two | Three | Four
    | Five | Six | Seven | Eight | Nine
    | DecimalSeparator
    | Add | Subtract | Multiply | Divide
    | Equals | Clear
```

なぜ入力に `char` を使わないのかと言う人もいるでしょう。しかし、上で説明したように、私のドメインでは、理想的なデータだけを扱いたいのです。
このように限られた選択肢を使うことで、想定外の入力に対処する必要がなくなります。

また、文字列ではなく抽象型を使うことの副次的な利点は、`DecimalSeparator`が "."であると想定されないことです。
実際のセパレータは、まず現在のカルチャを取得して(`System.Globalization.CultureInfo.CurrentCulture`)、次に`CurrentCulture`を使って取得します。
を取得し，次に `CurrentCulture.NumberFormat.CurrencyDecimalSeparator` を使用してセパレータを取得する必要があります．この実装の詳細をデザインから隠すことで
実際に使用するセパレータを変更しても、コードへの影響は最小限に抑えられます。

## デザインの改良: 数字の処理

以上で、デザインのファーストパスが完成しました。ここからは、より深く掘り下げて、内部処理を定義していきましょう。

まず、数字の処理について説明します。

数字のキーが押されたら、その数字を現在のディスプレイに追加したいと思います。これを表す関数型を定義しましょう。

```fsharp
type UpdateDisplayFromDigit = CalculatorDigit * CalculatorDisplay -> CalculatorDisplay
```

`CalculatorDisplay`は先ほど定義した型ですが、この`CalculatorDigit`は何でしょうか？

もちろん、入力として使用可能なすべての数字を表す型が必要です。
この関数では、`Add`や`Clear`などの他の入力は有効ではありません。

```fsharp
type CalculatorDigit =
    | Zero | One | Two | Three | Four
    | Five | Six | Seven | Eight | Nine
    | DecimalSeparator
```

さて、次の問題は、この型の値をどのようにして得るのかということです。次のように、`CalculatorInput`と`CalculatorDigit`を対応させる関数が必要でしょうか？

```fsharp
let convertInputToDigit (input:CalculatorInput) =
    match input with
        | Zero -> CalculatorDigit.Zero
        | One -> CalculatorDigit.One
        | etc
        | Add -> ???
        | Clear -> ???
```

多くの状況では、これは必要かもしれませんが、この場合はやりすぎのように思えます。
また、`Add`や`Clear`のような数字ではないものを、この関数はどう扱うのでしょうか？

そこで、`CalculatorInput`の型を再定義して、新しい型を直接使うようにしましょう。

```fsharp
type CalculatorInput =
    | Digit of CalculatorDigit
    | Add | Subtract | Multiply | Divide
    | Equals | Clear
```

ついでに、他のボタンも分類してみましょう。

私は、`Add｜Subtract｜Multiply｜Divide`を算術演算に分類します。
そして、`Equals | Clear`については、言葉が足りないので、「アクション」と呼ぶことにします。

以下は、新しい型 `CalculatorDigit`, `CalculatorMathOp`, `CalculatorAction` を含む、リファクタリングされた完全なデザインです。

```fsharp
type Calculate = CalculatorInput * CalculatorState -> CalculatorState
and CalculatorState = {
    display: CalculatorDisplay
    }
and CalculatorDisplay = string
and CalculatorInput =
    | Digit of CalculatorDigit
    | Op of CalculatorMathOp
    | Action of CalculatorAction
and CalculatorDigit =
    | Zero | One | Two | Three | Four
    | Five | Six | Seven | Eight | Nine
    | DecimalSeparator
and CalculatorMathOp =
    | Add | Subtract | Multiply | Divide
and CalculatorAction =
    | Equals | Clear

type UpdateDisplayFromDigit = CalculatorDigit * CalculatorDisplay -> CalculatorDisplay
```

これは唯一の方法ではありません。`Equals` と `Clear`を別々の選択肢として残すことも簡単にできました。

さて、`UpdateDisplayFromDigit`をもう一度見直してみましょう。何か他のパラメータが必要でしょうか？例えば、ステートの他の部分が必要でしょうか？

いいえ、他には何も思いつきません。これらの関数を定義するときは、できるだけ最小限にしたいと思っています。ディスプレイだけが必要なのに、なぜ電卓全体の状態を渡すのでしょうか？

また、`UpdateDisplayFromDigit`がエラーを返すことはありますか？例えば、無限に桁を増やすことはできないはずですが、それが許されない場合はどうなるのでしょうか？
また、エラーになるような他の入力の組み合わせはありますか? 例えば、10進数のセパレータだけを入力したとします。その場合はどうなるのでしょうか？

今回のプロジェクトでは、どちらも明示的なエラーにはならず、不正な入力は静かに拒否されると仮定します。
言い換えれば、例えば10桁の数字を入力した後、他の数字は無視されます。また、最初の小数点セパレータの後は、それ以降の小数点セパレータも無視されます。

残念ながら、このような要求をデザインに盛り込むことはできません。しかし、`UpdateDisplayFromDigit`
が明示的なエラータイプを返さないという事実は、少なくともエラーが静かに処理されることを示しています。

## Refining the design: the math operations

さて、次は算術演算に移りましょう。

これらはすべて二項演算で、2つの数値を受け取り、新しい結果を出力します。

これを表現する関数型は次のようになります。

```fsharp
type DoMathOperation = CalculatorMathOp * Number * Number -> Number
```

もし、`1/x`のような単項演算があれば、別の型が必要になるが、そうではないので、シンプルにすることができる。

次の決断：どのような数値型を使うべきか？汎用性を持たせるべきでしょうか？

ここでもシンプルに、`float`を使うことにしましょう。しかし、表現を少し切り離すために、`Number`というエイリアスを残しておきます。以下に更新したコードを示します。

```fsharp
type DoMathOperation = CalculatorMathOp * Number * Number -> Number
and Number = float
```


さて、上記の `UpdateDisplayFromDigit` と同様に、`DoMathOperation` について考えてみましょう。

質問1：これは最小限のパラメータセットですか？例えば、他の部分の状態が必要なのでしょうか？

回答 いいえ、他には何も思いつきません。

質問2： `DoMathOperation` がエラーを返すことはありますか？

回答 Yes! ゼロで割るのはどうですか？

では，エラーはどのように処理すればよいのでしょうか？
数学演算の結果を表す新しい型を作成して、それを `DoMathOperation` の出力としましょう。

新しい型である`MathOperationResult`は、`Success`と`Failure`の2つの選択肢(discriminated union)を持つことになります。

```fsharp
type DoMathOperation = CalculatorMathOp * Number * Number -> MathOperationResult
and Number = float
and MathOperationResult =
    | Success of Number
    | Failure of MathOperationError
and MathOperationError =
    | DivideByZero
```

ビルトインされている汎用の `Choice` 型や、完全な ["鉄道指向プログラミング"](/rop/) アプローチを使うこともできましたが、
これはデザインのスケッチなので、多くの依存関係なしにデザインを独立させたいので、ここで特定の型を定義することにしました。

他のエラーは？NaNとかアンダーフローとかオーバーフローとか？よくわかりません。`MathOperationError`という型があるので、必要に応じて簡単に拡張できそうです。

## 数字はどこから来るの？

`DoMathOperation` は `Number` 値を入力として使用するように定義しました。しかし、`Number`はどこから来るのでしょうか？

数字は入力された一連の数字から、その数字を浮動小数点に変換します。

1つの方法としては、文字列の表示と一緒に `Number` をステートに保存し、各桁が入力されるたびに更新するというものがあります。

ここでは、もっと単純な方法で、ディスプレイから直接数字を取得することにします。つまり、次のような関数が必要なのです。

```fsharp
type GetDisplayNumber = CalculatorDisplay -> Number
```

考えてみると、この関数は失敗する可能性があります。なぜなら、表示される文字列は「error」か何かだからです。そこで，代わりにオプションを返すことにしましょう。

```fsharp
type GetDisplayNumber = CalculatorDisplay -> Number option
```

同様に、結果が成功したときには、それを表示したいので、逆方向に動作する関数が必要です。

```fsharp
type SetDisplayNumber = Number -> CalculatorDisplay
```

この関数は絶対にエラーにならない(はず)なので、`option`は必要ありません。

## Refining the design: Handling a math operation input

演算の処理はまだ終わっていません。

入力が `Add` の場合、目に見える効果は何でしょうか？何もありません。

`Add` イベントは後で別の数字を入力する必要があるので、`Add` イベントは何らかの形で保留されています。
次の数字を待っているのです。

考えてみると、`Add`イベントを保留にしておくだけでなく、前の数字も保留にして、入力された最新の数字に追加できるようにしておかなければなりません。

これをどこで管理するのでしょうか？もちろん、`CalculatorState`の中です。

ここでは、新しいフィールドを追加する最初の試みを紹介します。

```fsharp
and CalculatorState = {
    display: CalculatorDisplay
    pendingOp: CalculatorMathOp
    pendingNumber: Number
    }
```

しかし，保留中の操作がない場合もあるので，オプションにする必要があります。

```fsharp
and CalculatorState = {
    display: CalculatorDisplay
    pendingOp: CalculatorMathOp option
    pendingNumber: Number option
    }
```

しかし、これも間違っています。 `pendingNumber`がないのに`pendingOp`があったり、その逆があったりするでしょうか？いいえ、これらは一緒に生きたり死んだりします。

これは、ステートがペアを含み、ペア全体がオプションであることを意味しています。

```fsharp
そしてCalculatorState = {
    display: CalculatorDisplay
    pendingOp: (CalculatorMathOp * Number) option
    }
```

しかし、ここでまだ1つ足りません。操作がpendingとして状態に追加された場合。
実際に操作が実行され、結果が表示されるのはいつですか？

答え: `Equals` ボタンが押されたとき、あるいは他の演算ボタンが押されたときです。これについては後ほど説明します。

## Refining the design: Handling the Clear button

もう一つのボタン、`Clear`ボタンを扱います。これは何をするものでしょうか？

もちろん、ディスプレイが空になるように状態をリセットし、保留されていた操作を削除します。

この関数を "clear"ではなく、`InitState`と呼ぶことにしました。

```fsharp
type InitState = unit -> CalculatorState
```

## サービスの定義

この時点で、ボトムアップ開発に切り替えるために必要なものがすべて揃いました。
私は、`Calculate`関数の試験的な実装を作ってみて、デザインが使えるかどうか、何か見落としていないかどうかを確認したいと思っています。

でも、全部を実装するのではなく、どうやって試験的な実装をすればいいのでしょうか？

そこで便利なのが、これらのタイプです。このタイプは、`calculate`関数が使用する一連の「サービス」を、実際には実装せずに定義することができます。

ここでは、次のように説明します。

```fsharp
type CalculatorServices = {
    updateDisplayFromDigit: UpdateDisplayFromDigit
    doMathOperation: DoMathOperation
    getDisplayNumber: GetDisplayNumber
    setDisplayNumber: SetDisplayNumber
    initState: InitState
    }
```

これで、`Calculate`関数の実装に注入できるサービスのセットができました。
これで、`Calculate`関数をすぐにコーディングして、サービスの実装は後回しにすることができます。

この時点では、小さなプロジェクトには無理があると思われるかもしれません。

確かに、これが[FizzBuzz Enterprise Edition](https://github.com/EnterpriseQualityCoding/FizzBuzzEnterpriseEdition)のようになってしまうことは避けたいですね。

しかし、私はここである原則を示しています。「サービス」をコア・コードから分離することで、すぐにプロトタイピングを始めることができます。
目標は本番用のコードベースを作ることではなく、設計上の問題点を見つけることです。今はまだ、要件発見の段階です。

このアプローチは馴染みがないかもしれませんが、これは、サービスのためにたくさんのインターフェースを作成し、
それらをコア・ドメインに注入するというOOOの原則と直接同じです。

## レビュー

サービスを追加したことで、初期設計は完了しました。
これまでのコードは以下の通りです。

```fsharp
type Calculate = CalculatorInput * CalculatorState -> CalculatorState
and CalculatorState = {
    display: CalculatorDisplay
    pendingOp: (CalculatorMathOp * Number) option
    }
and CalculatorDisplay = string
and CalculatorInput =
    | Digit of CalculatorDigit
    | Op of CalculatorMathOp
    | Action of CalculatorAction
and CalculatorDigit =
    | Zero | One | Two | Three | Four
    | Five | Six | Seven | Eight | Nine
    | DecimalSeparator
and CalculatorMathOp =
    | Add | Subtract | Multiply | Divide
and CalculatorAction =
    | Equals | Clear
and UpdateDisplayFromDigit =
    CalculatorDigit * CalculatorDisplay -> CalculatorDisplay
and DoMathOperation =
    CalculatorMathOp * Number * Number -> MathOperationResult
and Number = float
and MathOperationResult =
    | Success of Number
    | Failure of MathOperationError
and MathOperationError =
    | DivideByZero

type GetDisplayNumber =
    CalculatorDisplay -> Number option
type SetDisplayNumber =
    Number -> CalculatorDisplay

type InitState =
    unit -> CalculatorState

type CalculatorServices = {
    updateDisplayFromDigit: UpdateDisplayFromDigit
    doMathOperation: DoMathOperation
    getDisplayNumber: GetDisplayNumber
    setDisplayNumber: SetDisplayNumber
    initState: InitState
    }
```


## まとめ

これは非常に素晴らしいことだと思います。まだ「本当の」コードは書いていませんが、少し考えただけで、かなり詳細な設計ができています。

[次の投稿](/posts/calculator-implementation)では、この設計をテストして、実装を作ってみたいと思います。

*この記事のコードは、GitHubのこの[gist](https://gist.github.com/swlaschin/0e954cbdc383d1f5d9d3#file-calculator_design-fsx)にあります。

