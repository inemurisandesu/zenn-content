---
title: "第1章：AI時代の業務データ自動化で絶対に外せない安全設計と規約基準（実測テスト付き）"
free: true
---

# 第1章：AI時代の業務データ自動化で絶対に外せない安全設計と規約基準

生成AIやエージェントツールの普及により、スクリプトを書くスピードそのものは劇的に上がりました。しかし実務の自動化ツールを「継続的に動かし、価値を生み続ける仕組み」にするには、コード生成速度とは別の壁があります。

それが **「入力の揺れ」「セキュリティ（サプライチェーン・情報漏えい）」「利用規約・法令順守」** の3点です。

---

## 実務で必ず起きる「3大事故」と本書の解決策

| 現場で起きる事故 | よくある失敗実装 | 本書で採用する解決パターン |
| --- | --- | --- |
| **CSVが本番2日目に落ちる** | `open(f, encoding="utf-8")` ＋ `float()` 集計 | `utf-8-sig` → `cp932` 自動フォールバック、`Decimal` 計算、異常行の `anomalies.csv` 隔離 |
| **個人情報・APIキーがLLMへ漏洩** | 生のメールや社内メモをそのままAPI送信 | NFKC正規化＋決定的PII/シークレットマスキング（監査カウント付き）＋SHA-256重複排除 |
| **LLM出力JSONで後段がクラッシュ** | `json.loads()` 直呼びで例外停止 | `eval()` 不使用の5段階決定的修復＋`type(v) is int` 厳密型検証＋修復証跡ログ |

---

## 本書で一貫して守る「3つの安全設計原則」

### 1. 依存パッケージを最小限に抑える
無名サードパーティライブラリを大量に `pip install` する構成は、サプライチェーン攻撃やアップデート破損の温床になります。本書ではコアロジックを **Python標準ライブラリのみ** で実装し、クラウド接続層（Apify SDK等）と明確に分離します。

### 2. 規約違反スクレイピング・不正アクセスを扱わない
ログイン壁の突破、CAPTCHA回避、過剰なクローリングは、技術以前にアカウント停止・法的リスクが高すぎます。本書は **「呼び出し元が渡したデータの浄化・検証・変換」** と **「公式に利用許可されたAPI仕様」** だけに絞ることで、壊れにくく長期運用できる自動化を作ります。

### 3. テスト可能な純粋ロジックとI/O層を分離する
```text
[入力データ (CSV / JSON / Markdown / OpenAPI)]
        │
        ▼
┌──────────────────────────────────────────┐
│  純粋Pythonコア層（外部依存ゼロ）           │
│  ・文字コード/金額正規化・異常行隔離        │
│  ・決定的PIIマスキング・見出し保持チャンク  │
│  ・壊れたJSON修復・OpenAPIセキュリティ監査  │
└──────────────────────────────────────────┘
        │  ← unittest で1ミリ秒以内に全自動検証
        ▼
┌──────────────────────────────────────────┐
│  配信・実行アダプタ層                      │
│  ・ローカルCLI / 定期バッチ                │
│  ・Apify Pay-Per-Event / MCPサーバー      │
│  ・ブラウザ完結 Chrome拡張（権限ゼロ）      │
└──────────────────────────────────────────┘
```

---

## 実機テスト検証済みの即戦力コード例（無料プレビュー）

たとえば、日本の実務CSVで最も多い「UTF-8 BOM と Shift-JIS（CP932）の混在」と「全角 `￥`・半角 `¥`・カンマ・`円` の混在」は、以下の30行だけで安全に吸収できます。

```python
import csv
from decimal import Decimal, InvalidOperation
from pathlib import Path

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

実際にこの設計を組み込んだサンプル一式（`portfolio-automation-kit`）でユニットテストを回すと、混在ファイル・重複ID・不正日付・負数金額をすべて数ミリ秒で分離・集計できることを確認しています。

---

## 第2章〜第5章で手に入るもの（有料パート全4章）

- **第2章**：文字コード混在・金額誤差・異常行を監査可能に捌く4ファイル同時出力パターン（`category_summary.csv` / `anomalies.csv` / `summary.json` / `report.md`）とテスト設計
- **第3章**：LLM/RAG投入前の決定的PII・APIキーマスキング、見出し保持チャンキング、日本語対応トークン見積もり、SHA-256重複排除
- **第4章**：非スクレイピング型ユーティリティの **Apify Store（Pay-Per-Event＋MCP対応）** 実装・ディレクトリ構成・自動デプロイ運用
- **第5章**：**LLM出力の壊れたJSON修復器（`eval()`不使用）** と **OpenAPI 3.x セキュリティ監査＆破壊的変更diff検知** の実戦レシピ

※ すぐ手元で動かせる一括ZIPパック（Python CSV集計＋GAS自動通知＋オフラインPII/JSON修復Chrome拡張・導入マニュアル付き）はBOOTHでも配布しています：
https://inemuric.booth.pm/items/9195572

次章から、テスト通過済みの各レシピを順番に掘り下げていきます。
