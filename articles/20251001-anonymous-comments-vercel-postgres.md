---
title: "Vercel Postgresで匿名コメント欄を実装する（Next.js App Router）"
emoji: "📝"
type: "tech"
topics: ["nextjs","vercel","postgres","webdev"]
published: true
---


## なにを作ったか

- 認証なし（表示名＋任意ID）でコメント投稿
- 保存先は **Vercel Postgres**（`@vercel/postgres`）
- APIは **Next.js App Router の Route Handlers**（`/api/comments`）
- 最低限の対策（IPハッシュのレート制限, honeypot, 本文最大長, XSS回避）
- blog / portfolio 両ページにコメント一覧＋投稿フォームを組み込み

---

## データモデル

テーブル `comments` を用意。アプリ側で `UUID` を発行するため、DB拡張は不要です。

```sql
create table if not exists comments (
  id uuid primary key,
  site text not null,           -- 'blog' | 'portfolio'
  slug text not null,
  user_display_name text not null,
  user_public_id text not null,
  content text not null,
  created_at timestamptz not null default now(),
  ip_hash text,
  is_approved boolean not null default true,
  parent_id uuid references comments(id)
);
create index if not exists idx_comments_site_slug_created_at on comments(site, slug, created_at desc);
```

---

## APIのポイント

- GET `/api/comments?site=blog|portfolio&slug=...`：承認済みのみ返却
- POST `/api/comments`：本文バリデーション・honeypot・レート制限を適用
- PATCH/DELETE `/api/comments/[id]`：`Authorization: Bearer COMMENT_ADMIN_TOKEN` 必須

実装では `randomUUID()` で `id` を作り、IPは `sha256(ip + SALT)` によるハッシュのみ保存します。

---

## UI（`CommentSection`）

- 表示名 / ユーザーID / 本文 / 同意チェック
- 送信後に一覧を再取得
- スタイルは既存UIのボタンコンポーネントを利用

---

## 環境変数

`.env` / Vercel：

```env
POSTGRES_URL=postgres://USER:PASSWORD@HOST:PORT/DB
COMMENT_ADMIN_TOKEN=長いランダム値
COMMENT_HASH_SALT=長いランダム値
COMMENT_RATE_LIMIT_PER_MIN=3
COMMENT_MAX_LENGTH=1000
```

---

## まとめ

シンプルな要件でも「悪用されにくい設計」にすることで、運用負荷を抑えられます。今後はスレッド/返信、モデレーションUI、エクスポートなどに拡張予定です。


