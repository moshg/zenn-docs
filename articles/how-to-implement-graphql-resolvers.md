---
title: "GraphQLリゾルバの実装の仕方"
publication_name: "aidemy"
emoji: "🔗"
type: "tech"
topics: ["graphql"]
published: false
---

GraphQL のリゾルバの考え方がすぐには理解できなかったので TypeScript による実装例とともに解説します！！！

前提知識はぼんやり GraphQL を知っていることです。

GraphQL そのものについても勉強したい人は[勉強に使用したサイト](#勉強に使用したサイト)を載せているので参考にしてください。

実装例とともに解説していきます。

## スキーマ

以下のようなユーザーの名前と身長を管理するスキーマを考えます。

```graphql
type User {
  id: ID!
  name: String!
  height(unit: Unit!): Float!
}

enum Unit {
  METER
  CENTIMETER
}

type Query {
  getUser(id: ID!): User
}
```

`Query`がクライアントから直接呼び出すことのできる操作を表します。そして純粋にスキーマとして見たとき `Query` もそれ以外の型も変わりないというのが GraphQL の特徴です。
実際、身長は取得する際の単位をメートルかセンチメートルか指定できるようになっています。REST と異なり、GraphQL では最上層以外でも引数を取ることができます。

上記のスキーマがあったとき一般的な設定では GraphQL サーバーのエンドポイント `POST /graphql` に対して以下のような body を投げると[^1][^2]

```graphql
{
  getUser(id: "001") {
    id
    height(unit: "CENTIMETER")
  }
}
```

[^1]: 仕様のドキュメント: https://graphql.org/learn/serving-over-http/
[^2]: 必要なフィールドを指定しなければならないという仕様は、単に不要データを削るというだけでなく、再帰的な参照 (例: `User`が`friend`として`User`を参照する)を許容していることによる再帰的な取得の回避という意図があると思われる。

以下のようなレスポンスが返ってきます。

```json
{
  "data": {
    "getUser": {
      "id": "001",
      "height": 160
    }
  }
}
```

このレスポンスの構築を実装するものがリゾルバです。
リゾルバの仕様はどのフレームワークでもほぼ変わりありません。(GraphQL サーバーのリファレンス実装である[GraphQL.js](https://graphql.org/graphql-js/)を踏襲している？)
上記スキーマのリゾルバを実装していきます。

## リゾルバの実装

まず `Query` の `user` フィールドに対応するリゾルバは以下のようになります。

```ts
import { readUser } from "./db";

const Query = {
  getUser: (parent: {}, args: { id: string }) => {
    const usr: {
      userId: string;
      name: string;
      heightCm: number;
    } = readUser(args.id);
    return usr;
  },
};
```

`parent` はグラフの親から渡されてきたデータです。トップレベルの型 `Query` のフィールドである `getUser` には親が存在しないので空オブジェクトになります。
`args` はクライアントから渡される引数で、スキーマの定義通り `id` がオブジェクトとして渡されます。

実は `getUser` の引数は 4 つあり、 `context` と `info` が続きます。
`context`はリクエスト毎に生成したコンテキストを受け取るための引数です。
`info` についてはこの記事では扱いません。

次に `User` のリゾルバを実装します。

```ts
const User = {
  id: (parent: { userId: string; name: string; heightCm: number }) =>
    parent.userId,
  name: (parent: { userId: string; name: string; heightCm: number }) =>
    parent.name,
  height: (
    parent: {
      userId: string;
      name: string;
      heightCm: number;
    },
    args: { unit: "METER" | "CENTIMETER" }
  ) => {
    if (args.unit === "METER") {
      return height * 100;
    } else {
      return height;
    }
  },
};
```

これを見てわかるのが、リゾルバがマッパーのような役割を果たしているということです。
上の階層で取得したデータをスキーマに沿う形に変換しています。

`User`のリゾルバでマッピングを行うので、`User` をフィールドとして持つ型のリゾルバで `User` と同じ形のオブジェクトを返す必要はありません。
実際 `Query`の`getUser`のリゾルバでは `userId` という少し違う名前のフィールド名を持つオブジェクトを返しています。

既にお気づきかもしれませんが、スキーマと同様にリゾルバにおいても `Query` はただの型に過ぎません。
Query は親が存在しないのでリゾルバの parent が必ず空になる[^3]という違いしかありません。
このことは GraphQL に対するメンタルモデルを形成する上で重要な事実であると思われます。

[^3]: 厳密には Query をフィールドに持つ型を定義すると parent が空でなくなるかもしれない (未確認)

リゾルバとはデータ取得とマッパーを合わせたものだと考えてもいいかもしれません。

リゾルバが定義できればあとは簡単にサーバーを実装できます。例えば [GraphQL Yoga](https://the-guild.dev/graphql/yoga-server) であれば以下のようになります。

```ts
import { createServer } from "node:http";
import { createYoga } from "graphql-yoga";

const schema = createSchema({
  typeDefs: `
    type User {
      id: ID!
      name: String!
      height(unit: Unit!): Float!
    }

    enum Unit {
      METER
      CENTIMETER
    }

    type Query {
      getUser(id: ID!): User
    }
  `,
  resolvers: {
    Query,
    User,
  },
});

const yoga = createYoga({ schema });

const server = createServer(yoga);

server.listen(4000, () => {
  console.info("Server is running on http://localhost:4000/graphql");
});
```

## Query 以外でのデータ取得

上記と同じスキーマを題材に他の実装例を紹介します。

```graphql
type User {
  id: ID!
  name: String!
  height(unit: Unit!): Float!
}

enum Unit {
  METER
  CENTIMETER
}

type Query {
  getUser(id: ID!): User
}
```

前の実装例では `User` のリゾルバは単にマッピングするだけでしたが、他の方針も考えられます。
例えば `User` の身長が DB 上ではユーザー情報とは別のテーブルに分かれている場合を考えましょう。
その場合、以下のように身長の取得を遅延するというパターンも考えられます。

```ts
import { readUser, readHeightCm } from "./db";

const getUser = (parent: {}, args: { id: string }) => {
  // 前の例とは `readUser` の返り値の型が変わっていることに注意
  const usr: {
    userId: string;
    name: string;
  } = readUser(args.id);
  return usr;
};

const User = {
  id: (parent: { userId: string; name: string }) => parent.userId,
  name: (parent: { userId: string; name: string }) => parent.name,
  height: (
    parent: {
      userId: string;
      name: string;
    },
    args: { unit: "METER" | "CENTIMETER" }
  ) => {
    // ここが変更点
    const height = readHeightCm(parent.userId);

    if (args.unit === "METER") {
      return height * 100;
    } else {
      return height;
    }
  },
};
```

最初に紹介した例と異なり、`height`はクエリで実際に指定されたときのみ取得するように変更しました。
ここでは`height`はただの数値なので一括して取得しても大差ないですが、大きなオブジェクトであればこのように必要なときのみ取得することでパフォーマンスを向上させることができます。
また、このように必要なデータのみの取得が行われるように気をつけることで、クエリに特化した型を用意しなくてもパフォーマンスを損ねず開発をすることができるようになります。
例えば、REST ではユーザーの一括取得用にはデータを絞ったユーザー型、ユーザーの個別取得には多くのフィールドを持つ別のユーザー型を返すというような工夫が必要ですが、GraphQL ならどちらも同じユーザー型を返すようにし、一括取得では取得するフィールドを絞るという形にすることができます。[^4]

[^4]: クライアントで無駄にデータを取得するクエリを叩かないように気をつける、または persisted query を使用して変なクエリを叩けなくする必要はありますが

注意点として、子オブジェクト内で何もせずデータ取得してしまうと、親オブジェクトが配列だった場合に N+1 問題が発生します。
これを回避するには『[GraphQL で N+1 問題を解決する 4 つのアプローチ]』を参照してください。

## まとめ

リゾルバについて解説している記事があまりなかったので書きました。

上記では雰囲気で型を付けていましたが、リゾルバを実装する際には GraphQL Code Generator を使用すると型を自動生成することでスキーマと異なる型付けをしてしまうおそれがなくなります。そのうち GraphQL Code Generator についても解説記事を書こうと思います。

それでは、楽しい GraphQL サーバー実装をお楽しみください。

## 勉強に使用したサイト

- [GraphQL の公式ドキュメント](https://graphql.org/learn/schema/)
- [GraphQL で N+1 問題を解決する 4 つのアプローチ]

[GraphQL で N+1 問題を解決する 4 つのアプローチ]: https://zenn.dev/alea12/articles/15d73282c3aacc
