---
layout: post
title: "Binding with let, use, and do"
description: "How to use them"
date: 2012-05-17
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 4
---


すでに見てきたように、F#には「変数」はありません。代わりに値があります。

また、`let`、`use`、`do`などのキーワードが*束縛*として機能し、識別子と値や関数式を関連付けることも見てきました。

この記事では、これらの束縛について詳しく説明します。

## "let"束縛 ##

`let`バインディングはわかりやすく、通常の形式は次のようになります:

```fsharp
let aName = someExpression
```

しかし 、`let`には少し異なる2つの使い方があります。1つはモジュールのトップレベル*で名前付きの式を定義することで、もう1つは何らかの式の中で使われるローカル名を定義することです。これは、C#の"トップレベル"メソッド名と"ローカル"変数名の違いに似ています。
{{<footnote "*">}}。
また、後の連載でOO機能について説明しますが、クラスはトップレベルのlet束縛も持つことができます。
{{</footnote>}}。

以下は、両方のタイプの例です:

```fsharp
module MyModule =

    let topLevelName =
        let nestedName1 = someExpression
        let nestedName2 = someOtherExpression
        finalExpression
```


トップレベルにある名前はモジュールの一部として*定義*されており、`MyModule.topLevelName`といった完全修飾名でアクセスできます。ある意味、これはクラスメソッドに相当します。

しかし、入れ子になった名前には誰もアクセスできません――トップレベルの名前束縛のコンテキスト内でのみ有効です。

### "let"束縛内のパターン

束縛でパターンを直接使用する例はすでに見てきました。

```fsharp
let a,b = 1,2

type Person = {First:string; Last:string}
let alice = {First="Alice"; Last="Doe"}
let {First=first} = alice
```

関数定義内のパラメータ束縛も同様にパターンを含む場合があります:

```fsharp
// パラメータでパターンマッチ
let add (x, y) =x+y

// テスト
let aTuple = (1,2)
add aTuple
```

さまざまなパターン束縛の詳細は、束縛される型によって異なりますが、これについては今後のパターンマッチングに関する記事で説明する予定です。

### 式としての "let"束縛の入れ子

これまで、式は小さな式の集まりであると説明してきました。では、入れ子になった`let`はどうでしょうか?

```fsharp
let nestedName = someExpression
```

"let"はどうすれば式になりますか?その式の戻り値はどうなりますか?

その答えは、「入れ子になった"let"は決して単独で使用することは出来ない」です -- より大きなコードブロックの一部でなければなりません、そのため、次のように解釈することができます:

```fsharp
let nestedName= [何らかの式] in [nestedNameを含む他の式]
```

つまり、2番目の式 (*本体式*と呼ばれます) に記号 「nestedName」 が表示されるたびに、1番目の式に置き換えられます。

たとえば、次の式を考えてみましょう:

```fsharp
// 標準構文
let f () =
  let x = 1
  let y = 2
  x + y          //結果
```

実際は:

```fsharp
// "in"キーワードを使用する構文
let f () =
  let x = 1 in   // “in”キーワードはF#で使用可能
    let y = 2 in
      x + y      // 結果
```

置換が実行されると、最後の行は次のようになります。

    (xの定義) + (yの定義)
    //または
    (1) + (2) 

ある意味、入れ子になった名前はコンパイル時に消える "マクロ"や"プレースホルダ"にすぎません。したがって、入れ子になった`let`たちは式全体に影響を与えないことがわかるはずです。たとえば、入れ子になった`let`を含む式の型は、最終的な本体式と同じ型です。

入れ子になった`let`束縛がどのように機能するかを理解すると、いくつかのエラーが理解できるようになります。たとえば、入れ子になった "let"に対して"in"となるものがない場合、その式は完全ではありません。次の例は、let行の後に何もないため、エラーになります:

```fsharp
let f () =
  let x = 1
// error FS0588: この 'let' に続くブロックが完了していません。
//               すべてのコード ブロックは式であり、結果を持つ必要があります。
```

また、本体式は複数指定できないため、式の結果を複数指定することはできません。最終的な本体式の前に評価されるものはすべて"`do`"式である必要があり (下記参照) 、`unit`を返します:


```fsharp
let f () =
  2 + 2      // warning FS0020: この式の結果の型は 'int' で、暗黙的に無視されます。
             // 'ignore' を使用してこの値を明示的に破棄してください
  let x = 1
  x + 1      // これが最終的な結果です
```

このような場合、結果を"ignore"にパイプする必要があります。

```fsharp
let f () =
  2 + 2 |> ignore
  let x = 1
  x + 1      // これが最終的な結果です
```

{{< linktarget "use" >}}
## "use"束縛 ##

`use`キーワードは`let`と同じ目的で使用されます -- 式の結果を名前付きの値に束縛します。

