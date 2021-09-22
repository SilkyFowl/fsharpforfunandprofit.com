---
layout: post
title: "Designing with types: Introduction"
description: "Making design more transparent and improving correctness"
date: 2013-01-12
nav: thinking-functionally
seriesId: "Designing with types"
seriesOrder: 1
categories: [Types, DDD]
---

このシリーズでは、設計プロセスの一環として型を使用する方法のいくつかを見ていきます。
特に、型を思慮深く使用することで、設計の透明性を高め、同時に正しさを向上させることができます。

このシリーズでは、設計の「ミクロレベル」に焦点を当てます。つまり、個々の型や機能の最下位レベルでの作業です。
より高いレベルの設計アプローチや、それに伴う関数型やオブジェクト指向スタイルの使用に関する決定については、別のシリーズでご紹介します。

この提案の多くはC#やJavaでも実現可能ですが、F#の型は軽量なので、このようなリファクタリングを行う可能性が高くなります。

## 基本的な例 ##

型の様々な使い方を説明するために、非常に簡単な例、つまり以下のような`Contact`型を使ってみます。

```fsharp
type Contact =
    {
    FirstName: string;
    MiddleInitial: string;
    LastName: string;

    EmailAddress: string;
    //true if ownership of email address is confirmed
    IsEmailVerified: bool;

    Address1: string;
    Address2: string;
    City: string;
    State: string;
    Zip: string;
    //true if validated against address service
    IsAddressValid: bool;
    }

```

これはとても当たり前のことで、誰もが何度もこのようなものを見たことがあると思います。では、これで何ができるでしょうか？ 型システムを最大限に活用するためには、どのようにリファクタリングすればよいのでしょうか。

## "atomic" typeの作成 ##

まず最初にやるべきことは、データのアクセスと更新の使用パターンを見ることです。 例えば、`Zip`が更新されるときに、`Address1`も同時に更新されることはあるでしょうか？逆に、あるトランザクションが `EmailAddress` を更新しても `FirstName` を更新しないことはよくあることです。

これが、最初のガイドラインです。

* *ガイドライン：レコードやタプルを使って、一貫性が必要なデータ（つまり「アトミック」なデータ）をまとめるべきだが、関連性のないデータを不必要にまとめてはいけない。

このケースでは、3つの名前の値がセットになっていること、アドレスの値がセットになっていること、Eメールもセットになっていることが明らかになっています。

ここでは、`IsAddressValid`や`IsEmailVerified`などの追加フラグもあります。これらは関連するセットの一部とすべきでしょうか？ フラグは関連する値に依存しているので、今のところは確かにイエスです。

例えば、`EmailAddress` が変更された場合、`IsEmailVerified` も同時に false にリセットする必要があるでしょう。

`PostalAddress`については、コアとなる「住所」の部分は、`IsAddressValid`フラグがなくても有用な共通の型であることは明らかなようです。一方で、`IsAddressValid`は住所に関連付けられていて、住所が変更されると更新されます。

そこで、*2つの*タイプを作るべきだと思われます。1つは汎用の`PostalAddress`で、もう1つは連絡先の文脈における住所で、これは例えば`PostalContactInfo`と呼ぶことができます。

```fsharp
type PostalAddress =
    {
    Address1: string;
    Address2: string;
    City: string;
    State: string;
    Zip: string;
    }

type PostalContactInfo =
    {
    Address: PostalAddress;
    IsAddressValid: bool;
    }
```


最後に、option型を使って、`MiddleInitial`などの特定の値が本当にオプションであることを知らせることができます。

```fsharp
type PersonalName =
    {
    FirstName: string;
    // use "option" to signal optionality
    MiddleInitial: string option;
    LastName: string;
    }
```

## まとめ

以上の変更により、以下のようなコードになりました。

```fsharp
type PersonalName =
    {
    FirstName: string;
    // use "option" to signal optionality
    MiddleInitial: string option;
    LastName: string;
    }

type EmailContactInfo =
    {
    EmailAddress: string;
    IsEmailVerified: bool;
    }

type PostalAddress =
    {
    Address1: string;
    Address2: string;
    City: string;
    State: string;
    Zip: string;
    }

type PostalContactInfo =
    {
    Address: PostalAddress;
    IsAddressValid: bool;
    }

type Contact =
    {
    Name: PersonalName;
    EmailContactInfo: EmailContactInfo;
    PostalContactInfo: PostalContactInfo;
    }

```

まだひとつも関数を書いていませんが、すでにコードはドメインをよりよく表現しています。しかし、これはできることのほんの始まりに過ぎません。

次は、シングルケースユニオンを使って、プリミティブ型に意味を持たせることです。

