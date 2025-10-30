---
title: "マルチ投稿ブログを自動化してみた（第3回）— Vercel版に広告導入とAB最適化"
emoji: "📝"
type: "tech"
topics: ["automation","nextjs","vercel","adsense","blog"]
published: true
---


この記事は「マルチ投稿ブログを自動化してみた」シリーズの第3回です。今回は、Vercel でホストしている Next.js ブログに Google AdSense を導入し、記事末尾で Responsive と Multiplex を 50/50 で AB テストするところまでを実装しました。プライバシー配慮（同意後のみ読み込み）と CLS を避ける配置に注力しています。

## 目的と方針

- 安定収益（AdSense）＋記事に紐づく高単価（アフィリエイト）を両立
- まずは最小構成で導入し、データが取れたら拡張（Anchor/国内ネットワーク等）
- CMP はミニマム（Cookie バナー）で開始し、将来 IAB-CMP に置き換え可能な構造

## 実装概要

- 同意ゲート: `ConsentLoader` と `CookieBanner` を追加し、同意=granted のときだけ `adsbygoogle.js` を挿入
- コンポーネント化: In‑article/Responsive/Multiplex の 3 種を用意
- 配置: 記事上=In‑article、記事末尾=AB（Responsive vs Multiplex）、一覧/タグ下部=Responsive（低頻度）
- AB: `BottomAdAB` が localStorage でユーザーを 50/50 に固定わり当て
- パフォーマンス: `data-full-width-responsive=true`、`data-ad-format=fluid/autorelaxed`、遅延レンダリング
- 運用: `ads.txt` 追加・Vercel 環境変数（Preview/Production）でクライアント/スロットを管理

## 変更ファイル（主要）

- `src/components/CookieBanner.tsx`（最小バナー）
- `src/components/ConsentLoader.tsx`（同意後のみスクリプト挿入）
- `src/components/ads/AdSenseInArticle.tsx`
- `src/components/ads/AdSenseResponsive.tsx`
- `src/components/ads/AdSenseMultiplex.tsx`
- `src/components/ads/BottomAdAB.tsx`（記事末尾 AB）
- `src/app/blog/[slug]/page.tsx`（配置）
- `src/app/blog/page.tsx`, `src/app/tags/[tag]/page.tsx`（一覧/タグ下部）
- `src/app/privacy/page.tsx`, `src/app/disclaimer/page.tsx`
- `public/ads.txt`, `env.example`

## 必要な環境変数

環境変数は Vercel の Preview/Production 両方に設定し、ローカルは `.env.local` を使用。

```
NEXT_PUBLIC_ADSENSE_CLIENT=ca-pub-xxxxxxxxxxxxxxxx
NEXT_PUBLIC_ADSENSE_SLOT_INARTICLE=xxxxxxxxxx
NEXT_PUBLIC_ADSENSE_SLOT_RESPONSIVE=xxxxxxxxxx
NEXT_PUBLIC_ADSENSE_SLOT_MULTIPLEX=xxxxxxxxxx
```

## 実装ポイント（コード断片）

同意後のみスクリプトを読み込む:

```tsx
// ConsentLoader（抜粋）
{consent === 'granted' && (
  <Script src={`https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=${process.env.NEXT_PUBLIC_ADSENSE_CLIENT}`}
          strategy="afterInteractive" crossOrigin="anonymous" />
)}
```

記事末尾で AB（Responsive / Multiplex）:

```tsx
// BottomAdAB（概略）
const chosen = Math.random() < 0.5 ? 'multiplex' : 'responsive'
localStorage.setItem('ab-bottom-ad-variant', chosen)
return chosen === 'multiplex' ? <AdSenseMultiplex fullWidth /> : <AdSenseResponsive fullWidth />
```

## リリースフロー（メモ）

- feature ブランチ → develop に PR（CLI）→ マージ
- develop → main に PR（CLI）→ 承認後マージ
- 事前に `npm run build` を通し、Vercel 環境変数が設定済みか確認

## 運用と今後

- 1〜2 週間で RPM/CTR/フィル率を比較して勝者を固定化
- Auto Ads の Anchor は段階導入（まずは OFF で基線、必要なら一部 ON）
- アフィリエイトは記事種別で選択的に挿入（`<AffiliateBox />`）

次回は「記事テンプレートの改善と自動投稿の安定化（メタ・タグ生成の強化）」を予定しています。


