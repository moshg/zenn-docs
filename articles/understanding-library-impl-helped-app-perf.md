---
title: "アプリのパフォーマンス改善でライブラリの実装の理解が役立った話"
publication_name: "aidemy"
emoji: "🚀"
type: "tech"
topics: ["frontend", "javascript", "typescript", "performance", "react"]

published: false
---

## はじめに

技術で開発上の問題を突破できると嬉しいですよね。
しかもバックエンドで学んだことがフロントエンドで活用できると面白いですよね。
そこでGraphQLサーバーの開発でDataLoaderの実装を勉強したことが、フロントエンドのパフォーマンス改善で役に立った話を紹介します。[^dataloader-out-of-graphql]
（DataLoaderを知らなくても読めるようにこの記事は書いています！！！）

[^dataloader-out-of-graphql]: DataLoaderはGraphQLに限らず有用ですが: [Next.jsの考え方 - N+1とDataLoader](https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_1_data_loader)

# パフォーマンス問題

ReactでExcelのようなテーブルを持ったアプリを開発していました。
アプリには変更のあった行の色が変わる機能がありました。

画面の雰囲気:

![アプリの画面](/images/understanding-library-impl-helped-app-perf/app.webp =480x)

ある日、データ件数が大きいときにペーストを行うと非常に時間がかかるという問題が発覚しました。
1000件でコピペしたときの様子:

![コピペが長引く様子](/images/understanding-library-impl-helped-app-perf/long.webp =240x)

原因をChromeの開発者ツールで調査したところ、ペースト時にセルごとに呼び出されている`onChange`の処理が重すぎることがわかりました。

コードは抜粋すると概ね以下のようになっていました。

```ts
/**
 * データの差分がある行を検出して色を変更する。
 */
const onChange = (column: number, row: number, value: string) {
  // ...いろいろな処理

  // 全体の差分を計算してセルの色に反映する
  calculateDiffAndUpdateColor();
}
```

`onChange`はセルの更新ごとに呼び出されるコールバックです。
全体の差分を計算してセルの色に反映するという重い処理がペーストされたセルの数だけ (上記のアニメーションだと80回) 実行されるので、それは重いはずです。

## パフォーマンス改善

第一に検討すべきは、この重い`calculateDiffAndUpdateColor`をセルごとの変更で呼び出される`onChange`ではなく、セルの入力やペーストをフックに一度だけ呼び出されるコールバックに移動して処理の回数を減らすことです。[^onafterchanges]
もう一つ検討すべきは、変更された部分が影響する行のみを再計算して一回の処理を軽くすることです。
しかしこれら両方とも、諸般の事情でできませんでした。

