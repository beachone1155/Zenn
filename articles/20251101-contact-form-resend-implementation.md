---
title: "Next.js + Resend APIでお問い合わせフォームを実装した話"
emoji: "📝"
type: "tech"
topics: ["nextjs","resend","typescript","webdev"]
published: true
---


## はじめに

この記事では、Next.js（App Router）とResend APIを使用して、お問い合わせフォームを実装した過程をまとめます。フロントエンドのバリデーションから、バックエンドのメール送信、XSS対策、エラーハンドリングまで、実装の詳細を解説します。

## 実装の背景と目的

技術ブログに読者からのお問い合わせを受け付ける機能を追加したいと考えました。要件は以下の通りです：

- 名前、メールアドレス、メッセージを送信
- メールで通知を受け取る
- フロントエンドでバリデーション
- XSS対策を実装
- エラーハンドリングを適切に行う

## Resend APIの選択理由

メール送信サービスとしてResendを選択した理由：

- **無料プランが充実**: 月3,000通まで送信可能
- **シンプルなAPI**: 実装が簡単
- **Next.jsとの相性**: TypeScript対応、ドキュメントが充実
- **開発者フレンドリー**: 無料プランでも`onboarding@resend.dev`から送信可能

## Resend APIの設定

### 1. アカウント作成とAPIキー取得

