---
title: "国税庁法人番号・gBizINFO・EDINETの公式APIを横断統合するゼロ依存MCPサーバーの作り方"
emoji: "🏢"
type: "tech"
topics: ["python", "mcp", "opendata", "api", "edinet"]
published: true
---

AIエージェントに日本企業の調査・与信チェック・開示情報確認をさせたいとき、企業サイトや外部サービスを無断スクレイピングするのは利用規約・法的リスクが大きすぎます。

実は日本政府の一次オープンデータAPIは、出典を明記すれば**商用加工・再配布まで合法的に利用できる強力なデータ基盤**が揃っています。

- **国税庁 法人番号公表サイト**（13桁法人番号の公式チェックデジット仕様）
- **経済産業省 gBizINFO API v1**（企業基本情報・資本金・従業員数・補助金/調達実績）
- **金融庁 EDINET API v2**（有価証券報告書・開示書類一覧）

この記事では、外部ライブラリを一切使わず **Python標準ライブラリだけ** で「13桁法人番号の公式検証」「gBizINFO・EDINETの横断正規化」「出典自動付与」「stdio JSON-RPC 2.0 MCPサーバー」を実装する構成を解説します。

実装全体（MCPサーバー＋Apify Actor＋n8nワークフローJSON＋ユニットテスト）は公開リポジトリに置いています。
- リポジトリ：[inemurisandesu/japan-corporate-intel-mcp](https://github.com/inemurisandesu/japan-corporate-intel-mcp)

## 1. 13桁法人番号の公式チェックデジット検証

APIを叩く前に、入力された法人番号が本当に有効な13桁かをオフラインで即座に検証すると、無駄なAPIリクエストと入力ミスを防げます。国税庁の公式計算式は以下の通りです。

- 先頭1桁がチェックデジット、後ろ12桁が基礎番号
- 基礎番号の右端から1始まりで位置 `i`（1〜12）を数え、奇数は重み `1`、偶数は重み `2` を掛けて合計する
- `9 - (合計 % 9)` が先頭1桁と一致すれば有効

```python
def validate_corporate_number(raw: str) -> bool:
    cleaned = raw.replace("-", "").replace(" ", "")
    if len(cleaned) != 13 or not cleaned.isdigit():
        return False
    check_digit = int(cleaned[0])
    total = sum(
        int(ch) * (1 if i % 2 == 1 else 2)
        for i, ch in enumerate(reversed(cleaned[1:]), start=1)
    )
    return check_digit == (9 - (total % 9))
```

## 2. 出典（Attribution）をコードレベルで必ず付与する

公的オープンデータを二次利用・商用提供する際の最大の注意点は「出典記載漏れ」です。レコード生成関数（`normalize_gbiz_record` / `normalize_edinet_documents`）の戻り値JSON自体に `attribution` フィールドを強制同梱しておくことで、どの下流システム（MCPクライアント、n8n、Apify）から呼ばれても出典が欠落しない設計にできます。

## 3. Python標準ライブラリだけでMCPサーバー（JSON-RPC 2.0）を書く

MCPのstdioトランスポートは標準入力から1行ずつJSON-RPC 2.0を受け取り、`initialize` / `tools/list` / `tools/call` に対してJSONを標準出力へ返すシンプルな仕様です。

外部フレームワークに依存せず純粋Pythonで実装しておくことで、サプライチェーンリスクを最小化しつつ、同じコアモジュールを **ローカルMCP・Apify Actor・n8nワークフロー** の3経路でそのまま再利用できます。

## まとめ

「スクレイピングに頼らず、公式に許可された一次オープンデータを横断名寄せしてAIが使いやすい形に整える」設計は、規約・セキュリティ・継続性のすべてにおいて最も強い自動化アプローチです。
