---
title: "OpenAPI 3.xのセキュリティ監査と破壊的変更検知をPython標準ライブラリだけで自動化する"
emoji: "🛡️"
type: "tech"
topics: ["python", "openapi", "security", "api", "ci"]
published: true
---

REST APIの仕様書（OpenAPI 3.x）が大きくなると、目視レビューだけでは次のような重大ミスが必ずすり抜けます。

- `http://` の非TLSサーバーURLが本番仕様に残っている
- `api_key` や `token` のような機密情報がクエリ文字列（`in: query`）に渡されている（アクセスログやリファラに漏洩する）
- 内部参照（`$ref: "#/components/schemas/..."`）がリンク切れしている
- 新バージョンで既存エンドポイント削除や新規必須パラメータ追加（破壊的変更）を入れてしまった

この記事では、外部の重いLinterを入れず**Python標準ライブラリだけ**で「構造検証・セキュリティ監査・旧仕様との破壊的変更diff・0〜100スコア算出」を自動化する設計と実装ポイントを解説します。

実装全体とユニットテストは公開リポジトリに置いています。
- リポジトリ：[inemurisandesu/apify-openapi-spec-auditor](https://github.com/inemurisandesu/apify-openapi-spec-auditor)

## 1. 壊れた内部 `$ref` を外部通信なしで検証する

JSON Pointer（`#/components/schemas/...`）を辿る処理は15行ほどで書けます。外部URLの `$ref` を勝手にfetchしない設計にしておけば、CIやサンドボックス環境でもSSRFリスクゼロで高速に動きます。

```python
def resolve_local_ref(root: dict, ref: str) -> bool:
    if not ref.startswith("#/"):
        return True
    cur = root
    for part in ref[2:].split("/"):
        part = part.replace("~1", "/").replace("~0", "~")
        if isinstance(cur, dict) and part in cur:
            cur = cur[part]
        else:
            return False
    return True
```

## 2. クエリ文字列への機密情報混入と認証漏れを機械検知する

各エンドポイントの `parameters`（パス共通＋メソッド固有）を結合し、`in == "query"` かつ名前に `password / secret / token / api_key / auth` を含むものを `critical` として即座に警告します。また、グローバル `security` もオペレーション固有 `security` も空のエンドポイントを `NO_SECURITY_SCHEME` として可視化します。

## 3. 旧版仕様との「破壊的変更」自動diff

PRレビューで最も事故が多いのが破壊的変更です。`previous_spec` と `spec` を比較し、以下の3パターンを自動検出します。

1. **REMOVED_PATH（critical）**：既存パスが削除された
2. **REMOVED_OPERATION（critical）**：既存パスのHTTPメソッド（例：`DELETE`）が削除された
3. **NEW_REQUIRED_PARAM（high）**：既存エンドポイントに新しく `required: true` のパラメータが追加された

## 4. 決定的なヘルススコア（0〜100）でCIゲートに組み込む

重大度ごとに減点ルール（例：`critical: -25`, `high: -12`, `medium: -5`, `low: -2`）を固定しておけば、「スコア80未満はマージブロック」といったシンプルなCIゲートを即座に作れます。

## まとめ

OpenAPIのセキュリティ監査と破壊的変更検知は、外部依存なしの純粋Pythonロジックにしておくことで、GitHub Actions・ローカル開発・AIエージェント向けMCPツールのどこからでも安全に呼び出せます。

---

### 関連する実践ガイド（第1章無料公開中）

この記事を含む「壊れない業務データ自動化・PIIマスキング・OpenAPI監査・Apify/MCP公開」の設計とテスト済みコードを一冊にまとめました：

https://zenn.dev/inemuri/books/safe-ai-data-automation-kit
