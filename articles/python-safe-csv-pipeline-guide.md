---
title: "Python標準ライブラリだけで作る、壊れない日本業務CSV集計パイプラインの設計"
emoji: "🔧"
type: "tech"
topics: ["python", "csv", "automation", "dataengineering"]
published: true
---

日本の実務で扱う業務CSV（EC受注データ、基幹システム出力、Excel保存ファイルなど）は、単純な `csv.reader` や `float()` で書くと**本番投入から数日で必ず壊れます**。

この記事では、外部ライブラリを一切使わず **Python標準ライブラリだけ** で「文字コード混在・金額記号・重複行・異常行」を安全に処理し、集計レポートと異常リストを分離出力する実戦パターンを、テスト検証済みのコードとともに解説します。

実装全体は公開リポジトリにも置いています。
- サンプル実装：[inemurisandesu/portfolio-automation-kit](https://github.com/inemurisandesu/portfolio-automation-kit)

## 実務CSVで必ず踏む4つの地雷

1. **UTF-8（BOM付き）と Shift-JIS（CP932）の混在**
   - 同じ取引先でも、Excelで一度開いて保存し直したファイルだけ `cp932` に変わっていることが頻繁に起きます。
   - 特に `utf-8` ではなく `utf-8-sig` を先に試さないと、先頭カラム名に `\ufefforder_id` のようなBOMが混入して `KeyError` になります。
2. **全角 `￥`（`\uffe5`）と半角 `¥`（`\xa5`）、カンマ、`円` の混在**
   - `cp932` 環境では半角 `¥` ではなく全角 `￥` が出力されるケースがあり、片方しか除去していないと変換に失敗します。
3. **`float` による金額誤差**
   - 金額計算に `float` を使うと微小な丸め誤差が発生するため、金額は必ず `decimal.Decimal` を使い、`is_finite()` で `NaN` / `Infinity` を弾きます。
4. **1行の不正データでバッチ全体が落ちる問題**
   - 不正な日付やマイナス金額が1行混ざっただけで例外停止させると、運用が止まります。正常行と異常行を分離し、`anomalies.csv` として監査可能に出力する設計が実務では圧倒的に強いです。

## 実装のコアパターン

### 1. 複数文字コードの安全な読み込み

```python
ENCODINGS_TO_TRY = ("utf-8-sig", "cp932", "utf-8")

def read_csv_rows(path: Path) -> tuple[list[dict[str, str]], str]:
    last_error: Exception | None = None
    for enc in ENCODINGS_TO_TRY:
        try:
            with path.open("r", encoding=enc, newline="") as fh:
                reader = csv.DictReader(fh)
                if reader.fieldnames is None:
                    return [], enc
                return [dict(row) for row in reader], enc
        except UnicodeDecodeError as exc:
            last_error = exc
    raise ValueError(f"Failed to decode {path.name}: {last_error}")
```

`utf-8-sig` → `cp932` → `utf-8` の順で試すことで、BOM付きUTF-8とWindows Excel由来のShift-JISの両方を1関数で吸収できます。

### 2. 金額パースと異常値の隔離

```python
from decimal import Decimal, InvalidOperation

def parse_amount(raw_value: str) -> Decimal:
    cleaned = (
        raw_value.strip()
        .replace(",", "")
        .replace("¥", "")
        .replace("￥", "")
        .replace("円", "")
    )
    if not cleaned:
        raise InvalidOperation("empty amount")
    value = Decimal(cleaned)
    if not value.is_finite():
        raise InvalidOperation("non-finite amount")
    return value
```

ここで `InvalidOperation` や負数チェックに引っかかった行は例外で落とさず、`AnomalyRecord(source_file, row_number, reason, raw_row)` として別配列へ退避します。

### 3. Excelで文字化けしない出力

集計結果のCSV出力は `encoding="utf-8-sig"` で書き出すと、WindowsのExcelでダブルクリックしても文字化けしません。あわせて機械可読な `summary.json` と人間確認用の `report.md` を同時出力しておくと、後続の自動処理にも人間レビューにも対応できます。

## 自動テストで必ず押さえるべきケース

実運用前に最低限以下の4ケースを `unittest` で自動検証しておくと安心です。

- UTF-8 BOM ファイルと CP932（Shift-JIS）ファイルを同一バッチで混在処理できること
- 同一 `order_id` の重複行が二重計上されずスキップ件数に記録されること
- 不正日付・非数値金額・負数がバッチ全体を止めず `anomalies.csv` へ分離されること
- 全角 `￥`・半角 `¥`・カンマ区切り・`円` 付き金額がすべて正確に `Decimal` 集計されること

## まとめ

業務自動化で本当に価値が出るのは「動くスクリプト」ではなく、**「おかしな入力が来ても静かに壊れず、異常を報告してくれる仕組み」**です。標準ライブラリだけで組んでおけば依存パッケージ由来の脆弱性や環境構築トラブルも最小化できます。

より発展的な内容（LLM/RAG投入前の決定的PIIマスキング、見出し保持チャンキング、非スクレイピング型ユーティリティのAPI/MCP化）については、以下のリポジトリでも公開・検証を進めています。
- [inemurisandesu/apify-rag-data-normalizer](https://github.com/inemurisandesu/apify-rag-data-normalizer)

---

### 関連する実践ガイド（第1章無料公開中）

この記事を含む「壊れない業務データ自動化・PIIマスキング・OpenAPI監査・Apify/MCP公開」の設計とテスト済みコードを一冊にまとめました：

https://zenn.dev/inemuri/books/safe-ai-data-automation-kit
