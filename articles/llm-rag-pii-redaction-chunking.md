---
title: "LLM/RAG投入前の決定的PIIマスキングと構造保持チャンキングをPython標準ライブラリで作る"
emoji: "🔒"
type: "tech"
topics: ["python", "rag", "llm", "security", "mcp"]
published: true
---

社内ドキュメントや業務CSV・JSONをLLMやRAGへ投入するとき、そのままテキストを切り出して埋め込むと次の3つの事故が起きます。

1. **個人情報やAPIキーの意図しない送信**（メールアドレス、電話番号、IPアドレス、`sk-...` / `ghp_...` などのトークン）
2. **見出しと本文の分断による検索精度の劣化**（固定文字数でぶつ切りにすると「何の項目についての記述か」が消える）
3. **重複チャンクの大量登録によるトークン・ベクトルDB課金の浪費**

これらをLLM自身に前処理させるとコストもレイテンシも跳ね上がり、出力も非決定的になります。そこで**Python標準ライブラリだけで完結する決定的な前処理パイプライン**を実装・検証しました。

実装全体とテストコードは公開リポジトリに置いています。
- リポジトリ：[inemurisandesu/apify-rag-data-normalizer](https://github.com/inemurisandesu/apify-rag-data-normalizer)

## 1. NFKC正規化を最初に通す

日本語・英語が混ざる業務文書では、全角英数字・全角スペース・機種依存記号の揺れが正規表現や重複判定を壊します。最初に `unicodedata.normalize("NFKC", text)` を通し、改行コード（`\r\n` / `\r`）と過剰な空行を統一します。

```python
import re
import unicodedata

def normalize_whitespace(text: str) -> str:
    text = unicodedata.normalize("NFKC", text)
    text = text.replace("\r\n", "\n").replace("\r", "\n")
    lines = [re.sub(r"[ \t]+", " ", ln).strip() for ln in text.split("\n")]
    return re.sub(r"\n{3,}", "\n\n", "\n".join(lines)).strip()
```

## 2. 決定的PII・シークレットのマスキングと監査カウント

単に置換するだけでなく、「どの種別を何件マスクしたか」をチャンク単位で返しておくと、後段の監査ログや品質監視に使えます。

```python
PII_PATTERNS = [
    ("email", re.compile(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b"), "[REDACTED_EMAIL]"),
    ("api_token", re.compile(r"\b(?:sk-[A-Za-z0-9_-]{16,}|gh[pousr]_[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|AIza[0-9A-Za-z_-]{30,})\b"), "[REDACTED_TOKEN]"),
    ("credit_card", re.compile(r"(?<!\d)(?:\d{4}[-\s]?){3}\d{4}(?!\d)"), "[REDACTED_CARD]"),
    ("phone", re.compile(r"(?<!\d)(?:\+?\d{1,3}[-\s]?)?(?:\(0?\d{1,4}\)|0\d{1,4})[-\s]?\d{1,4}[-\s]?\d{3,4}(?!\d)"), "[REDACTED_PHONE]"),
    ("ipv4", re.compile(r"\b(?:(?:25[0-5]|2[0-4]\d|1?\d?\d)\.){3}(?:25[0-5]|2[0-4]\d|1?\d?\d)\b"), "[REDACTED_IP]"),
]
```

## 3. 見出し保持チャンキング ＋ SHA-256重複排除

Markdown文書は `#`〜`######` の見出し単位で先にセクション分割し、各チャンクに `heading` メタデータを保持します。セクションが目標サイズを超えた場合のみ、**段落（`\n\n`）→ 改行（`\n`）→ 空白**の優先順で自然な境界を探してオーバーラップ分割します。

さらに、浄化済みテキストの `hashlib.sha256` ダイジェストを集合管理することで、複数ファイルにまたがる重複チャンク（定型フッターやコピーされた規定文など）を確実にスキップできます。

## 4. 日本語混在テキストのトークン見積もり

英語前提の「4文字＝1トークン」計算は日本語だと大幅に過小評価されます。ASCII文字（`len / 4`）と非ASCII文字（`len * 1.1`）を分けて見積もることで、コンテキスト長超過の事故を防げます。

```python
import math

def estimate_tokens(text: str) -> int:
    if not text:
        return 0
    ascii_chars = sum(1 for ch in text if ord(ch) < 128)
    non_ascii_chars = len(text) - ascii_chars
    return max(1, math.ceil(ascii_chars / 4 + non_ascii_chars * 1.1))
```

## まとめ

RAGの前処理は「外部依存を増やさず、決定的にテストできる純粋Python層」として切り出しておくと、ローカルCLI・社内バッチ・MCP/Apify Actorのいずれにもそのまま載せ替えられます。

---

### 関連する実践ガイド（第1章無料公開中）

この記事を含む「壊れない業務データ自動化・PIIマスキング・OpenAPI監査・Apify/MCP公開」の設計とテスト済みコードを一冊にまとめました：

https://zenn.dev/inemuri/books/safe-ai-data-automation-kit
