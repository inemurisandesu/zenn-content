---
title: "第5章：n8n並列自動分析ワークフローと安全なAPIキー運用チェックリスト"
---

# 第5章：n8n並列自動分析ワークフローと安全なAPIキー運用チェックリスト

同じ公式API連携を、エンジニア以外のチームメンバーや定期バッチでも回せるようにするには **n8n ワークフロー** への展開が非常に効果的です。

## 1. n8n 並列取得フローの構成

リポジトリ同梱の `n8n/japan-corporate-due-diligence-workflow.json` は以下の4ノード構成になっています。

1. **Manual Trigger**（または Schedule / Webhook Trigger へ差し替え可能）
2. **Set Target Company**：調査対象の13桁法人番号とEDINET基準日を設定
3. **Fetch METI gBizINFO（並列ブランチA）**：`X-hojinInfo-api-token` を環境変数 `{{$env.GBIZ_API_TOKEN}}` から安全に参照
4. **Fetch FSA EDINET v2（並列ブランチB）**：`Subscription-Key` を環境変数 `{{$env.EDINET_API_KEY}}` から安全に参照

後段に Merge ノードと Slack / Google Sheets / LLM要約ノードを繋ぐだけで、新規取引先の定期与信チェックや開示監視パイプラインが数分で完成します。

## 2. 本番運用前のセキュリティ＆コンプライアンスチェックリスト

- [ ] APIトークンをワークフローJSONやGitリポジトリに直書きしていない（環境変数またはCredentials機能のみ使用）
- [ ] リクエストには明確な `User-Agent` と適切なタイムアウト（15秒等）を設定している
- [ ] 過剰な連続アクセスを避け、バッチ実行時は待機間隔（スリープ／レート制御）を設けている
- [ ] レポート・スプレッドシート・AI要約の出力末尾に、国税庁・経済産業省gBizINFO・金融庁EDINETの出典を明記している
- [ ] APIレスポンス内の外部テキストをLLMへ渡す際は、システム指示とデータ領域を明確に分離し、プロンプトインジェクションを防いでいる

## まとめ

本書で解説した「ゼロ依存Pythonコア＋MCPサーバー＋n8nワークフロー＋自動テスト」の全ソースコードは、[inemurisandesu/japan-corporate-intel-mcp](https://github.com/inemurisandesu/japan-corporate-intel-mcp) にて参照できます。ぜひ安全な一次オープンデータ活用の土台としてご活用ください。
