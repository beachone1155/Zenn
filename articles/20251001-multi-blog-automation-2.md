---
title: "AIと無料APIで\"マルチ投稿ブログ\"を自動化してみた（第2回）— 記事内容に合うカバー画像を自動生成"
emoji: "📝"
type: "tech"
topics: ["nextjs","automation","huggingface","unsplash"]
published: true
---


> 第1回では「1枚のMarkdownからマルチ投稿」を整えました。第2回は、記事に“合う”カバー画像を**無料の生成AI×画像API**で自動生成します。

---

## なにを実現したか（概要）

- 記事のタイトル・要約・タグからプロンプトを生成し、**Hugging Face Inference Providers**（Stable Diffusion XL）で画像を生成
- 生成に失敗・レート制限等の際は**Unsplash API**に自動フォールバック
- 取得/生成した画像は**OGP推奨サイズ（1200x630）に自動リサイズ**
- 画像は`public/images/[slug]-cover.jpg`へ保存し、Frontmatterの`cover`を自動更新
- ローカル実行で確認後にコミット→Vercelに自動反映

---

## 技術スタックと優先順位

- 生成AI: **Hugging Face Inference Providers**（最優先）
  - エンドポイント: `https://router.huggingface.co/hf-inference/models/...`
  - モデル例: `stabilityai/stable-diffusion-xl-base-1.0`
- フォールバック（任意）:
  - Fal.ai（FLUX.1 Schnell）→ Stability AI API → Unsplash API
- 画像処理: **sharp**（1200x630にトリミング/リサイズ）

---

## 実装ポイント（今回の差分）

### 1) プロンプト生成

記事の`title`/`summary`/`tags`から技術文脈を抽出して、生成AIに渡す**短い英語プロンプト**を作成。

```text
modern web development, Next.js framework, clean minimal illustration
```

基準:
- プラットフォーム名（qiita, zenn, devto）や日本語の汎用語は除外
- 技術タグ（nextjs, awsなど）を優先
- 人物/顔などは避け、著作権や生成リスクを下げるテイストに寄せる

### 2) 生成→フォールバックの順序

1. Hugging Face（生成AI）
2. Replicate（クレジット不足/429等は自動スキップ）
3. Fal.ai（statusポーリングで完了判定）
4. Stability AI（multipart/form-data + `Accept: image/*`）
5. Unsplash（検索API）

すべて失敗時はエラーログを出し、処理を中断します。

### 3) OGP最適化

- すべての画像を**1200x630**に正規化（中央トリミング）
- `app/blog/[slug]/page.tsx`のOpen Graph/Twitterで`post.cover`が設定されていればOGに載る構成

---

## 使い方（ローカル）

### 1) APIキーの設定（最低1つ）

`.env.local` に以下のいずれかを設定（Hugging Face推奨）

```env
HUGGING_FACE_API_KEY=xxxx
# 任意
REPLICATE_API_TOKEN=xxxx
FAL_API_KEY=xxxx
STABILITY_API_KEY=xxxx
UNSPLASH_ACCESS_KEY=xxxx
```

### 2) 画像生成を実行

```bash
npm run generate-images:dry-run   # ダウンロードなしで候補を確認
npm run generate-images          # 実際に保存＆Frontmatter更新
```

処理対象は、Frontmatterの`cover`が空/未設定の記事。保存先は`public/images/[slug]-cover.jpg`です。

---

## 実装ファイル

- `scripts/generate-cover-images.mjs`
  - 記事スキャン、プロンプト生成、Frontmatter更新
- `scripts/utils/prompt-generator.mjs`
  - タイトル/要約/タグから英語プロンプトを生成
- `scripts/utils/image-fetcher.mjs`
  - Hugging Face→Fal.ai→Stability→Unsplashの順で画像取得
  - Stabilityは`multipart/form-data`/`Accept: image/*`で実装
  - 取得画像を`sharp`で1200x630へ

---

## コストとライセンス

- Hugging Face: 無料枠あり（Inference Providers）
- Replicate/Fal.ai/Stability: 無料枠やトライアルあり（枯渇時は自動スキップ）
- Unsplash: 無料（クレジット/規約に従う）

記事配信は従来どおり**Qiita / Zenn / DEV.to**へ自動。OGP画像が安定して付与され、リンクプレビューがリッチになりました。

---

## まとめ

- 記事1枚から**「配信」と「見栄え（OGP）」**まで自動化
- 無料枠を中心に構成し、失敗時のフォールバックで運用を安定化
- 画像サイズを固定化してSNS/検索の表示品質を担保

次回は、**記事の内容要約や目次の自動生成**を予定。生成AIを“安全に”仕込む設計を解説します。


