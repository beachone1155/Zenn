---
title: "AI駆動型販売・在庫管理システム「OpsPilot Console」の実装と展望"
emoji: "📝"
type: "tech"
topics: ["nextjs","prisma","neon","typescript"]
published: true
---


# AI駆動型販売・在庫管理システム「OpsPilot Console」の実装と展望

## はじめに

こんにちは！今回は、現在開発中の「OpsPilot Console」というプロジェクトについてご紹介します。
これは、中小企業向けの販売・受発注・在庫管理を一つのNext.jsアプリケーションで完結させるという、野心的な試みです。

「業務システムって堅苦しくて使いにくい...」そんなイメージを払拭すべく、モダンな技術スタックでサクサク動く、そしてAIの力で業務を効率化するシステムを目指しています。

![OpsPilot Console Dashboard Mockup](https://beachone1155.vercel.app/images/ai-driven-sales-inventory-system-intro-dashboard.png)
*開発中のダッシュボード画面イメージ*

## アーキテクチャと技術スタック

OpsPilot Consoleは、**モノリポ構成**を採用しています。フロントエンドとAPIをNext.jsのApp Router内で同居させることで、開発効率を最大化しています。

主な技術スタックは以下の通りです。

- **Framework**: Next.js 16 (App Router) / React 19
- **Styling**: Tailwind CSS v4
- **Database**: Neon Postgres
- **ORM**: Prisma ORM
- **Language**: TypeScript

### なぜこのスタックなのか？

最大の理由は**「開発スピードと保守性の両立」**です。
Next.jsのApp Routerを使えば、サーバーサイドとクライアントサイドの境界を意識しつつも、シームレスにデータを扱えます。
また、DBにはサーバーレスPostgresであるNeonを採用。Prismaとの相性も抜群で、スキーマ定義から型安全なDB操作までがスムーズに行えます。

![OpsPilot Architecture Diagram](https://beachone1155.vercel.app/images/ai-driven-sales-inventory-system-intro-arch.png)
*OpsPilot Consoleのアーキテクチャ概要*

## 実装の詳細

### ディレクトリ構造

プロジェクトの構造は非常にシンプルです。

```
src/
  app/                   # App Router エントリ
    (dashboard)/         # 業務画面のルートグループ
    api/                 # Route Handlers
  components/            # UI コンポーネント
  lib/                   # 共通ユーティリティ
prisma/
  schema.prisma          # Prisma スキーマ
```

### Container Use / git worktree フロー

開発フローにもこだわりがあります。
すべての開発は**Container Use (Dagger MCP)** のコンテナ内で行い、ホストマシンの環境を汚しません。
また、`git worktree`を活用することで、ブランチの切り替えコストをゼロにしています。

```bash
# 例: 新機能開発の流れ
container-use new feature-xyz --branch feature/xyz
container-use enter feature-xyz
# ... 開発 ...
container-use merge feature-xyz
```

![Container Use Development Flow](https://beachone1155.vercel.app/images/ai-driven-sales-inventory-system-intro-flow.png)
*Container UseとGit Worktreeを活用した開発フロー*

これにより、複数のタスクを並行して進める際も、環境の競合を気にせず集中できます。

## 今後の展望 (Roadmap)

さて、ここからが本題です。現在は基礎的なCRUD機能の実装が進んでいますが、今後は以下の機能を追加していく予定です。

### 1. トランザクションと在庫履歴の実装

販売管理システムの肝となる、受注・発注のトランザクション処理と、それに伴う入出庫履歴の記録機能を実装します。
Prismaのトランザクション機能を活用し、データの整合性を担保しつつ、高速な処理を目指します。

### 2. APIの拡張とワークフロー

Route Handlersを拡張し、より複雑な業務ロジックをAPIとして提供します。
特に、発注承認フローなどのワークフロー機能は、システムの実用性を大きく高める重要な機能です。

### 3. 品質保証の自動化

現在、GitHub Issuesでも管理していますが、品質保証の自動化は急務です。

- **統合テストの追加**: VitestとPrisma Test DBを使って、API層の振る舞いを自動検証します（Issue #11）。
- **CI/CDパイプライン**: GitHub Actionsでlint、test、build、そしてPrismaのチェックを自動化し、常にデプロイ可能な状態を保ちます（Issue #12）。
- **E2Eテスト**: Playwrightを導入し、実際のユーザー操作を模したテストを行うことで、UIの不具合を未然に防ぎます。

## まとめ

OpsPilot Consoleは、単なる管理画面ではなく、**「開発者が楽しみながら作れる、ユーザーが快適に使える」**システムを目指しています。
モダンな技術を積極的に取り入れつつ、業務システムの要件もしっかり満たす。そんなバランスの取れた開発を続けていきます。

今後も開発の進捗や技術的な知見をブログで発信していきますので、ぜひチェックしてください！
