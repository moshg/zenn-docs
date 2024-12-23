---
title: "TypeScriptの原因不明の型エラーの原因 コールバック編"
publication_name: "aidemy"
emoji: "❓"
type: "tech"
topics: ["typescript", "型システム"]

published: true
---

TypeScriptでなんでこの型チェック通らないんだと思うことありますよね。
この記事ではよく見るエラーの一つであるコールバックでの型エラーについて、なぜその型エラーを通すとまずいのかを解説します。[^excuse]

[^excuse]: TypeScriptのここでドキュメントされている挙動ですという形の説明はありません

## エラー

以下のように紙書籍と電子書籍のデータを題材に考えます。

```ts
// 紙書籍はISBN (International Standard Book Number) をIDとして利用する
type PaperBook = {
  type: "paper";
  isbn: string;
  // ...その他いろんなフィールド
};

// 電子書籍はDOI (Digital Object Identifier) をIDとして利用する
type EBook = {
  type: "ebook";
  doi: string;
  // ...その他いろんなフィールド
};

type Book = PaperBook | EBook;
```

ここで注文に対応して本のリストから本を取り出す処理を考えます。

```ts
type PaperBookId = {
  type: "paper";
  isbn: string;
};

type EBookId = {
  type: "ebook";
  doi: string;
};

type BookId = PaperBookId | EBookId;

/** 注文に対応して本のリストから本を取り出す。 */
const getBookByOrder = (books: Book[], order: { bookId: BookId }): Book | undefined => {
  if (order.bookId.type === 'paper') {
    return books.find((book) => {
      book.type === 'paper' && book.isbn === order.bookId.isbn;// Property 'isbn' does not exist on type 'BookId'.
    });
  } else {
    // 同様の処理
  }
};
```

特に型エラーの起きそうにないコードに見えますが、コールバック内の `order.bookId.isbn` の部分で `Property 'isbn' does not exist on type 'BookId'.` という型エラーが発生します。
VSCodeなどで変数にマウスカーソルを合わせて確認してみると、`if (order.bookId.type === 'paper') {`で型チェックをした直後は`order.bookId`が`PaperBookId`と推論されているのに、コールバックの中では`BookId`と推論されています。
このようにTypeScriptでは謎の型エラーが発生することがあります。

## 原因

コールバック内で`order.bookId`の型を`PaperBookId`と推論するとまずいことは、以下のコールバックを`find`以外の関数に渡す例を考えてみるとわかります。

```ts
const logBookIdAndModifyOrder = (order: { bookId: BookId }) => {
  if (order.bookId.type === 'paper') {
    setTimeout(() => {
      console.log(order.bookId.isbn);
    }, 1000);
  } else {
    // ...
  }

  // ここでbookIdを変更する
  order.bookId = {
    type: 'ebook',
    doi: '1234567890',
  };

  // setTimeoutのコールバック関数はこの辺りのタイミングで呼び出される
};
```

上記の例ではコールバック呼び出し時には`order.bookId`は`EBookId`になっているので、`PaperBookId`と推論しないのが正しいことがわかります。
TypeScriptには渡したコールバックが即時呼び出しされることを表す文法がなく[^type-inference]、`find`と`setTimeout`の区別がつかないため、このように型推論するしかないということになります。

[^type-inference]: 仮にコールバックを即時呼び出しすることを表す文法があったとしても、関数がコールバックとしてしか使われないことをコンパイラが確認する必要があるなど、精密な型推論のためには他の障壁もありますが

## 解決方法

以下のように`order.bookId`を変数に代入すると、bookIdが書き換えられる恐れがなくなり、コールバック内でもbookIdの型が`PaperBookId`と推論されるようになります。[^other-solution]

[^other-solution]: 今回のケースでは`find`の中で分岐を行うことでも解決できますが、本筋とは関係ないので省略

```ts
const getBookByOrder = (books: Book[], order: { bookId: BookId }): Book | undefined => {
  const bookId = order.bookId;
  if (bookId.type === 'paper') {
    return books.find((book) => {
      book.type === 'paper' && book.isbn === bookId.isbn;
    });
  } else {
    // 同様の処理
  }
};
```

他の解決方法としてそもそも`bookId`を引数にするという方法があります。
上記の例では`order`のフィールドを`bookId`だけとしていましたが、実際には他のフィールドもあるはずで、スタンプ結合 ([Wikipedia](https://ja.wikipedia.org/wiki/%E7%B5%90%E5%90%88%E5%BA%A6)) になっています。
`bookId`だけを引数にするとスタンプ結合が解消されるので、可能であればこちらの方が望ましい設計となります。

```ts
const getBookById = (books: Book[], bookId: BookId): Book | undefined => {
  if (bookId.type === 'paper') {
    return books.find((book) => {
      // コンパイルが通る
      book.type === 'paper' && book.isbn === bookId.isbn;
    });
  } else {
    // ...
  }
};
```

## 脇道

上記の`bookId`を引数に取る場合には少し面白いところがあります。
その話の前提として、`bookId`を関数内に`const`, `let`で宣言してみましょう。

```ts
const getBookLet = (books: Book[]): Book | undefined => {
  let bookId: BookId = {
    type: 'paper',
    isbn: '1234567890',
  };
  if (bookId.type === 'paper') {
    return books.find((book) => {
      // コンパイルエラー
      book.type === 'paper' && book.isbn === bookId.isbn;
    });
  } else {
    // ...
  }
};

const getBookConst = (books: Book[]): Book | undefined => {
  const bookId: BookId = {
    type: 'paper',
    isbn: '1234567890',
  };
  if (bookId.type === 'paper') {
    return books.find((book) => {
      // コンパイルが通る
      book.type === 'paper' && book.isbn === bookId.isbn;
    });
  } else {
    // ...
  }
};
```

[原因](#原因)で述べたのと同じ理由で、`let`で宣言した変数は書き換えられる恐れがあるのでコールバック内では`BookId`と推論され、`const`で宣言した変数のみがコールバック内でも`PaperBookId`と推論されています。
ここで引数に`bookId`を取る場合を考えると、引数は書き換え可能なので`let`と同様にコンパイルエラーが発生しそうですが実際には発生しません。
より詳細な挙動は以下のコードで確認できます。

```ts
const getBookById = (
  books: Book[],
  bookId: BookId
): Book | undefined => {
  if (bookId.type === 'paper') {
    return books.find((book) => {
      // コンパイルが通る
      book.type === 'paper' && book.isbn === bookId.isbn;
    });
  } else {
    // 同様の処理
  }
};

const getBookByIdAndModify = (
  books: Book[],
  bookId: BookId
): Book | undefined => {
  if (bookId.type === 'paper') {
    return books.find((book) => {
      // コンパイルエラーが発生する
      book.type === 'paper' && book.isbn === bookId.isbn;
    });
  } else {
    // 同様の処理
  }

  bookId = {
    type: 'ebook',
    doi: '1234567890',
  };
};
```

引数に代入を行うと`let`の場合と同様に振る舞い、引数に代入を行わないと`const`の場合と同様に振る舞っています。
TypeScriptは利便性のためか、引数については代入が実際に行われているかによってスマートにチェックするようです。

## まとめ

TypeScriptでよく見かけるコールバックの型エラーの原因は呼び出しタイミングによる書き換え可能性が原因だということを解説しました。
これでTypeScriptの挙動の謎が一つ解けると幸いです。[^scrap]

[^scrap]: 他にもTypeScriptの挙動で遊んでいるスクラップがあります [TypeScriptの挙動メモ](https://zenn.dev/mosh/scraps/728ff095fae08c)

