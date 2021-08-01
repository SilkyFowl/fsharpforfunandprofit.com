---
layout: post
title: "Formatted text using printf"
description: "Tips and techniques for printing and logging"
date: 2012-07-12
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 10
---

今回は少し回り道をして、書式付きのテキストを作成する方法を見てみましょう。 出力や書式設定の関数は、技術的にはライブラリ関数です。
しかし、実際にはコア言語の一部であるかのように使用されます。

F#は2つの異なるスタイルのテキストフォーマットをサポートしています。

* .NETの標準的な技術である["複合書式指定"](http://msdn.microsoft.com/en-us/library/txafckwd.aspx)は、`String.Format`や`Console.WriteLine`などで見られます。
* C スタイルの技術である `printf` と `printfn`, `sprintf` などの [関連する関数群](http://msdn.microsoft.com/en-us/library/ee370560)。

## String.Format vs printf

複合書式指定は、すべての.NET言語で利用可能であり、C#ではお馴染みでしょう。

```fsharp
Console.WriteLine("文字列です。{0}. int: {1}. float: {2}. a bool: {3}","hello",42,3.14,true)
```

一方，`printf`は，C言語のフォーマット文字列を利用した手法です。

```fsharp
printfn "A string: %s. int: %i. a float: %f. A bool: %b" "hello" 42 3.14 true
```

ご覧になったように、F#では、`printf`というテクニックが非常に一般的で、`String.Format`や`Console.Write`などはほとんど使われません。

なぜ F# では `printf` が好まれ、慣用的と考えられているのでしょうか。理由は以下の通りです。

* 静的に型チェックされている。
* F#の関数としてよくできているので、部分適用などをサポートしています。
* F#のネイティブな型をサポートしています。

### printf は静的に型チェックされます。

`String.Format` とは異なり、`printf` はパラメータの型と数値の両方に対して *静的な型チェック* を行います。

例えば、`printf`を使った2つのスニペットは、コンパイルに失敗します。

```fsharp
// 間違ったパラメータタイプ
printfn "A string: %s" 42

// パラメータの数がおかしい
printfn "文字列: %s" "Hello" 42
```

複合書式指定を使用した同等のコードは，コンパイルは正常に行われますが，正しくない動作を静かに行うか，ランタイムエラーが発生します。

```fsharp
// 間違ったパラメータタイプ
Console.WriteLine("A string: {0}", 42) //動作します!

// パラメータの数がおかしい
Console.WriteLine("A string: {0}","Hello",42) //うまくいきました!
Console.WriteLine("A string: {0}. An int: {1}", "Hello") //FormatException
```

### printf は部分適用をサポート

.NETの書式設定関数では、すべてのパラメータを*同時に*渡す必要があります。

しかし、`printf`は標準的でお行儀の良いF#の関数なので、[部分適用](/posts/partial-application)をサポートしています。

以下に例を示します。

```fsharp
// 部分適用 - 明示的なパラメータ
let printStringAndInt s i =  printfn "A string: %s. An int: %i" s i
let printHelloAndInt i = printStringAndInt "Hello" i
do printHelloAndInt 42

// 部分適用 - ポイントフリースタイル
let printInt = printfn "A int: %i"
do printInt 42
```

もちろん、`printf`は、標準的な関数が使えるところならどこでも、関数のパラメータに使うことができます。

```fsharp
let doSomething printerFn x y =
    let result = x + y
    printerFn "result is" result

let callback = printfn "%s %i"
do doSomething callback 3 4
```

これには，リストなどの高次関数も含まれます．

```fsharp
[1..5] |> List.map (sprintf "i=%i")
```

### printfがF#のネイティブ型をサポート

非プリミティブ型の場合、.NETの書式設定関数は、`ToString()`の使用しかサポートしていませんが、`printf`は、`%A`指定子を使って、F#のネイティブ型をサポートしています。

```fsharp
// タプルの出力
let t = (1,2)
Console.WriteLine("A tuple: {0}", t)
printfn "A tuple: %A" t

// レコード出力
type Person = {First:string; Last:string}
let johnDoe = {First="John"; Last="Doe"}
Console.WriteLine("A record: {0}", johnDoe )
printfn "A record: %A" johnDoe

// 判別共用体の出力
type Temperature = F of int | C of int
let freezing = F 32
Console.WriteLine("A union: {0}", freezing )
printfn "A union: %A" freezing
```

ご覧のように、タプル型には素晴らしい `ToString()` がありますが、他のユーザー定義型にはありません。
したがって、タプル型を.NETのフォーマット関数で使用したい場合は、明示的に`ToString()`メソッドをオーバーライドする必要があります。

## printfの問題点

printf`を使用する際には、いくつかの "問題"に注意する必要があります。

まず，パラメータが多すぎるのではなく，*少なすぎる*場合，コンパイラはすぐには*文句を言いませんが，後で不可解なエラーを出すことがあります．

```fsharp
// パラメータが少なすぎる
printfn "A string: %s An int: %i" "Hello"
```

理由はもちろん、これはエラーではなく、`printf`が部分適用されているだけだからです。
なぜこのようなことが起こるのかよくわからない場合は、[部分適用の議論](/posts/partial-application)を参照してください。

もう一つの問題は、"フォーマット文字列"が実際には文字列ではないということです。

.NETの書式モデルでは、書式文字列は通常の文字列なので、それを渡したり、リソースファイルに保存したりすることができます。
つまり、以下のコードは問題なく動作するということです。

```fsharp
let netFormatString = "A string: {0}"
Console.WriteLine(netFormatString, "hello")
```

一方で、`printf`の第一引数である"フォーマット文字列"は、実際には文字列ではなく、`TextWriterFormat`と呼ばれるものです。
つまり、以下のコードは **動かない** ということです。

```fsharp
let fsharpFormatString = "A string: %s"
printfn fsharpFormatString "Hello"
```

コンパイラは、文字列定数`"A string: %s"`を適切なTextWriterFormatに変換するために、舞台裏でいくつかのマジックを行います。
TextWriterFormatは，`string->unit`や`string->int->unit`などのフォーマット文字列の型を"知る"重要なコンポーネントで，これによって
これにより、`printf`がタイプセーフになります。

コンパイラをエミュレートしたい場合は、`Microsoft.FSharp.Core.Printf`モジュールの`Printf.TextWriterFormat`型を使って、文字列から独自のTextWriterFormat値を作成することができます。

フォーマット文字列が "inline "の場合は、コンパイラがバインディング時に型を推測してくれます。

```fsharp
let format:Printf.TextWriterFormat<_> = "A string: %s"
printfn format "Hello"
```

しかし，フォーマット文字列が本当に動的なものである場合（たとえば，リソースに格納されていたり，その場で作成されたりする場合），コンパイラは型を推測することができません。
コンストラクタで明示的に型を指定する必要があります。

以下の例では、最初のフォーマット文字列は文字列のパラメータをひとつ持ち、ユニットを返すので、フォーマットの型として `string->unit` を指定しなければなりません。
また、2番目の例では、フォーマットタイプとして、`string->int->unit`を指定する必要があります。

```fsharp
let formatAString = "A string: %s"
let formatAStringAndInt = "A string: %s. A int: %i"

//TextWriterFormatに変換します。
let twFormat1 = Printf.TextWriterFormat<string->unit>(formatAString)
printfn twFormat1 "Hello"
let twFormat2 = Printf.TextWriterFormat<string->int->unit>(formatAStringAndInt)
printfn twFormat2 "Hello" 42
```

ここでは、`printf`と`TextWriterFormat`がどのように連携しているかについては詳しく説明しませんが、単に単純なフォーマット文字列が渡されるだけではないことをご理解ください。

最後に、`printf` とその仲間はスレッドセーフではありませんが、`Console.Write` とその仲間はスレッドセーフであることを覚えておいてください。

{{< book_page_pdf >}}。

## フォーマットの指定方法

フォーマットの"%"の指定は、C言語で使われているものとよく似ていますが、F#用にいくつか特別なカスタマイズがされています。

Cと同様に、`%`の直後の文字は以下のように特定の意味を持っています。

    %[flags][width][.precision]specifier

これらの各属性については、以下で詳しく説明します。

### Formatting for dummies

最も一般的に使用される形式指定子は次のとおり:

* `%s`は文字列用
* `%b`はブール値
* `%i`は整数
* `%f`は浮動小数点
* `%A`は、タプル、レコードおよび判別共用体をきれいに出力します。
* `%O`は他のオブジェクト用で、`ToString () `を使用します。

これらの6つで、基本的なニーズのほとんどを満たすことができるでしょう。

### %のエスケーピング

`%`は、それだけではエラーになります。エスケープするには、2重にすればいいのです。

```fsharp
printfn "unescaped: %" // エラー
printfn "escape: %%"
```

### 幅と配置を制御する

固定幅のカラムやテーブルをフォーマットする際には、アライメントや幅をコントロールする必要があります。

そのためには、"width"と"flags"オプションを使用します。

* `%5s`, `%5i`. 数値で値の幅を設定します。
* `%*s`, `%*i`. 星印は、値の幅を動的に設定します (フォーマットするための param の直前にある追加のパラメータから)
* `%-s`, `%-i`. ハイフンは、値を正当化します。

これらの使用例を紹介します。

```fsharp
let rows = [ (1, "a"); (-22, "bb"); (333, "ccc"); (-4444, "dddd") ]

// アライメントなし
for (i,s) in rows do
    printfn "|%i|%s|" i s

// アライメントあり
for (i,s) in rows do
    printfn "|%5i|%5s|" i s

// 2列目の左揃えで
for (i,s) in rows do
    printfn "|%5i|%-5s|" i s

// 列1の動的列幅=20の場合
for (i,s) in rows do
    printfn "|%*i|%-5s|" 20 i s

// 第1列と第2列の動的な列幅に合わせて
for (i,s) in rows do
    printfn "|%*i|%-*s|" 20 i 10 s
```

### 整数の書式設定

基本的な整数型にはいくつかの特別なオプションがあります。

* `%i`や`%d`は符号付き整数
* `%u`は符号なし整数
* `%x`と`%X` は小文字、大文字の16進数
* `%o` は8進数

以下に例を示します。

```fsharp
printfn "signed8: %i unsigned8: %u" -1y -1y
printfn "signed16: %i unsigned16: %u" -1s -1s
printfn "signed32: %i unsigned32: %u" -1 -1
printfn "signed64: %i unsigned64: %u" -1L -1L
printfn "uppercase hex: %X lowercase hex: %x octal: %o" 255 255 255
printfn "byte: %i " 'A'B
```

この指定子は、整数型の型安全性を強制するものではありません。上の例からわかるように、符号付きのintを符号なしの指定子に渡すことは問題なくできます。
違うのは、フォーマットの仕方です。符号なし指定子は、intが実際にどのように型付けされているかに関わらず、intを符号なしとして扱います。

なお，`BigInteger`は基本的な整数型ではないので，`%A`や`%O`でフォーマットする必要があります。

```fsharp
printfn "bigInt: %i " 123456789I // エラー
printfn "bigInt: %A " 123456789I // OK
```

フラグを使って、符号やゼロパディングのフォーマットを制御することができます。

* `%0i` はゼロでパディングします。
* `%+i` はプラス記号を表示します。
* `% i` はプラス記号の代わりに空白を表示します。

以下に例を示します。

```fsharp
let rows = [ (1, "a"); (-22, "bb"); (333, "ccc"); (-4444, "ddd") ]

// アライメントあり
for (i,s) in rows do
    printfn "|%5i|%5s|" i s

// プラス記号の場合
for (i,s) in rows do
    printfn "|%+5i|%5s|" i s

// ゼロパッド付き
for (i,s) in rows do
    printfn "|%0+5i|%5s|" i s

// 左寄せの場合
for (i,s) in rows do
    printfn "|%-5i|%5s|" i s

// 左寄せとプラスで
for (i,s) in rows do
    printfn "|%+-5i|%5s|" i s

// 左寄せにして、プラスの代わりにスペースを入れる
for (i,s) in rows do
    printfn "|% -5i|%5s|" i s
```

### 浮動小数点と小数点のフォーマット

浮動小数点型には、いくつかの特別なオプションがあります。

* `%f`は標準フォーマット
* `%e`や`%E`は指数形式
* `%g`と`%G`は`f`と`e`をよりコンパクトに表示
* `%M`は小数用

以下にいくつかの例を示します。

```fsharp
let pi = 3.14
printfn "float: %f exponent: %e compact: %g" pi pi pi

let petabyte = pown 2.0 50
printfn "float: %f exponent: %e compact: %g" petabyte petabyte petabyte
```

10進数型は浮動小数点指定子と一緒に使うことができますが、精度が落ちることがあります。
`M`という指定子を使うと，精度が失われないようにすることができます。 以下の例でその違いがわかります。

```fsharp
let largeM = 123456789.123456789M // 小数点以下の値
printfn "float: %f decimal: %M" largeM largeM
```

浮動小数点数の精度は，`%.2f`や`%.4f`のような精度指定を使って制御することができます。
`f`と`%e`の精度指定では、小数点以下の桁数に影響し、`%g`では全体の桁数に影響します。
以下に例を示します。

```fsharp
printfn "2 digits precision: %.2f. 4 digits precision: %.4f." 123.456789 123.456789
// output => 2 digits precision: 123.46. 4 digits precision: 123.4568.
printfn "2 digits precision: %.2e. 4 digits precision: %.4e." 123.456789 123.456789
// output => 2 digits precision: 1.23e+002. 4 digits precision: 1.2346e+002.
printfn "2 digits precision: %.2g. 4 digits precision: %.4g." 123.456789 123.456789
// output => 2 digits precision: 1.2e+02. 4 digits precision: 123.5.
```

アライメントと幅のフラグは，浮動小数点と小数に対しても機能します。

```fsharp
printfn "|%f|" pi     // 通常
printfn "|%10f|" pi   // 幅
printfn "|%010f|" pi  // ゼロパッド
printfn "|%-10f|" pi  // 左寄せにする
printfn "|%0-10f|" pi // 左のゼロパッド
```

### カスタムフォーマット機能

2つの特別な書式指定子があり、単純な値ではなく関数を渡すことができます。

* `%t` は、入力されていないテキストを出力する関数を指定します。
* `%a` は、与えられた入力から何らかのテキストを出力する関数を意味します。

以下に，`%t`の使用例を示します:

```fsharp
open System.IO

//関数を定義する
let printHello (tw:TextWriter) = tw.Write("hello")

//テスト
printfn "custom function: %t" printHello
```

明らかに、コールバック関数はパラメータを取らないので、他の値を参照するクロージャになるでしょう。
ここでは，乱数を表示する例を示します:

```fsharp
open System
open System.IO

//クロージャを使って関数を定義します。
let printRand =
    let rand = new Random()
    // 実際の出力関数を返す
    fun (tw:TextWriter) -> tw.Write(rand.Next(1,100))

//テスト
for i in [1..5] do
    printfn "rand = %t" printRand
```

`%a`指定の場合、コールバック関数は追加のパラメータを受け取ります。つまり，`%a`指定子を使うときには，フォーマットする関数と値の両方を渡さなければなりません。

以下にタプルをカスタムフォーマットする例を示します。

```fsharp
open System
open System.IO

//コールバック関数を定義する
//データパラメータがTextWriterの後にあることに注意してください。
let printLatLong (tw:TextWriter) (lat,long) =
    tw.Write("lat:{0} long:{1}", lat, long)

// テスト
let latLongs = [ (1,2); (3,4); (5,6)]
for latLong in latLongs do
    // 関数と値の両方をprintfnに渡す
    printfn "latLong = %a" printLatLong latLong
```


### 日付の書式設定

F#では、日付に特別なフォーマット指定子はありません。

日付をフォーマットする場合、いくつかの選択肢があります。

* `ToString` を使用して日付を文字列に変換し、`%s` 指定子を使用します。
* 前述のように `%a` 指定子を使ってカスタムコールバック関数を使用する。

ここでは、使用している2つのアプローチを紹介します。

```fsharp
// 日付をフォーマットする関数
let yymmdd1 (date:DateTime) = date.ToString("yy.MM.dd")

// 日付をTextWriterで整形する関数
let yymmdd2 (tw:TextWriter) (date:DateTime) = tw.Write("{0:yy.MM.dd}", date)

// テスト
for i in [1..5] do
    let date = DateTime.Now.AddDays(float i)

    // %sを使用
    printfn "using ToString = %s" (yymmdd1 date)

    // %aを使用しています
    printfn "using a callback = %a" yymmdd2 date
```

どの方法が良いのでしょうか？

%s`を使った`ToString`は、テストや使用が簡単ですが、TextWriterに直接書き込むよりも効率が悪くなります。


## printf 関数の種類

`printf`関数には多くのバリエーションがあります。ここにクイックガイドがあります。

F#関数|C#の同等関数|説明
-------------|---------|----
`printf`および`printfn`|`Console.Write`および`Console.WriteLine`|"print"で始まる関数は標準出力に書き込みます。
`eprintf`および`eprintfn`|`Console.Error.Write`および`Console.Error.WriteLine`|"eprint"で始まる関数は、標準エラー出力に書き込みます。
`fprintf`および`fprintfn`|`TextWriter.Write`および`TextWriter.WriteLine`|"fprint"で始まる関数は、TextWriterに書き込みます。
`sprintf`|`String.Format`|"sprint"で始まる関数は文字列を返します。
`bprintf`|`StringBuilder.AppendFormat`|"bprint"で始まる関数は、StringBuilderに書き込みます。
`kprintf`,`kfprintf`,`ksprintf`および`kbprintf`|該当なし|継続を受け入れる関数。詳細は、次のセクションを参照してください。

*`bprintf`と`kXXX`ファミリ以外はすべて自動的に使用可能です ( [Microsoft.FSharp.Core.ExtraTopLevelOperators](http://msdn.microsoft.com/en-us/library/ee370230))
しかし、モジュールを使用してアクセスする必要がある場合は、 [`Printf`module] (http://msdn.microsoft.com/en-us/library/ee370560)*にあります。

使用法は明白です (後述の`kXXX`ファミリを除く) 。

特に有用なテクニックは、部分適用を使ってTextWriterやStringBuilderに "ベイク処理"することです。

StringBuilderの使用例を次に示します:

```fsharp
let printToSb s i =
    let sb = new System.Text.StringBuilder()

    // 部分適用を使って、StringBuilderを修正する
    let myPrint format = Printf.bprintf sb format

    do myPrint "A string: %s. " s
    do myPrint "An int: %i" i

    //結果の取得
    sb.ToString()

// テスト
printToSb "hello" 42
```

また，TextWriterを使った例を示します。

```fsharp
open System
open System.IO

let printToFile filename s i =
    let myDocsPath = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments)
    let fullPath = Path.Combine(myDocsPath, filename)
    use sw = new StreamWriter(path=fullPath)

    // 部分適用を使ってTextWriterを修正する
    let myPrint format = fprintf sw format

    do myPrint "A string: %s. " s
    do myPrint "An int: %i" i

    //結果の取得
    sw.Close()

// テスト
printToFile "myfile.txt" "hello" 42
```

### printfの部分適用の詳細

上記の2つのケースでは、部分的なアプリケーションを作成する際に、formatパラメータを渡す必要がありました。

つまり，次のようにしなければなりませんでした。

```fsharp
let myPrint format = fprintf sw format
```

ポイントフリー版でなく:

```fsharp
let myPrint = fprintf sw
```

これにより、コンパイラが不正な型だと文句を言うことがなくなります。その理由ははっきりしません。`printf`の最初のパラメータとして、上記の`TextWriterFormat`について簡単に説明しました。
`printf`は`String.Format`ではなく、"本物"にするためにTextWriterFormat (または類似のStringFormat) でパラメータ化する必要のある汎用関数です。

ですから、安全のためには、部分適用に対して過度に積極的になるのではなく、常に`printf`をformatパラメータと組み合わせた方が良いでしょう。

## kprintf 関数

4つの`kXXX`関数は、同類の関数と似ていますが、追加のパラメータを受け取ることができます。つまり、フォーマットが完了した直後に呼び出される関数です。

以下に，簡単な例を示します。

```fsharp
let doAfter s =
    printfn "Done"
    // 結果を返す
    s

let result = Printf.ksprintf doAfter "%s" "Hello"
```

なぜこのようなことが必要なのでしょうか？ いくつかの理由があります。

* ロギングフレームワークなど、何か有用なことをする別の関数に結果を渡すことができます。
* TextWriterをフラッシュするなどの処理を行うことができます。
* イベントを発生させることができる

それでは、外部のロギングフレームワークとカスタムイベントを使用したサンプルを見てみましょう。

まず、log4netやSystem.Diagnostics.Traceのようなシンプルなロギングクラスを作成します。
実際には、これを実際のサードパーティのライブラリで置き換えることになります。

```fsharp
open System
open System.IO

// log4netのようなロギングライブラリ
// またはSystem.Diagnostics.Trace
type Logger(name) =

    let currentTime (tw:TextWriter) =
        tw.Write("{0:s}",DateTime.Now)

    let logEvent level msg =
        printfn "%t %s [%s] %s" currentTime level name msg

    member this.LogInfo msg =
        logEvent "INFO" msg

    member this.LogError msg =
        logEvent "ERROR" msg

    static member CreateLogger name =
        new Logger(name)
```

次に、私のアプリケーションコードでは、次のようにします。

* ロギングフレームワークのインスタンスを作成します。ここではファクトリーメソッドをハードコーディングしましたが、IoCコンテナを使用することもできます。
* ロギングフレームワークを呼び出す `logInfo` と `logError` というヘルパー関数を作成し、`logError` の場合は、ポップアップメッセージも表示します。

```fsharp
// 私のアプリケーションコード
module MyApplication =

    let logger = Logger.CreateLogger("MyApp")

    // Logger クラスを使って logInfo を作成する
    let logInfo format =
        let doAfter s =
            logger.LogInfo(s)
        Printf.ksprintf doAfter format

    // Logger クラスを使って logError を作成します。
    let logError format =
        let doAfter s =
            logger.LogError(s)
            System.Windows.Forms.MessageBox.Show(s) |> ignore
        Printf.ksprintf doAfter format

    // ロギングを実行する関数
    let test() =
        do logInfo "Message #%i" 1
        do logInfo "Message #%i" 2
        do logError "Oops! an error occurred in my app"
```


最後に、`test`関数を実行すると、コンソールにメッセージが書き込まれ、ポップアップメッセージも表示されます。

```fsharp
MyApplication.test()
```

以下に示すように、ロギングライブラリのラッパークラス "FormattingLogger "を作成することで、ヘルパーメソッドのオブジェクト指向バージョンを作成することもできます。

```fsharp
type FormattingLogger(name) =

    let logger = Logger.CreateLogger(name)

    // Loggerクラスを使ってlogInfoを作成する
    member this.logInfo format =
        let doAfter s =
            logger.LogInfo(s)
        Printf.ksprintf doAfter format

    // Logger クラスを使って logError を作成します。
    メンバー this.logError フォーマット =
        let doAfter s =
            logger.LogError(s)
            System.Windows.Forms.MessageBox.Show(s) |> ignore
        Printf.ksprintf doAfter format

    static member createLogger name =
        new FormattingLogger(name)

// 私のアプリケーションコード
module MyApplication2 =

    let logger = FormattingLogger.createLogger("MyApp2")

    let test() =
        do logger.logInfo "Message #%i" 1
        do logger.logInfo "Message #%i" 2
        do logger.logError "Oops! an error occurred in app 2"

// テスト
MyApplication2.test()

```

オブジェクト指向のアプローチは、よく知られていますが、必ずしも良いものではありません。オブジェクト指向 (OO) メソッドと純粋関数の長所と短所については、 [ここ](/posts/type-extensions/#downsides-of-methods) で説明しています。


