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

## 実装の詳細

### AtomPub APIとは

AtomPub（Atom Publishing Protocol）は、ブログやWikiなどのコンテンツを公開・更新するための標準プロトコルです。はてなブログでは、このプロトコルを使用して記事の投稿・更新が可能です。

### WSSE認証の実装

はてなブログのAtomPub APIでは、WSSE（Web Services Security）認証を使用します。WSSE認証は、ユーザー名、APIキー、現在時刻、ランダムな文字列（nonce）を使用して認証ヘッダーを生成します。

```javascript
// scripts/publish.mjs の実装例
import crypto from 'crypto'

function buildWsseHeader(username, apiKey) {
  const nonce = crypto.randomBytes(16).toString('base64')
  const created = new Date().toISOString()
  const digest = crypto
    .createHash('sha1')
    .update(nonce + created + apiKey)
    .digest('base64')
  
  return `UsernameToken Username="${username}", PasswordDigest="${digest}", Nonce="${nonce}", Created="${created}"`
}
```

### AtomEntry XMLの生成

はてなブログに投稿するためには、AtomEntry形式のXMLを生成する必要があります：

```javascript
function generateAtomEntry(post) {
  const atomEntry = `<?xml version="1.0" encoding="utf-8"?>
<entry xmlns="http://www.w3.org/2005/Atom"
       xmlns:app="http://www.w3.org/2007/app">
  <title>${escapeXml(post.title)}</title>
  <author><name>beachone1155</name></author>
  <content type="text/x-markdown">${escapeXml(post.content)}</content>
  <app:control>
    <app:draft>no</app:draft>
  </app:control>
  <category term="技術" />
</entry>`
  
  return atomEntry
}

function escapeXml(text) {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;')
}
```

### 投稿処理の実装

はてなブログへの投稿処理：

```javascript
async function publishToHatena(post, config) {
  const { username, apiKey, blogId } = config
  
  // WSSE認証ヘッダーを生成
  const wsseHeader = buildWsseHeader(username, apiKey)
  
  // AtomEntry XMLを生成
  const atomEntry = generateAtomEntry(post)
  
  // はてなブログのAtomPub APIエンドポイント
  const endpoint = `https://blog.hatena.ne.jp/${username}/${blogId}/atom/entry`
  
  const response = await fetch(endpoint, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/xml',
      'X-WSSE': wsseHeader
    },
    body: atomEntry
  })
  
  if (!response.ok) {
    throw new Error(`はてなブログ投稿失敗: ${response.statusText}`)
  }
  
  const responseXml = await response.text()
  // レスポンスから記事URLを抽出
  const entryUrl = extractEntryUrl(responseXml)
  
  return {
    platform: 'hatena',
    url: entryUrl,
    success: true
  }
}
```

### Frontmatterの処理

はてなブログ用にFrontmatterを処理する際の注意点：

- **タイトル**: XMLエスケープが必要
- **本文**: Markdown形式のまま投稿可能
- **タグ**: `<category>` 要素として追加
- **公開設定**: `<app:draft>` で制御（`yes`/`no`）

```javascript
function processHatenaPost(post) {
  return {
    title: post.title,
    content: post.content,
    tags: post.tags || [],
    draft: post.draft || false
  }
}
```

## GitHub Actionsワークフローの統合

はてなブログへの投稿をGitHub Actionsワークフローに統合：

```yaml
# .github/workflows/publish.yml の抜粋
- name: Publish to Hatena
  env:
    HATENA_USERNAME: ${{ secrets.HATENA_USERNAME }}
    HATENA_API_KEY: ${{ secrets.HATENA_API_KEY }}
    HATENA_BLOG_ID: ${{ secrets.HATENA_BLOG_ID }}
  run: |
    node scripts/publish.mjs
