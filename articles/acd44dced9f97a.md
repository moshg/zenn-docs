---
title: "Reactでｎ個のコールバックをどうメモ化する問題"
publication_name: "aidemy"
emoji: ""
type: "tech"
topics: ["react", "パフォーマンス"]
published: true
---

## 導入

React でパフォーマンスを意識するとコールバック関数をメモ化することになります。
しかしコールバック関数を渡す対象がテーブルのセルなど動的に生成される要素の場合、途端にメモ化の方法が自明ではなくなります。
`useCallback` を覚えた後、自分はその次の一歩で困ったので、ここで知っている方法を紹介しようと思います。

まずは `useCallback` について軽くおさらいした後、本題に入ります。
`useCallback` ぐらい知っとるわという方は『[問題](#問題)』の節へジャンプしてください。

## React のおさらい

React でパフォーマンスを向上するには値を変化させないことが重要です。
そこで力を発揮するのがメモ化です。
例として、以下のように数値を入力するとその 2 倍を計算して隣に表示する行が 2 つ並んだコンポーネントを考えてみましょう。

@[codepen](https://codepen.io/moshg/pen/ZEMXYxd)

```tsx
import { useState } from "react";

function App(): JSX.Element {
  const [value1, setValue1] = useState(0);
  const [value2, setValue2] = useState(0);

  return (
    <table>
      <thead>
        <tr>
          <th>x</th>
          <th>x * 2</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>
            <input
              type="number"
              value={value1}
              onChange={(e) => setValue1(e.currentTarget.valueAsNumber)}
            />
          </td>
          <td>{value1 * 2}</td>
        </tr>
        <tr>
          <td>
            <input
              type="number"
              value={value2}
              onChange={(e) => setValue2(e.currentTarget.valueAsNumber)}
            />
          </td>
          <td>{value2 * 2}</td>
        </tr>
      </tbody>
    </table>
  );
}
```

1 行目の `input` の値を編集すると、1 行目の計算結果も DOM 上で更新されますが、2 行目の計算結果は値が変わらないため DOM 上で更新されません。
これが React の力です。しかし実は 1 行目を編集したときに 2 行目の `input` も更新されています。
`onChange` にコールバックとして渡しているアロー関数はコンポーネント内で作成されているため、レンダリングの度に新たに作成され、前回のレンダリングと別物になります。(一番簡潔な例を上げて説明すると `() => {} !== () => {}` ということです)
`onChange` に渡したい処理は変化していないのに、レンダリングの度に別のコールバックに変更するという不要なことを行っています。
そこで行うのがメモ化です。以下のようにコードを書き換えることで、DOM の更新を減らすことができます。

```tsx
import { useCallback, useState } from "react";

function App(): JSX.Element {
  const [value1, setValue1] = useState(0);
  const [value2, setValue2] = useState(0);
  const handleChange1 = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) =>
      setValue1(e.currentTarget.valueAsNumber),
    []
  );
  const handleChange2 = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) =>
      setValue2(e.currentTarget.valueAsNumber),
    []
  );

  return (
    <table>
      <thead>
        <tr>
          <th>x</th>
          <th>x * 2</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>
            <input type="number" value={value1} onChange={handleChange1} />
          </td>
          <td>{value1 * 2}</td>
        </tr>
        <tr>
          <td>
            <input type="number" value={value2} onChange={handleChange2} />
          </td>
          <td>{value2 * 2}</td>
        </tr>
      </tbody>
    </table>
  );
}
```

`onChange` に渡すコールバック関数を `useCallback` で包むようにしました。
(それに伴って、見づらくなるのでコールバック関数を JSX の手前で定義するようにしました)
`useCallback` はコールバック関数をメモ化するためのものです。
動作の雰囲気は次のような感じです。(厳密には違うかも)

1. 初回レンダリング時は第一引数の関数をそのまま返す。
2. 二回目以降のレンダリング時には、第二引数 (依存配列) の中身が変化していれば第一引数の関数を返し、変化していなければ前回のレンダリングと同じ関数を返す
   これにより、`onChange` に渡すコールバック関数がレンダリング毎に別物になることがなくなり、更新されていないはずの `input` が DOM 上で更新されることがなくなりました。

## 問題

そしてここからが本題ですが、今の話には 1 つ問題があります。
動的な n 個の行を扱いたい場合にそのまま適用できないということです。
まず n 個の行を扱うとどうなるかメモ化を気にせず書いてみましょう。
(ここでの n 個というのは個数が動的であるという意味です。以下の例では簡単のために 5 個としていますが、この「5」はプロパティで渡される (動的な) 値に容易に書き換えられます)

```tsx
import { useState } from "react";

function App(): JSX.Element {
  const [values, setValues] = useState<number[]>(
    Array.from({ length: 5 }, () => 0)
  );
  const handleChange = (
    e: React.ChangeEvent<HTMLInputElement>,
    index: number
  ) =>
    setValues((values) =>
      values.map((value, i) => {
        if (i !== index) {
          return value;
        }
        return e.target.valueAsNumber;
      })
    );

  return (
    <table>
      <thead>
        <tr>
          <th>x</th>
          <th>x * 2</th>
        </tr>
      </thead>
      <tbody>
        {values.map((value, i) => (
          <tr key={i}>
            <td>
              <input
                type="number"
                value={value}
                onChange={(e) => handleChange(e, i)}
              />
            </td>
            <td>{value * 2}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

@[codepen](https://codepen.io/moshg/pen/KKxXGZw)

`onChange={(e) => handleChange(e, i)}` の部分で渡しているコールバックを `onChange={useCallback((e) => handleChange(e, i), [i])}` のようにメモ化することはできません。
ループ文の中で `useCallback` のような hooks を呼び出すことはできないからです。
これは `useCallback` を知った人全員が衝突する問題なのではないでしょうか。
この問題への対処法をいくつか紹介していきます。

## 行をコンポーネントにする

行をコンポーネントにすることで、`useCallback` を n 回呼び出したいという要求を実現できます。

```tsx
import { useCallback, useState } from "react";

function App(): JSX.Element {
  const [values, setValues] = useState(Array.from({ length: 5 }, () => 0));

  const handleChange = useCallback(
    (value: number, index: number) =>
      setValues((values) =>
        values.map((v, i) => {
          if (i !== index) {
            return v;
          }
          return value;
        })
      ),
    []
  );

  return (
    <table>
      <thead>
        <tr>
          <th>x</th>
          <th>x * 2</th>
        </tr>
      </thead>
      <tbody>
        {values.map((value, i) => (
          <Row key={i} index={i} value={value} onChange={handleChange} />
        ))}
      </tbody>
    </table>
  );
}

function Row(props: {
  index: number;
  value: number;
  onChange: (value: number, index: number) => void;
}): JSX.Element {
  const handleChange = useCallback(
    (e: React.ChangeEvent<HTMLInputElement>) => {
      props.onChange(e.target.valueAsNumber, props.index);
    },
    [props.onChange, props.index]
  );

  return (
    <tr>
      <td>
        <input type="number" value={props.value} onChange={handleChange} />
      </td>
      <td>{props.value * 2}</td>
    </tr>
  );
}
```

`Row` コンポーネント内で `useCallback` を使うことは普通に可能なので、コンポーネントを切り出すことで実質的に `useCallback` を n 回呼び出すことが実現できています。
複雑なコンポーネントを書くときには、スコープを小さくする意味でも有効な方法です。

## データ属性を使う

HTML の要素には[データ属性](https://developer.mozilla.org/ja/docs/Learn/HTML/Howto/Use_data_attributes)という形でデータを持たせることができます。
これにより、インデックスを要素自身に持たせ、コールバック内でインデックスを取得することができます。

```tsx
import { useCallback, useState } from "react";

function App(): JSX.Element {
  const [values, setValues] = useState<number[]>(
    Array.from({ length: 5 }, () => 0)
  );

  const handleChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const indexStr = e.target.dataset.index;
    const index = indexStr ? Number(indexStr) : undefined;

    setValues((values) =>
      values.map((value, i) => {
        if (i !== index) {
          return value;
        }
        return e.target.valueAsNumber;
      })
    );
  }, []);

  return (
    <table>
      <thead>
        <tr>
          <th>x</th>
          <th>x * 2</th>
        </tr>
      </thead>
      <tbody>
        {values.map((value, i) => (
          <tr key={i}>
            <td>
              <input
                data-index={i}
                type="number"
                value={value}
                onChange={handleChange}
              />
            </td>
            <td>{value * 2}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

大本のコールバックに注目すると、インデックスを取得できるのでコールバック関数が一つで済むという点で行をコンポーネントにするパターンと共通しています。
ただし、今回の例では `input` 要素を扱っているのでデータ属性を使用できましたが、カスタムコンポーネントの場合は利用できないのが欠点です。

## フォームライブラリ を使う

最後に`react-hook-form` を使う方法をさっと紹介します。
ライブラリのユースケースからはみ出ない限りこれが一番書きやすいです。

フォームライブラリは他にも React Final Form 等があるのですが、使ったことがないためここでは触れません。
まずコードは以下のようになります。

```tsx
import { useFieldArray, useForm } from "react-hook-form";

function App(): JSX.Element {
  const { register, control, watch } = useForm({
    defaultValues: { values: Array.from({ length: 5 }, () => ({ value: 0 })) },
  });
  const { fields } = useFieldArray({ control, name: "values" });
  const values = watch("values");

  return (
    <table>
      <thead>
        <tr>
          <th>x</th>
          <th>x * 2</th>
        </tr>
      </thead>
      <tbody>
        {fields.map((field, i) => (
          <tr key={field.id}>
            <td>
              <input type="number" {...register(`values.${i}.value`)} />
            </td>
            <td>{values[i].value * 2}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

もはや自分でメモ化する必要もなく、ライブラリに乗っかるだけの素直なコードになっています。
`react-hook-form` uncontrolled 志向なためパフォーマンスもよく、便利です。
普通のフォームを超えたことをやろうとすると途端に面倒になりがちなので使い所には気をつけましょう。

## まとめ

駆け足になってしまいましたが、以上が私の知っているコールバックのメモ化方法です。
`useCallback` を覚えたときにいきなりぶつかった壁なので、対処法をまとめてみました。
私の場合、使い分けは以下のように行っています。

1. 適切ならば `react-hook-form` を使用する
1. パフォーマンス的に問題がなさそうであればメモ化を気にしない
1. 行をコンポーネントにして `useCallback` を使う

他にもよい方法があればコメントで教えてください。
