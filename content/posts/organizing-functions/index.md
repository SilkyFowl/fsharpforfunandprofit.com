---
layout: post
title: "Organizing functions"
description: "Nested functions and modules"
date: 2012-05-10
nav: thinking-functionally
seriesId: "Thinking functionally"
seriesOrder: 10
categories: [Functions, Modules]
---

関数を定義する方法がわかったところで、それをどのように整理すればよいのでしょうか。

F#には3つの選択肢があります:

* 関数は他の関数内へ入れ子にすることができます。
* アプリケーション・レベルでは、最上位の関数は「モジュール」 としてグループ化されます。
* あるいは、オブジェクト指向の手法を用いて、型に関数をメソッドとして付加する方法もあります。

この記事では最初の2つの選択肢を、次の記事では3番目の選択肢を取り上げます。

## ネストされた関数 ##

F#では、他の関数の内部で関数を定義できます。これは、メイン関数に必要だが外部に公開すべきでない「ヘルパー」関数をカプセル化する優れた方法です。

以下の例では、`add`は`addThreeNumbers`の中に入れ子になっています:

```fsharp
let addThreeNumbers x y z  =

    //入れ子のヘルパー関数を作る
    let add n =
       fun x -> x + n

    // use the helper function
    x |> add y |> add z

// テスト
addThreeNumbers 2 3 4
```

入れ子になった関数は、親関数のパラメータがスコープ内にあるため、親関数のパラメータに直接アクセスすることができます。
つまり、以下の例では、`printError`という入れ子になった関数は、それ自身のパラメータを持つ必要はなく、`n`と`max`の値に直接アクセスすることができます。

```fsharp
let validateSize max n  =

    //パラメータを持たない入れ子のヘルパー関数を作る
    let printError() =
        printfn "Oops: '%i' is bigger than max: '%i'" n max

    // ヘルパー関数を使う
    if n > max then printError()

// test
validateSize 10 9
validateSize 10 11
```

よくあるパターンは、メイン関数がネストされた再帰的なヘルパー関数を定義し、それを適切な初期値で呼び出すというものです。
次のコードはその例です:

```fsharp
let sumNumbersUpTo max =

    // recursive helper function with accumulator
    let rec recursiveSum n sumSoFar =
        match n with
        | 0 -> sumSoFar
        | _ -> recursiveSum (n-1) (n+sumSoFar)

    // call helper function with initial values
    recursiveSum max 0

// test
sumNumbersUpTo 10
```


関数を入れ子にする場合は、深い入れ子を避けるようにしてください、特に入れ子になった関数にパラメータが渡さず、親スコープの変数に直接アクセスする場合は特に注意してください。
不適切な入れ子構造の関数は、最悪に深い入れ子の命令型分岐と同じくらい複雑になります。

以下のようで*やってはいけません*:

```fsharp
// この関数は何をするの？
let f x =
    let f2 y =
        let f3 z =
            x * z
        let f4 z =
            let f5 z =
                y * z
            let f6 () =
                y * x
            f6()
        f4 y
    x * f2 x
```


## モジュール ##

モジュールは、グループ化された関数の集合にすぎず、通常は同じ種類のデータを処理するためのものです。

モジュール定義は関数定義によく似ています。`module`キーワードで始まり、`=`記号が続き、その後にモジュールの内容がリストされます。
関数定義内の式はインデントされなければならないように、モジュールの内容は*必ず*インデントされなければなりません。

以下は、2つの関数を含むモジュールです:

```fsharp
module MathStuff =

    let add x y  = x + y
    let subtract x y  = x - y
```

Visual Studioで`add`関数にカーソルを合わせると、`add`関数のフルネームが実際には`MathStuff.add`であることがわかります。

実際、まさにその通りなのです。F#コンパイラは舞台裏で、静的なメソッドを持つ静的なクラスを生成します。つまり、以下のC#コードに相当します:

```csharp
static class MathStuff
{
    static public int add(int x, int y)
    {
        return x + y;
    }

    static public int subtract(int x, int y)
    {
        return x - y;
    }
}
```

モジュールが単なる静的クラスであり、関数が静的メソッドであることがわかれば、F#のモジュールがどのように動作するかを理解するための最初のステップは既に踏み出してせています。
静的クラスに適用されるルールのほとんどは、モジュールにも適用されます。

また、C#と同様に、すべての独立した関数はクラスの一部である必要があり、F#では、それぞれの独立した関数はモジュールの一部*でなければなりません*。

