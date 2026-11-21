---
title: "第2章：13桁法人番号の公式チェックデジット検証をゼロ依存Pythonで実装する"
---

# 第2章：13桁法人番号の公式チェックデジット検証をゼロ依存Pythonで実装する

法人番号を入力として受け取るAPIやMCPツールでは、外部APIへリクエストを飛ばす前に**ローカルで瞬時にチェックデジット検証を行う**ことが極めて重要です。これにより、AIエージェントの桁打ち間違い（ハルシネーション）やユーザー入力ミスを0ミリ秒・API消費ゼロで弾けます。

## 国税庁の公式計算式

13桁の法人番号は、**先頭1桁（チェックデジット）＋ 後ろ12桁（基礎番号）** で構成されています。

1. 後ろ12桁の右端から1始まりで位置 `i = 1..12` を数える。
2. 位置 `i` が奇数なら重み `1`、偶数なら重み `2` を掛けて合計 `total` を求める。
3. `expected = 9 - (total % 9)` を計算し、先頭1桁と一致すれば有効な法人番号。

## 検証済み実装コード

```python
import re

def validate_corporate_number(raw: str) -> dict:
    cleaned = re.sub(r"[\s-]", "", str(raw or ""))
    if not re.fullmatch(r"\d{13}", cleaned):
        return {"input": raw, "normalized": cleaned, "valid": False, "reason": "13 digits required"}

    check_digit = int(cleaned[0])
    total = sum(
        int(ch) * (1 if idx % 2 == 1 else 2)
        for idx, ch in enumerate(reversed(cleaned[1:]), start=1)
    )
    expected = 9 - (total % 9)
    return {
        "input": raw,
        "normalized": cleaned,
        "valid": check_digit == expected,
        "expectedCheckDigit": expected,
        "actualCheckDigit": check_digit,
    }
```

## ユニットテストで必ず押さえる3ケース

- 実在する13桁番号（例：`7000012050002`）が `valid: True` かつ `expectedCheckDigit: 7` になること
- 先頭1桁だけ改変した番号（`6000012050002`）が `valid: False` になること
- ハイフン・スペース混在入力の正規化、および13桁未満の拒否が正しく動くこと

この前段ガードがあるだけで、後続のgBizINFO・EDINET連携の信頼性が劇的に向上します。
