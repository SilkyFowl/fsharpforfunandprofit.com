---
layout: post
title: "Dr Frankenfunctor and the Monadster"
description: "Or, how a 19th century scientist nearly invented the state monad"
date: 2015-07-07
categories: [Partial Application, Currying, Combinators]
image: "/posts/monadster/monadster_horror.jpg"
seriesId: "Handling State"
seriesOrder: 1
---

*UPDATE: [Slides and video from my talk on this topic](/monadster/)*

*Warning! この記事には、陰惨なトピック、苦しいアナロジー、モナドの議論が含まれています*。

何世代にもわたって、私たちはフランケンフアンクター博士の悲劇的な物語に魅了されてきました。
生命の力に魅了されています。
電気とガルバニズムの初期の実験。
そして最終的には、死んだ体の一部を集めて命を吹き込むという画期的な方法で、モナドスターを完成させました。

しかし、ご存知のように、その生物は逃げ出し、自由なモナドスターはコンピュータサイエンスの学会で暴れまわりました。
熟練したプログラマーの心にも恐怖を与えたのです。

![The horror, The horror](./monadster_horror.jpg)

*1990年に開催されたACM Conference on LISP and Functional Programmingで起きた恐ろしい出来事を紹介します。

その詳細はここでは繰り返しません。この話はまだ思い出すにはあまりにもひどいものです。

しかし、この悲劇に捧げられた何百万もの言葉の中で、一つのトピックが満足に取り上げられたことはありません。

*その生物はどのようにして組み立てられ、命を吹き込まれたのでしょうか？*

フランケンファンクター博士は、死んだ体のパーツからクリーチャーを作り、生命力を生み出すために稲妻を使って一瞬にしてそれらを動かしたことは分かっています。

しかし、体の各部分は全体に組み合わされなければならず、生命力は適切な方法で組み立てられた中を伝わらなければなりませんでした。
それも、雷が落ちた瞬間の一瞬である。

私はこの問題を長年にわたって研究し、最近になって、フランケンファンクター博士の個人的な実験ノートを、多大な費用をかけて入手することができました。

これでようやく、フランケンファンクター博士の技術を世に問うことができるようになりました。 お好きなようにお使いください。私はその道徳性については何も判断しない。
結局のところ、我々が作ったものが実際にどのような影響を及ぼすのかを問うことは、単なる開発者にはできないことなのです。

## 背景

まず始めに、基本的なプロセスを理解する必要があります。

まず知っておかなければならないのは、フランケンファンクター博士には全身が用意されていなかったということです。代わりに、クリーチャーは体のパーツの集合体から作られました。
-- 腕、足、脳、心臓など、その出所は曖昧で、語られないのが最善である。

フランケンファンクター博士は、死んだ体の一部から始めて、それにある種の生命力を注入した。その結果、2つのことが起こった。
その結果、生きている体の部分と、生きている部分に生命力の一部が伝達されたために残った、減少した生命力の2つが得られたのです。

この原理を示した図があります。

![原理](./monadster1.png)

しかし、これでは体の一部は1つしか作れません。どうすれば、複数のパーツを作ることができるのでしょうか？これが、Dr. Frankenfunctorが直面した課題です。

最初の問題は、私たちが持っている生命力の量には限りがあるということです。
つまり、2つ目の体のパーツをアニメーション化する必要があるとき、前のステップで得た残りのバイタルフォースしか利用できないということです。

では、2つのステップをつなげて、1つ目のステップのバイタルフォースを2つ目のステップの入力に送り込むにはどうしたらいいのでしょうか？

![ステップをつなぐ](./monadster_connect.png)

ステップを正しく連結したとしても、さまざまな生きた体のパーツを、何らかの方法で組み合わせる必要があります。しかし、私たちが「生きている」身体のパーツにアクセスできるのは、創造の瞬間だけです。
その一瞬の間に、どうやって組み合わせるのか。

![各ステップの出力を組み合わせる](./monadster_combine.png)

この2つの問題を解決するエレガントなアプローチを導き出したのは、フランケンファンクター博士の天才的な才能であり、今から紹介するアプローチです。

