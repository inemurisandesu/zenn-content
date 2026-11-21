---
title: "第4章：Python標準ライブラリだけで書くゼロ依存MCPサーバー（JSON-RPC 2.0）"
---

# 第4章：Python標準ライブラリだけで書くゼロ依存MCPサーバー（JSON-RPC 2.0）

MCP（Model Context Protocol）サーバーを作る際、重いフレームワークや大量のサードパーティ依存を入れると、環境構築エラーやサプライチェーンリスクの原因になります。

実はローカルの stdio MCPサーバーは、**標準入力から1行ずつJSON-RPC 2.0を受け取り、標準出力へ1行JSONを返すだけ**なので、Python標準ライブラリのみで100行前後で堅牢に実装できます。

## 必須の3メソッド

1. **`initialize`**：`protocolVersion`, `serverInfo`, `capabilities: {"tools": {}}` を返す
2. **`tools/list`**：公開するツール定義（`name`, `description`, `inputSchema`）を配列で返す
3. **`tools/call`**：引数を受け取り、純粋Pythonのビジネスロジックを実行して `content: [{"type": "text", "text": ...}]` と `isError` を返す

（※ `notifications/initialized` などの通知メッセージは `id` を持たないためレスポンスを返さず無視します）

## Claude Desktop / Cursor への登録例

```json
{
  "mcpServers": {
    "japan-corporate-intel": {
      "command": "python3",
      "args": ["-m", "src.mcp_server"],
      "cwd": "/path/to/japan-corporate-intel-mcp",
      "env": {
        "GBIZ_API_TOKEN": "YOUR_FREE_GBIZ_TOKEN",
        "EDINET_API_KEY": "YOUR_FREE_EDINET_KEY"
      }
    }
  }
}
```

## プロトコル層もオフラインで自動テストする

`handle_mcp_request(req)` を純粋関数として切り出しておけば、標準入出力を起動せずに `unittest` から `initialize` → `tools/list` → `tools/call` の一連のハンドシェイクをミリ秒単位で自動検証できます。
