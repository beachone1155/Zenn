---
title: "Next.jsでQiita風の見出しアンカーリンクを実装する方法"
emoji: "📝"
type: "tech"
topics: ["nextjs","react","markdown","rehype"]
published: true
---


# Next.jsでQiita風の見出しアンカーリンクを実装する方法

## はじめに

技術ブログを運営していると、Qiitaのような見出しにホバーするとリンクアイコンが表示される機能が欲しくなりますよね。実は、これって意外と簡単に実装できるんです。

今回は、Next.jsと`rehype`を使って、Qiita風の見出しアンカーリンクを実装した際の経験を共有します。最初は位置がずれてしまったり、ハイドレーションエラーに悩まされたりしましたが、最終的には綺麗に動作するようになりました。

## 実装の背景

私のブログでは、Markdownで記事を書いて、`unified`と`rehype`を使ってHTMLに変換しています。見出しに自動でアンカーリンクを追加する機能は`rehype-autolink-headings`というプラグインで実現できるのですが、Qiitaのように見出しの左側にアイコンを表示するには、いくつか工夫が必要でした。

## 実装手順

### 1. 必要なパッケージのインストール

まず、必要なパッケージをインストールします。

```bash
npm install unified remark-parse remark-gfm remark-rehype rehype-stringify rehype-highlight rehype-slug rehype-autolink-headings
```

### 2. Markdownコンポーネントの実装

`rehype-autolink-headings`を使って、見出しに自動でリンクを追加します。ポイントは`behavior: 'prepend'`を使うことです。

```typescript:src/components/mdx/Markdown.tsx
import { unified } from 'unified';
import remarkParse from 'remark-parse';
import remarkGfm from 'remark-gfm';
import remarkRehype from 'remark-rehype';
import rehypeStringify from 'rehype-stringify';
import rehypeHighlight from 'rehype-highlight';
import rehypeSlug from 'rehype-slug';
import rehypeAutolinkHeadings from 'rehype-autolink-headings';

export function Markdown({ content, className }: MarkdownProps) {
  const htmlContent = unified()
    .use(remarkParse)
    .use(remarkGfm)
    .use(remarkRehype, { allowDangerousHtml: false })
    .use(rehypeSlug)
    .use(rehypeAutolinkHeadings, {
      behavior: 'prepend', // 見出しの内部（最初の子要素）にリンクを追加
      properties: {
        className: ['anchor-link'],
        'aria-label': '見出しへのリンク',
      },
      content() {
        // SVGアイコンを返す
        return {
          type: 'element',
          tagName: 'svg',
          properties: {
            className: ['anchor-link-icon'],
            width: '16',
            height: '16',
            viewBox: '0 0 16 16',
            fill: 'currentColor',
            'aria-hidden': 'true',
          },
          children: [
            {
              type: 'element',
              tagName: 'path',
              properties: {
                d: 'M7.775 3.275a.75.75 0 001.06 1.06l1.25-1.25a2 2 0 112.83 2.83l-2.5 2.5a2 2 0 01-2.83 0 .75.75 0 00-1.06 1.06 3.5 3.5 0 004.95 0l2.5-2.5a3.5 3.5 0 00-4.95-4.95l-1.25 1.25zm-4.69 9.64a2 2 0 010-2.83l2.5-2.5a2 2 0 012.83 0 .75.75 0 001.06-1.06 3.5 3.5 0 00-4.95 0l-2.5 2.5a3.5 3.5 0 004.95 4.95l1.25-1.25a.75.75 0 00-1.06-1.06l-1.25 1.25a2 2 0 11-2.83-2.83l2.5-2.5z',
              },
              children: [],
            },
          ],
        };
      },
    })
    .use(rehypeHighlight, {
      detect: true,
      ignoreMissing: true,
    })
    .use(rehypeStringify, { allowDangerousHtml: true })
    .processSync(content);

  return (
    <div 
      className={className || ''}
      dangerouslySetInnerHTML={{ __html: String(htmlContent) }}
    />
  );
}
```

### 3. CSSスタイルの実装

見出しをFlexboxにして、アイコンとテキストを同じ行に配置します。また、アイコンは通常は非表示にして、ホバー時に表示するようにします。

