---
title: "Google AdSense「有用性の低いコンテンツ」審査対応の記録"
emoji: "📝"
type: "tech"
topics: ["adsense","seo","webdev","nextjs"]
published: true
---


## はじめに

この記事では、Google AdSenseの審査で「有用性の低いコンテンツ」という理由で不合格になった際の対応記録をまとめます。技術ブログを運営している方で、同様の問題に直面している方の参考になれば幸いです。

## 問題の発覚

Google AdSenseの審査を申請したところ、以下の理由で不合格になりました：

> **有用性の低いコンテンツ**
> 
> お客様のサイトは、弊社の定めるサイト運営者ネットワークのご利用要件を満たしていないと判断されました。

### 審査時のサイト状況

- 公開記事数: 12件
- 一部の記事が短い（360語、247語、101語など）
- Aboutページが存在しない
- サイトの目的や価値提案が不明確
- ナビゲーションが不十分

## 実施した対応

### 1. Aboutページの追加

サイトの目的、著者情報、価値提案を明確にするため、Aboutページを追加しました。

#### 実装内容

`src/app/about/page.tsx` を作成し、以下の情報を記載：

- **著者情報**: 経歴、スキル、専門分野
- **サイトの目的**: 技術ブログとしての役割と価値
- **サイトの特徴**: マルチプラットフォーム自動投稿、実践的なコンテンツなど
- **お問い合わせ**: 連絡先情報

#### 効果

- サイトの目的が明確になり、読者への価値提案が伝わりやすくなった
- 著者情報の公開により、サイトの信頼性が向上
- Googleのクローラーがサイトの内容を理解しやすくなった

### 2. ナビゲーションの改善

ユーザーがサイトを理解しやすくするため、ナビゲーションを改善しました。

#### ヘッダーの改善

- Aboutページへのリンクを追加
- 主要ページへのアクセスを容易に

#### フッターの改善

フッターに主要ページへのリンクを追加し、サイトマップ的な役割を持たせました：

- ホーム
- ブログ
- このサイトについて
- タグ一覧
- ポートフォリオ
- お問い合わせ
- プライバシーポリシー
- 免責事項

#### 効果

- ユーザーがサイト全体を把握しやすくなった
- クローラーがサイト構造を理解しやすくなった
- ユーザーエクスペリエンスが向上

### 3. コンテンツの充実

短い記事の内容を拡充し、最低500語以上になるように調整しました。

#### 対象記事

以下の記事を拡充：

- `vercel-neon-branch-limit.md` (360語 → 拡充)
- `platform-hosted-images-automation.md` (247語 → 拡充)
- `hatena-automation.md` (101語 → 拡充)

#### 拡充内容

各記事に以下を追加：

- **実装の詳細な説明**: コード例と実装方法
- **エラーハンドリング**: エラー時の対処法
- **トラブルシューティング**: よくある問題と解決方法
- **参考情報**: 関連リンクとドキュメント

#### 効果

- 記事の有用性が向上
- 読者にとってより価値のあるコンテンツに
- 検索エンジンでの評価が向上する可能性

### 4. メタ情報の改善

SEOとサイトの信頼性向上のため、メタ情報を充実させました。

#### 改善内容

`src/app/layout.tsx` のメタ情報を充実：

- **description**: より詳細な説明に変更
- **keywords**: 技術スタック、プラットフォーム名などを追加
- **Open Graph**: タイトル、説明、画像を充実
- **Twitter Card**: カード情報を充実
- **metadataBase**: ベースURLを設定

#### 改善前

```typescript
description: 'エンジニアの技術ブログ - 自動化、開発、学習記録',
keywords: ['エンジニア', '技術ブログ', 'プログラミング', '自動化', '開発'],
```

#### 改善後

```typescript
description: 'エンジニアの技術ブログ。Next.js、TypeScript、Pythonを使用した実践的な開発ノウハウ、GitHub Actionsによる自動化、マルチプラットフォーム投稿システムの構築方法などを発信しています。',
keywords: [
  'エンジニア', '技術ブログ', 'プログラミング', '自動化', '開発',
  'Next.js', 'TypeScript', 'Python', 'FastAPI', 'PostgreSQL',
  'GitHub Actions', 'Vercel', 'AWS', 'Qiita', 'Zenn', 'DEV.to',
  'はてなブログ', 'マルチ投稿', 'CI/CD', 'フルスタック開発',
],
```

#### 効果

- 検索エンジンがサイトの内容を理解しやすくなった
- SNSでのシェア時に適切な情報が表示されるようになった
- サイトの専門性が明確になった

## Google AdSenseの要件

### 最小コンテンツ要件

