---
title: "ZennのGitHub連携テンプレート作った"
emoji: "📚"
type: "tech"
topics: ["zenn"]
published: false
---

Zenn は GitHub で記事を管理することができます。
そのためのテンプレートを作成しました：https://github.com/moshg/zenn-docs-template
既にWebで管理している記事をGitHub管理に移行する方法も紹介します。

## 前提

GitHub 連携には[Zenn CLI](https://zenn.dev/zenn/articles/install-zenn-cli)を使用します。
Zenn CLI の前提は以下のようになっています。

> Zenn CLI を使うには Node.js 14 以上が必要です。Node.js をはじめて使う場合は[インストール](https://nodejs.org/ja)する必要があります。

## Zenn の記事を GitHub で管理するまで

まず[テンプレート](https://github.com/moshg/zenn-docs-template)の「Create a new repository」からテンプレートをコピーしてリポジトリを作成します。
上記方法でリポジトリを作成すると「generated from moshg/zenn-docs-template」と表示されてしまうので、気になるなら適宜いい感じに使用してください。

作成したリポジトリを Zenn 公式の『[GitHub リポジトリで Zenn のコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github#github%E3%81%A8%E3%81%AE%E9%80%A3%E6%90%BA%E6%89%8B%E9%A0%86)』に従って連携します。

これで GitHub 連携ができました。

## 記事の作成

初めに `npm install` で依存 (Zenn CLI) をインストールしてください。

テンプレートでは Zenn CLI のコマンドを npm scripts として登録しています。
以下のコマンドで記事を作成できます。

```sh
npm run article
```

このコマンドで`articles` フォルダに記事の雛形が作成されます。
この作成されたファイルを編集して push することで記事を作成することができます。

生成されたファイルにはメタデータが以下のようなYAML front matterとして設定されています。

```
---
title: ""
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
```

このpublishedをtrueにすることで記事を公開することができます。

上記のように生成するのではなく、手動で記事を作成することもできます。
このときファイル名 ([slug](https://zenn.dev/zenn/articles/what-is-slug)) は自由に付けることができて、ファイル名はURLに反映されます。
ファイル名を既に公開している記事のURLに合わせることで、既に公開している記事もGitHubで管理することができます。

books についても同様です。

以下のコマンドでプレビューを起動できます。

```sh
npm run preview
```

`http://localhost:8000`でプレビューを閲覧することができます。

詳しくは『[Zenn CLI で記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)』を参照してください。

## テンプレートの説明

『Zenn CLI をインストールする』の手順通りに GitHub 連携リポジトリを作成した場合との違いは以下です。

- 公式ドキュメントでは `npm init` で `package.json` を初期化しているが、その方法で初期化した場合から不要な項目を削っています
- Zenn CLI を npm scripts に登録しています

## 参考

- [GitHub リポジトリで Zenn のコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github)
- [Zenn CLI をインストールする](https://zenn.dev/zenn/articles/install-zenn-cli)
- [Zenn CLI で記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)
