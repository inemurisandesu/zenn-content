---
title: "暗号資産の確定申告前に取引所CSV（Shift-JIS）とEtherscan CSVを完全オフラインで整理・異常検知するPython設計"
emoji: "📊"
type: "tech"
topics: ["python", "crypto", "csv", "automation", "security"]
published: true
---

暗号資産の確定申告シーズンになると、多くの人が「国内取引所のShift-JIS CSVと、EtherscanのUTC CSVの形式がバラバラ」「重複行や未分類の報酬・手数料行が混ざって損益計算ツールでエラーになる」という前処理地獄にぶつかります。

しかも取引履歴CSVには個人情報・資産額・ウォレット履歴が詰まっているため、**得体の知れないWebサービスへアップロードするのはセキュリティ上避けたい**ところです。

そこで、**外部通信ゼロ・Python標準ライブラリだけ** でローカルPC上で取引CSVを統合し、重複除外・異常行隔離・要確認フラグ付与まで行う設計と実装をまとめました。

実装全体とユニットテストは公開リポジトリに置いています。
- リポジトリ：[inemurisandesu/crypto-tax-csv-cleaner](https://github.com/inemurisandesu/crypto-tax-csv-cleaner)

## 1. 確定申告前CSVで必ず踏む4つの罠

1. **文字コードの混在**：国内取引所の多くが `cp932`（Shift-JIS）、Etherscanは `utf-8-sig`（BOM付きUTF-8）。
2. **オンチェーン送金の誤認**：自分の別ウォレットへの移動（非課税）と報酬受取・売却（課税対象）が同じ列に入るため、オンチェーン行には必ず `needs_review=YES` を立てて人間・税理士の確認を強制する。
3. **浮動小数点の丸め誤差**：`0.00000001 BTC` のような極小数量や `wei` 換算を `float` で足すと誤差が出るため `Decimal` 必須。
4. **重複取り込み**：月次CSVと年次CSVを重ねて読んだ際の二重計上を `tx_id` / `eth:<txhash>` の集合管理で自動スキップする。

## 2. 出力を3ファイルに分離する

1回の実行で以下3ファイルを生成します。

- `unified_transactions.csv`（Excel文字化けしない `utf-8-sig`・時系列ソート・`needs_review` 列付き）
- `anomalies.csv`（日付破損・数量不正などのエラー行を生JSONごと隔離）
- `tax_prep_report.md`（通貨別の購入/売却/入出金合計と要確認件数のサマリー）

> ※ 本ツールは確定申告前のデータ整理・異常検知を補助するものであり、最終的な税額計算・申告判断は国税庁の公式ガイドラインおよび税理士へご確認ください。

---

### 関連する実践ガイド＆実務自動化パック

- **Zenn Book（第1章無料公開中・800円）**：  
  https://zenn.dev/inemuri/books/safe-ai-data-automation-kit
- **BOOTH（日本政府公式API3種統合 ローカルMCP＆n8n自動分析キット）**：  
  https://inemuric.booth.pm/items/9195542
- **BOOTH（Python業務CSV集計＋GAS自動通知＋オフラインPII/JSONツール 3点セット）**：  
  https://inemuric.booth.pm/items/9195572