大きな違いは、値がスコープ外になると*自動的に破棄*されることです。

このことは、 `use`が入れ子状態でのみ適用されることを意味します。トップレベルに`use`を設定することはできず、設定しようとするとコンパイラが警告します。

```fsharp
module A =
    use f () =  // Error
      let x = 1
      x + 1
```

適切な`use`束縛がどのように動作するかを見るため、まずは`IDisposable`をオンザフライで作成するヘルパー関数を作成しましょう。

```fsharp
// IDisposableを実装した新しいオブジェクトを作成する
let makeResource name =
   {new System.IDisposable
     with member this.Dispose() = printfn "%s disposed" name }
```

では入れ子になった`use`束縛を使ってテストしてみましょう:

```fsharp
let exampleUseBinding name =
    use myResource = makeResource name
    printfn "done"

//テスト
exampleUseBinding "hello"
```

"done"が出力され、その直後に`myResource`がスコープ外になり、その`Dispose`が呼び出されるため、"hello disposed"も出力されます。

一方、通常の `let`束縛を使用してテストした場合、同じ効果は得られません。

```fsharp
let exampleLetBinding name =
    let myResource = makeResource name
    printfn "done"

//テスト
exampleLetBinding "hello"
```

この場合、"done"は出力されますが、`Dispose`は呼び出されません。

### "use"はIDisposableでしか使えない

"use"束縛は、`IDisposable`を実装した型でのみ動作し、そうでない場合はコンパイラが文句を言うことに注意しましょう。

```fsharp
let exampleUseBinding2 name =
    use s = "hello"  // Error: 型の制約が一致しません。次の型'string'は
                     // 次の型と互換性がありません'System.IDisposable'
    printfn "done"
```


### "使われた"値を返してはいけない

重要なのは、*宣言した式のスコープから*外れると即座に値が破棄されることです。
別の関数で使用するために値を返そうとしても、その返り値は無効になります。

次のように*しない*でください:

```fsharp
let returnInvalidResource name=
    use myResource = makeResource name
    myResource // これはしないでください!

//テスト
let resource = returnInvalidResource  "hello"
```

関数の"外側"にあるdisposableな関数を扱う必要がある場合は、おそらくコールバックを使用するのが最善の方法です。

この場合、関数は次のように動作します。

* disposableが作成されます。
* disposableでコールバックが評価されます。
* disposableで `Dispose`を呼ぶ。

次に例を示します。

```fsharp
let usingResource name callback =
    use myResource = makeResource name
    callback myResource
    printfn "done"

let callback aResource = printfn "Resource is %A" aResource
do usingResource "hello" callback
```

このアプローチでは、disposableが作成された関数と同じ関数でdisposeも処理され、リークの可能性はありません。

もう1つの方法は、作成時に`use`束縛を使用*せず*、代わりに`let`束縛を使用し、呼び出し側に廃棄の責任を負わせることです。

次に例を示します:

```fsharp
let returnValidResource name=
    //"use"ではなく"let"で束縛
    let myResource=makeResource name
    myResource//まだ有効です

let testValidResource=
    //"let"ではなく"use"で束縛
    use resource = returnValidResource  "hello"
    printfn "done"
```

個人的にはこの方法は好きではありません。対称的ではなく、createとdisposeが分離されているため、リソースリークにつながる可能性があるからです。

### "using "関数

上で紹介したdisposableを共有するための好ましいアプローチは、コールバック関数を使用していました。

同じように動作する組み込みの `using` 関数があります。2つのパラメータを受け取ります。

* 1つ目は、リソースを作成する式です。
* 2つ目は、リソースを使用する関数で、リソースをパラメータとして受け取ります。

先ほどの例を、`using`関数を使って書き直してみましょう。

```fsharp
let callback aResource = printfn "Resource is %A" aResource
using (makeResource "hello") callback
```

実際には、`using`関数はそれほど頻繁には使われません。なぜなら、先ほど見たように、自分でカスタムバージョンを作るのはとても簡単だからです。

### "use"の悪用

F#では、`use`キーワードを適切に設定することで、あらゆる種類の"停止"や"復帰"を自動的に実行させるという手法があります。

その方法は次のとおりです:

* ある型の [拡張メソッド](/posts/type-extensions) を作成します。
* このメソッド内で、必要な動作を開始し、その後、その動作を停止させる`IDisposable`を返します。

たとえば、タイマーを開始し、それを停止する`IDisposable`を返す拡張メソッドは次のとおりです:

```fsharp
module TimerExtensions =

    type System.Timers.Timer with
        static member StartWithDisposable interval handler =
            // タイマーの作成
            let timer = new System.Timers.Timer(interval)

            // ハンドラを追加してスタート
            do timer.Elapsed.Add handler
            timer.Start()

            // "Stop"を呼び出すIDisposableを返す
            { new System.IDisposable with
                member disp.Dispose() =
                    do timer.Stop()
                    do printfn "Timer stopped"
                }
```

