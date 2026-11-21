---
title: "第4章：非スクレイピング型ユーティリティのApify Actor化と自動運用設計"
---

# 第4章：非スクレイピング型ユーティリティのApify Actor化と自動運用設計

第3章で作ったような「純粋なデータ正規化・PIIマスキング・チャンキング」のモジュールは、自前でWebサーバーや決済システム（Stripe・請求・税務処理）を運用しなくても、**Apify Store の Pay-Per-Event（PPE）型ユーティリティActor**として公開できます。

## 1. なぜ「非スクレイピング型」を選ぶのか

ApifyというとWebスクレイピングのイメージが強いですが、2026年現在は **MCP（Model Context Protocol）経由でAIエージェントがツールを動的に呼び出し、イベント単位で従量課金する用途** が拡大しています。

他社サイトのスクレイピングではなく「呼び出し元が入力したデータを変換・検証するユーティリティ」に特化することで、以下の大きな利点があります。

- 対象サイトの利用規約変更やHTML構造変更で壊れない
- 法的・規約上のリスクが極めて低い
- 外部ネットワークアクセス不要のため実行が高速で計算コスト（CU）を極小化できる

## 2. 最小構成のディレクトリ設計

```text
apify-rag-data-normalizer/
├── .actor/
│   ├── actor.json          # Actorメタデータ
│   └── input_schema.json   # コンソールUI・入力バリデーション定義
├── Dockerfile              # apify/actor-python:3.12 ベース
├── requirements.txt        # 公式 apify SDK のみ
├── src/
│   ├── normalizer.py       # 純粋Pythonコアロジック（依存ゼロ）
│   └── main.py             # Apify SDK接続＋Pay-Per-Event課金層
└── tests/
    └── test_normalizer.py  # ローカル自動テスト
```

## 3. Pay-Per-Event（PPE）課金の実装要点

Python SDKでは、`Actor.push_data()` に `charged_event_name` を渡すだけで、出力チャンク1件ごとに透過的なイベント課金を記録できます。

```python
for chunk in chunks:
    await Actor.push_data(
        chunk.to_dict(),
        charged_event_name="normalized-chunk",
    )
```

コンソール側の Monetization 設定でイベント名 `normalized-chunk` に対応する単価を設定しておけば、実行回数に応じた課金・請求・スケーリングをプラットフォーム側が自動処理します。

## 4. 継続的な改善サイクルの回し方

1. GitHubリポジトリとApify（ビルド連携）およびZenn（コンテンツ連携）を接続する。
2. コアロジックを修正したら、まずローカルで `unittest` を通す。
3. `git push` することで、コード更新とドキュメント・技術書更新を安全に同期する。
4. 実行ログやエラー率を確認し、想定外の入力形式やエッジケースをテストケースに追加して再デプロイする。

この最小構成から始めることで、初期費用とセキュリティリスクを抑えながら、継続的に改善が回る自動化プロダクトの土台を作れます。
