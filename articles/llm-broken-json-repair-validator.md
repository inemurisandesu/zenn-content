---
title: "LLM出力の壊れたJSONをPython標準ライブラリで決定的に修復・型検証する実装パターン"
emoji: "🧩"
type: "tech"
topics: ["python", "llm", "json", "ai", "automation"]
published: true
---

LLMにJSON形式での出力を指示しても、実運用では次のような「あと一歩で壊れているJSON」が必ず混ざります。

- 前後に説明文や ```` ```json ... ``` ```` のコードフェンスが付く
- 末尾カンマ（trailing comma）や `//` コメントが入る
- シングルクォートや Python の `True / False / None` が混ざる
- 最大トークン上限に当たって末尾の `}` や `]` が欠けたまま途切れる

これを毎回LLMに「直して」と再プロンプトすると、API費用も遅延も倍増します。そこで **LLM呼び出しゼロ・Python標準ライブラリだけ** で壊れたJSONを決定的に修復し、必須項目・型チェック・CSV/Markdown表変換まで一気に行う実装パターンをまとめました。

実装全体とテストは公開リポジトリに置いています。
- リポジトリ：[inemurisandesu/apify-llm-json-repair-validator](https://github.com/inemurisandesu/apify-llm-json-repair-validator)

## 1. 段階的フォールバック修復パイプライン

いきなり強引な置換をすると正常なJSONまで壊すリスクがあるため、以下の5段階で安全にフォールバックします。

1. **そのまま `json.loads()`**（正常JSONなら無変換・最速通過）
2. **Markdownフェンス剥離＋前後の説明文トリム**（最外殻の `{...}` または `[...]` を抽出）
3. **行コメント（`//`）と末尾カンマ（`,}` / `,]`）の除去**
4. **途切れた文字列・閉じ括弧の自動補完**（スタックで未閉鎖の `{` / `[` を追跡して末尾に補完）
5. **安全な `ast.literal_eval` フォールバック**（`eval()` は絶対に使わず、`ast.literal_eval` のみでシングルクォートや `True/False/None` を安全に吸収）

※ セキュリティ上、`eval()` ではなく **リテラル以外は絶対に実行しない `ast.literal_eval()`** を使うのが必須ポイントです。

## 2. 監査可能な修復ログ（`repairSteps`）を残す

自動修復を行うシステムでは、「何が起きたのか分からず静かにデータが変わる」のが一番危険です。どの修復ルールが発動したかを `["stripped_code_fence", "removed_trailing_commas", "closed_truncated_brackets"]` のように配列で返しておくことで、プロンプト側の劣化も定量的に監視できます。

## 3. 型チェック時のPython特有の罠（`bool` は `int` のサブクラス）

Pythonでは `isinstance(True, int)` が `True` になるため、JSONの型検証で `isinstance(v, int)` と書くと `true` が整数として通過してしまいます。厳密な型判定では `type(v) is int` / `type(v) is bool` を使うのが安全です。

```python
TYPE_CHECKS = {
    "string": lambda v: isinstance(v, str),
    "integer": lambda v: type(v) is int,
    "number": lambda v: type(v) in (int, float),
    "boolean": lambda v: type(v) is bool,
    "array": lambda v: isinstance(v, list),
    "object": lambda v: isinstance(v, dict),
}
```

## まとめ

LLMパイプラインの安定性は、プロンプトを凝るよりも**「出力の後段に決定的な修復・型検証レイヤを1枚挟む」**だけで劇的に改善します。今回のコードも依存ゼロの純粋Pythonで完結しているため、既存プロジェクトやMCPツールへそのまま組み込めます。