## 共通のコンテキスト

ボディパーツの組み立て方を説明する前に、残りの手順で必要となる共通の機能について少し説明します。

まず、ラベルタイプが必要です。フランケンファンクター博士は、使用したすべての部品の出所をラベルで示すことに非常にこだわっていました。

```fsharp
タイプ ラベル = 文字列
```

生命力は、シンプルなレコード型でモデル化します。

```fsharp
type VitalForce = {units:int}.
```

バイタルフォースは頻繁に使用するので、1つのユニットを抽出し、ユニットと残りのフォースのタプルを返す関数を作成します。

```fsharp
let getVitalForce vitalForce =
   let oneUnit = {units = 1}.
   let remaining = {units = vitalForce.units-1} // デクリメント
   oneUnit, remaining // 両方を返す
```

## 左足

一般的なコードの説明が終わったので、中身に戻りましょう。

フランケンファンクター博士のノートには、まず下肢が作られたと記録されています。研究室には左足が転がっていて、それが出発点となった。

```fsharp
type DeadLeftLeg = DeadLeftLeg of Label（デッドレフトレッグオブラベル
```

この脚から、同じラベルと1単位のバイタルフォースを持つ生きた脚を作ることができた。

```fsharp
type LiveLeftLeg = LiveLeftLeg of Label * VitalForce
```

作成関数の型署名は次のようになる。

```fsharp
タイプ MakeLiveLeftLeg =
    DeadLeftLeg * VitalForce -> LiveLeftLeg * VitalForce
```

そして、実際の実装は次のようになります。

```fsharp
let makeLiveLeftLeg (deadLeftLeg,vitalForce) =.
    // パターンマッチを使って死んだ脚からラベルを取得する
    let (DeadLeftLeg label) = deadLeftLeg
    // 1単位のバイタルフォースを得る
    let oneUnit, remainingVitalForce = getVitalForce vitalForce
    // ラベルとバイタルフォースから生きている脚を作る
    let liveLeftLeg = LiveLeftLeg (label,oneUnit)
    // 脚と残りのバイタルフォースを返す
    liveLeftLeg, remainingVitalForce
```

見ての通り、この実装は先ほどの図と正確に一致しています。

![バージョン1](./monadster1.png)

この時点で、フランケンファンクター博士は2つの重要な洞察を得ました。

1つ目は、[currying](/posts/currying/)のおかげで、この関数はタプルを受け取る関数から、各パラメータを順番に渡す2パラメータの関数に変換できるということでした。

![バージョン2](./monadster2.png)

そして、コードは次のようになりました。

```fsharp
type MakeLiveLeftLeg =
    DeadLeftLeg -> VitalForce -> LiveLeftLeg * VitalForce

let makeLiveLeftLeg deadLeftLeg vitalForce =.
    let (DeadLeftLeg ラベル) = deadLeftLeg
    let oneUnit, remainingVitalForce = getVitalForce vitalForce
    LiveLeftLeg = LiveLeftLeg (label,oneUnit) とする。
    LiveLeftLeg, 残りのバイタルフォース
```

2つ目の発見は、この同じコードは、「beetAlive」関数を返す関数として解釈できるということでした。

つまり、死んだ部分は手元にあるが、最後の瞬間まで生命力はないので、今すぐ死んだ部分を処理して、生命力があるときに使える関数
を返せば、生命力が湧いてきたときに使うことができます。

つまり、死んだ部品を渡すと、生命力があれば生きた部品を作る関数が返ってくるのです。

![バージョン3](./monadster3.png)

これらの「生きている状態になる」関数は、何らかの方法で組み合わせることができれば、「レシピのステップ」として扱うことができます。

コードは次のようになっています。

```fsharp
type MakeLiveLeftLeg =
    DeadLeftLeg -> (VitalForce -> LiveLeftLeg * VitalForce)

let makeLiveLeftLeg deadLeftLeg =.
    // 内部の中間関数を作る
    let becomeAlive vitalForce =
        let (DeadLeftLeg label) = deadLeftLeg
        let oneUnit, remainingVitalForce = getVitalForce vitalForce
        LiveLeftLeg = LiveLeftLeg (label,oneUnit) とする。
        LiveLeftLeg, 残りのバイタルフォース
    // 戻す
    生きる
```

