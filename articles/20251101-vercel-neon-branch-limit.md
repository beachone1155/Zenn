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

## まとめ

- **問題**: Vercel統合Neonでプレビューデプロイごとに自動ブランチが作成され、無料プランの上限（10個）に達した
- **解決**: 固定プレビューブランチを使用する設定で、新しいブランチの作成を防止
- **設定**: Vercel CLIでPreview環境に固定ブランチの`POSTGRES_URL`を設定
- **運用**: 未使用ブランチを定期的にクリーンアップ

これで、プレビューデプロイ時に新しいブランチが作成されなくなり、ブランチ数の上限問題を回避できます。

---

## 参考リンク

- [Neon Console](https://console.neon.tech)
- [Vercel CLI Documentation](https://vercel.com/docs/cli)
- [Neon Branching Documentation](https://neon.tech/docs/guides/branching)


