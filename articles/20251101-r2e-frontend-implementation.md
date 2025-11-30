---
title: "R2E APIのデモUIを作りました！Next.js + Render でバックエンド連携する実装記"
emoji: "📝"
type: "tech"
topics: ["nextjs","typescript","vercel","render","fastapi"]
published: true
---


## はじめに

以前 [Research→Experience API を作りました](https://beachone1155.vercel.app/blog/research-to-experience-api) という記事で、FastAPI + PostgreSQL(pgvector) で「ビジネス課題 → 関連論文探索 → 示唆」を返す API を紹介しました。

今回は、この API を体験できるフロントエンド UI を Next.js（Vercel）ブログの `/portfolio/r2e` に追加した実装記です。バックエンドは Render の無料枠で動かしています。

## 完成したもの

- **URL**: `/portfolio/r2e`
- **機能**: 3タブUI（Search / Sources / Status）
- **バックエンド**: Render（`https://research-to-experience-api.onrender.com`）
- **フロントエンド**: Next.js（Vercel）

Search タブでビジネス課題を入力すると、Sources タブに論文リストが表示されます。

## アーキテクチャ設計

### 全体構成

![R2E API アーキテクチャ構成図](https://beachone1155.vercel.app/images/r2e-architecture.drawio.png)

### バックエンド構成

バックエンドは以下の構成で動作しています：

- **FastAPI**: Web フレームワーク
- **Neon PostgreSQL**: クラウド PostgreSQL（`pgvector` 拡張対応）
  - ベクトル次元: 1536（OpenAI `text-embedding-3-small` 対応）
- **OpenAI API**: 埋め込みモデル + LLM
  - Embedding: `text-embedding-3-small` (1536次元)
  - Chat: `gpt-4o-mini`

**なぜ Neon か？**

- Render の無料枠では PostgreSQL を直接ホストできないため
- Neon は無料枠があり、`pgvector` 拡張をサポートしている
- クラウドでスケーラブルなデータベースサービス

### 要件

1. フロントエンドからバックエンド API を叩く
2. 本番環境では Render、開発環境では localhost:8000 を使い分け
3. CORS を避けたい（同一オリジン扱いにしたい）

### 選択肢

最初は Next.js の `rewrites` を使うことを考えましたが、Vercel 上で POST リクエストが正しくプロキシされない問題が発生しました。最終的に **Next.js API Route** でバックエンドにプロキシする方式にしました。

```typescript
// src/app/api/r2e/[...path]/route.ts
export async function GET(request: NextRequest, { params }: { params: { path: string[] } }) {
  return proxyRequest(request, params.path, 'GET')
}

export async function POST(request: NextRequest, { params }: { params: { path: string[] } }) {
  return proxyRequest(request, params.path, 'POST')
}
```

### ディレクトリ構成

```
external/research-to-experience-api/
  ├── client.ts          # APIクライアント（search, health）
  └── README.md          # 使い方

src/app/
  ├── api/r2e/[...path]/
  │   └── route.ts        # バックエンドプロキシ
  └── portfolio/r2e/
      └── page.tsx        # メインUI
```

## 実装詳細

### 1. API クライアント（薄いラッパー）

`external/` フォルダに API クライアントを配置。ドメイン非依存の相対パスで実装しています。

```typescript:external/research-to-experience-api/client.ts
export type SearchResponse = {
  summary?: string[]
  business_implications?: string[]
  sources?: Source[]
}

export async function search(query: string): Promise<SearchResponse> {
  const r = await fetch('/api/r2e/search', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ query }),
  })
  if (!r.ok) {
    const errorData = await r.json().catch(() => ({}))
    if (r.status === 504) {
      throw new Error('バックエンドAPIのコールドスタート中です。30秒ほど待ってから再度お試しください。')
    }
    throw new Error(errorData.details || `HTTP ${r.status}`)
  }
  return r.json()
}
```

### 2. API Route（プロキシ実装）

Next.js API Route でバックエンド API にプロキシします。環境変数 `BACKEND_URL` で開発/本番を切り替えます。

```typescript:src/app/api/r2e/[...path]/route.ts
const BACKEND_URL = process.env.BACKEND_URL || 'https://research-to-experience-api.onrender.com'

async function proxyRequest(
  request: NextRequest,
  pathSegments: string[],
  method: 'GET' | 'POST'
) {
  const path = pathSegments.join('/')
  const url = `${BACKEND_URL}/${path}`
  
  const options: RequestInit = { method, headers: { 'Content-Type': 'application/json' } }
  
  if (method === 'POST') {
    const body = await request.text()
    options.body = body
  }
  // タイムアウト設定（Render のコールドスタート対応）
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), 90000)
  
  try {
    const response = await fetch(url, { ...options, signal: controller.signal })
    clearTimeout(timeoutId)
    const data = await response.text()
    const jsonData = JSON.parse(data)
    return NextResponse.json(jsonData, { status: response.status })
  } catch (fetchError) {
    clearTimeout(timeoutId)
    if (fetchError instanceof Error && fetchError.name === 'AbortError') {
      return NextResponse.json(
        { error: 'Request timeout', details: 'Backend API did not respond within 90 seconds...' },
        { status: 504 }
      )
    }
    throw fetchError
  }
}
```

### 3. メインUI（3タブ構成）

React の state でタブ管理、API クライアントでデータ取得します。

```typescript:src/app/portfolio/r2e/page.tsx
export default function R2EPage() {
  const [tab, setTab] = useState<Tab>('Search')
  const [query, setQuery] = useState('')
  const [loading, setLoading] = useState(false)
  const [resp, setResp] = useState<SearchResponse | null>(null)
  
  const doSearch = async () => {
    setLoading(true)
    setError(null)
    try {
      const data = await doApiSearch(query)
      setResp(data)
      setTab('Sources') // 成功時はSourcesへ切替
    } catch (e: unknown) {
      const message = e instanceof Error ? e.message : 'request failed'
      setError(message)
    } finally {
      setLoading(false)
    }
  }
  // ... UI実装
}
```

**タブ構成**:
- **Search**: ビジネス課題を入力、検索実行
- **Sources**: 検索結果の論文リスト（Title, URL, Confidence）
- **Status**: `/health` エンドポイントの疎通確認

## 環境設定

### Vercel 環境変数

Vercel CLI で環境変数を設定しました。

```bash
# Production / Preview
echo -n "https://research-to-experience-api.onrender.com" | vercel env add BACKEND_URL production
echo -n "https://research-to-experience-api.onrender.com" | vercel env add BACKEND_URL preview

# Development（ローカル開発用）
echo -n "http://localhost:8000" | vercel env add BACKEND_URL development
```

### Render 環境変数

バックエンド API は Render 上で以下の環境変数を設定しています：

```bash
# LLM/Embedding プロバイダー
PROVIDER=openai

# OpenAI API 設定
OPENAI_API_KEY=sk-...
OPENAI_EMBED_MODEL=text-embedding-3-small
OPENAI_CHAT_MODEL=gpt-4o-mini

# データベース（Neon PostgreSQL）
DATABASE_URL=postgresql://user:pass@ep-xxx.region.neon.tech/dbname?sslmode=require

# オプション: データベース初期化をスキップ（ヘルスチェック用）
SKIP_DB_INIT=0
```

**補足**:
- `PROVIDER=openai`: OpenAI API を使用（`oss` にすると Ollama を使用可能）
- `DATABASE_URL`: Neon から提供される接続文字列（`sslmode=require` 必須）
- ベクトル次元は 1536 に固定（`text-embedding-3-small` の出力次元）

### Neon データベース設定

1. [Neon Console](https://console.neon.tech) でプロジェクト作成
2. `pgvector` 拡張を有効化（Neon はデフォルトでサポート）
3. 接続文字列を取得して Render の `DATABASE_URL` に設定

**スキーマ**:
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE IF NOT EXISTS papers (
  id SERIAL PRIMARY KEY,
  paper_id TEXT,
  title TEXT,
  url TEXT,
  abstract_chunk TEXT,
  embedding VECTOR(1536),  -- OpenAI text-embedding-3-small は 1536 次元
  source TEXT,
  indexed_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_papers_embedding ON papers 
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

### Render デプロイ

バックエンドは Render の無料枠でデプロイ。**Docker で構築したものをそのまま Render にデプロイ**しています。

- **URL**: `https://research-to-experience-api.onrender.com`
- **Health Check Path**: `/health`
- **Auto-Deploy**: Yes（GitHub main ブランチ連携）
- **Instance Type**: Free
- **ビルド方式**: Dockerfile ベース（`docker compose` でローカル開発、同じ Dockerfile を Render でも使用）

ローカル開発時は `docker compose` で起動し、本番環境では同じ Dockerfile を Render でビルド・デプロイしています。これにより、ローカルと本番環境の一貫性を保っています。

## ハマりポイントと対応

### 1. POST リクエストが 405 エラー

**問題**: Next.js の `rewrites` を使ったが、Vercel 上で POST が正しくプロキシされなかった。

**対応**: `rewrites` をやめ、Next.js API Route でプロキシする方式に変更。

```typescript
// ✗ 最初の実装（rewrites）
// next.config.mjs
async rewrites() {
  return [{ source: '/r2e/:path*', destination: `${BACKEND_URL}/:path*` }]
}

// ✓ 最終実装（API Route）
// src/app/api/r2e/[...path]/route.ts
export async function POST(...) { return proxyRequest(...) }
```

### 2. Render のコールドスタート

**問題**: Render 無料枠は15分間アクセスがないとスリープし、初回アクセスで30〜60秒かかることがある。

**対応**:
- タイムアウトを90秒に延長
- 504エラー時に「コールドスタート中です。30秒ほど待ってから再度お試しください。」と案内

```typescript
if (r.status === 504) {
  throw new Error('バックエンドAPIのコールドスタート中です。30秒ほど待ってから再度お試しください。')
}
```

### 3. ビルド時の環境変数エラー

**問題**: Vercel ビルド時に `BACKEND_URL` が `undefined` になり、rewrites の destination が `undefined/:path*` になってエラー。

**対応**: API Route ではランタイムで `process.env.BACKEND_URL` を参照するため、ビルド時の問題は発生しない（rewrites から API Route に変更したので解決）。

### 4. ベクトル次元の不一致（バックエンド側）

**問題**: 初回デプロイ時に、ollamaのモデルだとデータベーススキーマが 768 次元だったが、OpenAI の `text-embedding-3-small` は 1536 次元を出力するためエラー。

**対応**: 
- スキーマを 1536 次元に更新（`app/db/models.sql`）
- 起動時に 768 次元テーブルを自動検出して再作成するマイグレーション機能を追加

## 学んだこと

1. **Next.js の rewrites は GET には向くが、POST は API Route が確実**
2. **無料ホスティング（Render）のコールドスタートを考慮した UX 設計が重要**
3. **環境変数はビルド時とランタイムで扱いが異なる**
4. **Neon のようなマネージド PostgreSQL を使うと、Render 無料枠でも pgvector が使える**

## まとめ

- Next.js API Route でバックエンドプロキシを実装
- 3タブUIで R2E API を体験できるUIを追加
- Render のコールドスタートに対応
- Vercel 環境変数で開発/本番を切り替え
- Neon PostgreSQL + OpenAI API でバックエンド構築

実際に `/portfolio/r2e` にアクセスして、ビジネス課題から論文探索までの体験を試してみてください！

## 参考リンク

- [バックエンドリポジトリ](https://github.com/beachone1155/research-to-experience-api)
- [デプロイ先（Render）](https://research-to-experience-api.onrender.com)
- [フロントエンドデモ](https://beachone1155.vercel.app/portfolio/r2e)
- [Neon PostgreSQL](https://neon.tech)

