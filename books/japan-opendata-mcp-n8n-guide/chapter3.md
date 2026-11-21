---
title: "第3章：gBizINFO API v1 と EDINET API v2 の横断名寄せ＆出典強制付与設計"
---

# 第3章：gBizINFO API v1 と EDINET API v2 の横断名寄せ＆出典強制付与設計

日本政府の公式APIを複数横断するときに現場で一番つまずくのは、**「項目名のバラつき」「未提供APIキー時の安全なスキップ」「二次利用時の出典記載漏れ」**の3点です。

## 1. APIキーは必ず呼び出し元の環境変数・引数から受け取る

オープンソースや配布キットに開発者自身のAPIキーを埋め込むのは厳禁です。本設計では以下のように安全に分離します。

- **gBizINFO API v1**：`X-hojinInfo-api-token` ヘッダーに呼び出し元の無料トークンを渡す
- **EDINET API v2**：`Subscription-Key` パラメータに呼び出し元の無料キーを渡す
- キーが未指定のソースは例外で落とさず、`{"status": "skipped", "reason": "..."}` として部分結果を返す

## 2. 英語＋日本語の統一ドシエ（Dossier）スキーマへ正規化

gBizINFOはスネークケース（`corporate_number`, `capital_stock`）、EDINETはキャメルケースと略語（`docID`, `JCN`, `pdfFlag: "1"`）が混在しています。これをAIエージェントが誤読しないクリーンな統一JSONへ変換します。

- `JCN`（EDINET側の法人番号）と `corporate_number`（gBizINFO側）を13桁数字に正規化して突合
- `"1"` / `"0"` の文字列フラグを厳密なboolean（`pdfFlag: true/false`）へ変換
- 企業名検索から開始した場合でも、gBizINFOで解決した `resolvedCorporateNumber` をそのままEDINETのフィルタキーへ引き継ぐ

## 3. 出典（Attribution）をレコード構造に強制同梱する

公的オープンデータの利用規約では出典明記が求められます。出力JSONの各レコードとルートオブジェクトの両方に `attribution` 配列を強制付与しておくことで、AIが要約やレポートへ転記する際も一次ソースの根拠を失わずに済みます。

```python
ATTRIBUTIONS = {
    "nta_corporate_number": "国税庁法人番号公表サイト (https://www.houjin-bangou.nta.go.jp/) の公開仕様に基づく検証",
    "gbizinfo": "経済産業省 gBizINFO (https://info.gbiz.go.jp/) のデータを加工して作成",
    "edinet": "金融庁 EDINET API v2 (https://disclosure2.edinet-fsa.go.jp/) のデータを加工して作成",
}
```