```

## エラーハンドリング

### 認証エラーの処理

WSSE認証が失敗した場合のエラーハンドリング：

```javascript
async function publishToHatenaWithRetry(post, config, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await publishToHatena(post, config)
    } catch (error) {
      if (error.message.includes('401') || error.message.includes('403')) {
        throw new Error(`認証エラー: ユーザー名またはAPIキーが正しくありません`)
      }
      
      if (i === maxRetries - 1) {
        throw error
      }
      
      // リトライ前に少し待機
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)))
    }
  }
}
```

### ネットワークエラーの処理

ネットワークエラーが発生した場合の処理：

```javascript
async function publishToHatenaWithErrorHandling(post, config) {
  try {
    return await publishToHatena(post, config)
  } catch (error) {
    if (error.code === 'ECONNREFUSED' || error.code === 'ETIMEDOUT') {
      throw new Error(`ネットワークエラー: はてなブログAPIに接続できませんでした`)
    }
    
    if (error.message.includes('400')) {
      throw new Error(`リクエストエラー: 送信したXMLが不正です`)
    }
    
    throw error
  }
}
```

## 他のプラットフォームとの比較

### Qiita/DEV.toとの違い

- **認証方式**: Qiita/DEV.toはBearer Token、はてなブログはWSSE認証
- **データ形式**: Qiita/DEV.toはJSON、はてなブログはXML（AtomEntry）
- **エンドポイント**: プラットフォームごとに異なるAPIエンドポイント

### Zennとの違い

- **同期方式**: ZennはGitHubリポジトリへのpush、はてなブログはAPI経由
- **画像処理**: Zennは画像ファイルをリポジトリにコピー、はてなブログはAPIでアップロード（別途実装が必要）

## トラブルシューティング

### よくあるエラーと対処法

1. **401 Unauthorized**: ユーザー名またはAPIキーが正しくありません
   - はてなブログの設定画面でAPIキーを再確認
   - 環境変数が正しく設定されているか確認

2. **400 Bad Request**: 送信したXMLが不正です
   - XMLのエスケープ処理を確認
   - Frontmatterの内容が正しいか確認

3. **403 Forbidden**: APIキーの権限が不足しています
   - はてなブログの設定でAtomPub APIを有効化
   - APIキーの権限を確認

### デバッグ方法

投稿処理をデバッグする際は、以下の情報をログに出力します：

```javascript
function debugHatenaPost(post, config) {
  console.log('はてなブログ投稿デバッグ情報:')
  console.log('- タイトル:', post.title)
  console.log('- 本文の長さ:', post.content.length)
  console.log('- タグ:', post.tags)
  console.log('- 下書き:', post.draft)
  console.log('- エンドポイント:', `https://blog.hatena.ne.jp/${config.username}/${config.blogId}/atom/entry`)
}
```

## 今後の拡張予定

### 画像アップロード機能

はてなブログでも画像を自動アップロードする機能を追加予定です：

- はてなフォトライフAPIを使用
- 記事内の画像パスを自動置換
- エラーハンドリングとフォールバック処理

### 記事更新機能

既存記事の更新機能も実装予定です：

- はてなブログの記事IDを管理
- 更新時は既存記事を上書き
- 新規投稿と更新を自動判別

## 補足: noteへの自動投稿について

note への自動投稿も検討しましたが、以下の理由で実装を見送りました：

1. **公式APIが存在しない**: noteは公式のAPIを提供していません
2. **非公式APIの不安定性**: 非公式APIは仕様変更のリスクがあります
3. **投稿機能の不明確性**: 非公式APIでも投稿機能が明確ではありません

一方、はてなブログは公式のAtomPub APIが提供されているため、安定して自動投稿できます。また、WSSE認証という標準的な認証方式を使用しているため、実装も比較的簡単です。

## まとめ

この記事では、はてなブログへの自動投稿機能を追加しました：

- **実装**: AtomPub APIとWSSE認証を使用
- **統合**: 既存のマルチ投稿パイプラインに統合
- **使い方**: Frontmatterの`publish_on`に`hatena`を指定するだけ
- **エラーハンドリング**: 認証エラーやネットワークエラーに対応

これにより、Qiita、Zenn、DEV.to、X（旧Twitter）に加えて、はてなブログへの自動投稿も可能になりました。1つのMarkdownファイルから、5つのプラットフォームに同時投稿できるようになりました。
