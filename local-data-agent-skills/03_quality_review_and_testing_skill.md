---
name: local-quality-review-and-testing
description: Mandatory quality review and regression testing workflow for local data cleaning pipelines.
license: Apache-2.0
metadata:
  version: v1
  publisher: local
---

# Local Quality Review and Testing Skill

このスキルは、AI が作った前処理コードをそのまま信用せず、変換前後のデータ品質を確認するための手順です。

## When to Use

- データクレンジング後
- 型変換後
- 重複削除後
- 欠損値処理後
- OCR 結果の補正後
- 機械学習用データセット作成後

## Hard Rules

1. 変換後データだけを見て完了にしない。必ず変換前と比較する。
2. 行数が変わった場合は理由を説明する。
3. 欠損率が増えた場合は理由を説明する。
4. 型変換失敗件数を隠さない。
5. 重複削除は削除前後の件数を記録する。
6. テストが失敗した状態で完了にしない。

## Required Checks

### 1. Row Count Check

```markdown
| Dataset | Row Count |
| --- | ---: |
| Before |  |
| After |  |
| Difference |  |
```

行数が減った場合、以下を記録する。

- フィルタ条件
- 削除対象の件数
- 削除した理由
- 削除前サンプル

### 2. Column Check

```markdown
| Column | Status | Notes |
| --- | --- | --- |
| source_column | kept / renamed / dropped / created |  |
```

列を削除した場合は理由を必ず書く。

### 3. Null Rate Check

```markdown
| Column | Null Rate Before | Null Rate After | Difference | Judgment |
| --- | ---: | ---: | ---: | --- |
```

欠損率が 1%以上増えた列は、原因を調べる。

### 4. Type Conversion Check

```markdown
| Column | Target Type | Success Count | Failure Count | Failure Samples |
| --- | --- | ---: | ---: | --- |
```

変換失敗サンプルは最低 5 件確認する。

### 5. Numeric Range Check

```markdown
| Column | Min Before | Min After | Max Before | Max After | Notes |
| --- | ---: | ---: | ---: | ---: | --- |
```

異常な値の例。

- 金額がマイナス
- 数量が 0 または極端に大きい
- 年齢が 150 以上
- 日付が未来すぎる、または古すぎる

### 6. Duplicate Check

```markdown
| Duplicate Type | Key | Count Before | Count After | Action |
| --- | --- | ---: | ---: | --- |
```

完全一致重複とキー重複を分ける。

### 7. Before / After Sample Check

最低 10 行のサンプルを比較する。

```markdown
| Source Row | Before | After | Judgment |
| ---: | --- | --- | --- |
```

### 8. Business Rule Check

データの意味に関するルールを確認する。

例:

- `total_amount = unit_price * quantity` に近いか
- 購入日は未来日でないか
- 税込金額が税抜金額より小さくないか
- カテゴリが定義済み一覧に含まれるか

## Recommended Test Files

AI がスクリプトを作る場合、可能なら以下を作る。

```text
tests/test_schema.py
tests/test_quality.py
tests/test_transformations.py
```

### pytest Example

```python
import pandas as pd


def test_required_columns_exist():
    df = pd.read_parquet("data/silver/cleaned.parquet")
    required = {"source_file", "source_row_number"}
    assert required.issubset(df.columns)


def test_no_empty_column_names():
    df = pd.read_parquet("data/silver/cleaned.parquet")
    assert all(str(c).strip() for c in df.columns)
```

## Quality Review Report Template

`reports/quality_review/quality_review.md` に以下を残す。

```markdown
# Quality Review Report

## Summary
- Source:
- Output:
- Review date:
- Result: pass / warning / fail

## Checks
| Check | Result | Notes |
| --- | --- | --- |
| Row count |  |  |
| Column mapping |  |  |
| Null rate |  |  |
| Type conversion |  |  |
| Numeric range |  |  |
| Duplicate |  |  |
| Before / after sample |  |  |
| Business rules |  |  |

## Remaining Risks
| Risk | Impact | Next Action |
| --- | --- | --- |
```

## Definition of Done

- 品質レビューレポートがある。
- 行数、列数、欠損率、型変換、重複の確認が終わっている。
- 失敗サンプルが確認されている。
- 重要な警告が `Remaining Risks` に残っている。
- 再実行できるテストがある。

## 用語

- **回帰テスト**: 修正によって以前できていたことが壊れていないか確認するテスト。
- **ビジネスルール**: データの意味に基づくルール。例: 合計金額は単価×数量に近い、など。
- **pytest（パイテスト）**: Python のテストを自動実行するためのライブラリ。