1. [Resend](https://resend.com/) にアクセスしてアカウントを作成
2. Dashboard → API Keys → Create API Key
3. 表示されたAPIキーをコピー（一度しか表示されないため注意）

### 2. 環境変数の設定

#### ローカル開発環境（.env.local）

```env
RESEND_API_KEY=re_xxxxxxxxxxxxx
CONTACT_EMAIL=your-email@example.com
RESEND_FROM_EMAIL=onboarding@resend.dev  # オプション（未設定時はデフォルト値を使用）
```

#### Vercel環境変数の設定

```bash
# Production環境
echo "your-resend-api-key" | vercel env add RESEND_API_KEY production
echo "your-email@example.com" | vercel env add CONTACT_EMAIL production

# Preview環境
echo "your-resend-api-key" | vercel env add RESEND_API_KEY preview
echo "your-email@example.com" | vercel env add CONTACT_EMAIL preview

# Development環境
echo "your-resend-api-key" | vercel env add RESEND_API_KEY development
echo "your-email@example.com" | vercel env add CONTACT_EMAIL development
```

### 3. Resendパッケージのインストール

```bash
npm install resend
```

## 実装の詳細

### フロントエンド（ContactForm.tsx）

#### コンポーネントの構造

```typescript
'use client'

import { useState } from 'react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card'
import { Send, Loader2 } from 'lucide-react'

export function ContactForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: '',
  })
  const [isSubmitting, setIsSubmitting] = useState(false)
  const [submitStatus, setSubmitStatus] = useState<'idle' | 'success' | 'error'>('idle')
  const [emailError, setEmailError] = useState<string>('')

  // メールアドレスのバリデーション（APIと同じロジック）
  const isValidEmail = (email: string): boolean => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    return emailRegex.test(email)
  }

  const handleEmailChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value
    setFormData({ ...formData, email: value })
    
    // リアルタイムバリデーション（入力中はエラーをクリア）
    if (emailError && value) {
      setEmailError('')
    }
  }

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    
    // メールアドレスのバリデーション
    if (!isValidEmail(formData.email)) {
      setEmailError('正しいメールアドレスを入力してください（例: example@email.com）')
      return
    }

    setEmailError('')
    setIsSubmitting(true)
    setSubmitStatus('idle')

    try {
      const response = await fetch('/api/contact', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(formData),
      })

      if (response.ok) {
        setSubmitStatus('success')
        setFormData({ name: '', email: '', message: '' })
        setEmailError('')
      } else {
        const data = await response.json().catch(() => ({}))
        if (data.error === 'Invalid email address') {
          setEmailError('正しいメールアドレスを入力してください（例: example@email.com）')
        }
        setSubmitStatus('error')
      }
    } catch (error) {
      console.error('Failed to submit contact form:', error)
      setSubmitStatus('error')
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <Card>
      <CardHeader>
        <CardTitle>お問い合わせ</CardTitle>
        <CardDescription>
          ご質問やご要望がございましたら、お気軽にお問い合わせください
        </CardDescription>
      </CardHeader>
      <CardContent>
        <form onSubmit={handleSubmit} className="space-y-4">
          {/* フォームフィールド */}
        </form>
      </CardContent>
    </Card>
  )
}
```

#### メールアドレスバリデーション

フロントエンドでメールアドレスのバリデーションを実装し、ユーザーに即座にフィードバックを提供します：

```typescript
// メールアドレスのバリデーション（APIと同じロジック）
const isValidEmail = (email: string): boolean => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return emailRegex.test(email)
}

// エラーメッセージの表示
{emailError && (
  <p className="mt-1 text-sm text-red-600">{emailError}</p>
)}
```

### バックエンド（/api/contact）

#### API Routeの実装

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { Resend } from 'resend'

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const { name, email, message } = body

    // バリデーション
    if (!name || !email || !message) {
      return NextResponse.json(
        { error: 'All fields are required' },
        { status: 400 }
      )
    }

    if (!isValidEmail(email)) {
      return NextResponse.json(
        { error: 'Invalid email address' },
        { status: 400 }
      )
    }

    // Resend APIキーの確認
    const resendApiKey = process.env.RESEND_API_KEY
    const contactEmail = process.env.CONTACT_EMAIL

    if (!resendApiKey || !contactEmail) {
      // 開発環境ではログ出力のみ、本番環境ではエラーを返す
      if (process.env.NODE_ENV === 'production') {
        return NextResponse.json(
          { error: 'Email service is not configured' },
          { status: 500 }
        )
      }
      console.log('Contact form submission (dev mode):', { name, email, message })
      return NextResponse.json({
        success: true,
        message: 'Contact form submitted successfully (dev mode - email not sent)',
      })
    }

    // メール送信
    const resend = new Resend(resendApiKey)
    
    // HTMLエスケープ（XSS対策）
    const escapeHtml = (text: string) => {
      return text
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#039;')
    }

    const escapedName = escapeHtml(name)
    const escapedEmail = escapeHtml(email)
    const escapedMessage = escapeHtml(message).replace(/\n/g, '<br>')

    // 送信元メールアドレス（環境変数で設定可能）
    const fromEmail = process.env.RESEND_FROM_EMAIL || 'onboarding@resend.dev'

    const emailResult = await resend.emails.send({
      from: fromEmail,
      to: contactEmail,
      replyTo: email, // 返信先を送信者のメールアドレスに設定
      subject: `お問い合わせ: ${escapedName}`,
      html: `
        <h2>お問い合わせ内容</h2>
        <p><strong>名前:</strong> ${escapedName}</p>
        <p><strong>メール:</strong> ${escapedEmail}</p>
        <p><strong>メッセージ:</strong></p>
        <p>${escapedMessage}</p>
        <hr>
        <p style="color: #666; font-size: 12px;">
          このメールは <a href="https://beachone1155.vercel.app/contact">beachone1155 Engineer Blog</a> のお問い合わせフォームから送信されました。
        </p>
      `,
    })

    if (emailResult.error) {
      console.error('Resend API error:', emailResult.error)
      return NextResponse.json(
        { error: 'Failed to send email' },
        { status: 500 }
      )
    }

    console.log('Contact form email sent successfully:', emailResult.data?.id)
    
    return NextResponse.json({
      success: true,
      message: 'Contact form submitted successfully',
    })
  } catch (error) {
    console.error('Contact form error:', error)
    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}

function isValidEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  return emailRegex.test(email)
}
```

## XSS対策（HTMLエスケープ）

ユーザー入力のHTMLタグをエスケープして、XSS攻撃を防ぎます：

```typescript
const escapeHtml = (text: string) => {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;')
}