よくわからないかもしれませんが、これは前のバージョンとまったく同じコード*で、書き方が少し違うだけです。

このcurried関数（2つのパラメータを持つ）は、通常の2つのパラメータを持つ関数として解釈することができます。
あるいは、もう一つの*1つの1つのパラメータ関数を返す*1つのパラメータ*関数と解釈することもできます。

もしこれがよくわからなければ，もっと単純な2パラメータの`add`関数の例を考えてみましょう．

```fsharp
let add x y =
    x + y
```

F#はデフォルトで関数をcurryするので，この実装はこの実装と全く同じです。

```fsharp
let add x =
    fun y -> x + y
```

また，中間関数を定義すると，以下のようになります．

```fsharp
let add x =
    レット addX y = x + y
    addX // 関数を返す
```

### モナドスター型の作成

今後は、生きた体のパーツを作るすべての関数に同じような手法を使えることがわかります。

これらの関数はすべて、次のようなシグネチャーを持つ関数を返します。VitalForce -> LiveBodyPart * VitalForce`.

簡単にするために、この関数のシグネチャに名前をつけましょう。`M`、これは "Monadster part generator "の略です。
そして、汎用的な型のパラメータ`'LiveBodyPart`を与えて、様々なボディパーツで使えるようにしましょう。

```fsharp
タイプ M<'LiveBodyPart> =
    VitalForce -> 'LiveBodyPart * VitalForce
```

これで、`makeLiveLeftLeg`関数の戻り値の型を、`:M<LiveLeftLeg>`で明示的にアノテーションすることができます。

```fsharp
let makeLiveLeftLeg deadLeftLeg :M<LiveLeftLeg> =.
    let becomeAlive vitalForce =
        let (DeadLeftLeg label) = deadLeftLeg
        let oneUnit, remainingVitalForce = getVitalForce vitalForce
        let liveLeftLeg = LiveLeftLeg (label,oneUnit)
        LiveLeftLeg, 残りのバイタルフォース
    生きる力
```

関数の残りの部分は、`becomeAlive`の戻り値がすでに`M<LiveLeftLeg>`と互換性があるので、変更はありません。

しかし、常に明示的にアノテーションをしなければならないのは好ましくありません。そこで、この関数を1つのcase union -- "M "と呼ぶ -- で囲み、独自の型を与えるのはどうでしょうか。こんな感じです。

```fsharp
タイプ M<'LiveBodyPart> =
    M of (VitalForce -> 'LiveBodyPart * VitalForce)
```

