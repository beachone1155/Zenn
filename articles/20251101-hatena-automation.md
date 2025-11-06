---
title: "はてなブログへの自動投稿に対応した（GitHub Actions + AtomPub API）"
emoji: "📝"
type: "tech"
topics: ["githubactions","automation","hatena"]
published: true
---


## 追加内容の概要

本ブログの自動配信パイプライン（Qiita/Dev.to/Zenn/X）に、はてなブログへの自動投稿機能を追加しました。

- はてなブログ: AtomPub API（WSSE認証）で投稿

Frontmatter の `publish_on` に `hatena` を書くだけで配信対象になります。

## 実装ポイント

- `scripts/publish.mjs`
  - `publishToHatena(post)` を追加（AtomEntry XML + X-WSSE認証）
  - `main()` に `hatena` を統合（X以外は並列処理）
- `.github/workflows/publish.yml`
  - Hatena ステップを追加
- バリデーション/型に `hatena` を追加

## 使い方

`/content/YYYY/MM/slug.md` の Frontmatter で配信先を指定します。

```md
publish_on: ["hatena"]
```

GitHub に push すると、Actions が記事を配信します。ローカルでは `PUBLISH_TARGETS=hatena node scripts/publish.mjs` のように個別実行も可能です。

## 補足
note への自動投稿も検討しましたが、公式APIが存在せず、非公式APIも投稿機能が不明確なため、今回は実装を見送りました。はてなブログは公式のAtomPub APIが提供されているため、安定して自動投稿できます。
