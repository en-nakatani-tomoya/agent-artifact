# bigquery cost guard for wildcard tables

## 概要
BigQueryのワイルドカードテーブル(events_*)クエリ時の大量データ取得を防ぐコスト管理手法をまとめた

## 詳細
## 1. クエリレベルのガード

### maximum_bytes_billed の設定（最も手軽）
クエリ単位でスキャン量を制限できる。制限を超えるクエリは実行前にエラーになり課金されない。

```sql
SELECT *
FROM \`project.dataset.events_*\`
WHERE _TABLE_SUFFIX BETWEEN '20260201' AND '20260224'
OPTION (maximum_bytes_billed = 10737418240);  -- 10GB
```

## 2. プロジェクト/ユーザーレベルのガード

### カスタムコスト管理（Admin Console）
- プロジェクト単位: 1日あたりのクエリ使用量(TB)の上限を設定
- ユーザー単位: 特定ユーザーの1日あたりのクエリ使用量を制限

設定場所: BigQuery → 管理 → カスタムコスト管理

## 3. クエリ時のベストプラクティス
- 必ず _TABLE_SUFFIX で期間を絞る
- SELECT * を避け必要なカラムだけ指定する

```sql
SELECT event_name, event_timestamp, user_pseudo_id
FROM \`project.dataset.events_*\`
WHERE _TABLE_SUFFIX = '20260224'
```

## 4. 組織レベルのポリシー
BigQuery の予約（Reservations）やスロット制限でコストを固定化できる（設定はやや重め）。

## おすすめの組み合わせ
_TABLE_SUFFIXでの期間指定を必須ルール化 + カスタムコスト管理でユーザー/プロジェクト単位の日次上限設定

## 参考
- https://cloud.google.com/bigquery/docs/custom-quotas
- https://cloud.google.com/bigquery/docs/best-practices-costs

---
Generated: 2026-02-24
