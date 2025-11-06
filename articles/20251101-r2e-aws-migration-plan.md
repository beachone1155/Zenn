---
title: "R2E APIをAWSに移行するプラン：Render + Vercel + Neon から AWS への移行戦略"
emoji: "📝"
type: "tech"
topics: ["aws","migration","ecs","rds"]
published: true
---


## はじめに

 [以前の記事](https://beachone1155.vercel.app/blog/r2e-frontend-implementation)でR2E（Research→Experience）API を Render（FastAPI）+ Vercel（Next.js）+ Neon（PostgreSQL/pgvector）の構成で公開していました。しかし無料枠の制約やコールドスタートで初回 30〜60 秒かかる体験は、プロダクション運用には耐えません。そこで、私は本番運用を見据えて、同機能を **AWS へ段階的に移行する** 計画を立てました。本稿は、私がこの移行で採用する構成、具体的な手順、運用の要点をまとめたプランです。

## 現在の構成の整理

現在の構成は以下の通りです：

![R2E API アーキテクチャ構成図](/images/r2e-architecture.drawio.png)

### フロントエンド
- **Next.js on Vercel**: ブログサイトと `/portfolio/r2e` のデモUI
- **API Route**: `/api/r2e/[...path]` でバックエンドにプロキシ

### バックエンド
- **FastAPI on Render**: Docker コンテナでデプロイ
- **URL**: `https://research-to-experience-api.onrender.com`
- **コールドスタート**: 15分間アクセスがないとスリープ、初回起動に30〜60秒

### データベース
- **Neon PostgreSQL**: クラウド PostgreSQL（`pgvector` 拡張対応）
- **ベクトル次元**: 1536（OpenAI `text-embedding-3-small` 対応）

### 課題
1. Render 無料枠のコールドスタート問題（30〜60秒の待機）
2. スケーラビリティの制限
3. 監視・ログ機能の不足
4. データベースとアプリケーションの分離（Neon は別サービス）

## AWS移行の目的とメリット

### メリット
1. **コールドスタートの解消**: ECS Fargate は常時起動可能（コストとトレードオフ）
2. **スケーラビリティ**: オートスケーリング対応
3. **統合監視**: CloudWatch でログ・メトリクスを一元管理
4. **セキュリティ**: VPC 内でリソースを分離、Secrets Manager で機密情報管理
5. **高可用性**: マルチAZ対応、自動バックアップ

### デメリット・考慮事項
- **コスト**: 無料枠から月額約 $80-120 へ（小規模運用）
- **運用複雑度**: インフラ管理の知識が必要
- **セットアップ工数**: 初期構築に時間がかかる

## 私が採用するAWS構成

![R2E API アーキテクチャ構成図(AWS版)](/images/r2e-aws-migration-plan.drawio.png)

### フロントエンド

フロントエンドは **AWS Amplify** を第一候補として採用します。Next.js の SSR/SSG をサポートし、GitHub 連携で CI/CD と環境変数管理が一か所にまとまります。もし SSR が不要なページのみを分離できる場合は、コスト最適化のために **S3 静的ホスティング + CloudFront** に切り替えるオプションも用意します（本計画では Amplify を本線、S3 + CloudFront を代替案として維持）。

### バックエンド

アプリケーション層は **ECS Fargate + ALB** に移行します。既存の Dockerfile を無変更で使え、常時起動でコールドスタートを排除できます。スケーリングは CPU/メモリのターゲット追跡で運用し、最初は 1 タスク常時起動、負荷に応じて最大 5 までの自動拡張を設定します。将来的に構成簡素化が必要になれば App Runner への置き換え、超低トラフィック帯では Lambda 実行（ただし本APIの性質上は非本命）も検討余地として残します。

### データベース

データ層は **RDS for PostgreSQL 15+（pgvector有効）** を採用します。自動バックアップとマルチAZで可用性を確保しつつ、Neon から `pg_dump/pg_restore` で段階移行します。Aurora Serverless v2 は将来のコスト最適化候補として調査継続に留めます。

### その他のAWSサービス

- **VPC**（パブリック/プライベート分離）
- **ALB**（HTTPS終端 + 負荷分散）
- **ECR**（Dockerイメージ保管）
- **CloudWatch**（ログ/メトリクス/アラーム）
- **Secrets Manager**（DATABASE_URL/OPENAI_API_KEY を安全管理）
- **NAT Gateway**（OpenAI/arXiv への外部アクセス）

## AWS移行後のアーキテクチャ

### 全体構成

![R2E API AWS移行後のアーキテクチャ構成図](/images/r2e-aws-migration-plan.drawio.png)

### データフロー

1. **クエリ受信**: Amplify（または CloudFront+S3）→ Next.js API Route → ALB → ECS Fargate
2. **論文取得**: ECS Fargate → NAT Gateway → arXiv
3. **埋め込み生成**: ECS Fargate → NAT Gateway → OpenAI API
4. **ベクトル検索**: ECS Fargate → RDS PostgreSQL (pgvector)
5. **要約生成**: ECS Fargate → NAT Gateway → OpenAI API
6. **レスポンス返却**: ECS Fargate → ALB → API Route → フロントエンド

## 移行手順（詳細）

### ステップ1: データベース移行（RDS作成）

#### 1-1. RDS for PostgreSQL インスタンス作成

```bash
# AWS CLI での作成例（コンソールでも可）
aws rds create-db-instance \
  --db-instance-identifier r2e-postgres \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.4 \
  --master-username postgres \
  --master-user-password <PASSWORD> \
  --allocated-storage 20 \
  --storage-type gp3 \
  --vpc-security-group-ids sg-xxx \
  --db-subnet-group-name r2e-db-subnet-group \
  --backup-retention-period 7 \
  --multi-az \
  --publicly-accessible false
```

**パラメータグループ設定**:
1. パラメータグループを作成（`r2e-pg-params`）
2. `shared_preload_libraries` に `vector` を追加
3. インスタンスにパラメータグループを適用

```sql
-- パラメータグループ設定後、RDSに接続して拡張を有効化
CREATE EXTENSION IF NOT EXISTS vector;
```

#### 1-2. Neon から RDS へのデータ移行

```bash
# 1. Neon からダンプ取得
pg_dump -h ep-xxx.region.neon.tech -U user -d dbname \
  --no-owner --no-acl -F c -f r2e_backup.dump

# 2. RDS にリストア
pg_restore -h r2e-postgres.xxx.rds.amazonaws.com \
  -U postgres -d postgres \
  --no-owner --no-acl r2e_backup.dump
```

**ダウンタイム最小化**:
- メンテナンスウィンドウで実施
- または、レプリケーション設定（より複雑）

### ステップ2: バックエンド移行（ECS Fargate）

#### 2-1. ECR に Docker イメージをプッシュ

```bash
# ECR リポジトリ作成
aws ecr create-repository --repository-name r2e-api

# ログイン
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com

# イメージビルド
docker build -t r2e-api .

# タグ付け・プッシュ
docker tag r2e-api:latest <ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/r2e-api:latest
docker push <ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/r2e-api:latest
```

#### 2-2. ECS クラスター・タスク定義作成

**タスク定義** (`task-definition.json`):
```json
{
  "family": "r2e-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [
    {
      "name": "r2e-api",
      "image": "<ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/r2e-api:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:<ACCOUNT_ID>:secret:r2e/database-url"
        },
        {
          "name": "OPENAI_API_KEY",
          "valueFrom": "arn:aws:secretsmanager:ap-northeast-1:<ACCOUNT_ID>:secret:r2e/openai-api-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/r2e-api",
          "awslogs-region": "ap-northeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

```bash
# タスク定義登録
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

#### 2-3. ALB 作成とターゲットグループ設定

```bash
# ALB 作成
aws elbv2 create-load-balancer \
  --name r2e-api-alb \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-alb-xxx

# ターゲットグループ作成
aws elbv2 create-target-group \
  --name r2e-api-tg \
  --protocol HTTP \
  --port 8000 \
  --vpc-id vpc-xxx \
  --target-type ip \
  --health-check-path /health

# リスナー作成
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN> \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=<CERT_ARN> \
  --default-actions Type=forward,TargetGroupArn=<TG_ARN>
```

#### 2-4. ECS サービス作成

```bash
aws ecs create-service \
  --cluster r2e-cluster \
  --service-name r2e-api-service \
  --task-definition r2e-api \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=<TG_ARN>,containerName=r2e-api,containerPort=8000"
```

### ステップ3: フロントエンド（Amplify もしくは S3+CloudFront）

本番運用では Amplify を使います。SSRが不要なページのみ切り出す場合は S3+CloudFront で静的ホスティングに置き換えます。

Amplify の設定:

```bash
# Vercel 環境変数更新
vercel env rm BACKEND_URL production
echo -n "https://r2e-api-alb-xxx.ap-northeast-1.elb.amazonaws.com" | \
  vercel env add BACKEND_URL production
```

```bash
# Amplify プロジェクト作成
amplify init

# 環境変数設定（Console でも可）
amplify env add
# BACKEND_URL=https://r2e-api-alb-xxx.ap-northeast-1.elb.amazonaws.com
```

S3+CloudFront 案（静的配信）:

```bash
# S3 バケット作成（静的ホスティング）
aws s3 mb s3://r2e-frontend-bucket
aws s3 website s3://r2e-frontend-bucket --index-document index.html --error-document 404.html

# CloudFront ディストリビューション作成
aws cloudfront create-distribution --origin-domain-name r2e-frontend-bucket.s3.amazonaws.com
```

### ステップ4: ネットワーク設定（VPC）

#### 4-1. VPC 作成（既存のVPCを使用する場合はスキップ）

```bash
# VPC作成
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=r2e-vpc}]'

# パブリックサブネット作成
aws ec2 create-subnet \
  --vpc-id vpc-xxx \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-northeast-1a

# プライベートサブネット作成
aws ec2 create-subnet \
  --vpc-id vpc-xxx \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-northeast-1a
```

#### 4-2. インターネットゲートウェイ・NAT Gateway 設定

```bash
# インターネットゲートウェイ作成・アタッチ
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --internet-gateway-id igw-xxx --vpc-id vpc-xxx

# パブリックサブネットのルートテーブル設定
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxx

# NAT Gateway 作成（Elastic IP 必要）
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway \
  --subnet-id subnet-public-xxx \
  --allocation-id eipalloc-xxx

# プライベートサブネットのルートテーブル設定
aws ec2 create-route --route-table-id rtb-private-xxx --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxx
```

#### 4-3. セキュリティグループ設定

```bash
# ALB セキュリティグループ（HTTP/HTTPS 許可）
aws ec2 create-security-group \
  --group-name r2e-alb-sg \
  --description "R2E API ALB Security Group" \
  --vpc-id vpc-xxx

# ECS セキュリティグループ（ALBからのみ許可、NAT Gateway経由で外部アクセス）
aws ec2 create-security-group \
  --group-name r2e-ecs-sg \
  --description "R2E API ECS Security Group" \
  --vpc-id vpc-xxx

# RDS セキュリティグループ（ECSからのみ許可）
aws ec2 create-security-group \
  --group-name r2e-rds-sg \
  --description "R2E API RDS Security Group" \
  --vpc-id vpc-xxx
```

### ステップ5: Secrets Manager 設定

```bash
# データベース接続文字列
aws secretsmanager create-secret \
  --name r2e/database-url \
  --secret-string "postgresql://user:pass@r2e-postgres.xxx.rds.amazonaws.com:5432/dbname"

# OpenAI API キー
aws secretsmanager create-secret \
  --name r2e/openai-api-key \
  --secret-string "sk-..."
```

## コスト比較

### Render + Neon（現在の構成）

- **Render**: 無料枠（制限あり）
- **Neon**: 無料枠（制限あり）
- **合計**: $0/月（制限内）

### AWS移行後（小規模運用）

| サービス | 月額コスト目安 | 備考 |
|---------|--------------|------|
| RDS (db.t3.micro) | $15-30 | 無料枠なし |
| ECS Fargate (0.5 vCPU, 1GB) | $10-20 | 常時起動 |
| ALB | $16 | 固定 |
| NAT Gateway | $32 | 固定 |
| CloudWatch Logs | $2-5 | 使用量ベース |
| データ転送 | $1-5 | 少量 |
| **合計** | **約 $80-120/月** | |

### コスト最適化のヒント

1. **ECS Fargate のスケールダウン**: 夜間は desired-count を 0 に（ただしコールドスタート発生）
2. **NAT Gateway の代替**: NAT Instance（EC2 t3.micro）で約 $7/月（運用コスト増）
3. **RDS の停止**: 開発環境のみ、停止可能（ただし起動に時間がかかる）

## ハマりポイントと対応

### 1. pgvector 拡張のインストール

**問題**: RDS のパラメータグループで `shared_preload_libraries` を設定しても、拡張が有効にならない。

**対応**:
1. パラメータグループ作成時に `vector` を追加
2. **インスタンス再起動が必要**（パラメータ変更反映のため）
3. 接続後に `CREATE EXTENSION vector;` を実行

```sql
-- 確認
SELECT * FROM pg_extension WHERE extname = 'vector';
```

### 2. VPC 内からの外部APIアクセス

**問題**: プライベートサブネットの ECS タスクから OpenAI API にアクセスできない。

**対応**:
- **NAT Gateway 必須**: プライベートサブネットからインターネットへのルーティング
- セキュリティグループでアウトバウンド HTTPS (443) を許可
- コストが高いため、開発環境では NAT Instance を検討

### 3. ECS タスクのヘルスチェック失敗

**問題**: ALB のヘルスチェックが失敗し、タスクが起動しない。

**対応**:
- ヘルスチェックパス: `/health` を返すエンドポイントを実装
- ヘルスチェック間隔: 30秒（デフォルト）
- タイムアウト: 5秒（デフォルト）
- 成功閾値: 2回（デフォルト）

```python
# FastAPI のヘルスチェック実装例
@app.get("/health")
async def health():
    return {"status": "healthy"}
```

### 4. Secrets Manager の IAM 権限不足

**問題**: ECS タスクが Secrets Manager から値を取得できない。

**対応**:
- ECS タスク実行ロールに `SecretsManagerReadWrite` ポリシーをアタッチ
- または、最小権限ポリシー:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-northeast-1:<ACCOUNT_ID>:secret:r2e/*"
      ]
    }
  ]
}
```

### 5. RDS への接続タイムアウト

**問題**: ECS タスクから RDS に接続できない。

**対応**:
- セキュリティグループで RDS のポート（5432）を許可
- サブネットグループが正しく設定されているか確認
- `publicly-accessible` は `false` に設定（プライベートサブネット経由）

## 監視・運用

### CloudWatch Logs 設定

ECS タスクのログは自動的に CloudWatch Logs に送信されます（タスク定義で設定済み）。

```bash
# ロググループ確認
aws logs describe-log-groups --log-group-name-prefix /ecs/r2e-api

# ログストリーム確認
aws logs describe-log-streams --log-group-name /ecs/r2e-api
```

### CloudWatch Metrics 設定

**ECS メトリクス**:
- `CPUUtilization`: CPU 使用率
- `MemoryUtilization`: メモリ使用率
- `RunningTaskCount`: 実行中のタスク数

**RDS メトリクス**:
- `CPUUtilization`: CPU 使用率
- `DatabaseConnections`: データベース接続数
- `FreeableMemory`: 利用可能メモリ

### アラーム設定

```bash
# ECS CPU 使用率アラーム（80%超過）
aws cloudwatch put-metric-alarm \
  --alarm-name r2e-ecs-high-cpu \
  --alarm-description "ECS CPU utilization exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold
```

### オートスケーリング設定

```bash
# ターゲット追跡スケーリングポリシー
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/r2e-cluster/r2e-api-service \
  --min-capacity 1 \
  --max-capacity 5

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/r2e-cluster/r2e-api-service \
  --policy-name r2e-cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    }
  }'
```

## まとめ

### 移行のメリット

1. **コールドスタート解消**: 常時起動で即座に応答
2. **スケーラビリティ**: オートスケーリングで負荷に対応
3. **統合監視**: CloudWatch でログ・メトリクスを一元管理
4. **セキュリティ強化**: VPC 内でリソース分離、Secrets Manager で機密情報管理
5. **高可用性**: マルチAZ対応、自動バックアップ

### コスト vs 機能のトレードオフ

- **無料枠（Render + Neon）**: 制限あり、コールドスタートあり
- **AWS移行（$80-120/月）**: 制限なし、コールドスタートなし、高可用性

小規模運用では Render + Neon でも十分ですが、本格運用やスケールアップを想定する場合は AWS 移行を検討すべきです。

### 段階的移行

1. **フェーズ1**: RDS のみ移行（Neon → RDS）
2. **フェーズ2**: バックエンド移行（Render → ECS Fargate）
3. **フェーズ3**: フロントエンド移行（Vercel → Amplify、オプション）

各フェーズで動作確認を行い、問題があればロールバック可能にすることが重要です。

### 次のステップ

1. **Terraform/CDK 化**: インフラをコード化して再現性を確保
2. **CI/CD パイプライン**: GitHub Actions で自動デプロイ
3. **コスト最適化**: リザーブドインスタンス、Spot インスタンスの検討
4. **マルチリージョン**: 高可用性のための複数リージョン展開

AWS 移行は工数がかかりますが、長期的な運用を考えると投資価値があります。段階的に移行を進め、各ステップで動作確認を行います。

## 参考リンク

- [AWS ECS Fargate ドキュメント](https://docs.aws.amazon.com/ecs/latest/userguide/what-is-fargate.html)
- [RDS for PostgreSQL ドキュメント](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html)
- [pgvector on RDS 手順](https://github.com/pgvector/pgvector)
- [AWS Secrets Manager ドキュメント](https://docs.aws.amazon.com/secretsmanager/)
- [VPC ネットワーキング ベストプラクティス](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)

