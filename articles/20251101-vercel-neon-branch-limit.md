---
title: "Vercel統合Neonでブランチ制限に達した話と対処方法"
emoji: "📝"
type: "tech"
topics: ["vercel","neon","postgres","webdev","devops"]
published: true
---


## 問題の発覚

Vercelでプレビューデプロイを実行したところ、以下のエラーが発生しました：

```
engineer_blog_comments: Create database branch for deployment

Branch limit reached. Upgrade your plan or delete unused branches.
```

**Neonの無料プランでは、データベースブランチが最大10個まで**という制限があり、それに達してしまったようです。

---

## 原因の調査

### Vercel統合Neonの動作

VercelとNeonを統合すると、デフォルトで以下のような動作になります：

- **Production環境（本番）**: メインのデータベースブランチを使用
- **Preview環境（プレビュー）**: 各プレビューデプロイごとに**新しいブランチを自動作成**

つまり、PRを作成するたび、またはプレビューデプロイのたびに、Neon側で新しいデータベースブランチが作成されます。

### なぜブランチが増えるのか

1. 新しいPRを作成 → プレビューデプロイ実行
2. Vercelが自動的にNeonブランチを生成（例: `preview/feature/r2e-frontend`）
3. PRをマージしてもブランチは残る
4. 新しいPRを複数作成 → ブランチが増え続ける
5. 10個に達するとエラー

---

## 対処方法: 固定プレビューブランチを使用

すべてのプレビューデプロイで**同じ固定ブランチ**を使用するように設定することで、新しいブランチが作成されなくなります。

### 設定手順

#### 1. Neon Consoleで固定ブランチを作成

