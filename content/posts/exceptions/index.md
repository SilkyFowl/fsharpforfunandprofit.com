---
layout: post
title: "Exceptions"
description: "Syntax for throwing and catching"
date: 2012-05-21
nav: thinking-functionally
seriesId: "Expressions and syntax"
seriesOrder: 8
---

他の.NET言語と同様に、F#は例外のthrowとcatchをサポートしてます。コントロールフロー式と同様、構文は見慣れたものですが、ここでもいくつか注意点があります。

## 独自例外の定義

例外をraying/throwする場合、`InvalidOperationException`などの標準型の例外を使用するか、次のような単純な構文を使用して独自の型を定義できます (例外の"内容"は任意のF#型です) 。

```fsharp
exception MyFSharpError1 of string
exception MyFSharpError2 of string * int
```

これで終わりです！新しい例外クラスの定義は、C#よりもずっと簡単です。

## 例外をスローする

例外をスローする基本的な方法は3つあります。

* "invalidArg"などの組み込み関数の使用
* 標準.NET例外クラスの1つの使用
* 独自のカスタム例外型の使用

### 例外を発生させる方法 1:組み込み関数を使用する

F#には4つの便利な例外キーワードが組み込まれています:

* `failwith`は汎用的な`System.Exception`をスローします
* `invalidArg`は`ArgumentException`をスローします
* `nullArg`は`NullArgumentException`をスローします
* `invalidOp`は`InvalidOperationException`をスローします

この4つで、通常スローされる例外のほとんどをカバーできます。使用方法は次の通りです:

```fsharp
// 一般的なSystem.Exceptionをスローする
let f x =
   if x then "ok"
   else failwith "message"

// ArgumentExceptionをスローする
let f x =
   if x then "ok"
   else invalidArg "paramName" "message"

// NullArgumentExceptionをスローする
let f x =
   if x then "ok"
   else nullArg "paramName" "message"

// InvalidOperationExceptionをスローする
let f x =
   if x then "ok"
   else invalidOp "message"
```

ところで、`failwith`の便利なバージョンとして、`failwithf`というものがあります。これは、`printf`スタイルのフォーマットを含んでおり、カスタムメッセージを簡単に作ることができます。

```fsharp
open System
let f x =
    if x = "bad" then
        failwithf "Operation '%s' failed at time %O" x DateTime.Now
    else
        printfn "Operation '%s' succeeded at time %O" x DateTime.Now

// テスト
f "good"
f "bad"
```

### 例外を発生させる方法 2: .NET 標準の例外クラスを使用する

任意の.NET例外を明示的に `raise` することができます:

```fsharp
// 例外の種類を制御する
let f x =
   if x then "ok"
   else raise (new InvalidOperationException("message"))
```

### 例外を発生させる方法 3: 独自の F# 例外型を使用する

最後に、先ほど定義したような独自の型を使うことができます。

```fsharp
// 独自のF#例外型を使う
let f x =
   if x then "ok"
   else raise (MyFSharpError1 "message")
```

以上で、例外を発生させる方法はほぼ完了です。

## 例外を発生させると、関数の型にどのような影響があるのでしょうか？

先ほど、if-then-else式の両ブランチは同じ型を返さなければならないと言いました。しかし、この制約の中で例外を発生させるとどうなるのでしょうか？

その答えは、例外を発生させるコードは、式の型を決定するためには無視されるということです。 つまり、関数のシグネチャは、例外の場合ではなく、通常の場合のみに基づいて決定されるということです。

たとえば、次のコードでは例外は無視され、関数全体のシグネチャは予想どおり`bool->int`になります:

```fsharp
let f x =
   if x then 42
   elif true then failwith "message"
   else invalidArg "paramName" "message"
```

質問：両方の分岐で例外が発生した場合、関数のシグネチャはどうなると思いますか？

```fsharp
let f x =
   if x then failwith "error in true branch"
   else failwith "error in false branch"
```

使ってみてください。

## 例外の処理

例外は、他の言語のようにtry-catchブロックを使用してキャッチできません。F#は代わりに`try-with`を呼び出し、標準的なパターンマッチング構文を使用して例外型毎に処理を行います。

```fsharp
try
    failwith "fail"
with
    | Failure msg -> "caught: " + msg
    | MyFSharpError1 msg -> " MyFSharpError1: " + msg
    | :? System.InvalidOperationException as ex -> "unexpected"
```

捕捉する例外が`failwith` (例:System.Exception) またはカスタムF#例外でthrowされた場合、上記の単純なタグアプローチを使用して照合できます。

一方、特定の.NET例外クラスを捕捉するには、より複雑な構文を使用して一致させる必要があります。

```fsharp
:? (exception class) as ex
```

ここでも、if-then-elseやループと同様に、try-withブロックはある値を返す式です。これは、`try-with`式の全枝は*同じ型を返さなければならない*ことを意味します。

次の例を考えてみます:

```fsharp
let divide x y=
    try
        (x+1) / y                      // ここでエラー -- 下記参照
    with
    | :? System.DivideByZeroException as ex ->
          printfn "%s" ex.Message
```

評価しようとすると、エラーが発生ます:

    error FS 0043:型 'unit' は型 'int' と一致しません

これは、"`with`"分岐の型が`unit`であり、"try`"分岐の型が`int`であることに起因します。つまり、2つの分岐の型には互換性がありません。

これを修正するには、"`with`"ブランチが`int`を返すようにする必要があります。これは、セミコロンを使って式を1行で連結することで簡単に行うことができます。

```fsharp
let divide x y=
    try
        (x+1) / y
    with
    | :? System.DivideByZeroException as ex ->
          printfn "%s" ex.Message; 0            //ここに0を追加しました!

//テスト
divide 1 1
divide 1 0
```

`try-with`式の型が定義されたので、関数全体に型、つまり`int->int->int`を割り当てることができます。

以前と同様に、例外をスローする分岐は、型が決定されるときには無視されます。

### 例外の再スロー

必要に応じて、補足した例外の処理中に"`reraise()`"関数を呼び出して、その例外を呼び出しチェーンに伝播させることができます。これは、C#の`throw`キーワードと同じです。

```fsharp
let divide x y=
    try
        (x+1) / y
    with
    | :? System.DivideByZeroException as ex ->
          printfn "%s" ex.Message
          reraise()

//テスト
divide 1 1
divide 1 0
```

## Try-finally

もうひとつのおなじみの表現は `try-finally` です。 ご想像のとおり、"finally"節は何があっても呼び出されます。

```fsharp
let f x =
    try
        if x then "ok" else failwith "fail"
    finally
        printf "this will always be printed"
```

try-finally式全体の戻り型は、"try"節自体の戻り型と常に同じです。"finally"節は式全体の型には影響しません。上記の例では、式全体の型は`string`です。

"finally"句は常にunitを返す必要があるため、unit以外の値はコンパイラによって警告されます。

```fsharp
let f x =
    try
        if x then "ok" else failwith "fail"
    finally
        1+1  // この式の結果の型は 'int' で、暗黙的に無視されます。
```

## try-withとtry-finallyの組み合わせ

try-with式とtry-finally式は別々であり、1つの式として直接まとめることはできません。この場合、必要に応じて入れ子にする必要があります。

```fsharp
let divide x y=
   try
      try
         (x+1) / y
      finally
         printf "this will always be printed"
   with
   | :? System.DivideByZeroException as ex ->
           printfn "%s" ex.Message; 0
```

## 関数は例外を発生させるべきか、エラー構造体を返すべきか?

関数を設計するとき、例外を発生させるべきか、エラー情報を格納した構造体を返すべきか。本章では、2つの異なるアプローチについて説明します。

### ペア関数アプローチ

1つの方法は、2つの関数を提供することです。1つは、すべてが正常に動作すると仮定し、それ以外の場合は例外をスローする関数で、もう1つは、何か問題が発生した場合に欠損値を返す関数です。

次に、例外を処理しない関数と、例外を処理する関数の2つのライブラリ関数の例を示します:

```fsharp
// 例外を処理しないライブラリ関数
let divideExn x y = x / y

// 例外をNoneに変換するライブラリ関数
let tryDivide x y =
   try
       Some (x / y)
   with
   | :? System.DivideByZeroException -> None // 欠損を返す
```

Note: この値が有効かどうかをクライアントに知らせるために、`tryDividev`コードで`Some`と`None`のOption型が使われています。

最初の関数では、クライアント・コードは例外を明示的に処理する必要があります。

```fsharp
// クライアントコードは例外を明示的に処理する必要がある
try
    let n = divideExn 1 0
    printfn "result is %i" n
with
| :? System.DivideByZeroException as ex -> printfn "divide by zero"
```

クライアントにこれを強制する制約はないため、この方法は障害の原因になる可能性があることに注意してください。

2番目の関数を使用すると、クライアント・コードは単純になり、通常の場合とエラーの場合の両方を処理するように制限されます。

```fsharp
//クライアントコードは両方のケースをテストする必要がある
match tryDivide 1 0 with
| Some n -> printfn "result is %i" n
| None -> printfn "divide by zero"
```

この"normal vs.try"は.NET BCLでは非常によく使用され、F#ライブラリでも少数のケースで使用されます。たとえば、`List`モジュールでは次のようになります。

* 'List.find`は、キーが見つからない場合に`KeyNotFoundException`をスローします。
* しかし、`List.tryFind`はOption型を返し、キーが見つからない場合は`None`を返します。

この方法を使用する場合は、命名規則を定めてください。たとえば:

* "doSomethingExn":クライアントが例外をキャッチすることを想定した関数。
* "tryDoSomething":ユーザに代わって通常の例外を処理する関数。

Note: 筆者は "doSomething"に"Exn"という接尾辞を付けるほうが良いと考えています。これにより、通常の場合でもクライアントが例外をキャッチすることを想定していることが明確になります。

この方法の全体的な問題は、関数のペアを作成するために余分な作業が必要になることです。また、クライアントが安全でない関数を使用している場合には、例外の捕捉をクライアントに委ねることになるため、システムの安全性が低下します。

### エラーコード方式

> "適切なエラーコードベースのコードを書くのは難しいが,適切な例外ベースのコードを書くのは極めて難しい。"
> [*Raymond Chen*](https://devblogs.microsoft.com/oldnewthing/20050114-00/?p=36693)

関数型の世界では、例外を発生させるよりもエラー・コード (つまりエラー*型*) を返す方が通常は好まれるため、 (ユーザーが処理すべきと考えられる) 一般的なケースをエラー型に変換し、非常に特殊な例外はそのままにしておくというのが、標準的なハイブリッド・アプローチです。

通常、最も単純なやり方は、成功の場合は`Some`、エラーの場合は`None`というoption型を使うことです。`tryDivide`や`tryParse`のようにエラーケースが明白な場合は、詳細なエラーケースを記述する必要はありません。

しかし、複数のエラーが発生する可能性があり、それぞれ異なる方法で処理する必要がこともあります。この場合、各エラーに対応するケース識別子を持つ判別共用体型が有効です。

次の例では、SqlCommandを実行します。3つの非常にありふれたエラー・ケースは、ログインエラー、制約エラー、および外部キーエラーです。その他のエラーはすべて例外として発生させます。

```fsharp
open System.Data.SqlClient

type NonQueryResult =
    | Success of int
    | LoginError of SqlException
    | ConstraintError of SqlException
    | ForeignKeyError of SqlException

let executeNonQuery (sqlCommmand:SqlCommand) =
    try
       use sqlConnection = new SqlConnection("myconnection")
       sqlCommmand.Connection <- sqlConnection
       let result = sqlCommmand.ExecuteNonQuery()
       Success result
    with
    | :?SqlException as ex ->     // SqlExceptionの場合
        ex.Numberと一致
        | 18456 ->                // ログインに失敗
            LoginError ex
        | 2601 | 2627 ->          // 制約エラーの処理
            constraintError ex
        | 547 ->                  // 外部キーエラーを処理
            ForeignKeyError ex
        | _ ->                    // 他のケースを処理しない
            reraise()
       // 他の SqlExceptions 以外の例外は正常にスローされる
```

クライアントは一般的なケースを処理するように強制されますが、まれな例外は呼び出しチェーンの上位にあるハンドラによって捕捉されます。

```fsharp
let myCmd = new SqlCommand("DELETE Product WHERE ProductId=1")
let result =  executeNonQuery myCmd
match result with
| Success n -> printfn "success"
| LoginError ex -> printfn "LoginError: %s" ex.Message
| ConstraintError ex -> printfn "ConstraintError: %s" ex.Message
| ForeignKeyError ex -> printfn "ForeignKeyError: %s" ex.Message
```

従来のエラーコードアプローチとは異なり、関数の呼び出し側はエラーをすぐに処理する必要がなく、次に示すように、処理方法を知っている人に届くまで構造体をそのまま渡すだけで済みます:

```fsharp
let lowLevelFunction commandString =
  let myCmd = new SqlCommand(commandString)
  executeNonQuery myCmd          //結果を返す

let deleteProduct id =
  let commandString = sprintf "DELETE Product WHERE ProductId=%i" id
  lowLevelFunction commandString  //エラーを処理せずに返す

let presentationLayerFunction =
  let result = deleteProduct 1
  match result with
  | Success n -> printfn "success"
  | errorCase -> printfn "error %A" errorCase
```

一方、C#とは異なり、式の結果が誤って破棄されることはありません。そのため、関数がエラー結果を返した場合、呼び出し元はそれを処理しなければなりません (ただし、どうしても不正な動作をさせるつもりであれば、`ignore`に送ってください) 。

```fsharp
let presentationLayerFunction =
  do deleteProduct 1    // error: 結果コードの破棄!
```

