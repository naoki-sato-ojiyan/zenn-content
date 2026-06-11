---
title: "【まなびクエスト開発日記】S3・CloudFront・ECSでAWSにフルデプロイした"
emoji: "☁️"
type: "tech"
topics: ["aws", "cloudfront", "ecs", "s3", "django"]
published: true
---

## はじめに

前回はECS（Fargate）でDjangoをAWSにデプロイしました。今回はフロントエンドをS3+CloudFrontでデプロイし、バックエンドAPIとつなぎ合わせて、まなびクエストをインターネットで動かせるようにしました。

最終的に `https://d1dymxm4w0e1jw.cloudfront.net` でアクセスできる状態になりました。

## 今回やったこと

```
✅ S3バケット作成・フロントエンドをアップロード
✅ CloudFront作成（S3+ECSプロキシ）
✅ config.jsで環境ごとにAPIのURLを切り替え
✅ .env.prodで本番/ローカル環境を分離
✅ ドメイン取得（manabi-quest.com）
✅ Route53・ACM設定
```

## 構成図

```
ブラウザ
　↓ HTTPS
CloudFront（d1dymxm4w0e1jw.cloudfront.net）
　├── /* → S3（index.html・JS・画像など）
　└── /api/* → ECS（Django・8000番ポート）
                    ↓
                  RDS（PostgreSQL）
```

CloudFrontが「フロントエンドのファイル置き場（S3）」と「APIサーバー（ECS）」の両方への入口になっています。

## S3バケットの作成

フロントエンドの静的ファイルを置くバケットを作成します。

```bash
aws s3api create-bucket \
  --bucket manabi-quest-frontend \
  --region ap-northeast-1 \
  --create-bucket-configuration LocationConstraint=ap-northeast-1
```

静的ウェブサイトホスティングを有効にします。

```bash
aws s3 website s3://manabi-quest-frontend \
  --index-document index.html \
  --error-document index.html
```

## OAC（Origin Access Control）の作成

OACとは「CloudFrontだけがS3にアクセスできる通行証」です。S3を直接公開せず、必ずCloudFront経由でアクセスさせるセキュリティ設定です。

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "manabi-quest-oac",
    "Description": "OAC for manabi-quest-frontend",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```

## CloudFrontの作成

S3をオリジンにしたCloudFrontディストリビューションを作成します。

```bash
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "manabi-quest-2026",
    "Origins": {
      "Quantity": 1,
      "Items": [{
        "Id": "manabi-quest-s3",
        "DomainName": "manabi-quest-frontend.s3.ap-northeast-1.amazonaws.com",
        "S3OriginConfig": {"OriginAccessIdentity": ""},
        "OriginAccessControlId": "（OACのID）"
      }]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "manabi-quest-s3",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "AllowedMethods": {
        "Quantity": 2,
        "Items": ["GET", "HEAD"]
      }
    },
    "DefaultRootObject": "index.html",
    "Enabled": true,
    "Comment": "manabi-quest frontend"
  }'
```

`ViewerProtocolPolicy: redirect-to-https` でHTTPアクセスを自動でHTTPSにリダイレクトします。

## S3バケットポリシーの設定

CloudFrontからだけS3にアクセスできるようにポリシーを設定します。

```bash
aws s3api put-bucket-policy \
  --bucket manabi-quest-frontend \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "AllowCloudFrontAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::manabi-quest-frontend/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::（アカウントID）:distribution/（ディストリビューションID）"
        }
      }
    }]
  }'
```

`AWS:SourceArn` で自分のCloudFrontからのアクセスだけを許可しています。他人のCloudFront経由のアクセスは拒否されます。

## フロントエンドをS3にアップロード

```bash
# public/フォルダをアップロード
aws s3 sync ~/app/manabi-quest/public/ s3://manabi-quest-frontend/ \
  --region ap-northeast-1

# src/フォルダをアップロード
aws s3 sync ~/app/manabi-quest/src/ s3://manabi-quest-frontend/src/ \
  --region ap-northeast-1
```

:::message
`index.html`のscriptタグのパスは `../src/main.js`（ローカル用）から `./src/main.js`（S3用）に変更が必要でした。
:::

## config.jsで環境ごとにAPIのURLを切り替える

ローカルと本番でAPIのURLが異なるため、`config.js`で切り替えます。

```javascript
// src/config.js

// localhost以外（CloudFront）でアクセスされたら本番とみなす
const isProduction = window.location.hostname !== 'localhost';

const API_BASE_URL = isProduction
  ? 'https://（CloudFrontのドメイン）'
  : 'http://localhost:8000';

export default API_BASE_URL;
```

各シーン（LoginScene.js・FieldScene.js・BattleScene.js）でimportして使います。

```javascript
import API_BASE_URL from '../config.js';