呼び出し側のコードでは、タイマーを作成して`use` でバインドしています。タイマーの値がスコープ外になると、自動的に停止します。

```fsharp
open TimerExtensions
let testTimerWithDisposable =
    let handler = (fun _ -> printfn "elapsed")
    use timer = System.Timers.Timer.StartWithDisposable 100.0 handler
    System.Threading.Thread.Sleep 500
```

この手法は、次のような一般的な操作の組合せにも使用できます。

* リソースを開く/接続してから閉じる/切断する(これは`IDisposable`を使用することが推奨されていますが、ターゲット型が実装していない可能性もあります)
* イベントハンドラの登録と登録解除 (`WeakReference`を使用する代わりに)
* UIの場合、コードの先頭でスプラッシュ画面を表示させます。その後、コードの末尾で自動的に閉じられます。

この方法は、状況を隠蔽するので一般的にはお勧めしませんが、場合によっては非常に便利です。

## "do"束縛 ##

関数や値の定義とは別にコードを実行したい場合があります。これは、モジュールの初期化やクラスの初期化などに役立ちます。

つまり、"`let x=do something`"ではなく単に"`do something`"としたいのです。これは命令型言語の文に似ています。

これを行うには、コードの先頭に”`do`”を付けます。

```fsharp
do printf "logging"
```

多くの場合、`do`キーワードは省略できます。

```fsharp
printf 「ロギング」
```

ただし、いずれの場合も、式はunit値を返す必要があります。さもなければエラーが発生します。

```fsharp
do 1 + 1    // warning: この式は関数です
```

通常どおり、結果を"`ignore`"へパイプしてunitでない結果を破棄することができます。

```fsharp
do ( 1+1 |> ignore )
```

"`do`"キーワードも同様にループ内で使用できます。

省略可能な場合もありますが、"`do`"を常に明示的に記述することをお勧めします。"`do`"は結果ではなく、副作用のみを意味するドキュメントとして機能します。


### モジュール初期化のための "do"

`let`と同様に、`do`は入れ子のコンテキストでも、モジュールやクラスのトップレベルでも使用することができます。

モジュールレベルで使用する場合、`do`式はモジュールが最初にロードされたときに一度だけ評価されます。

```fsharp
module A =

    module B =
        do printfn "Module B initialized"

    module C =
        do printfn "Module C initialized"

    do printfn "Module A initialized"
```

これは、C#の静的クラスのコンストラクタに似ていますが、複数のモジュールがある場合、初期化の順序が固定され、宣言の順序に従って初期化される点が異なります。

## let! and use! and do!

`let!`, `use!`および`do!`(つまり、感嘆符付きの) が、中括弧`{..}`ブロックに含まれている場合、それらは "コンピュテーション式"の一部として使用されています。`let!`,`use!`,`do!`は、コンピュテーションの式そのものに依存します。一般的なコンピュテーション式を理解するには、今後のシリーズを待たなければなりません。

最も一般的なコンピュテーション式は*非同期ワークフロー*で、`async{...}`ブロックを使用します。
このコンテキストでは、非同期操作が終了するのを待機してから結果値にバインドするために使用されています。

["Why use F#?"シリーズの記事](/posts/concurrency-async-and-parallel) で以前見たいくつかの例を以下に示します:

```fsharp
//このシンプルなワークフローは、2秒間スリープするだけです。
open System
let sleepWorkflow  = async{
    printfn "Starting sleep workflow at %O" DateTime.Now.TimeOfDay

    // do! means to wait as well
    do! Async.Sleep 2000
    printfn "Finished sleep workflow at %O" DateTime.Now.TimeOfDay
    }

//テスト
Async.RunSynchronously sleepWorkflow


// 他の非同期ワークフローがネストされたワークフロー。
/// 中括弧の中で、入れ子になったワークフローはlet！やuse！の構文を使ってブロックオンできます。
let nestedWorkflow  = async{

    printfn "Starting parent"

    // let! は、待機してから childWorkflow の値にバインドすることを意味します。
    let! childWorkflow = Async.StartChild sleepWorkflow

    // 子供にチャンスを与え、その後作業を続ける
    do! Async.Sleep 100
    printfn "Doing something useful while waiting "

    // 子をブロックする
    let! result = childWorkflow

    // 完了
    printfn "Finished parent"
    }

// ワークフロー全体を実行
Async.RunSynchronously nestedWorkflow
```

## letとdoの束縛への属性

モジュールのトップレベルにある場合、`let`および`do`束縛は属性を持つことができます。F#では`[]`の構文を使用します。

C#とF#の同価なコードの例を次に示します:

