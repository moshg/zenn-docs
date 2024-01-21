---
title: "今更JavaScriptのclass, prototypeを勉強する"
publication_name: "aidemy"
emoji: "👴"
type: "tech"
topics: ["javascript"]
published: false
---

## はじめに

典型的な開発をしている分には、JavaScriptの`class`など多くの機能は他の言語と同じ感覚で使えます。
しかし場合によってはJS固有の機能に踏み込む必要に迫られることがあります。
私の場合、JSのイケてるWebフレームワークである[Hono](https://hono.dev/)に少し前に入った高速化 (解説記事: [HonoのNode.jsアダプタが2.7倍速くなりました](https://zenn.dev/yusukebe/articles/7ac501716ae1f7)) に興味を持ってPRを読もうとしたところ、prototypeなどJSの基礎に関する知識が必要になりました。

勉強してみると面白かったのですが、ドキュメントを前後することになったので、学んだことをまとめておきます。

## this

参考: [MDN web docs, this][MDN_this]

`this`は、その関数がバインドされているオブジェクトを指します。

```js
function getName() {
  return this.name;
}

const fooObj = {
  name: 'foo',
  getName: getName
}

console.log(fooObj.getName()); // foo
```

JSの`this`は他の多くの言語と異なり呼び出し時に決定されます。

```js
const barObj = {
  name: 'bar',
  getName: fooObj.getName
}

console.log(barObj.getName()); // bar
```

グローバルスコープの`this`はグローバルオブジェクトと呼ばれるオブジェクトを指します。
ブラウザにおいては`window`オブジェクトがグローバルオブジェクトになります。

```js
console.log(this === window); // true
```

アロー関数の場合は`function`と異なり、定義時のスコープの`this`が関数内の`this`になります。

```js
const returnThis = () => this;

const bazObj = {
  name: 'baz',
  getThis: returnThis
};

// bazObjではなくグローバルオブジェクトが返る
console.log(fooObj.returnThis() == this); // true
```

すべての関数には[call](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Function/call)というメソッドが定義され、第一引数に`this`としてバインドするオブジェクトを渡すことができます。

```js
function addValue(v) {
  return this.value + v;
}

const valueObj = {
  value: 10
}

addValue.call(valueObj, 20); // 30
```

ただしアロー関数の場合は`call`を使っても`this`を変更することはできません。

```js
const returnThis = () => this;

const valueObj = {
  value: 10
}

// valueObjではなくグローバルオブジェクトが返る
console.log(returnThis.call(valueObj) == this); // true
```

より詳しい`this`の説明は[MDN web docs, this][MDN_this]を参照してください。

## 参考

MDN web docs

* [this][MDN_this]
* [オブジェクトのプロトタイプ](https://developer.mozilla.org/ja/docs/Learn/JavaScript/Objects/Object_prototypes)
* [new 演算子](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/new)
* [クラス](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Classes)
* [instanceof](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/instanceof)
* [継承とプロトタイプチェーン](https://developer.mozilla.org/ja/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)

[MDN_this]: https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/this
