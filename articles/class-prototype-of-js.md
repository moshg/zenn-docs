---
title: "いまさらJavaScriptのclassとprototypeを勉強する"
publication_name: "aidemy"
emoji: "🦖"
type: "tech"
topics: ["javascript", "typescript"]
published: true
---

## はじめに

JavaScriptのプロトタイプ理解してますか？
`class`はプロトタイプのシンタックスシュガーであるみたいな記述を見たり、MDNのドキュメントでメソッドを調べるとprototypeが現れたり (e.g. [Array.prototype.map](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/map)) でなんとなく目にしますが、典型的なアプリケーション開発をしている分にはプロトタイプや`class`の裏側を知らずに乗り切れてしまいます。
しかしいざ必要になって[^necessity][^necessity2]勉強してみたところ面白かったのですが、どこから勉強すればいいのか前後することになりました。[^yak_shaving]
そこで、順番に読めるようドキュメントをまとめ、私の理解を残しておきます。

[^necessity]: JSのイケてるWebフレームワークである[Hono](https://hono.dev/)に入った最適化PR (解説記事: [HonoのNode.jsアダプタが2.7倍速くなりました](https://zenn.dev/yusukebe/articles/7ac501716ae1f7)) を読もうとしたときに必要になりました
[^necessity2]: 実はけっこう前に『[JavaScriptのカスタムエラーはこれでOK](https://www.wantedly.com/companies/wantedly/post_articles/492456)』を理解する上でも必要になっていたのですが、なんとなく雰囲気でわかったつもりになっていました
[^yak_shaving]: こういうのを[yak shaving](http://0xcc.net/blog/archives/000196.html)というらしいです

## ドキュメント

以下がクラスとプロトタイプの関係を理解するまでに読んだMDN web docsです。

1. [this](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/this)
1. [オブジェクトのプロトタイプ](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Objects/Object_prototypes)
1. [new 演算子](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/new)
1. [クラス](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Classes)
2. [extends](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Classes/extends)
3. [instanceof](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/instanceof)
4. [継承とプロトタイプチェーン](https://developer.mozilla.org/ja/docs/Web/JavaScript/Inheritance_and_the_prototype_chain) (これは重かったので読み切っていない)

次のセクションでは各ドキュメントに対する要約と私の理解をまとめています。

## 各ドキュメントの要約

### this

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/this

`function`内の`this`は関数を定義時ではなく呼び出し時のコンテキストに依存します。

```js
function getName() {
  return this.name;
}

const obj = { name: "foo", getName: getName };

console.log(obj1.getName()); // foo
```

ラムダ式で定義された関数の場合は、定義時のコンテキストに依存します。

### オブジェクトのプロトタイプ

https://developer.mozilla.org/ja/docs/Learn/JavaScript/Objects/Object_prototypes

すべてのオブジェクトはプロトタイプと呼ばれるオブジェクトを持ち、`Object.getPrototypeOf`で取得できます。
プロトタイプは「aのプロトタイプ→aのプロトタイプのプロトタイプ→…」のようにチェインになっており、最後は`null`になります。

プロトタイプはプロパティアクセスに対して継承のような機能を提供します。
`obj.foo` のようにプロパティにアクセスしたとき、`obj`のプロトタイプチェーンをたどって`foo`という名前のプロパティを探します。

```js
const prototype = {
  name: "foo",
};

// prototypeをプロトタイプとして持つオブジェクトを作成する
const obj = Object.create(prototype);

console.log(obj); // {}
console.log(obj.name); // foo
```

関数は`prototype`というプロパティを持ちます。これはその関数自身のプロトタイプではなく[^prototype]、その関数をコンストラクタとして使ったときに生成されるオブジェクトのプロトタイプになります。

[^prototype]: 何もわかってなかったのでこれは衝撃でした

```js
function Foo() {}
Foo.prototype.name = "foo";

Object.getPrototypeOf(Foo) !== Foo.prototype; // true (Foo.prototypeはFooのプロトタイプではない)

const obj = new Foo();

console.log(Object.getPrototypeOf(obj) === Foo.prototype); // true
console.log(obj.name); // foo
```

このように`new`はクラスに限らず関数に対して使用することができます。
`new`の詳細は次の[new 演算子](#new-演算子)で説明されます。

### new 演算子

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/new

new演算子は以下のように関数をコンストラクタとして扱ってオブジェクトを作成します。

```js
function MyClass(foo, bar) {
  this.foo = foo;
  this.bar = bar;
}

const obj = new MyClass("value1", "value2");

console.log(obj); // MyClass { foo: 'value1', bar: 'value2' }
```

前の[オブジェクトのプロトタイプ](#オブジェクトのプロトタイプ)で説明したように、`new MyClass`で作成されるオブジェクトのプロトタイプは`MyClass.prototype`になるという話と合わせると、`new`は概ね以下のようなコードと等価になります。

```js
// new MyClass("value1", "value2") はほぼ以下のコードと同じ

// MyClass.prototypeをプロトタイプとして持つオブジェクトを作成する
const obj = Object.create(MyClass.prototype);
// オブジェクトをthisとしてバインドしてMyClass関数を呼び出す
MyClass.call(obj, "value1", "value2");
```

`prototype` にメソッドを定義して、コンストラクタでオブジェクトにフィールドとしてのプロパティをセットすることで `class` っぽいことが実現できます。

### クラス

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Classes

クラス定義の構文は以下のようになります。

```js
class MyClass {
  constructor(foo, bar) {
    this.foo = foo;
    this.bar = bar;
  }

  static baz = "value3";

  fooBar() {
    return this.foo + this.bar;
  }  
}
```

上記のクラスは概ね以下のような関数を定義することと等価です。

```js
function MyClass(foo, bar) {
  this.foo = foo;
  this.bar = bar;
}

MyClass.baz = "value3";

MyClass.prototype.fooBar = function() {
  return this.foo + this.bar;
};
```

TypeScriptにおいては`MyClass`を値として使うときは上記の`MyClass`そのものを指し、`MyClass`を型として使うときは`MyClass`のインスタンスの型を指すということになります。

`class`は`function`の単純なシンタックスシュガーとして完全に提供できるわけではなく、例えば[プライベートプロパティ](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Classes/Private_properties)を提供します。

### extends

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Classes/extends

`ChildClass`が`ParentClass`を継承している場合、`ChildClass`のプロトタイプとして`ParentClass`が設定され、
`ChildClass.prototype`のプロトタイプとして`ParentClass.prototype`が設定されます。

### instanceof

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/instanceof

`obj instanceof Class`は`obj`が`Class`のインスタンスであるかどうかを、`obj`のプロトタイプチェーンに`Class.prototype`と一致するものが存在するかどうかによって判定します。

## まとめ

以上でクラスとプロトタイプを概ね理解できたと思います。
ちょっと可能性が広がった感じがしていろいろ試してみたくなったのではないでしょうか。[^try]

この記事は同じことを勉強する必要が現れた人がスムーズに勉強できるようにしたい気持ち半分、私が味わった楽しさを共有したい気持ち半分で書きました。
どちらも楽しくスムーズに勉強できていれば非常に嬉しいです！！！


[^try]: 私はKotlinのスコープ関数っぽいものをTypeScriptで書いてみました: [スクラップ](https://zenn.dev/link/comments/1b712c20378d56)

