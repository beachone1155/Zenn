---
title: "AIとGitHub Actionsで\"マルチ投稿ブログ\"を自動化してみた（第1回）"
emoji: "📝"
type: "tech"
topics: ["nextjs","githubactions","qiita","zenn"]
published: true
---



> GitHub Pages、Vercel、Qiita、Zenn、DEV.to —  
> 1つのMarkdownから全部に同時投稿できるようにしてみました。

---

## 1. やりたかったこと

エンジニアブログを書いても、  
「ZennにもQiitaにも出したいけど、毎回コピペが面倒…」  
「自分のブログも持ちたい…けど管理が増える…」

そう思って、**1つのMarkdownファイルから全プラットフォームに自動投稿**できる仕組みを作りました。

- 記事はローカルで書くだけ  
- GitHubにpushすると、GitHub Actionsが動く  
- Qiita / Zenn / DEV.to に自動投稿  
- Vercel で自前ブログも自動更新

---

## 2. 公開しているリポジトリ

📦 GitHub:  
👉 [beachone1155/engineer-blog-automation](https://github.com/beachone1155/engineer-blog-automation)

🌐 自前ブログ:   
- [Vercel版](https://beachone1155.vercel.app/)

---

## 3. 技術構成（ざっくり図）

```

Markdown (.md)
│
▼
GitHub Actions
├─ Qiita API 投稿
├─ DEV.to API 投稿
├─ Zenn 連携 (GitHub push)
└─ Next.js (Vercel) でブログ更新

````

使用スタック：
- **Next.js 14 + MDX**：自前ブログ用（Vercel）
- **gray-matter**：Frontmatterの抽出
- **GitHub Actions**：投稿トリガー
- **Qiita / DEV.to API**：自動投稿
- **Zenn CLI**：GitHub連携で記事反映
- **Vercel**：ホスティング

---

## 4. 記事の書き方

記事は `/content/YYYY/MM/slug.md` に Markdown で書くだけ。  
先頭に「Frontmatter」というメタ情報をつけます。

```md
---
title: "AIとGitHub Actionsでマルチ投稿ブログを自動化してみた"
slug: "multi-blog-automation"
summary: "1つのMarkdownからQiita/Zenn/DEV.to/Vercelに自動投稿できる仕組みを構築しました。"
tags: ["nextjs","githubactions","qiita","zenn"]
cover: "/images/cover.png"
lang: "ja"
canonical_url: "https://beachone1155.vercel.app/blog/multi-blog-automation"
publish_on:
  - qiita
  - zenn
  - devto
  - hatena
  - x
draft: false
created_at: "2025-10-25"
updated_at: "2025-10-25"
---
````

pushすると、自動ですべてに反映されます。

---

## 5. 自動投稿の仕組み（裏側）

### 5.1 GitHub Actions の流れ

`content/**/*.md` が更新されたら実行されます。

1. Qiita / DEV.to に API 投稿
2. `Zenn` リポジトリに記事をコピーしてコミット
3. Vercelが自動ビルド

```yaml
# .github/workflows/publish.yml（抜粋）
on:
  push:
    paths:
      - "content/**/*.md"

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Publish to Qiita / Dev.to
        run: node scripts/publish.mjs
      - name: Sync to Zenn
        run: node scripts/toZenn.mjs
```

---

### 5.2 投稿スクリプトの中身

Node.jsでQiitaとDEV.toのAPIを叩いています。

```js
// scripts/publish.mjs（抜粋）
import matter from "gray-matter";
import fetch from "node-fetch";

const articles = getMarkdownFiles("content");
for (const file of articles) {
  const { data, content } = matter.read(file);
  if (data.draft) continue;

  if (data.publish_on.includes("qiita"))
    await postQiita(data, content);
  if (data.publish_on.includes("devto"))
    await postDevto(data, content);
}
```

---

## 6. 自前ブログ（Next.js + MDX）

Next.jsの`app/blog/[...slug]/page.tsx`でMarkdownをレンダリング。
MDX対応でコードブロックもハイライト。

```tsx
// app/blog/[...slug]/page.tsx
import { getPostBySlug } from "@/lib/posts";
import { MDXRemote } from "next-mdx-remote/rsc";

export default async function BlogPost({ params }) {
  const post = await getPostBySlug(params.slug);
  return (
    <article className="prose">
      <h1>{post.title}</h1>
      <MDXRemote source={post.content} />
    </article>
  );
}
```

---

## 7. ホスティング（Vercel）

* Vercel：GitHub連携で自動デプロイ

---

## 8. ここまでやってみて

正直、**Qiita / Zenn / DEV.to すべてを一元管理**できるのは快感。
Markdown一枚で一気に公開できるから、書くことに集中できる。

今後は：

* Vercel版のWebデザインをリッチにする
* OGP画像の自動生成
* 「Originally published at ...」リンク自動追加
* AI下書き生成（Cursor連携）****

このあたりを仕込んで、完全自動化まで持っていく予定です。

---

## 9. 参考リンク

* [Qiita API v2 Docs](https://qiita.com/api/v2/docs)
* [DEV.to API Docs](https://developers.forem.com/api)
* [Zenn CLI Docs](https://zenn.dev/zenn/articles/zenn-cli-guide)
* [Next.js Docs](https://nextjs.org/docs)
* [GitHub Actions](https://docs.github.com/actions)

---

## まとめ

> Markdownを1枚書いてpushするだけ。
> それがQiitaにも、Zennにも、自分のブログにも反映される。

もう「どこに書こうかな」って悩まなくていい。
あとは**書く習慣だけ**を磨けばいい。