// 使用例
const escapedName = escapeHtml(name)
const escapedMessage = escapeHtml(message).replace(/\n/g, '<br>')
```

これにより、`<script>alert('XSS')</script>`のような入力も安全に処理されます。

## エラーハンドリング

### フロントエンド

```typescript
// 送信成功時
{submitStatus === 'success' && (
  <div className="p-3 bg-green-50 border border-green-200 rounded-md text-green-800 text-sm">
    お問い合わせありがとうございます。正常に送信されました。
  </div>
)}

// 送信失敗時
{submitStatus === 'error' && (
  <div className="p-3 bg-red-50 border border-red-200 rounded-md text-red-800 text-sm">
    送信に失敗しました。しばらく時間をおいて再度お試しください。
  </div>
)}
```

### バックエンド

- 環境変数未設定時の処理（開発環境ではログ出力のみ）
- Resend APIエラー時の処理
- ネットワークエラー時の処理

## テスト項目と確認事項

### 1. 基本動作の確認

- ✅ フォーム送信が正常に動作するか
- ✅ メールが正しく送信されるか
- ✅ 送信成功メッセージが表示されるか

### 2. バリデーションの確認

- ✅ 空のフィールドで送信 → エラー表示
- ✅ 不正なメールアドレス（`a@a`など） → エラー表示
- ✅ 正しいメールアドレス → 送信成功

### 3. XSS対策の確認

- ✅ `<script>`タグを含む入力 → エスケープされる
- ✅ HTMLタグを含む入力 → エスケープされる
- ✅ 特殊文字（`&`, `<`, `>`, `"`, `'`） → エスケープされる

### 4. エラーハンドリングの確認

- ✅ ネットワークエラー時の処理
- ✅ Resend APIエラー時の処理
- ✅ 環境変数未設定時の処理

### 5. 本番環境での確認

- ✅ Vercel環境変数が正しく設定されているか
- ✅ 本番環境でメール送信が正常に動作するか
- ✅ エラーログが適切に記録されるか

## 今後の改善点

### 1. 独自ドメインの設定

現在は`onboarding@resend.dev`を使用していますが、独自ドメインを設定することで：

- 送信元の信頼性が向上
- ブランドの一貫性が保たれる
- メールの到達率が向上する可能性

設定方法：

1. Resend Dashboardでドメインを追加
2. DNSレコードを設定（SPF、DKIM、CNAME）
3. 環境変数`RESEND_FROM_EMAIL`に設定（例: `contact@yourdomain.com`）

### 2. レート制限の実装

スパム対策として、レート制限を実装することを検討：

```typescript
// IPアドレスベースのレート制限
const rateLimit = new Map<string, number>()

const checkRateLimit = (ip: string): boolean => {
  const now = Date.now()
  const lastRequest = rateLimit.get(ip) || 0
  const timeDiff = now - lastRequest
  
  if (timeDiff < 60000) { // 1分以内
    return false
  }
  
  rateLimit.set(ip, now)
  return true
}
```

### 3. reCAPTCHAの追加

ボット対策として、Google reCAPTCHAを追加することも検討できます。

### 4. メールテンプレートの改善

現在のHTMLメールを、より洗練されたテンプレートに改善することも可能です。

## まとめ

Next.jsとResend APIを使用して、お問い合わせフォームを実装しました。主なポイントは以下の通りです：

- **フロントエンド**: リアルタイムバリデーション、エラーメッセージ表示
- **バックエンド**: Resend APIを使用したメール送信、XSS対策
- **エラーハンドリング**: 適切なエラーメッセージとログ記録
- **セキュリティ**: HTMLエスケープによるXSS対策

無料プランでも十分に使える実装となっており、個人ブログや小規模なサイトに適しています。同様の機能を実装する際の参考になれば幸いです。

---

## 参考リンク

- [Resend Documentation](https://resend.com/docs)
- [Next.js API Routes](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Resend API Reference](https://resend.com/docs/api-reference/emails/send-email)