```csharp
クラスAttributeTest
{
    [Obsolete]
    public static int MyObsoleteFunction(int x, int y)
    {
        return x + y;
    }

    [CLSCompliant(false)]
    public static void NonCompliant()
    {
    }
}
```

```fsharp
module AttributeTest =
    []
    let myObsoleteFunction x y = x + y

    []
    let nonCompliant () = ()
```

3つの属性の例を簡単に見てみましょう:

* "main"関数を示すために使用されるEntryPoint属性。
* さまざまなAssemblyInfo属性。
* アンマネージコードと対話するためのDllImport属性。

### EntryPoint 属性

C#では、`static void Main`メソッドのように、スタンドアロンアプリのエントリーポイントを示すために、特別な`EntryPoint`属性が使用されます。

おなじみのC#バージョンは以下の通りです。

```csharp
class Program
{
    static int Main(string[] args)
    {
        foreach (var arg in args)
        {
            Console.WriteLine(arg);
        }

        //Environment.Exit(code)と同じです。
        return 0;
    }
}
```

F#の場合は以下のようになります。

```fsharp
module Program

[<EntryPoint>]
let main args =
    args |> Array.iter printfn "%A"

    0 // リターンが必要です!
```

C#と同じように、argsは文字列の配列になっています。しかし、静的な`Main`メソッドが`void`であることができるC#とは異なり、F#の関数は*必ず*intを返さなければなりません。

また、大きな問題として、この属性を持つ関数は、プロジェクトの最後のファイルの一番最後の関数でなければなりません。さもなければ、このようなエラーが発生します。

    エラー FS0191: `EntryPointAttribute'属性を持つ関数は、コンパイルシーケンスの最後のファイルの最後の宣言でなければなりません。

なぜF#コンパイラはそんなにうるさいんでしょう?C#の場合、クラスは任意の場所へ移動できます。

有用な類推は次のとおりです: ある意味では、アプリケーション全体は`main`に束縛されている単一の巨大な式である。
この`main`には、他の部分式が含まれている部分式が含まれています。

```fsharp
[<EntryPoint>]
let main args =
    // アプリケーション全体を部分式の集合として表現する
```

さて，F#プロジェクトでは，前方参照が許されていません．つまり、他の式を参照する式は、その式の後に宣言しなければなりません。
そのため、論理的には、最も高い最上位の関数である `main` は、最後に宣言しなければなりません。

### AssemblyInfo 属性

C#プロジェクトでは、すべてのアセンブリレベルの属性を含む`AssemblyInfo.cs`ファイルがあります。

F#では、これらの属性でアノテーションされた`do`式を含むダミーモジュールでこれを行うのが同等の方法です。

```fsharp
オープン System.Reflection

モジュール AssemblyInfo =
    <assembly: AssemblyTitle("MyAssembly")>] [<assembly: AssemblyVersion("MyAssembly")
    <assembly: AssemblyVersion("1.2.0.0")>] [<assembly: AssemblyFileVersion("1.2.0.0")
    [<assembly: AssemblyFileVersion("1.2.3.4152")>] 。
    do () // 何もしない -- 単なる属性のプレースホルダー
```

### DllImport属性

もう一つの有用な属性として、`DllImport`属性があります。以下にC#の例を示します。

```csharp
using System.Runtime.InteropServices;

[TestFixture]
public class TestDllImport
{
    [DllImport("shlwapi", CharSet = CharSet.Auto, EntryPoint = "PathCanonicalize", SetLastError = true)]
    private static extern bool PathCanonicalize(StringBuilder lpszDst, string lpszSrc);

    [Test]
    public void TestPathCanonicalize()
    {
        var input = @"A:\name_1\.\name_2\..\name_3";
        var expected = @"A:\name_1\name_3";

        var builder = new StringBuilder(260);
        PathCanonicalize(builder, input);
        var actual = builder.ToString();

        Assert.AreEqual(expected,actual);
    }
}
```

F#でもC#と同じように動作します。注意すべき点は、`extern宣言...`はC言語のようにパラメータの前に型を置くことです。

```fsharp
open System.Runtime.InteropServices
open System.Text

[<DllImport("shlwapi", CharSet = CharSet.Ansi, EntryPoint = "PathCanonicalize", SetLastError = true)>]
extern bool PathCanonicalize(StringBuilder lpszDst, string lpszSrc)

let TestPathCanonicalize() =
    let input = @"A:\name_1\.\name_2\..\name_3"
    let expected = @"A:\name_1\name_3"

    let builder = new StringBuilder(260)
    let success = PathCanonicalize(builder, input)
    let actual = builder.ToString()

    printfn "actual=%s success=%b" actual (expected = actual)

// テスト
TestPathCanonicalize()
```

アンマネージコードとの相互連携は、独自のシリーズを必要とする大きなテーマです。