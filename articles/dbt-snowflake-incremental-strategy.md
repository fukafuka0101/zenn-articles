---
title: "dbt × Snowflake のインクリメンタルモデル、strategy 3種類を整理してみた"
emoji: "❄️"
type: "tech"
topics: ["dbt", "Snowflake", "datawarehouse", "SQL"]
published: true
---

## はじめに

dbt でインクリメンタルモデルを使い始めたとき、最初は `merge` しか使っていませんでした。でも Snowflake では他にも strategy があって、使い分けると効率が上がるケースがあります。

この記事では、Snowflake で使える3つの strategy を整理します。自分自身の理解整理も兼ねているので、「一つのパターンしか使ったことない」という方の参考になれば嬉しいです。

## インクリメンタルモデルの strategy とは

dbt のインクリメンタルモデルは、テーブル全体を毎回再構築せず差分だけを処理する仕組みです。その「差分をどう適用するか」を決めるのが `incremental_strategy` です。

```sql
{{ config(
    materialized='incremental',
    unique_key='id',
    incremental_strategy='merge'  -- ここで strategy を指定
) }}
```

Snowflake で使える strategy は主に3つあります。

| strategy | 動作のイメージ |
|---|---|
| `merge` | 既存行は UPDATE、新規行は INSERT |
| `delete+insert` | 対象行を削除してから INSERT |
| `insert_overwrite` | パーティションごと丸ごと上書き |

## merge（デフォルト）

```sql
{{ config(
    materialized='incremental',
    unique_key='id',
    incremental_strategy='merge'
) }}
```

Snowflake の `MERGE` 文を使います。`unique_key` で一致した行は UPDATE、なければ INSERT する動きです。

**向いているケース:**

- レコードが後から更新されるデータ（注文ステータス、ユーザー情報など）
- 主キーが明確に決まっている

**注意点:**

大量データに対しては重くなりやすいです。`unique_key` を複数列にする場合はリストで指定します。

```sql
{{ config(
    unique_key=['order_id', 'product_id']
) }}
```

## delete+insert

```sql
{{ config(
    materialized='incremental',
    unique_key='id',
    incremental_strategy='delete+insert'
) }}
```

`unique_key` に一致する行をまず DELETE してから INSERT します。`merge` より単純な分、Snowflake のパーティションプルーニングが効きやすくパフォーマンスが出やすいです。

**向いているケース:**

- データ量が多く `merge` が遅い
- `unique_key` でフィルタしやすいデータ

**注意点:**

DELETE と INSERT の間に一時的にデータが存在しない瞬間があります。通常は問題ないですが、このモデルに対してリアルタイムにクエリが走る場合は注意が必要です。

## insert_overwrite

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    cluster_by=['date_day']
) }}
```

パーティション（クラスタリングキー）単位でごっそり上書きします。`unique_key` は不要です。

**向いているケース:**

- 日次バッチで「この日付のデータをまるごと入れ直す」パターン
- イベントログなど、日次単位で再処理が必要なデータ

**注意点:**

`cluster_by` の設計が必要になります。また、モデル内の WHERE 句で上書き対象のパーティションを絞り込むことが重要です。

```sql
-- モデル内での絞り込み例（直近3日分を対象にする）
{% if is_incremental() %}
WHERE date_day >= dateadd('day', -3, current_date)
{% endif %}
```

## 使い分けまとめ

| ケース | おすすめ strategy |
|---|---|
| レコードが後から更新される | `merge` |
| 主キーがあるが `merge` が遅い | `delete+insert` |
| 日次でパーティションごと入れ直す | `insert_overwrite` |
| 追記のみ（更新なし） | `merge`（`unique_key` なし）でも可 |

## おわりに

最初は `merge` だけ使っていましたが、データの特性によって strategy を変えると、パフォーマンスが改善したり設計がシンプルになったりします。

自分もまだ全パターンを実務で試せているわけではないので、また新しい発見があれば続編で書きます。