// 使用例
const response = await fetch(`${API_BASE_URL}/api/player/`, { ... });
```

## Mixed Contentエラーへの対処

最初はECSのIPを直接指定していたため、以下のエラーが発生しました。

```
Mixed Content: The page at 'https://...' was loaded over HTTPS,
but requested an insecure resource 'http://（ECSのIP）:8000/api/register/'.
```

HTTPSのページからHTTPのAPIは呼べないというブラウザのセキュリティルールです。

解決策として、**CloudFrontのAPIプロキシ機能**を使いました。`/api/*` へのリクエストをCloudFrontがECSに転送するように設定することで、ブラウザはすべてHTTPSで通信できます。

```
ブラウザ
　↓ HTTPS（安全）
CloudFront
　↓ HTTP（内部通信なので問題なし）
ECS（Django）
```

CloudFrontの設定にECSオリジンとキャッシュビヘイビアを追加します。

```json
"Origins": {
  "Quantity": 2,
  "Items": [
    { "Id": "manabi-quest-s3", ... },
    {
      "Id": "manabi-quest-api",
      "DomainName": "（ECSのパブリックDNS）",
      "CustomOriginConfig": {
        "HTTPPort": 8000,
        "OriginProtocolPolicy": "http-only"
      }
    }
  ]
},
"CacheBehaviors": {
  "Quantity": 1,
  "Items": [{
    "PathPattern": "/api/*",
    "TargetOriginId": "manabi-quest-api",
    "ViewerProtocolPolicy": "https-only",
    "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"
  }]
}
```

`CachePolicyId: 4135ea2d-...` はAWSが用意している「キャッシュなし」ポリシーです。APIのレスポンスはキャッシュしてはいけないので必ずこれを使います。

## .env.prodで本番/ローカル環境を分離する

ローカル開発用の`.env`はそのままにして、本番用の`.env.prod`を別途作成します。

```
manabi-quest-api/
├── .env          ← ローカル開発用（DB_HOST=db）
└── .env.prod     ← 本番用（DB_HOST=RDSのエンドポイント）
```

`.env.prod`の主な設定：

```bash
DEBUG=False
DB_HOST=（RDSのエンドポイント）
ALLOWED_HOSTS=localhost,127.0.0.1,（CloudFrontのドメイン）,（ECSのDNS）
CORS_ALLOWED_ORIGINS=http://localhost:5500,https://（CloudFrontのドメイン）
```

`Dockerfile`で`.env.prod`を`.env`としてコピーします。

```dockerfile
FROM python:3.14-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
# 本番用.envを読み込む
COPY .env.prod .env
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

:::message alert
`.env.prod`には本番のDBパスワードなどが含まれるため、必ず`.gitignore`に追加してGitHubにpushしないようにしてください。
:::

新しいイメージをビルドしてECRにpushします。

```bash
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin \
  （ECRのURI）

docker build --platform linux/amd64 -t manabi-quest-api .
docker tag manabi-quest-api:latest （ECRのURI）:latest
docker push （ECRのURI）:latest
```

`--platform linux/amd64` はApple Silicon（M1/M2）のMacでビルドする際に必要です。

## ドメイン取得とDNS設定

`manabi-quest.com`をお名前.comで取得し、Route53に接続します。

### Route53ホストゾーン作成

```bash
aws route53 create-hosted-zone \
  --name manabi-quest.com \
  --caller-reference manabi-quest-2026
```

作成されたネームサーバー（4つ）をお名前.com Naviで設定します。反映まで24〜72時間かかります。

### ACMでSSL証明書を取得

CloudFront用の証明書は**us-east-1リージョン**で作成する必要があります。

```bash
aws acm request-certificate \
  --domain-name manabi-quest.com \
  --subject-alternative-names "*.manabi-quest.com" \
  --validation-method DNS \
  --region us-east-1
```

DNS検証レコードをRoute53に追加します。

```bash
aws acm describe-certificate \
  --certificate-arn （証明書ARN） \
  --region us-east-1 \
  --query 'Certificate.DomainValidationOptions'
```

返ってきたCNAMEレコードをRoute53に追加すれば検証が自動で完了します。

## 現在の課題：ECSのIPが変わる問題

ECSのFargateタスクはサービス更新や再起動のたびにパブリックIPが変わります。現在はCloudFrontのオリジンを手動で更新しています。

**根本的な解決策はALB（Application Load Balancer）の導入**です。ALBは固定のDNSを持ち、タスクのIPが変わっても自動で追跡してくれます。

```
現在:
CloudFront → ECSのIP（再起動のたびに変わる）

ALB導入後:
CloudFront → ALB（固定DNS） → ECSタスク（IPが変わっても自動追跡）
```

次回はALBを作成してこの問題を解決します。

## まとめ

| やったこと | 使ったサービス |
|-----------|-------------|
| フロントエンドの公開 | S3 + CloudFront |
| APIサーバーの公開 | ECS（Fargate） |
| HTTPSの解決 | CloudFrontプロキシ |
| 環境ごとのURL切り替え | config.js |
| 本番/ローカルの環境分離 | .env / .env.prod |
| ドメイン取得 | お名前.com |
| DNS設定 | Route53 |
| SSL証明書 | ACM |

次回：ALB作成・カスタムドメイン設定（manabi-quest.com）