```css:src/app/globals.css
/* Qiita風の見出しアンカーリンク */
.article-content .anchor-link,
.prose .anchor-link {
  display: inline-flex;
  align-items: center;
  text-decoration: none;
  color: #94a3b8;
  opacity: 0; /* 通常は非表示 */
  transition: opacity 0.2s, color 0.2s;
  flex-shrink: 0;
}

.article-content .anchor-link-icon,
.prose .anchor-link-icon {
  width: 16px;
  height: 16px;
  display: inline-block;
}

/* ホバー時にアイコンを表示 */
.article-content h2:hover .anchor-link,
.article-content h3:hover .anchor-link,
.article-content h4:hover .anchor-link,
.prose h2:hover .anchor-link,
.prose h3:hover .anchor-link,
.prose h4:hover .anchor-link {
  opacity: 1;
}

.article-content .anchor-link:hover,
.prose .anchor-link:hover {
  color: #3b82f6; /* ホバー時は青色に */
  opacity: 1;
}

/* 見出しをFlexboxにして、アイコンとテキストを同じ行に配置 */
.article-content h2,
.article-content h3,
.article-content h4,
.prose h2,
.prose h3,
.prose h4 {
  color: inherit;
  text-decoration: none;
  position: relative;
  padding-bottom: 0.3em;
  border-bottom: 1px solid #e2e8f0; /* 下線を追加 */
  margin-top: 1.5em;
  margin-bottom: 0.8em;
  display: flex; /* Flexboxに変更 */
  align-items: center;
  gap: 0.5rem; /* アイコンとテキストの間隔 */
}

.dark .article-content h2,
.dark .article-content h3,
.dark .article-content h4,
.dark .prose h2,
.dark .prose h3,
.dark .prose h4 {
  border-bottom-color: #475569;
}
```

## つまずいたポイントと解決方法

### 問題1: リンクアイコンが見出しの上に表示されてしまう

最初は`behavior: 'before'`を使っていたのですが、これだとリンクが見出し要素の**外側**（兄弟要素）に配置されてしまい、見出しの`display: flex`が効きませんでした。

**解決方法**: `behavior: 'prepend'`に変更することで、リンクが見出し要素の**内部**（最初の子要素）に配置されるようになり、Flexboxが正しく機能するようになりました。

```typescript
// ❌ これだと見出しの外側に配置される
behavior: 'before'

// ✅ これで見出しの内部に配置される
behavior: 'prepend'
```

### 問題2: ハイドレーションエラーが発生する

リンクアイコンをクリックすると、以下のようなエラーが発生していました。

```
A tree hydrated but some attributes of the server rendered HTML didn't match the client properties.
```

原因を調べたところ、クライアント側で見出しのIDを上書きする`AssignHeadingIds`コンポーネントが原因でした。`rehypeSlug`がサーバー側で生成したIDと、クライアント側で変更されたIDが一致しなかったのです。

**解決方法**: `AssignHeadingIds`コンポーネントを削除し、`TableOfContents`コンポーネントを修正して、DOMから実際の見出しIDを取得するようにしました。

```typescript:src/components/TableOfContents.tsx
useEffect(() => {
  // DOMから実際の見出し要素を取得してIDを取得（rehypeSlugが生成したIDを使用）
  const headings = Array.from(document.querySelectorAll('.article-content h2, .article-content h3')) as HTMLElement[]
  if (headings.length > 0) {
    const tocItems: TOCItem[] = headings.map((heading) => {
      const level = heading.tagName === 'H2' ? 2 : 3
      const text = heading.textContent?.trim() || ''
      const id = heading.id || '' // rehypeSlugが生成したIDをそのまま使用
      return { id, text, level }
    }).filter(item => item.id)
    setToc(tocItems)
  }
}, [content])
```

これで、サーバー側とクライアント側で同じIDが使用されるため、ハイドレーションエラーが解消されました。

## 実装のポイント

1. **`behavior: 'prepend'`を使う**: 見出しの内部にリンクを配置することで、Flexboxが正しく機能します。
2. **Flexboxでレイアウト**: 見出しを`display: flex`にして、アイコンとテキストを同じ行に配置します。
3. **ホバーで表示**: `opacity: 0`で通常は非表示にし、ホバー時に`opacity: 1`で表示します。
4. **IDの一貫性**: サーバー側とクライアント側で同じIDを使用することで、ハイドレーションエラーを防ぎます。

## まとめ

Qiita風の見出しアンカーリンクを実装するのは、思ったより簡単でした。`rehype-autolink-headings`の`behavior`オプションを適切に設定し、Flexboxでレイアウトすることで、綺麗に動作するようになりました。

ハイドレーションエラーには少し悩まされましたが、サーバー側とクライアント側で同じIDを使用することで解決できました。同じような問題に遭遇した方の参考になれば幸いです。

もし、さらにカスタマイズしたい場合は、SVGアイコンのデザインを変更したり、アニメーションを追加したりすることもできます。ぜひ試してみてください！