### モジュール境界を越えた関数へのアクセス

他のモジュールの関数にアクセスしたい場合は、修飾名で参照します。

```fsharp
module MathStuff =

    let add x y  = x + y
    let subtract x y  = x - y

module OtherStuff =

    // MathStuffモジュールの関数を使う
    let add1 x = MathStuff.add x 1
```

また、`open`ディレクティブを使って、他のモジュールのすべての関数をインポートすることもできます。
その後は，修飾名を指定するのではなく，短い名前を使うことができます．

```fsharp
module OtherStuff =
    open MathStuff // すべての関数にアクセスできるようにする

    let add1 x = add x 1
```

修飾名を使用するための規則は、ご想像のとおりです。つまり、完全修飾名を使用していつでも関数にアクセスできます。
スコープ内の他のモジュールに基づいて、相対名または非修飾名も使用できます。

### 入れ子のモジュール

静的クラスと同様に、モジュールにも次に示すような子モジュールを入れ子にすることができます。

```fsharp
module MathStuff =

    let add x y  = x + y
    let subtract x y  = x - y

    // 入れ子のモジュール
    module FloatLib =

        let add x y :float = x + y
        let subtract x y :float  = x - y
```

また、他のモジュールは、必要に応じて完全名または相対名を使用して、ネストされたモジュール内の関数を参照できます。

```fsharp
module OtherStuff =
    open MathStuff

    let add1 x = add x 1

    // 完全修飾
    let add1Float x = MathStuff.FloatLib.add x 1.0

    //相対パスの場合
    let sub1Float x = FloatLib.subtract x 1.0
```

### トップレベルのモジュール

入れ子になった子モジュールがあるならば、その連鎖をさかのぼると、常に何らかの*トップレベル*の親モジュールが存在しなければならないことを意味します。確かにそうですね。

最上位レベルのモジュールは、これまで見てきたモジュールとは少し異なります。

* `module MyModuleName` の行は、ファイル内の最初の宣言でなければなりません。
* `=`の記号はありません
* モジュールの内容はインデント*されません*。

一般的に、すべての `.FS` ソースファイルには、トップレベルのモジュール宣言がなければなりません。いくつかの例外はありますが、いずれにしても、これは良い習慣です。
モジュール名は、ファイル名と同じである必要はありませんが、2つのファイルが同じモジュール名を共有することはできません。

`.FSX`のスクリプトファイルでは、モジュール宣言は必要ありません。この場合、モジュール名は自動的にスクリプトのファイル名に設定されます。

以下は、`MathStuff`をトップレベルのモジュールとして宣言した例です．

```fsharp
// トップレベルモジュール
module MathStuff

let add x y  = x + y
let subtract x y  = x - y

// 入れ子のモジュール
module FloatLib =

    let add x y :float = x + y
    let subtract x y :float  = x - y
```

トップレベルのコード(`module MathStuff` の内容)にはインデントがありませんが、`FloatLib` のような入れ子のモジュールの内容にはインデントが必要であることに注意してください。

### その他のモジュールの内容

モジュールには、関数だけでなく、型宣言、単純な値、初期化コード(静的コンストラクタなど)を含むことができます。

```fsharp
module MathStuff =

    // 関数
    let add x y  = x + y
    let subtract x y  = x - y

    // 型の定義
    type Complex = {r:float; i:float}
    type IntegerFunction = int -> int -> int
    type DegreesOrRadians = Deg | Rad

    // "定数"
    let PI = 3.141

    // "変数"
    let mutable TrigType = Deg

    // 初期化／静的コンストラクタ
    do printfn "module initialized"

```

{{<alertinfo>}}
ところで、これらの例をインタラクティブウィンドウで遊んでいる場合は、コードが新鮮で、以前の評価で汚染されないように、時々右クリックして「セッションのリセット」をするといいでしょう。
{{</alertinfo>}}

### シャドーイング

もう一度、モジュールの例を示します。`MathStuff`には`add`関数があり、`FloatLib`*にも*`add`関数があることに注目してください。

```fsharp
module MathStuff =

    let add x y  = x + y
    let subtract x y  = x - y

    // ネストしたモジュール
    module FloatLib =

        let add x y :float = x + y
        let subtract x y :float  = x - y
```

では、これらの*両方*をスコープに取り込んでから`add`を使うとどうなるのでしょうか?

```fsharp
open  MathStuff
open  MathStuff.FloatLib

let result = add 1 2  // Compiler error: This expression was expected to
                      // have type float but here has type int
```

