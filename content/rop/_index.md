---
layout: page
title: "Railway Oriented Programming"
description: Slides and videos explaining a functional approach to error handling
hasComments: 1
image: "/rop/rop427.jpg"
date: 2020-01-01
---

このページでは、私の講演「鉄道指向プログラミング」のスライドとコードへのリンクを掲載しています。

この講演の宣伝文句は以下の通りです。

> 関数型プログラミングの多くの例では、常に「幸せな道」を歩むことを前提としています。
> しかし、堅牢な現実世界のアプリケーションを作成するためには、検証、ロギング、ネットワークやサービスのエラーなどに対処しなければなりません。
> しかし、堅牢な現実世界のアプリケーションを作るためには、検証、ロギング、ネットワークやサービスのエラー、その他の厄介な問題に対処しなければなりません。
> \
> \
> では、これらをきれいな機能的な方法で処理するにはどうすればよいのでしょうか？
> \
> \
> 本講演では、このテーマについて
> 楽しくてわかりやすい鉄道の例えを使って、簡単に紹介します。

また、近日中にこれらのトピックに関する記事をいくつかアップする予定です。一方で、似たような内容を扱った[recipe for a functional app](/series/a-recipe-for-a-functional-app.html)シリーズもご覧ください。

もし、実際のコードを見たいのであれば、私は
[普通のC#とF#をROPアプローチで比較したGithubのプロジェクト](https://github.com/swlaschin/Railway-Oriented-Programming-Example)

警告: これはエラー処理のための便利なアプローチですが、極端なことはしないでください! [Against Railway-Oriented Programming](/posts/against-railway-oriented-programming/)をご覧ください。

## ビデオ

NDC London 2014でこのトピックについて発表しました(画像をクリックすると動画が表示されます)

[![Video from NDC London 2014](rop427.jpg)](https://goo.gl/Lv5ZAo)

この講演の他のビデオは、[NDC Oslo 2014](http://vimeo.com/97344498)
および[Functional Programming eXchange, 2014](https://skillsmatter.com/skillscasts/4964-railway-oriented-programming)


## スライド

Functional Programming eXchange（2014年3月14日）のスライド

{{< slideshare "9eUxEVfdTUTTh8" "railway-oriented-programming" "Railway Oriented Programming" >}}

パワーポイントのスライドは[Github](https://github.com/swlaschin/RailwayOrientedProgramming)からも公開しています。ご自由にご利用ください。

{{< book_page_explain >}}

{{< linktarget "monads" >}}

## EitherモナドとKleisli compositionとの関係 ##

これを読んでいるハスケラーならば、このアプローチが[`Either`型](http://book.realworldhaskell.org/read/error-handling.html)であることがすぐにわかるでしょう。
Leftケースにカスタムエラータイプのリストを使用するように特化されています。Haskellでは次のようになります。`type TwoTrack a b = Either [a] (b,[a])`

確かに、私がこの方法を発明したと主張するつもりは全くありません（くだらないアナロジーは主張していますが）。 では、なぜ私は標準的なHaskellの用語を使わなかったのでしょうか？

まず、**この投稿はモナドのチュートリアル**ではなく、エラー処理という特定の問題を解決することに焦点を当てています。

F#に来た人のほとんどは、モナドに慣れていません。私はむしろ、視覚的で威圧感がなく、多くの人にとってより直感的なアプローチを提示したいと考えています。

私は、["具体的なことから始めて、抽象的なことに移る"](https://byorgey.wordpress.com/2009/01/12/abstraction-intuition-and-the-monad-tutorial-fallacy/)を強く信じています。
教育的なアプローチです。私の経験では、いったんこの特別なアプローチに慣れてしまえば、後からより高度な抽象的概念を理解することが容易になります。

第二に、bindを使った2トラックの型がモナドであると主張するのは間違っています。

第三に、そして最も重要なことですが、`Either`はあまりにも一般的な概念です。**私が提示したかったのはレシピであって、ツールではありません。**

例えば、パンを作るためのレシピを知りたいとき、「小麦粉とオーブンを使えばいい」と言っても、あまり役に立ちませんよね。

それと同じように、エラー処理のレシピを知りたければ、「Eitherとbindを使えばいい」と言ってもあまり意味がありません。

そこで、このアプローチでは、一連のテクニックを紹介します。

* Eitherの左辺と右辺の両方で、カスタムエラータイプのリストを使用する（たとえば、`Either String a`ではなく）。
* モナディック関数をパイプラインに統合するための「bind」（`>>=`）。
* モナド関数を合成するための "Kleisli composition" (`>=>`) です。
* "map" (`fmap`): 非単項式関数をパイプラインに統合します。
(F# は IO モナドを使用しないため) * 単位関数をパイプラインに統合するための "tee" 。
* 例外からエラーケースへのマッピング。
* モナド関数を並列に結合するための `&&&` (例：検証用)
* ドメイン駆動設計のためのカスタムエラータイプの利点。
* ロギング、ドメインイベント、トランザクションの補償など、明らかな拡張機能。

「Eitherモナドを使えばいい」というものではなく、より包括的なアプローチであることがお分かりいただけると思います。

ここでの私の目標は、ほとんどすべての状況で使用可能な汎用性を持ち、かつ制約のあるテンプレートを提供することです。
しかし、一貫したスタイルを実現するためには十分な制約が必要です。
つまり、コードを書く方法は基本的に1つしかないということです。
これは、後でコードをメンテナンスしなければならない人にとって、どのようにまとめられているかをすぐに理解することができるので、非常に便利です。

これが唯一の方法だとは言いません。しかし、このアプローチは良いスタートだと思います。

余談ですが、Haskellコミュニティでも、[エラー処理に対する一貫したアプローチはありません](http://www.randomhacks.net/2007/03/10/haskell-8-ways-to-report-errors/)。
これは[初心者を混乱させるものです](http://programmers.stackexchange.com/questions/252977/cleanest-way-to-report-errors-in-haskell)。
[たくさんのコンテンツ](http://www.fpcomplete.com/school/starting-with-haskell/basics-of-haskell/10_Error_Handling)があることは知っています。
しかし、これらすべてのツールを
包括的にまとめたドキュメントを私は知りません。

## どうやって自分のコードに使えばいいの？

* NuGetで動作する既製のF#ライブラリが欲しい場合は、 [Chessieプロジェクト](https://fsprojects.github.io/Chessie/)をチェックしてください。
* これらの技術を使ったウェブサービスのサンプルを見たい場合は、 [GitHubにプロジェクトを作成しました](https://github.com/swlaschin/Railway-Oriented-Programming-Example)をご覧ください。
* また、[FizzBuzzに適用されたROPアプローチを見る](/posts/railway-oriented-programming-carbonated/)こともできます!

F#は型クラスを持っていないので、モナドを行うための再利用可能な方法はありません（ただし、[FSharpXライブラリ](https://github.com/fsprojects/fsharpx/blob/master/src/FSharpx.Core/ComputationExpressions/Monad.fs)
には便利な方法があります）．)  このため，`Rop.fs`ライブラリは，すべての関数をゼロから定義しています．
(しかし，ある意味では，外部への依存性が全くないため，この孤立化が役に立つこともあります)．

## 続きを読む

> *"One bind does not a monad make" -- アリストテレス*.

上に書いたように、私がモナドから遠ざかっていた理由の一つは、モナドを正しく定義することは、単に「bind」と「return」を実装するだけの問題ではないからです。
モナドを正しく定義することは、単に "bind "と "return "を実装することではなく、モナド則（それは特定の状況における[monoid laws](/posts/monoids-without-tears/)に従う必要がある代数的な構造です。
これは、今回の講演では避けたかった道です。

しかし、`Either`やKleisi compositionについてもっと詳しく知りたいという方には、以下のリンクが役に立つかもしれません。

* **Monads in general**.
  * [Stack overflow answer on monads](http://stackoverflow.com/questions/44965/what-is-a-monad)
  * [Stack overflow answer by Eric Lippert](http://stackoverflow.com/questions/2704652/monad-in-plain-english-for-the-oop-programmer-with-no-fp-background/2704795#2704795)
  * [Monads in pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)
  * ["You Could Have Invented Monads"](http://blog.sigfpe.com/2006/08/you-could-have-invented-monads-and.html)
  * [Haskell チュートリアル](https://www.haskell.org/tutorial/monads.html)
  * [The hardcore definition on nLab](http://ncatlab.org/nlab/show/monad)
* **The `Either` monad**.
  * [School of Haskell](http://www.fpcomplete.com/school/starting-with-haskell/basics-of-haskell/10_Error_Handling)
  * [Real World Haskell on error handling](http://book.realworldhaskell.org/read/error-handling.html) (途中まで)
  * [LYAH on error handling](http://learnyouahaskell.com/for-a-few-monads-more) (中途半端)
* **Kleisli categories and composition**.
  * [FPComplete への投稿](http://www.fpcomplete.com/user/Lkey/kleisli)
  * [Bartosz Milewski の投稿](http://bartoszmilewski.com/2014/12/23/kleisli-categories/)
  * [The hardcore definition on nLab](http://ncatlab.org/nlab/show/Kleisli+category)
* **包括的なエラー処理のアプローチ**。
  * [この投稿の項目5](http://www.randomhacks.net/2007/03/10/haskell-8-ways-to-report-errors/)
  * この講演で取り上げた技術をすべてカバーする他のアプローチを私は知りません。
    もし知っていたら、コメントで教えていただければ、このページを更新します。