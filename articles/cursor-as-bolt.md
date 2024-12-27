---
title: "CursorをBoltやv0のように使う"
publication_name: "aidemy"
emoji: "🤖"
type: "tech"
topics: ["cursor", "bolt", "v0", "ai", "llm"]

published: true
---

## はじめに

世は大生成AI時代。
みなさんも開発に大なり小なりAIを使っていると思います。
その中でも、[Bolt](https://bolt.new/)や[v0](https://v0.dev/)は強烈なツールです。

![Boltのデモ画像](/images/cursor-as-bolt/bolt.png)

このように指示を出すと即座にアプリが作成され、触ってみて意図と違う部分やブラッシュアップしたい部分を指示するとどんどんアプリが完成していきます。
0-1でアプリを作成する場合には非常に強力なツールとなります。

しかし、1日にたくさん出力するにはもちろん課金が必要ですが、普段開発に使用しているCursorに加えてBoltやv0にも課金するとなるとお財布が気になってしまいます。
また、ライブラリの選定など自分で細かい制御を効かせたいときや、AIが把握していないような情報をドキュメントとして与えたいとき、Cursorを使いたくなります。

そこで、CursorをBoltやv0のように使う方法を紹介します。

## CursorをBoltやv0のように使う

CursorのComposerとVSCodeのSimple Browserがこの話のコアになります。
完成イメージは以下のようになります。

![完成イメージ](/images/cursor-as-bolt/tic-tac-toe.png)

この完成イメージまでの手順を紹介します。

### 1. プロジェクトの初期化

最近実装されたComposerのAgent機能を使うとプロジェクトの初期化から任せることができるはずなのですが、あまりスムーズにいかなかったので手動でプロジェクトを初期化します。
（初期化は自分でやったほうがいろいろ便利なので、私はあまりデメリットだと感じていません）

せっかくなので、AIが知らなそうな[TanStack Router](https://tanstack.com/router/latest)を使ってみます。
他のフレームワークを使用する場合は適宜読み替えてください。

```sh
pnpm create @tanstack/router
```

### 2. プロジェクトの起動とブラウザの起動

作成されたプロジェクトを起動してみましょう。

```sh
pnpm dev
```

Boltの使用感を再現するためにはエディタ内でブラウザを起動したいです。
VSCodeにはSimple Browserという組み込みのブラウザがあり、VSCodeベースのCursorからも使用できます。
コマンドパレットからSimple Browserを起動します。
コマンドパレットはWindowsでは`Ctrl + Shift + P`、Macでは`Cmd + Shift + P`で開きます。

![Simple Browserの起動](/images/cursor-as-bolt/launch-simple-browser.png)

URLの入力を求められるので、開発サーバーが起動しているURL `http://localhost:3001` を入力して起動します。

![Simple Browserの表示](/images/cursor-as-bolt/simple-browser.png)

### 3. Composerをエディタとして開く

Composerを大きく開くためにエディタとして開きます。[^open-composer]

[^open-composer]: ComposerはWindowsでは`Ctrl + I`、Macでは`Cmd + I`で開きます。

![Composerをエディタとして開く](/images/cursor-as-bolt/open-composer-as-editor.png)

サイドバーを隠したい場合は、上部のボタンまたは`Ctrl + B`でサイドバーを隠せます。[^frequent-commit]

[^frequent-commit]: Boltのように細かく履歴を残すためには頻繁にコミットする必要があるので、サイドバーは隠さないほうが便利かもしれません。

![サイドバーを隠す](/images/cursor-as-bolt/hide-sidebar.png)

最後に適宜タブをペインに整理してComposerから指示を行うと最初の完成イメージのようになります。

![完成イメージ](/images/cursor-as-bolt/tic-tac-toe.png)

## おわりに

ちょっとケチくさい気がしなくもないですが、CursorをBoltやv0のような見た目で使うことができました。
本家に比べると会話のたびにスナップショットが作成されなかったり、編集の成功率だったり劣っているところはありますが、軽く使う分には気楽に使えていいんじゃないでしょうか！
