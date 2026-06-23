---
name: receipt-ocr-analysis
description: Receipt OCR cleaning and analysis skill for transforming OCR output into analytics-ready purchase records.
license: Apache-2.0
metadata:
  version: v1
  publisher: local
---

# Receipt OCR Analysis Skill

このスキルは、レシート画像や OCR 結果から、分析に使える購買データを作るための専用手順です。

## When to Use

- レシート画像から OCR したテキストを整理する
- OCR 結果の誤認識を確認する
- 商品名、数量、単価、金額、日付、店舗名を抽出する
- 商品カテゴリを付ける
- 月別、店舗別、カテゴリ別に支出を集計する
- 機械学習用の購買データを作る

## Hard Rules

1. OCR 結果をそのまま信用しない。
2. 元画像、元 OCR テキストは必ず保存する。
3. 商品名の補正は対応表に残す。
4. 金額の合計がレシート合計と合うか確認する。
5. 判断できない行は捨てずに `needs_review` フラグを付ける。
6. 個人情報らしき項目は分析用データから分離する。

## Recommended Data Structure

### Bronze

OCR 直後のデータ。

```text
data/bronze/receipts_ocr_lines__bronze.parquet
```

推奨列:

- `receipt_id`
- `source_image`
- `ocr_engine`
- `line_number`
- `raw_text`
- `confidence`
- `bbox_x1`
- `bbox_y1`
- `bbox_x2`
- `bbox_y2`
- `ingested_at`

### Silver

商品行や合計行を整理したデータ。

```text
data/silver/receipt_items__silver.parquet
```

推奨列:

- `receipt_id`
- `store_name`
- `purchase_date`
- `item_name_raw`
- `item_name_clean`
- `category`
- `quantity`
- `unit_price`
- `line_amount`
- `tax_type`
- `payment_method`
- `needs_review`
- `review_reason`

### Gold

分析用の集計データ。

例:

```text
data/gold/receipt_monthly_category_summary.parquet
data/gold/receipt_store_summary.parquet
data/gold/receipt_item_frequency.parquet
```

## Workflow

### Step 1: OCR Quality Profile

最初に OCR 品質を確認する。

```markdown
| Receipt ID | OCR Engine | Lines | Avg Confidence | Low Confidence Lines | Notes |
| --- | --- | ---: | ---: | ---: | --- |
```

確認すること。

- 信頼度が低い行
- 数字の誤認識
- 店舗名の欠落
- 日付の欠落
- 合計金額の欠落
- 商品行とキャンペーン行の混在

### Step 2: Receipt Header Extraction

抽出対象:

- 店舗名
- 購入日
- 購入時刻
- 電話番号
- 住所
- レシート番号

分析に不要な個人情報や住所は、必要がなければ silver から外し、別ファイルに分離する。

### Step 3: Item Line Extraction

商品行を抽出する。

典型パターン:

```text
商品名  数量  単価  金額
商品名  金額
商品名
      金額
```

注意する行:

- 小計
- 合計
- 税額
- 値引き
- ポイント
- クーポン
- 支払い方法
- 釣銭

判断できない場合は削除せず、`needs_review = true` にする。

### Step 4: Amount Reconciliation

金額整合性を確認する。

```markdown
| Receipt ID | Sum of Item Amounts | Subtotal | Tax | Total | Difference | Judgment |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
```

差額がある場合の候補。

- 税込み / 税抜きの違い
- 値引き行の扱い漏れ
- クーポン行の扱い漏れ
- OCR の数字誤認識
- 商品行の抽出漏れ

### Step 5: Item Name Normalization

商品名を正規化する。

ルール:

- 元の商品名は `item_name_raw` に残す。
- 補正後の商品名は `item_name_clean` に入れる。
- 補正ルールは `reports/quality_review/item_name_mapping.csv` に保存する。
- 不明な商品は無理に分類しない。

例:

```csv
item_name_raw,item_name_clean,reason
コ-ラ,コーラ,OCR hyphen correction
ﾊﾟﾝ,パン,half-width kana normalization
```

### Step 6: Category Assignment

カテゴリ付与は段階的に行う。

1. ルールベースで分類する。
2. 不明なものを `unknown` にする。
3. 必要なら AI に候補を出させる。
4. AI が分類したものは `category_source = ai_suggested` として記録する。

推奨カテゴリ例:

- food
- drink
- daily_goods
- medicine
- clothing
- transport
- entertainment
- other
- unknown

### Step 7: Quality Review

最低限確認する。

- レシートごとの商品行数
- 合計金額との差額
- 日付欠損率
- 店舗名欠損率
- `needs_review` 件数
- カテゴリ `unknown` 件数
- 金額が 0 またはマイナスの行
- 極端に高い金額の行

## Output Reports

```text
reports/profiling/receipt_ocr_quality.md
reports/quality_review/receipt_amount_reconciliation.md
reports/quality_review/item_name_mapping.csv
reports/quality_review/category_assignment_summary.md
reports/quality_review/receipt_remaining_risks.md
```

## Gold Analysis Examples

作成できる分析。

- 月別支出
- 店舗別支出
- カテゴリ別支出
- 商品別購入回数
- 曜日別支出
- OCR エンジン別の誤認識率比較

## Definition of Done

- 元画像と元 OCR 結果が保存されている。
- bronze, silver, gold が分かれている。
- 商品名補正表がある。
- カテゴリ付与ルールがある。
- レシート合計と商品行合計の差額が確認されている。
- `needs_review` 行が分かる。
- 集計用 gold データが作成されている。

## 用語

- **OCR（光学文字認識）**: 画像の中の文字をテキストに変換する技術。
- **bbox（バウンディングボックス）**: OCR で検出した文字や単語の位置情報。
- **信頼度**: OCR がその文字認識にどれだけ自信を持っているかを表す値。
- **正規化**: 表記揺れをそろえること。
- **needs_review**: 人間の確認が必要な行であることを示すフラグ。