[^onafterchanges]: [jspreadsheet](https://bossanova.uk/jspreadsheet/v4/)というライブラリを使用していたので、[onafterchanges](https://bossanova.uk/jspreadsheet/v4/docs/events)がそれに当たります

そこでふとGraphQLでよく利用されるDataLoaderのことを思い出しました。
DataLoaderは処理を溜め込んで一括実行することで、細かく呼び出したいがバッチで行った方がパフォーマンス上有利な処理を便利に扱うためのライブラリです。
DataLoaderからアイデアを拝借して、処理をバッチで実行してパフォーマンス改善を行うと今回のケースに対処できそうです。[^dataloader]

[^dataloader]: DataLoaderの実装の参考: [N+1問題を解決するDataLoaderの仕組みとサンプル実装](https://dev.classmethod.jp/articles/graphql-dataloader-sample/)

### queueMicrotask

改善では[queueMicrotask](https://developer.mozilla.org/ja/docs/Web/API/Window/queueMicrotask)を利用したので軽く説明しておきます。[^not-queueMicrotask]

[^not-queueMicrotask]: DataLoaderで使用されているのは`queueMicrotask`ではなく`process.nextTick`ですが: https://github.com/graphql/dataloader/issues/348

queueMicrotaskはめちゃくちゃ雑に言ってしまうと、現在実行中のコールバックが終了したタイミングで実行する処理を登録する関数です。

例:

```ts
queueMicrotask(() => console.log('execute'));
console.log("queued");
// 出力:
// queued
// execute
```

アプリケーションではタイミングズレのHACKな対応で使うのが最もオーソドックスな使い方ではないかと思います。（よくない）[^queueMicrotask-hack]

[^queueMicrotask-hack]: `onchange`はペースト時に実際にセルが書き換わるより少し早く実行されます。そのため、元々`calculateDiffAndUpdateColor`ではタイミング問題へのHACKな対処として`queueMicrotask`が使用されています😇

### コード

Reactを使用していたので、`queueMicrotask`を利用して処理をバッチ化したコードは以下のようになりました。

```ts
/**
 * シート全体の差分計算をマイクロタスクキューに追加する。
 * タスクの実行までにこの関数が複数回呼び出された場合、タスクは1度だけ実行される。
 *
 * セルの変更ごとに実行するとペースト時にパフォーマンスが悪いため、まとめて実行する。
 */
const queueCalculateDiffAndUpdateColor = useMemo(() => {
  let executed = true;
  return () => {
    executed = false;
    queueMicrotask(() => {
      if (executed) {
        return;
      }
      executed = true;
      calculateDiffAndUpdateColor();
    });
  };
}, [calculateDiffAndUpdateColor]);

const onChange = (column: number, row: number, value: string) {
  // ...いろいろな処理

  // 全体の差分を計算してセルの色に反映する
  queueCalculateDiffAndUpdateColor();
}
```

変更が起きたとき`queueMicrotask`で差分計算をキューし、まとめて一度だけ実行するという戦略です。
例えば、ペーストによって(0, 0), (0, 1), (0, 2)の3つのセルが同時に変更されたとき、以下のような順序で処理が実行されます。

```
onChange(0, 0, "a"); // executed = false を代入して、処理をキューに追加
onChange(0, 1, "b"); // executed = false を代入して、処理をキューに追加
onChange(0, 2, "c"); // executed = false を代入して、処理をキューに追加
// 連続するonChangeの呼び出しが完了し、キューに追加された処理が実行される
(0, 0)のコールバック実行 // 差分計算が行われ、executed = true になる
(0, 1)のコールバック実行 // executed = true なので、差分計算は行われない
(0, 2)のコールバック実行 // executed = true なので、差分計算は行われない
```

この変更によってペースト時のパフォーマンスが改善されました。

![ペースト時のパフォーマンスが改善された様子](/images/understanding-library-impl-helped-app-perf/optimized.webp =240x)

:::details 今回のコード全体

簡易的な確認のため、Reactではなく素のHTMLで書いています。

```html
<html>
<script src="https://bossanova.uk/jspreadsheet/v4/jexcel.js"></script>
<link rel="stylesheet" href="https://bossanova.uk/jspreadsheet/v4/jexcel.css" type="text/css" />
 
<script src="https://jsuites.net/v4/jsuites.js"></script>
<link rel="stylesheet" href="https://jsuites.net/v4/jsuites.css" type="text/css" />
 
<div id="spreadsheet"></div>
 
<script>
const columns = [
  {
    type: 'numeric',
    title: 'col1',
    width: 80,
  },
  {
    type: 'numeric',
    title: 'col2',
    width: 80,
  },
  {
    type: 'numeric',
    title: 'col3',
    width: 80,
  },
  {
    type: 'numeric',
    title: 'col4',
    width: 80,
  }
]

const originalData = Array.from({ length: 1000 }, (_, i) => Array.from({ length: 4 }, (_, j) => i * 4 + j + 1));
const data = originalData.map(row => row.slice());

/**
 * データの差分がある行を検出して色を変更する。
 *
 * @param {HTMLDivElement} instance 
 * @param {HTMLTableCellElement} cell 
 * @param {string} column 
 * @param {string} row 
 * @param {string} value 
 */
// const onchange = (instance, cell, column, row, value) => {
//   queueMicrotask(() => {
//     data.forEach((row, rowIndex) => {
//       const hasChanges = row.some((cell, colIndex) => cell != originalData[rowIndex][colIndex]);
//       const color = hasChanges ? '#FFE4C4' : '';
//       const cells = document.querySelectorAll(`td[data-y="${rowIndex}"]`);
//       cells.forEach(cell => {
//         cell.style.backgroundColor = color;
//       });
//     });
//   });
// }

let executed = true;
const onchange = (instance, cell, column, row, value) => {
  executed = false;
  queueMicrotask(() => {
    if (executed) {
      return;
    }
    executed = true;
    data.forEach((row, rowIndex) => {
      const hasChanges = row.some((cell, colIndex) => cell != originalData[rowIndex][colIndex]);
      const color = hasChanges ? '#FFE4C4' : '';
      const cells = document.querySelectorAll(`td[data-y="${rowIndex}"]`);
      cells.forEach(cell => {
        cell.style.backgroundColor = color;
      });
    });
  });
}

jspreadsheet(document.getElementById('spreadsheet'), {
  data:data,
  columns: columns,
  onchange: onchange
});
</script>
</html>
```

:::

## まとめ

このようにまとめて実行したほうがいい処理では、`queueMicrotask`を使用してバッチで実行することでパフォーマンス改善でき（ることがあり）ます。

今回の件ではバックエンド開発でDataLoaderを掘り下げたことが役に立ちました。
実は重い箇所の調査でISUCON[^isucon]の経験も役立っています。
これらの経験がなければ、今回の改善はかなり手間取っていたはずです。
深堀った経験が必要になる機会は確実にあり、学んだことは意外に広く助けてくれるので、興味があることはどんどん学んでいきましょう！

[^isucon]: ISUCONはWebサーバーをチューニングするコンテストです（[公式サイト](https://isucon.net/)）