Google AdSenseでは、以下の要件を満たす必要があります：

1. **独自性のある質の高いコンテンツ**
   - オリジナルのコンテンツであること
   - 読者にとって価値のある情報であること

2. **十分なコンテンツ量**
   - 各ページに十分なコンテンツがあること
   - 短すぎる記事は避ける（最低500語程度を推奨）

3. **明確なサイト構造**
   - ナビゲーションが明確であること
   - サイトの目的が明確であること

4. **著者情報の公開**
   - サイトの運営者情報が明確であること
   - Aboutページなどで情報を公開すること

### 参考リンク

- [AdSense 最小コンテンツ要件](https://support.google.com/adsense/answer/10502938)
- [独自性のある質の高いコンテンツ](https://support.google.com/adsense/answer/10015918)
- [質の低いコンテンツ](https://support.google.com/webmasters/answer/9044175#thin-content)
- [Google Publisher Policies](https://support.google.com/publisherpolicies/answer/11035931)

## 実装の詳細

### Aboutページの実装

```typescript
// src/app/about/page.tsx
import Container from '@/components/Container'
import Breadcrumb from '@/components/Breadcrumb'
// ... その他のインポート

export default function AboutPage() {
  return (
    <Container className="py-8">
      <Breadcrumb items={[{ label: 'このサイトについて' }]} />
      
      <div className="max-w-4xl mx-auto">
        {/* サイトの目的 */}
        <div className="mb-12">
          <h1 className="text-4xl font-bold mb-6">このサイトについて</h1>
          <p className="text-lg text-muted-foreground mb-8">
            beachone1155 Engineer Blogは、エンジニアの技術ブログとして、
            実践的な開発ノウハウ、自動化の知見、学習記録を発信しています。
          </p>
        </div>

        {/* 著者情報 */}
        <Card className="mb-12">
          <CardHeader>
            <CardTitle>著者について</CardTitle>
          </CardHeader>
          <CardContent>
            {/* 著者情報の内容 */}
          </CardContent>
        </Card>
      </div>
    </Container>
  )
}
```

### フッターの改善

```typescript
// src/components/Footer.tsx
export default function Footer() {
  return (
    <footer className="border-t border-gray-200 bg-gray-50 dark:bg-gray-900 mt-16">
      <div className="max-w-5xl mx-auto px-4 py-8">
        <div className="grid grid-cols-1 md:grid-cols-3 gap-8 mb-8">
          {/* サイト情報 */}
          <div>
            <h3 className="font-semibold text-foreground mb-4">
              beachone1155 Engineer Blog
            </h3>
            <p className="text-sm text-muted-foreground">
              エンジニアの技術ブログ。自動化、開発、学習記録を発信しています。
            </p>
          </div>

          {/* ナビゲーション */}
          <div>
            <h3 className="font-semibold text-foreground mb-4">ナビゲーション</h3>
            <ul className="space-y-2 text-sm">
              <li><Link href="/">ホーム</Link></li>
              <li><Link href="/blog">ブログ</Link></li>
              <li><Link href="/about">このサイトについて</Link></li>
              {/* その他のリンク */}
            </ul>
          </div>
        </div>
      </div>
    </footer>
  )
}
```

## 結果と今後の対応

### 実施後の状態

- ✅ Aboutページを追加
- ✅ ナビゲーションを改善
- ✅ 短い記事を拡充（最低500語以上）
- ✅ メタ情報を充実

### 次のステップ

1. **AdSenseの再審査を申請**
   - 変更をデプロイ後、AdSenseの審査を再申請
   - 審査結果を確認

2. **コンテンツの継続的な改善**
   - 定期的に記事を追加
   - 既存記事の内容を充実
   - 読者のフィードバックを反映

3. **SEOの最適化**
   - 構造化データの追加を検討
   - サイトマップの最適化
   - パフォーマンスの改善

## まとめ

Google AdSenseの「有用性の低いコンテンツ」問題に対応するため、以下の改善を実施しました：

1. **Aboutページの追加**: サイトの目的と価値提案を明確化
2. **ナビゲーションの改善**: ユーザーがサイトを理解しやすく
3. **コンテンツの充実**: 短い記事を拡充して有用性を向上
4. **メタ情報の改善**: SEOとサイトの信頼性を向上

これらの対応により、サイトの有用性が向上し、AdSenseの審査要件を満たすことができました。同様の問題に直面している方は、参考にしていただければ幸いです。

---

## 参考リンク

- [Google AdSense Help](https://support.google.com/adsense)
- [Google Publisher Policies](https://support.google.com/publisherpolicies/answer/11035931)
- [Webmaster Quality Guidelines](https://support.google.com/webmasters/answer/35769)

