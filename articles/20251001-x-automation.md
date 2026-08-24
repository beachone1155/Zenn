---
title: "X(旧Twitter)にも自動投稿できるようにした話"
emoji: "📝"
type: "tech"
topics: ["nextjs","githubactions","twitter","automation","api"]
published: true
---


> Qiita、Zenn、DEV.toだけじゃ物足りない。  
> X(旧Twitter)にも自動投稿できるようにしました。

---

## 1. やりたかったこと

既存のマルチ投稿ブログシステムでは、Qiita、Zenn、DEV.toへの自動投稿に対応していましたが、SNSのX(旧Twitter)への自動投稿は実装されていませんでした。記事公開時にXにも自動投稿できるようにして、より多くの読者にリーチできるようにしたいと考えました。

## 2. 実装の課題

### X API v2の認証方式

X API v2でツイートを投稿するには、以下の認証方式のいずれかが必要です：

- **OAuth 2.0 User Context**: Access Tokenが必要（Bearer Tokenだけでは不可）
- **OAuth 1.0a User Context**: API Key、API Secret、Access Token、Access Token Secretの4つが必要

当初はBearer Token（Application-Only）での投稿を試みましたが、`POST /2/tweets`エンドポイントは`OAuth 2.0 Application-Only`認証をサポートしていないため、`403 Unsupported Authentication`エラーが発生しました。

### 必要な認証情報

OAuth 1.0a認証を使用する場合、以下の4つの値が必要です：

1. **API Key** (Consumer Key)
2. **API Secret** (Consumer Secret)
3. **Access Token**
4. **Access Token Secret**

これらはX Developer Portalの「Keys and tokens」ページから取得できます。

### 権限の設定

重要なのは、アプリの権限を「Read and write」に設定することです。デフォルトでは「Read only」になっているため、権限を変更した後、**必ずAccess TokenとAccess Token Secretを再生成**する必要があります。

## 3. 実装内容

### OAuth 1.0a認証の実装

`oauth-1.0a`パッケージを使用して、OAuth 1.0a署名を生成します：

```javascript
import OAuth from 'oauth-1.0a'
import crypto from 'crypto'

const oauth = new OAuth({
  consumer: {
    key: X_API_KEY,
    secret: X_API_SECRET
  },
  signature_method: 'HMAC-SHA1',
  hash_function(baseString, key) {
    return crypto.createHmac('sha1', key).update(baseString).digest('base64')
  }
})

const requestData = {
  url: 'https://api.x.com/2/tweets',
  method: 'POST'
}

const token = {
  key: X_ACCESS_TOKEN,
  secret: X_ACCESS_TOKEN_SECRET
}

const authHeader = oauth.toHeader(oauth.authorize(requestData, token))
```

### 投稿テキストの生成

280文字制限に対応するため、タイトル、要約、URLを組み合わせてテキストを生成し、制限を超える場合は要約を切り詰めます：

```javascript
function generateXPostText(post) {
  const siteUrl = process.env.NEXT_PUBLIC_SITE_URL || 'https://beachone1155.vercel.app'
  const articleUrl = post.canonical_url || `${siteUrl}/blog/${post.slug}`
  
  const urlLength = articleUrl.length
  const maxContentLength = 280 - urlLength - 4 // URL + 改行2つ + 余裕
  
  let text = post.title
  if (post.summary) {
    text += '\n\n' + post.summary
  }
  
  // 280文字制限を超える場合、要約を切り詰める
  if ((text + '\n\n' + articleUrl).length > 280) {
    const availableLength = maxContentLength - post.title.length - 2
    if (availableLength > 0) {
      text = post.title + '\n\n' + post.summary.slice(0, availableLength - 3) + '...'
    } else {
      text = post.title.slice(0, maxContentLength - 3) + '...'
    }
  }
  
  return text + '\n\n' + articleUrl
}
```

### 型定義の更新

`publish_on`に`'x'`を追加し、型定義を更新しました：

```typescript
export interface PostFrontmatter {
  publish_on: ('qiita' | 'zenn' | 'devto' | 'x')[]
  // ...
}
```

## 4. セットアップ手順

### 1. X Developer Portalで認証情報を取得

1. [X Developer Portal](https://developer.x.com/)にログイン
2. アプリを選択（または新規作成）
3. 「Settings」タブで「User authentication settings」を設定
   - 「App permissions」を「**Read and write**」に変更
   - 「Type of App」は「Web App, Automated App or Bot」を選択
   - 「Callback URL」と「Website URL」を設定
4. 「Keys and tokens」タブで以下を取得：
   - **Consumer Keys**: API KeyとAPI Secret
   - **Access Token and Secret**: 「Regenerate」をクリックして生成（**Read and write権限で**）

### 2. 環境変数の設定

#### ローカル環境（.env.local）

```env
X_API_KEY=your_x_api_key_here
X_API_SECRET=your_x_api_secret_here
X_ACCESS_TOKEN=your_x_access_token_here
X_ACCESS_TOKEN_SECRET=your_x_access_token_secret_here
```

#### GitHub Secrets

GitHub Actionsから使用する場合は、リポジトリのSettings > Secrets and variables > Actionsで以下を登録：

- `X_API_KEY`
- `X_API_SECRET`
- `X_ACCESS_TOKEN`
- `X_ACCESS_TOKEN_SECRET`

### 3. 記事のFrontmatterに`x`を追加

```markdown
---
publish_on:
  - qiita
  - zenn
  - devto
  - hatena
  - x
---
```

## 5. 実行結果

実装後、5つの既存記事をXに投稿したところ、4件が成功しました：

- ✅ Vercel Postgresで匿名コメント欄を実装する
- ✅ AIと無料APIで"マルチ投稿ブログ"を自動化してみた（第2回）
- ✅ マルチ投稿ブログを自動化してみた（第3回）
- ✅ SAA模試の後半バテを防ぐ
- ⚠️ 第1回の記事（既に投稿済みのため重複エラー。想定通りの内容）

投稿されたツイートの形式：

```
記事タイトル

記事の要約（280文字制限を考慮して自動調整）

https://beachone1155.vercel.app/blog/article-slug
```

## 6. 学んだこと

### OAuth 1.0a認証の重要性

X API v2でツイートを投稿するには、OAuth 1.0a認証またはOAuth 2.0 User Context認証が必要です。Bearer Token（Application-Only）では読み取り専用のため、投稿はできません。

### 権限変更後の再生成必須

アプリの権限を「Read and write」に変更した後は、**必ずAccess TokenとAccess Token Secretを再生成**する必要があります。古いトークンはRead Only権限のままなので、投稿しようとすると`401 Unauthorized`エラーが発生します。

### 文字数制限への対応

Xの280文字制限に対応するため、URLの長さを考慮して、タイトルと要約を動的に調整する必要があります。日本語は1文字=1文字としてカウントされるため、切り詰め処理を適切に実装しました。

## 7. 今後の改善点

- **二重投稿防止**: 既に投稿済みの記事を検知してスキップする機能
- **エラーハンドリングの強化**: より詳細なエラーメッセージとリトライ機能
- **投稿スケジューリング**: 特定の時間に投稿できるようにする機能

## まとめ

X(旧Twitter)への自動投稿機能を追加し、Qiita、Zenn、DEV.toに加えて、より多くの読者にリーチできるようになりました。OAuth 1.0a認証を使用して実装し、280文字制限にも対応しています。

これで、1つのMarkdownファイルから、複数のプラットフォームとXへの自動投稿が可能になりました。