何が起こったかというと、`MathStuff.FloatLib`モジュールが、オリジナルの`MathStuff`モジュールを覆い隠したり、オーバーライドしたりして、`FloatLib`に「影」を落としてしまったのです。

その結果、最初のパラメータ `1` が float であることが期待されるため、[FS0001 compiler error](/troubleshooting-fsharp/#FS0001) が発生します。この問題を解決するには、`1`を`1.0`に変更する必要があります。

残念ながら、これは目に見えず、見落としがちです。サブクラス化のようにクールなトリックが使えることもありますが、多くの場合、同名の関数 (非常に一般的な `map`など) があると面倒です。

これを回避するには、`RequireQualifiedAccess`属性を使用してアクセスを停止させる方法があります。これは同じ例で、両モジュールとも装飾する場合です。

```fsharp
[<RequireQualifiedAccess>]
module MathStuff =

    let add x y  = x + y
    let subtract x y  = x - y

    // nested module
    [<RequireQualifiedAccess>]
    module FloatLib =

        let add x y :float = x + y
        let subtract x y :float  = x - y
```

これで、`open`が許されなくなりました。

```fsharp
オープン MathStuff // エラー
open MathStuff.FloatLib // エラー
```

しかし、関数の修飾名を使って（曖昧さを排除して）アクセスすることはできます。

```fsharp
let result = MathStuff.add 1 2
let result = MathStuff.FloatLib.add 1.0 2.0
```


### アクセスコントロール

F#は、`public`, `private`, `internal`といった、.NET標準のアクセスコントロールキーワードの使用をサポートしています。
詳細は[MSDNドキュメント](http://msdn.microsoft.com/en-us/library/dd233188)に記載されています。

* これらのアクセス指定子は、モジュール内の最上位にある ( 「let束縛された」 ) 関数、値、型、その他の宣言に置くことができます。モジュール自体に指定することもできます(たとえば、プライベートな入れ子モジュールが必要な場合など) 。
* デフォルトでは (いくつかの例外を除いて) すべてが公開されているので、保護したい場合は`private`か`internal`を使う必要があります。

これらのアクセス指定子は、F#でアクセス制御を行う1つの方法にすぎません。これとは全く別の方法として、Cのヘッダファイルに少し似た 「シグネチャ」 ファイルを使う方法があります。モジュールの内容を抽象的に記述します。シグニチャは、本格的なカプセル化を行う場合は非常に便利です、カプセル化と機能ベースの保護に関する一連の投稿が予定されています。


##名前空間

F#の名前空間はC#の名前空間に似ています。名前の衝突を避けるために、モジュールや型を整理するために使用できます。

名前空間は、次に示すように、`namespace`キーワードで宣言されます。

```fsharp
namespace Utilities

module MathStuff =

    // functions
    let add x y  = x + y
    let subtract x y  = x - y
```

この名前空間のおかげで、`MathStuff` モジュールの完全修飾名は `Utilities.MathStuff` となり、`add` 関数の完全修飾名は `Utilities.MathStuff.add` となります。
また、関数 `add` の完全修飾名は `Utilities.MathStuff.add` となります。

名前空間では、インデントのルールが適用されるので、上で定義したモジュールは、ネストされたモジュールであるかのように、その内容をインデントしなければなりません。

また、モジュール名にドットを付けることで、暗黙的に名前空間を宣言することができます。つまり，上のコードは次のように書くこともできます。

```fsharp
module Utilities.MathStuff

// functions
let add x y  = x + y
let subtract x y  = x - y
```

`MathStuff`モジュールの完全修飾名は`Utilities.MathStuff`と同じですが、
この場合、モジュールは最上位レベルのモジュールであり、内容をインデントする必要はありません。

名前空間を使用する際に注意すべき点がいくつかあります。

* モジュールの名前空間はオプションです。C#とは異なり、F#プロジェクトにはデフォルトの名前空間がないので、名前空間のないトップレベルのモジュールはグローバルレベルになります。
再利用可能なライブラリを作成する場合は、必ず何らかの名前空間を追加して、他のライブラリ内のコードとの名前衝突を回避してください。
* 名前空間には型宣言を直接含めることができますが、関数宣言は含めることができません。前述したように、すべての関数と値の宣言はモジュールの一部である必要があります。
* 最後に、スクリプトでは名前空間がうまく機能しないことに注意してください。たとえば、下記の`namespace Utilities`などの名前空間の宣言をインタラクティブウィンドウに送信しようとすると、エラーが発生します。


### 名前空間の階層化

名前をピリオドで区切るだけで、名前空間の階層を作ることができます。

```fsharp
namespace Core.Utilities

module MathStuff =
    let add x y  = x + y
```

*2つ*の名前空間を同一ファイルに置きたいなら、それも可能です。なお、すべての名前空間は完全に修飾されていないと*いけません*――入れ子はありません。

```fsharp
名前空間 Core.Utilities

モジュール MathStuff =
    let add x y = x + y

名前空間Core.Extra

モジュール MoreMathStuff =
    xとyを足す = x + y
```

1つできないことがあります、名前空間とモジュール間で名前を衝突させることです。


```fsharp
namespace Core.Utilities

module MathStuff =
    let add x y  = x + y

namespace Core

// fully qualified name of module
// is Core.Utilities
// Collision with namespace above!
module Utilities =
    let add x y  = x + y
```


## モジュール内での型と関数の混合 ##

これまで見てきたように、モジュールは通常、あるデータ型に関連する一連の関数で構成されています。

オブジェクト指向のプログラムでは、データ構造とそれに作用する関数はクラス内で結合されます。
しかし、関数型のF#では、データ構造とそれに作用する関数はモジュール内で結合されます。

型と関数を組み合わせて使用するには、一般的に2つのパターンがあります。

* 型を関数とは別に宣言する
* 型を関数と同じモジュール内で宣言する

最初の方法では、型は*外部*のいずれかのモジュール (ただし同じ名前空間内) で宣言され、
その型で動作する関数は同じ名前のモジュールに入れられます。

```fsharp
// トップレベルモジュール
namespace Example

// モジュールの外で型を宣言する
type PersonType = {First:string; Last:string}

// この型で動作する関数のためのモジュールを宣言する
module Person =

    // コンストラクタ
    let create first last =
        {First=first; Last=last}

    // 型に作用するメソッド
    let fullName {First=first; Last=last} =
        first + " " + last

// test
let person = Person.create "john" "doe"
Person.fullName person |> printfn "Fullname=%s"
```

別の方法では、型はモジュール*内*で宣言され、`T`やモジュール名のような単純な名前が与えられます。
したがって、関数は`MyModule.Func1`および`MyModule.Func2`という名前を持ち、
型自体は`MyModule.T`です。次に例を示します。

```fsharp
module Customer =

    //顧客.Tは、このモジュールのプライマリタイプです。
    type T={アカウントID:int;名前:string}

    //コンストラクタ
    let作成ID名=
        {T.AccountId=ID;T.Name=名前}

    //型に対して動作するメソッド
    let isValid{T.AccountId=id;}=
        ID>0

//テスト
let customer=Customer.create 42 「ボブ」
Customer.isValid customer|>printfn 「有効ですか?=%b」
```

どちらの場合も、その型の新しいインスタンスを生成するコンストラクタ関数 (必要ならばファクトリメソッド) が必要です。
これは、クライアントのコードで型を明示的に指定する必要はめったにないことを意味します。ですから、モジュール内に存在するかどうかは気にする必要はありません。

では、どのアプローチを選ぶべきなのでしょうか?

* 前者のアプローチは.NETに似ており、エクスポートされたクラス名は期待通りであるため、F#以外のコードとライブラリを共有したい場合には非常に適しています。
* 後者のアプローチは、他の関数型言語で使用される言語では一般的なものです。モジュール内の型はネストされたクラスにコンパイルされるため、相互運用性には優れていません。

ご自身のために、どちらも試してみてください。また、チームプログラミングの状況においては、ひとつのスタイルを選んで一貫性を保つ必要があります。


### 型だけのモジュール

関連する関数のない型のセットを宣言する必要がある場合、わざわざモジュールを使用する必要はありません。型は名前空間に直接宣言でき、入れ子のクラスを回避することができます。

たとえば、次のようにします:

```fsharp
//最上位モジュール
module Example

// モジュール内で型を宣言する
type PersonType = {First:string; Last:string}

// モジュール内に関数はなく、型のみを定義します..
```

別の方法もあります。`module`キーワードを`namespace`に置き換えるだけです。

```fsharp
//名前空間を使用する
namespace Example

//任意のモジュールの外部で型を宣言します
type PersonType = {First:string; Last:string}
```

どちらの場合も、`PersonType`が完全修飾名になります。

これは型でのみ動作することに注意してください。関数は常にモジュール内に存在しなければなりません。