1. [Neon Console](https://console.neon.tech)にアクセス
2. プロジェクト「engineer_blog_comments」を選択
3. 左サイドバーの「Branches」をクリック
4. 「Create Branch」をクリック
5. ブランチ名: `preview`（任意の名前でOK）
6. 「Branch from」: `main`を選択
7. 作成ボタンをクリック

#### 2. 固定ブランチの接続文字列を取得

作成したブランチを選択し、Connection Detailsから`POSTGRES_URL`をコピーします。

```
postgresql://user:pass@ep-xxx-pooler.region.neon.tech/neondb?sslmode=require
```

#### 3. Vercel CLIで環境変数を設定

Preview環境とDevelopment環境に、固定ブランチの`POSTGRES_URL`を設定します：

```bash
# Preview環境に設定
echo "固定ブランチのPOSTGRES_URL" | vercel env add POSTGRES_URL preview

# Development環境にも設定（ローカル開発用）
echo "固定ブランチのPOSTGRES_URL" | vercel env add POSTGRES_URL development
```

これにより：
- ✅ **Production環境**: 本番データベース（mainブランチ）を使用
- ✅ **Preview環境**: 固定プレビューブランチを使用（新しいブランチは作成されない）
- ✅ **Development環境**: 固定プレビューブランチを使用

---

## 設定の確認

環境変数が正しく設定されたか確認：

```bash
vercel env ls | grep POSTGRES_URL
```

出力例：

```
POSTGRES_URL       Encrypted           Development         X ago
POSTGRES_URL       Encrypted           Preview              X ago
```

**重要**: Production環境には`POSTGRES_URL`を設定**しない**ことで、本番データベースが使用されます。

---

## 既存の未使用ブランチの削除

既に10個に達している場合は、未使用のブランチを削除する必要があります。

### Neon Consoleでの削除手順

1. [Neon Console](https://console.neon.tech) → プロジェクト「engineer_blog_comments」
2. 「Branches」をクリック
3. 未使用のプレビューブランチを確認（例: `preview/feature/*`, `preview/fix/*`など）
4. 各ブランチの右側「...」メニューから「Delete」を選択

**注意**: `main`ブランチは削除しないでください。本番環境で使用されています。

---

## 推奨される運用

### 定期的なクリーンアップ

- マージ済みPRのプレビューブランチは定期的に削除
- アクティブなPRのブランチのみ保持
- ブランチ数が上限（9個）に近づいたら未使用ブランチを削除

### スクリプト化（オプション）

削除手順をスクリプト化することも可能です：

```bash
# scripts/cleanup-neon-branches.sh
#!/bin/bash
# 未使用のNeonプレビューブランチを削除するためのスクリプト
# 使用法: Neon Console (https://console.neon.tech) で手動削除を行うか、
# Neon CLIを使用してブランチをリスト・削除します

set -e

echo "=========================================="
echo "Neonプレビューブランチ削除ガイド"
echo "=========================================="
echo ""
echo "このスクリプトは、Neon Consoleでのブランチ削除手順を案内します。"
echo ""
echo "手順:"
echo "1. https://console.neon.tech にアクセス"
echo "2. プロジェクト「engineer_blog_comments」を選択"
echo "3. 左サイドバーの「Branches」をクリック"
echo "4. 以下のブランチを削除（mainブランチは削除しない）:"
echo ""
echo "   削除対象（未使用のプレビューブランチ）:"
echo "   - preview/feature/*"
echo "   - preview/fix/*"
echo "   - preview/chore/*"
echo "   - その他の古いプレビューブランチ"
echo ""
echo "5. 各ブランチの右側にある「...」メニューから「Delete」を選択"
echo ""
echo "⚠️  注意:"
echo "   - mainブランチは削除しないでください"
echo "   - 現在使用中のプレビューブランチは削除しないでください"
echo "   - 削除後、新しいプレビューデプロイで新しいブランチが作成されます"
echo ""
echo "現在のNeonブランチ数が10/10の場合は、"
echo "上記の手順で削除してから新しいデプロイを行ってください。"
echo ""


```

---

## トラブルシューティング

### 環境変数が反映されない場合

Vercel CLIで環境変数を設定した後、すぐに反映されない場合があります。以下の手順を確認してください：

1. **環境変数の確認**:
   ```bash
   vercel env ls
   ```
   すべての環境（Production、Preview、Development）で`POSTGRES_URL`が正しく設定されているか確認します。

2. **デプロイの再実行**:
   環境変数を設定した後、新しいデプロイを実行してください：
   ```bash
   vercel --prod  # Production環境
   # または
   git push origin main  # Preview環境は自動デプロイ
   ```

3. **Vercel Dashboardでの確認**:
   [Vercel Dashboard](https://vercel.com/dashboard) → プロジェクト → Settings → Environment Variables で設定を確認できます。

### 固定ブランチが作成されない場合

固定ブランチを作成した後も、新しいプレビューデプロイでブランチが作成される場合は、以下の点を確認してください：

- Preview環境の`POSTGRES_URL`が正しく設定されているか
- 環境変数の値が固定ブランチの接続文字列になっているか
- Vercelの統合設定でNeonの自動ブランチ作成が無効になっているか

### データの分離について

固定プレビューブランチを使用する場合、すべてのプレビューデプロイが同じデータベースブランチを共有することになります。これは以下の点に注意が必要です：

- **データの競合**: 複数のPRが同時に同じデータベースを使用するため、データの競合が発生する可能性があります
- **テストデータの混在**: 異なるPRのテストデータが混在する可能性があります
- **推奨**: 本番データに影響を与えない軽量なテストのみを実行することを推奨します

本番データを完全に分離したい場合は、有料プランにアップグレードしてブランチ数の上限を増やすか、各PRで手動でブランチを作成する方法を検討してください。

## 代替案

### 1. 有料プランへのアップグレード

Neonの有料プランでは、ブランチ数の上限が大幅に増加します：

- **Freeプラン**: 10ブランチまで
- **Launchプラン**: 100ブランチまで
- **Scaleプラン**: 無制限

頻繁にプレビューデプロイを行う場合は、有料プランへのアップグレードを検討する価値があります。

### 2. 手動ブランチ管理

固定ブランチを使用せず、重要なPRのみ手動でブランチを作成する方法もあります：

1. Neon Consoleで手動でブランチを作成
2. そのブランチの接続文字列をVercelの環境変数に設定
3. PRマージ後にブランチを削除

この方法は手間がかかりますが、データの完全な分離が可能です。

### 3. データベースの分離

プレビューデプロイ用に別のNeonプロジェクトを作成する方法もあります：

- 本番用プロジェクト: Production環境で使用
- プレビュー用プロジェクト: Preview環境で使用

この方法では、完全に独立したデータベース環境を構築できますが、コストが2倍になります。

## パフォーマンスへの影響

固定プレビューブランチを使用することで、パフォーマンスへの影響はほとんどありません：

- **接続速度**: 固定ブランチも通常のブランチと同様のパフォーマンスを提供します
- **クエリ速度**: データ量が少ない場合、クエリ速度への影響は無視できます
- **スケーラビリティ**: プレビュー環境は通常、本番環境よりもトラフィックが少ないため、スケーラビリティの問題は発生しにくいです

ただし、大量のテストデータを投入する場合は、パフォーマンスの低下に注意してください。

## まとめ

- **問題**: Vercel統合Neonでプレビューデプロイごとに自動ブランチが作成され、無料プランの上限（10個）に達した
- **解決**: 固定プレビューブランチを使用する設定で、新しいブランチの作成を防止
- **設定**: Vercel CLIでPreview環境に固定ブランチの`POSTGRES_URL`を設定
- **運用**: 未使用ブランチを定期的にクリーンアップ
- **注意**: 固定ブランチを使用する場合、データの分離に注意が必要

これで、プレビューデプロイ時に新しいブランチが作成されなくなり、ブランチ数の上限問題を回避できます。データの完全な分離が必要な場合は、有料プランへのアップグレードや別のNeonプロジェクトの使用を検討してください。

---

## 参考リンク

- [Neon Console](https://console.neon.tech)
- [Vercel CLI Documentation](https://vercel.com/docs/cli)
- [Neon Branching Documentation](https://neon.tech/docs/guides/branching)
- [Neon Pricing](https://neon.tech/pricing)
- [Vercel Environment Variables](https://vercel.com/docs/concepts/projects/environment-variables)





