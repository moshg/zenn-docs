---
title: "関係の制約をSQLで表現する方法を「ひらめく」のはやめます！"
publication_name: "aidemy"
emoji: "1️⃣"
type: "tech"
topics: ["rdb", "sql"]
published: true
---

:::message
この記事は[Aidemy Advent Calendar 2023](https://qiita.com/advent-calendar/2023/aidemy) 11日目の記事となります。
10日目は[マルチテナント構成のTerraform戦略](https://zenn.dev/aidemy/articles/eaed2ad54b55ca)でした！
:::

こんにちは、moshです。
[株式会社アイデミー](https://aidemy.co.jp/)でLab Bankという化学業界の研究室向けSaaSを開発しています。

https://labbank.jp/

## はじめに

多対多の関係を表現する際には中間テーブルを使いましょうといったことはデータベースの本を読めば自然と身につきますが、一対一の関係をどう表現するかといったことはあまり本には書かれていません。
そのためSQLで制約を表現する方法を設計中に「ひらめく」ことになってしまっていました。
そこでこの記事の中で整理しようと思います。主に自分のために！

## 一対多

まずはSQLで表現するのが最も簡単な1:nの関係をSQLで表現する方法から始めましょう。
例として「部署↔社員」を考えましょう。
ある部署に所属する社員は複数います。ある社員が所属する部署は一つだけです。[^1]
よってこれは一対多の関係になっています。

[^1]: どの会社にも当てはまるわけではないかもしれませんが、そういう会社を考えることにします

これは外部キーで表現できます。

部署テーブル

|id|name|
|--|----|
|001|法務部|
|002|人事部|

社員テーブル[^2]

[^2]: 視認性のために3桁にしていますが、3桁だと社員1000人で桁が溢れるので問題があります

|id|department_id|name|
|--|-------------|----|
|001|001|佐藤陽葵|
|002|001|鈴木碧|
|003|002|高橋凛|

SQLで表現すると以下のようになります。

```sql
CREATE TABLE departments (
    id VARCHAR(31) PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE employees (
    id VARCHAR(31) PRIMARY KEY,
    department_id VARCHAR(31) NOT NULL,
    name TEXT NOT NULL,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

`FOREIGN KEY (department_id) REFERENCES departments(id)` の部分で外部キー制約をかけているため、社員に紐づく部署が存在することが保証されます。

必ずしもすべての社員が部署に所属するわけではない場合、つまり1対多ではなく0…1対多の関係になる場合は、`department_id`をNULL許容にすることで表現できます。

### 一対多を表現する他の方法

他にも少し複雑になりますが、『ほんまに一対多でええんか？(https://zenn.dev/praha/articles/65afb28caacd0b)』で紹介されるように、中間テーブルを作るという方法もあります。
多対多に移行しやすいことと、テーブル同士が疎結合になるというメリットがあります。
この場合、DBの制約としては0…1対多の関係になります。

## 多対多

「ユーザー↔グループ」を例として考えましょう。
あるグループに所属するユーザーは複数います。あるユーザーは複数のグループに所属することができます。
よってこれは多対多の関係になっています。
これは以下のように中間テーブルを作ることで表現できます。

ユーザーテーブル

|id|name|
|--|----|
|001|佐藤陽葵|
|002|鈴木碧|

グループテーブル

|id|name|
|--|----|
|001|猫好きの会|
|002|最強†ギルド|

ユーザーグループテーブル (中間テーブル)

|user_id|group_id|
|-------|--------|
|001|001|
|001|002|
|002|001|

SQLで書くと以下のようになります。

```sql
CREATE TABLE users (
    id VARCHAR(31) PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE user_groups (
    id VARCHAR(31) PRIMARY KEY,
    title TEXT NOT NULL
);

CREATE TABLE user_user_group_relations (
    user_id VARCHAR(31) NOT NULL,
    user_group_id VARCHAR(31) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (user_group_id) REFERENCES user_groups(id)
);
```

## 一対一

これはエンティティの一部の属性を別のエンティティに分離したい場合に現れるパターンです。
ひとつのテーブルが大きくなりすぎることによる、クエリのパフォーマンスに悪くなったり、把握しづらさへの対応として現れます。

例として、ユーザーと身体データを考えます。

ユーザーテーブル

|id|name|body_measurement_id|
|--|----|------------------|
|001|佐藤陽葵|001|
|002|鈴木碧|002|

身体データテーブル
|id|height_cm|weight_kg|
|--|------|------|
|001|160|50|
|002|170|60|

```sql
CREATE TABLE users (
    id VARCHAR(31) PRIMARY KEY,
    name TEXT,
    body_measurement_id VARCHAR(31) NOT NULL UNIQUE,
    FOREIGN KEY (body_measurement_id) REFERENCES body_measurements(id)
);

CREATE TABLE body_measurements (
    id VARCHAR(31) PRIMARY KEY,
    height INTEGER,
    weight INTEGER
);
```

ここで注意すべきことは、この対応が必ずしも一対一の関係になるとは限らないということです。
ユーザーに対応する身体データは必ず一つ存在しますが、身体データに対応するユーザーは存在しない場合があります。
つまり、「ユーザー↔身体データ」は「0…1:1」になってしまっているということになります。
このようにSQLは一対一の関係を表現することができません。（私が知らないだけかもしれないのでご存知でしたらコメントください）
これに対する対応をいくつか挙げます。

### 諦める

そもそも一対一の関係をDBで保証することを諦めるというのも一つの手です。

ユーザーに紐づかない身体データを作れてしまいますが、ユーザーの身体データを取り出すことはあっても、身体データからユーザーを逆引きすることはないので、問題になることはないと割り切ってしまうこともできるでしょう。

### 実は一対一ではない

巨大テーブルの管理のために分割しているとき、そもそも身体データのような分割した属性はオプショナルなことも多いと思います。
その場合は「身体データ↔ユーザー」が「0…1対1」なので、身体データがユーザーを参照する外部キーを持つようにするとうまく表現できます。
また、身体データを入力するときは身長体重を両方入力しなければならないといった制約も表現できるので、ユーザーが身長と体重をオプショナルな属性として持つよりも表現力が高いです。

この場合は、[一対多を表現する他の方法](#一対多を表現する他の方法)と同様のバリエーションがあります。

### 一つのテーブルに収める

確実に一対一の関係を表現したい場合は、以下のように一つのテーブルに収めるしかありません。

|id|name|height_cm|weight_kg|
|--|----|---------|---------|
|001|佐藤陽葵|160|50|
|002|鈴木碧|170|60|

この場合もアプリケーションからは属性を分割して管理することができます。
例えばGoのORMであるgormでは`embeddedPrefix`というタグをサポートしています。 ([参照](https://gorm.io/ja_JP/docs/models.html#%E6%A7%8B%E9%80%A0%E4%BD%93%E3%81%AE%E5%9F%8B%E3%82%81%E8%BE%BC%E3%81%BF))

# まとめ

ここまでで、紹介した内容をまとめると以下のようなマトリックスになります。

|-|0…1|1|0…n|
|-|---|-|---|
|0…1|[一対一][not-1-by-1]|[一対一][not-1-by-1]|[一対多][1:n]|
|1  |-|[一対一][single-table]|[一対多][1:n]|
|0…n|-|-|[多対多][m:n]|

これでもう迷いません！

[1:1]: #一対一
[single-table]: #一つのテーブルに収める
[not-1-by-1]: #実は一対一ではない
[1:n]: #一対多
[m:n]: #多対多