こうすることで、「モナドスターのパーツジェネレータ」と「タプルを返す普通の関数」を[区別することができます](https://stackoverflow.com/questions/2595673/state-monad-why-not-a-tuple)。

この新しい定義を使うためには、中間関数を返すときに、シングルケースユニオン `M` で包むように、コードを次のように調整する必要があります。

```fsharp
let makeLiveLeftLegM deadLeftLeg = (左脚)
    let becomeAlive vitalForce =
        let (DeadLeftLeg label) = deadLeftLeg
        let oneUnit, remainingVitalForce = getVitalForce vitalForce
        let liveLeftLeg = LiveLeftLeg (label,oneUnit)
        LiveLeftLeg, 残りのバイタルフォース
    // 変化した!
    M becomeAlive // この関数を1つのケースにまとめる union
```

この最後のバージョンでは，型シグネチャを明示的に指定しなくても正しく推論されます：死んだ左脚を受け取り，生きた脚の "M "を返す関数です．

```fsharp
val makeLiveLeftLegM : DeadLeftLeg -> M<LiveLeftLeg>.
```

なお、関数名を `makeLiveLeftLegM` に変更して、`LiveLeftLeg` の `M` を返すことを明確にしています。

### Mの意味

では、この「M」という型は具体的にどのような意味を持つのでしょうか？どうやって意味を理解すればいいのでしょうか？

ひとつの方法として、`M<T>`を`T`を作るための*レシピ*と考えるとよいでしょう。あなたが私に生命力を与えてくれたら、私はあなたに`T`を返します。

しかし、`M<T>`はどうやって無から`T`を作り出すことができるのでしょうか？

そこで、`makeLiveLeftLegM`のような関数が非常に重要になってきます。この関数はパラメータを受け取り、それを結果に「焼き付ける」のです。
その結果、似たようなシグネチャーを持つ「Mを作る」関数がたくさん出てきますが、それらはすべて次のようなものです。

![](./monadster5.png)

コードで言えば、次のようになります。

```fsharp
デッドパート -> M<ライブパート
```

今後の課題は、これらをいかにしてエレガントに組み合わせるかということです。

### 左足のテスト

それでは、これまでに得られた成果をテストしてみましょう。

まず、死んだ足を作り、それに`makeLiveLeftLegM`を使って、`M<LiveLeftLeg>`を取得します。

```fsharp
let deadLeftLeg = DeadLeftLeg "Boris"
let leftLegM = makeLiveLeftLegM deadLeftLeg
```

leftLegM "とは何でしょうか？何らかの生命力を与えて、生きた左足を作るためのレシピです。

便利なのは、このレシピを*前もって*、つまり雷が落ちる前に作ることができることです。

嵐が来て、雷が落ちて、10ユニットのバイタルフォースが使えるようになったとしましょう。

```fsharp
let vf = {units = 10}.
```

さて、`leftLegM`の中には、バイタルフォースに適用できる関数が入っています。
しかし，まず，パターンマッチを使ってラッパーから関数を取り出す必要がある．

```fsharp
let (M innerFn) = leftLegM
```

そして、内部の関数を実行して、生きている左足と残りのバイタルフォースを得ることができる。

```fsharp
let liveLeftLeg, remainingAfterLeftLeg = innerFn vf
```

結果は次のようになります。

```テキスト
val liveLeftLeg : LiveLeftLeg = ライブレフトレッグ
   LiveLeftLeg ("Boris",{units = 1;})
val remainingAfterLeftLeg : VitalForce = {units = 9;}.
   {units = 9;}
```

LiveLeftLeg "の生成に成功し，残りのバイタルフォースが9個になったことがわかります．

このパターンマッチは厄介なので、内部の関数をアンラップして一度に呼び出すヘルパー関数を作ってみましょう。

これを`runM`と呼び、次のようにします。

```fsharp
let runM (M f) vitalForce = f vitalForce
```

上のテストコードは次のように簡単になります。

```fsharp
let liveLeftLeg, remainingAfterLeftLeg = runM leftLegM vf
```

これでようやく、生きた左足を作ることができる関数ができました。

動作させるのに時間がかかりましたが、今後使える便利なツールやコンセプトを構築することができました。

# #右足

何をしているのかがわかったので、今度は他の体の部位にも同じテクニックを使えるようになるはずです。

では、右足はどうだろう？

残念ながら、ノートによると、フランケンファンクター博士は実験室で右足を見つけることができませんでした。この問題はハックで解決しましたが...その話は後ほど。

# #左腕

次に、腕が作られた。まずは左腕だ。

しかし、問題がありました。 研究室には、壊れた左腕しかありませんでした。最終的なボディに使用するには、この腕を治さなければならない。

フランケンファンクター博士は医者なので、折れた腕の治し方は知っていたが、それは生きている腕に限られていた。 死んだ腕の骨折を治そうとしても、それは不可能です。

コードで言えば、こうなります。

```fsharp
type DeadLeftBrokenArm = DeadLeftBrokenArm of Label

// 壊れた腕のライブバージョン。
type LiveLeftBrokenArm = LiveLeftBrokenArm of Label * VitalForce

// 死んだバージョンが存在しない、健常者の腕のライブバージョン
タイプ LiveLeftArm = LiveLeftArm of Label * VitalForce

// 壊れた左腕を健常者の左腕にすることができる手術
type HealBrokenArm = LiveLeftBrokenArm -> LiveLeftArm
```

そこで課題となったのは、「手持ちの材料でどうやって生きた左腕を作るか」ということです。

まず、「DeadLeftUnbrokenArm」から「LiveLeftArm」を作ることはできません。また、「DeadLeftBrokenArm」を健康な「LiveLeftArm」に直接変換することもできません。

![Map dead to dead](./monadster_map1.png)

しかし、私たちが*できることは、`DeadLeftBrokenArm`を*生きた*壊れた腕に変えて、生きた壊れた腕を治すことですよね？

![生きた腕を直接作ることはできません](./monadster_map2.png)

残念ながら、それはできません。 ライブパーツを直接作成することはできません。ライブパーツは、`M`レシピのコンテキストの中でのみ作成できます。

そこで必要なのは、`M<LiveBrokenArm>`を`M<LiveArm>`に変換する特別バージョンの`healBrokenArm`（`healBrokenArmM`と呼ぶ）を作ることです。

![ライブブロークンアームを直接作ることはできません](./monadster_map3.png)

しかし、このような関数をどうやって作るのでしょうか？ また、その一部として `healBrokenArm` を再利用するにはどうすればよいでしょうか？

まずは、最も簡単な実装方法から説明します。

まず、この関数は `M` を返すので、先ほどの `makeLiveLeftLegM` 関数と同じ形になります。
vitalForceのパラメータを持つ内側の関数を作り、それを`M`で包んで返す必要があります。

しかし、先ほどの関数とは異なり、この関数にはパラメータとして`M`もあります（`M<LiveBrokenArm>`）。 この入力から必要なデータを取り出すにはどうすればいいでしょうか？

単純に、vitalForceで動かすだけです。 では、そのvitalforceはどこから得られるのでしょうか？それは、内部関数のパラメータからです。

というわけで、完成版は以下のようになります。

```fsharp
// HealBrokenArmの実装
let healBrokenArm (LiveLeftBrokenArm (label,vf)) = LiveLeftArm (label,vf)

/// M<LiveLeftBrokenArm>をM<LiveLeftArm>に変換する。
let makeHealedLeftArm brokenArmM =.

    // vitalForceのパラメータを受け取る新しい内部関数を作成する
    let healWhileAlive vitalForce =.
        // 入力されたbrokenArmMをvitalForceで実行する
        // 壊れた腕を得るために
        let brokenArm,remainVitalForce = runM brokenArmM vitalForce

        // 壊れた腕を治す
        let healedArm = healBrokenArm brokenArm

        // 癒された腕と残りのVitalForceを返す
        healedArm, remainingVitalForce

    // 内部の関数をラップして返す
    M HealWhileAlive
```

このコードを評価すると，シグネチャが得られます．

```fsharp
val makeHealedLeftArm : M<LiveLeftBrokenArm> -> M<LiveLeftArm>.
```

これはまさに私たちが望むものです。

しかし、そうはいきません。もっと良い方法があります。

HealBrokenArm "という変換をハードコーディングしてしまったのです。もし、他の体の部位に、他の変換をしたい場合はどうしたらいいでしょうか？
この関数をもう少し汎用的にすることはできないでしょうか？

はい、簡単にできます。次のように、体の一部を変換する関数("f")を渡せばいいのです。

```fsharp
let makeGenericTransform f brokenArmM =.

    // vitalForceのパラメータを受け取る新しい内部関数を作る
    let healWhileAlive vitalForce =
        let brokenArm,remainVitalForce = runM brokenArmM vitalForce

        // 渡されたfを使って壊れた腕を治す
        let healedArm = f brokenArm
        癒された腕、残りのバイタルフォース

    M HealWhileAlive
```

驚くべきことに、この一つの変換を `f` パラメータでパラメータ化することで、*全体* の関数がジェネリックになるのです!

他には何も変更していませんが、`makeGenericTransform`のシグネチャはもはや腕を参照していません。どんなものでも使えます。

```fsharp
val makeGenericTransform : f:('a -> 'b) -> M<'a> -> M<'b>)
```

### mapMの紹介

今はとても汎用的なので、名前が混乱しています。名前を変えてみましょう。
mapM`と呼ぶことにします。 これは *あらゆる* ボディパーツと *あらゆる* トランスフォームに対応します。

内部の名前も修正した実装は以下の通りです。

```fsharp
let mapM f ボディパーツM =
    let transformWhileAlive vitalForce =
        let bodyPart,remainingVitalForce = runM bodyPartM vitalForce
        更新されたボディ・パート = f ボディ・パート
        更新された体の部分、残りのバイタルフォース
    M トランスフォームウォリアライヴ
```

特に、`healBrokenArm`関数と連動しているので、`M`を使えるようにしたバージョンの "heal "を作るには、次のように書けばよいでしょう。

```fsharp
let healBrokenArmM = mapM healBrokenArm
```

![mapM with heal](./monadster_map4.png)

### mapMの重要性

mapM`を考える一つの方法は、「関数変換器」であることです。任意の「通常の」関数が与えられると、それを入力と出力が `M` である関数に変換します。

![mapM](./monadster_mapm.png)

mapM`に似た関数は様々な場面で登場します。例えば、`Option.map`は、「通常の」関数を、入力と出力がオプションである関数に変換します。
同様に、`List.map`は、「通常の」関数を、入力と出力がリストである関数に変換します。他にもたくさんの例があります。

```fsharp
// マップはオプションで動作する
let healBrokenArmO = Option.map healBrokenArm
// LiveLeftBrokenArmオプション -> LiveLeftArmオプション

// マップはリストで動作します
let healBrokenArmmL = List.map healBrokenArm
// LiveLeftBrokenArmのリスト -> LiveLeftArmのリスト
```

ラッパー」タイプの `M` には、Option や List のような単純なデータ構造ではなく、*関数* が含まれていることが、皆さんにとって新しい発見かもしれません。頭が痛くなるかもしれませんね。

さらに、上の図は、`M` は *あらゆる* 通常の型をラップすることができ、`mapM` は *あらゆる* 通常の関数をマッピングすることができることを意味しています。

試しにやってみましょう。

```fsharp
let isEven x = (x%2 = 0) // int -> bool
// 写す
let isEvenM = mapM isEven // M<int> -> M<bool>

let isEmpty x = (String.length x)=0 // string -> bool
// 写す
let isEmptyM = mapM isEmpty // M<string> -> M<bool>
```

というわけで、はい、動きました!

### 左腕のテスト

ここまででできたことをもう一度テストしてみましょう。

まず、壊れた腕を作り、`makeLiveLeftBrokenArm`を使って、`M<BrokenLeftArm>`を取得します。

```fsharp
let makeLiveLeftBrokenArm deadLeftBrokenArm = (デッドレフトブロッケンアーム)
    let (DeadLeftBrokenArm label) = deadLeftBrokenArm
    let becomeAlive vitalForce =
        let oneUnit, remainingVitalForce = getVitalForce vitalForce
        let liveLeftBrokenArm = LiveLeftBrokenArm (label,oneUnit)
        LiveLeftBrokenArm, remainingVitalForce
    M becomeAlive

/// 死んだ左ブロークンアームを作る
let deadLeftBrokenArm = DeadLeftBrokenArm "Victor"

/// 死んだものからM<BrokenLeftArm>を作る
let leftBrokenArmM = makeLiveLeftBrokenArm deadLeftBrokenArm
```

あとは、`mapM`と`healBrokenArm`を使って、`M<BrokenLeftArm>`を`M<LeftArm>`に変換することができる。

```fsharp
let leftArmM = leftBrokenArmM |> mapM healBrokenArm
```

leftArmM`には、壊れていない生きた左腕を作るためのレシピが入っています。あとは、生命力を加えるだけです。

以前のように、雷が落ちる前に、これらのことをすべて前もって行うことができます。

嵐が来て、稲妻が落ちて、バイタルフォースが使えるようになったら、次は
leftArmM`にバイタルフォースを加えて実行します。

```fsharp
let vf = {units = 10}.

let liveLeftArm, remainingAfterLeftArm = runM leftArmM vf
```

...そして、次のような結果が得られます。

```text
val liveLeftArm : LiveLeftArm = ライブレフトアーム
    LiveLeftArm ("Victor",{units = 1;})
val remainingAfterLeftArm :
    VitalForce = {units = 9;} です。
```

思い通りの左腕ができあがりました。

## ♪ ♪ The Right Arm

次は右腕です。

ここでも問題がありました。 フランケンファンクター博士のノートには、腕全体がなかったと記録されています。
しかし、下腕と上腕はあった。

```fsharp
type DeadRightLowerArm = ラベルのDeadRightLowerArm
type DeadRightUpperArm = DeadRightUpperArm of Label（デッドライトアッパーアームオブラベル
```

...これは、対応する生きているものに変えることができる。

```fsharp
type LiveRightLowerArm = ラベルのLiveRightLowerArm * VitalForce
type LiveRightUpperArm = LiveRightUpperArm of Label * VitalForce
```

フランケンファンクター博士は、2つの腕の部分を1つの腕にする手術をすることにしました。

```fsharp
// 腕全体の定義
type LiveRightArm = {
    lowerArm : LiveRightLowerArm
    upperArm : LiveRightUpperArm
    }

// 2つの腕のパーツを結合する手術
let armSurgery lowerArm upperArm = {lowerArm=lowerArm; upperArm
    {lowerArm=lowerArm; upperArm=upperArm}。
```

壊れた腕と同様に、この手術は*生きている*パーツでしかできません。死んだパーツを使って手術をするのは、嫌で嫌で仕方がありません。

しかし、腕が折れたときと同様に、生きているパーツには直接アクセスできず、`M`ラッパーのコンテキストの中でしかアクセスできませんでした。

言い換えれば、通常のライブパーツで動作する `armSurgery` 関数を `M` で動作する `armSurgeryM` 関数に変換する必要があります。

![armSurgeryM](./monadster_armsurgeryM.png)

先ほどと同じ方法を使うことができます。

* vitalForce パラメータを受け取る内部関数を作成します。
* 入力されたパラメータを vitalForce で実行し、データを抽出します。
* 内側の関数から、手術後の新しいデータを返します。
* 内側の関数を "M "で囲み、それを返す

以下はそのコードです。

```fsharp
/// M<LiveRightLowerArm>とM<LiveRightUpperArm>をM<LiveRightArm>に変換する。
let makeArmSurgeryM_v1 lowerArmM upperArmM =.

    // vitalForceのパラメータを受け取る新しい内部関数を作る
    let becomeAlive vitalForce =
        // 入力されたlowerArmMをvitalForceで動かす
        // 下アームを取得する
        let liveLowerArm,remainingVitalForce = runM lowerArmM vitalForce

        // 入ってくる上腕部を残りのバイタルフォースで走らせる
        // 上腕を取るために
        let liveUpperArm,remainingVitalForce2 = runM upperArmM remainingVitalForce

        // liveRightArmを作るための手術を行う
        let liveRightArm = armSurgery liveLowerArm liveUpperArm

        // 腕全体と2番目に残ったバイタルフォースを返す
        ライブRightArm, 残りのバイタルフォース2

    // 内部の関数をラップして返す
    M 生き返る
```

壊れた腕の例との大きな違いは、もちろんパラメータが*2つ*あることです。
2番目のパラメータ（`liveUpperArm`を取得する）を実行するときには、元のものではなく、最初のステップの後の*残っているバイタルフォース*を渡すようにしなければなりません。

そして、内部の関数から戻るときには、他のものではなく、必ず `remainingVitalForce2` (第2ステップの後の残り) を返さなければなりません。

このコードをコンパイルすると、次のようになります。

```fsharp
M<RightLowerArm> -> M<RightUpperArm> -> M<LiveRightArm>となります。
```

となり、まさに求めていたシグネチャーとなります。

### map2Mの紹介

しかし、以前のように、これをもっと一般的にしてはどうでしょうか？ armSurgery`をハードコーディングする必要はなく、パラメータとして渡せばいいのです。

ここでは、より汎用性の高い関数として、`map2M`を呼び出します。

これがその実装です。

```fsharp
let map2M f m1 m2 =
    let becomeAlive vitalForce =
        let v1,reverentVitalForce = runM m1 vitalForce
        let v2,remainingVitalForce2 = runM m2 remainingVitalForce
        let v3 = f v1 v2
        v3, 残りのバイタルフォース2
    M becomeAlive
```

そして、これには署名があります。

```fsharp
f:('a -> 'b -> 'c) -> M<'a> -> M<'b> -> M<'c>)
```

mapM`と同様に、この関数は「関数コンバータ」と解釈することができ、「通常の」2パラメータ関数を`M`の世界の関数に変換します。

![map2M](./monadster_map2m.png)


### 右腕のテスト

今回も、これまでに得たものをテストしてみましょう。

いつものように、死んだパーツを生きたパーツに変換するための関数が必要です。

```fsharp
let makeLiveRightLowerArm (DeadRightLowerArm label) = （右腕を生きた状態にする
    let becomeAlive vitalForce =
        let oneUnit, remainingVitalForce = getVitalForce vitalForce
        let liveRightLowerArm = LiveRightLowerArm (label,oneUnit)
        ライブ右腕、残りのバイタルフォース
    M 生きている状態になる

let makeLiveRightUpperArm (DeadRightUpperArm label) =.
    let becomeAlive vitalForce =
        let oneUnit, remainingVitalForce = getVitalForce vitalForce
        let liveRightUpperArm = LiveRightUpperArm (label,oneUnit)
        LiveRightUpperArm, remainingVitalForce
    M 生きる力
```

*ところで、これらの関数には多くの重複があることに気付きましたか？私もです。これは後で修正します。

次に、パーツを作ります。

```fsharp
let deadRightLowerArm = DeadRightLowerArm "Tom"
let lowerRightArmM = makeLiveRightLowerArm deadRightLowerArm

let deadRightUpperArm = DeadRightUpperArm "Jerry"
let upperRightArmM = makeLiveRightUpperArm deadRightUpperArm
```

そして、腕全体を作る関数を作ります。

```fsharp
let armSurgeryM = map2M armSurgery
let rightArmM = armSurgeryM lowerRightArmM upperRightArmM
```

いつものように、雷が落ちる前にこれらのことを前もって行い、いざというときに必要なことをすべて行うためのレシピ（あるいは計算）を作っておくことができます。

バイタルフォースが使えるようになったら、バイタルフォースを使って`rightArmM`を実行することができます...。

```fsharp
let vf = {units = 10}.

let liveRightArm, remainingFromRightArm = runM rightArmM vf
```

...そして、次のような結果が得られました。

```text
val liveRightArm : LiveRightArm = {lowerArm = LiveRightArm}.
    {lowerArm = LiveRightLowerArm ("Tom",{units = 1;});
     upperArm = LiveRightUpperArm ("Jerry",{units = 1;});}。

val remainingFromRightArm : VitalForce =.
    {units = 8;}
```

必要に応じて2つのサブコンポーネントで構成された生きた右腕。

また、残りのバイタルフォースが*8*になっていることにも注目してください。正しく2ユニットのバイタルフォースを使い切ったことになります。

## まとめ

この記事では、「生きている状態になる」関数をラップした `M` 型を作成する方法を見ました。この関数は、雷が落ちたときにのみ起動することができます。

また、`mapM`(折れた腕の場合)や`map2M`(2つに分かれた腕の場合)を使って、様々なM値を処理したり組み合わせたりする方法も見てきました。

*この記事で使用したコードサンプルは[GitHubで公開されています](https://gist.github.com/swlaschin/54489d9586402e5b1e8a)*.

## 次回予告

このワクワクするような物語には、まだまだ衝撃が待っています。次回は、頭部と胴体がどのようにして作られたのかを明らかにしますので、お楽しみに。

