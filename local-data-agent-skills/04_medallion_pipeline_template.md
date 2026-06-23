---
name: local-medallion-pipeline-template
description: Local Bronze/Silver/Gold pipeline template for repeatable data processing from raw files to analytics-ready datasets.
license: Apache-2.0
metadata:
  version: v1
  publisher: local
---

# Local Medallion Pipeline Template

このテンプレートは、ローカルファイルを `raw -> bronze -> silver -> gold` の 3 段階で処理するための設計書です。

## Purpose

AI エージェントに、毎回その場の思いつきで前処理させないために、保存場所・処理順序・成果物を固定します。

## Directory Structure

```text
project-root/
  data/
    raw/        # 元ファイル。絶対に上書きしない
    bronze/     # 抽出直後。最小限の構造化のみ
    silver/     # クレンジング済み。分析に使いやすい形
    gold/       # 集計・可視化・機械学習用
  reports/
    profiling/
    quality_review/
  scripts/
    ingest.py
    transform_silver.py
    build_gold.py
  notebooks/
  tests/
  implementation_plan.md
  walkthrough.md
```

## Layer Definitions

### Raw Layer

元ファイルをそのまま保存する場所。

ルール:

- 上書きしない。
- ファイル名を変える場合はコピーを作る。
- ダウンロード元 URL や取得日が分かる場合は記録する。

### Bronze Layer

抽出直後のデータ。

目的:

- Excel、CSV、PDF などを表形式にする。
- 元の構造をなるべく残す。
- まだ強いクレンジングはしない。

必須メタデータ列:

- `source_file`
- `source_sheet`
- `source_page`
- `source_row_number`
- `ingested_at`

### Silver Layer

クレンジング済みデータ。

目的:

- 列名をそろえる。
- 型をそろえる。
- 欠損値を標準化する。
- 表記揺れを整理する。
- 分析に使いやすい 1 行 1 観測の形にする。

ルール:

- 型変換失敗件数を記録する。
- 削除した行数を記録する。
- 補完した値の件数を記録する。

### Gold Layer

用途別の最終データ。

例:

- 月別売上集計
- 商品別購入回数
- カテゴリ別支出
- 機械学習用特徴量テーブル
- 可視化アプリ用データ

## Script Responsibilities

### `scripts/ingest.py`

担当:

- raw ファイルを読む
- bronze データを作る
- ファイルプロファイルを出す

### `scripts/transform_silver.py`

担当:

- bronze を読む
- 型変換する
- 列名を正規化する
- 欠損値を標準化する
- silver データを作る

### `scripts/build_gold.py`

担当:

- silver を読む
- 用途別に集計する
- gold データを作る

## `implementation_plan.md` Template

```markdown
# Implementation Plan

## Goal

## Input Files
| File | Type | Notes |
| --- | --- | --- |

## Output Files
| Layer | File | Purpose |
| --- | --- | --- |

## Profiling Evidence
| Dataset | Rows | Columns | Main Issues |
| --- | ---: | ---: | --- |

## Transformation Plan
| Step | Input | Output | Description | Risk |
| --- | --- | --- | --- | --- |

## Verification Plan
| Check | Method | Expected Result |
| --- | --- | --- |
```

## `walkthrough.md` Template

```markdown
# Walkthrough

## Summary

## Source Files

## Processing Steps
| Step | Input | Output | What Changed |
| --- | --- | --- | --- |

## Transformations
| Field | Issue Detected | Transformation Applied | Benefit | Risk |
| --- | --- | --- | --- | --- |

## Quality Review
| Check | Result | Notes |
| --- | --- | --- |

## Remaining Risks

## How to Re-run
```bash
python scripts/ingest.py
python scripts/transform_silver.py
python scripts/build_gold.py
pytest
```
```

## Naming Rules

ファイル名は次の形式を推奨する。

```text
<data_name>__bronze.parquet
<data_name>__silver.parquet
<data_name>__gold_<purpose>.parquet
```

例:

```text
receipts__bronze.parquet
receipts__silver.parquet
receipts__gold_monthly_summary.parquet
```

## Definition of Done

- raw, bronze, silver, gold の役割が分かれている。
- 各層の入力と出力が説明できる。
- スクリプトを再実行できる。
- `implementation_plan.md` と `walkthrough.md` がある。
- 品質レビューが残っている。

## 用語

- **メダリオン・アーキテクチャ**: データを bronze、silver、gold の段階に分けて整える設計。
- **raw**: 元ファイルそのもの。
- **bronze**: 抽出直後のデータ。
- **silver**: クレンジング済みデータ。
- **gold**: 集計や機械学習に使う最終データ。
