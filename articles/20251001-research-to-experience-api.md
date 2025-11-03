---
title: "Research→Experience API を作りました！課題から論文→示唆→次アクションまで一気通貫"
emoji: "📝"
type: "tech"
topics: ["fastapi","pgvector","llm","arxiv","docker"]
published: true
---


## Research→Experience API とは

一言でいうと、課題を自然文で投げるだけで、関連論文を探してくれて、埋め込み＋類似検索で根拠を集め、
LLMが「示唆（So what?）」「ビジネス含意」「出典リンク」を返す API（PoC）です。

- スタック: FastAPI + PostgreSQL(pgvector)
- データフロー: arXiv → 埋め込み保存 → 類似検索 → LLM要約
- コアAPI: `POST /search`（入力: `{ query: string }`）
- 出力: `summary[]`, `business_implications[]`, `sources[]`

「調べた」で終わらせず、「やってみる」に届くよう、根拠つきの要約と示唆を返します。

---

## すごいポイント（ここが気持ちいい）

- 1リクエストで論文探索→示唆まで到達: 手元のメモから次アクションに一歩進める
- 出典リンクが必ず返る: 会話で終わらず、根拠に戻れる（再現性が担保しやすい）
- OSS/クラウドをスイッチ可能: `PROVIDER=oss|openai` でモデル切替（開発コスト/精度を選べる）
- ベクトルDBはpgvector: Postgresベースで運用の心理的障壁が低い

---

## デモ（ローカル）

1) `.env` にモード設定（既定はOSS）
```bash
PROVIDER=oss
OLLAMA_BASE_URL=http://ollama:11434
OSS_EMBED_MODEL=nomic-embed-text
OSS_CHAT_MODEL=qwen2.5:1.5b-instruct
# OpenAIに切替する場合
# PROVIDER=openai
# OPENAI_API_KEY=sk-...
# OPENAI_EMBED_MODEL=text-embedding-3-small
# OPENAI_CHAT_MODEL=gpt-4o-mini
```

2) 起動
```bash
docker compose -f infra/docker-compose.yml up --build -d
```

3) 確認
- `http://localhost:8000/docs`
- `POST /search` のサンプル
```json
{ "query": "高齢者の服薬アドヒアランスを改善するデジタル介入の実証結果は？" }
```

---

## 技術構成（ざっくり）

- `app/api/main.py`: FastAPI エントリ
- `app/api/routes_search.py`: `/search` 実装
- `app/api/schemas.py`: 入出力スキーマ
- `app/core/config.py`: モデル/プロバイダ設定
- `app/core/paper_fetcher.py`: arXiv 取得
- `app/core/embed_client.py`: 埋め込み
- `app/core/retriever.py`: 類似検索（pgvector）
- `app/core/llm_client.py`: 要約・示唆生成
- `app/db/models.sql`: `vector` 拡張 + `papers` テーブル
- `infra/docker-compose.yml`: app + db

補足: 既定の `nomic-embed-text` は 768次元。OpenAI へ切替時はベクトル次元差に注意。

---

## 今後の改善点（やります）

1. 簡易フロントエンド（Vercel）
   - 最短パスのUIを用意して、検索→示唆→出典確認までを1画面で完結
   - 共有リンク/履歴/テンプレ保存（社内の“同じ質問”を減らす）

2. 役割別エージェントの合議制（LangGraph）
   - リサーチ / ビジネス翻訳 / UX提案 の3エージェントをオーケストレーション
   - 最終レポートは合議（RAG + 要約 + アクション）で確度を底上げ
   - API失敗時は状態管理・チェックポイントでリトライ/フォールバック（LangGraph）

3. ハイブリッド検索の実装
   - キーワード × ベクトル の組み合わせで再現性と網羅性を両取り

4. PDF全文取り込み/チャンク分割
   - 実論文PDFを取り込んで章・段落単位の根拠提示を強化

5. 多言語対応（日英）
   - 日本語の質問 → 英語の学術検索 → 日本語の示唆 の往復最適化

6. 監査性とメトリクス
   - ソース/モデル/パラメータ/トークン消費の記録、再現性のためのスナップショット

7. AWS移行の選択肢
   - Aurora PostgreSQL(pgvector), ECS/Fargate, S3キャッシュ, CloudWatch/X-Ray

---

## 使い方（最小のcurl）

```bash
curl -X POST http://localhost:8000/search \
  -H 'Content-Type: application/json' \
  -d '{"query":"LLM を用いた医療領域のデジタル介入の実証は？"}'
```

---

## まとめ

「知った」で止まらず、「やってみる」に届く API を目指しています。
出典に戻れる設計と、OSS/クラウドの柔軟な切替、将来のエージェント合議で
“調査から体験へ”の距離をできるだけ短くしていきます。