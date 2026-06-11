---
title: "【まなびクエスト開発日記】ALB・独自ドメイン・HTTPSでまなびクエストを本番公開した"
emoji: "🌐"
type: "tech"
topics: ["aws", "alb", "cloudfront", "route53", "django"]
published: false
---

## はじめに

前回の記事ではECS（Fargate）でDjangoをAWSにデプロイし、CloudFrontでS3+ECSを配信する構成を作りました。

しかし、この時点ではまだいくつか課題が残っていました。

- ECSタスクが再起動するたびにIPが変わり、CloudFrontのオリジンを手動更新する必要があった
- `https://d1dymxm4w0e1jw.cloudfront.net` というCloudFrontのデフォルトURLでしか動かない
- 独自ドメイン（manabi-quest.com）でアクセスできない
- HTTPSが設定されていない

この記事では以下の作業を行い、`https://manabi-quest.com` で本番公開するまでの手順を紹介します。

## 今回やること

```
✅ ALB（Application Load Balancer）作成
✅ ACM（AWS Certificate Manager）でSSL証明書を取得
✅ Route53でドメインのDNS設定
✅ CloudFrontに独自ドメイン・証明書を設定
✅ https://manabi-quest.com で本番公開
```

## 構成図

```
ユーザー
  ↓ https://manabi-quest.com
Route53（DNS）
  ↓
CloudFront（CDN・HTTPS終端）
  ├── /api/* → ALB → ECS（Django API）→ RDS
  └── /* → S3（フロントエンド静的ファイル）
```

## STEP1：ALB（Application Load Balancer）を作成する

### ALBとは？

ECSタスクの前に置くロードバランサーです。ECSタスクが再起動してIPが変わっても、ALBのDNS名は固定されます。

```
これまで：CloudFront → ECSのパブリックDNS（再起動のたびに変わる😢）
ALB導入後：CloudFront → ALB（固定） → ECS（IPが変わってもOK👍）
```

### ALB用セキュリティグループを作成

```bash
# VPC IDを取得
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text)

# ALB用セキュリティグループを作成
aws ec2 create-security-group \
  --group-name manabi-quest-alb-sg \
  --description "Security group for manabi-quest ALB" \
  --vpc-id $VPC_ID
```

HTTP（80）とHTTPS（443）を開放します。

```bash
# HTTPS（443）を開放
aws ec2 authorize-security-group-ingress \
  --group-id sg-0cdfc01fe92148d78 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# HTTP（80）を開放
aws ec2 authorize-security-group-ingress \
  --group-id sg-0cdfc01fe92148d78 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

### サブネットを確認

ALBは複数のアベイラビリティゾーン（AZ）に配置する必要があります。

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].[SubnetId,AvailabilityZone]" \
  --output table
```

```
-------------------------------------------------
|                DescribeSubnets                |
+---------------------------+-------------------+
|  subnet-0b8186f25fcf9e96c |  ap-northeast-1d  |
|  subnet-0b49c6d3a198bdd07 |  ap-northeast-1a  |
|  subnet-00df28d33b60a975f |  ap-northeast-1c  |
+---------------------------+-------------------+
```

### ALB本体を作成

```bash
aws elbv2 create-load-balancer \
  --name manabi-quest-alb \
  --subnets subnet-0b8186f25fcf9e96c subnet-0b49c6d3a198bdd07 subnet-00df28d33b60a975f \
  --security-groups sg-0cdfc01fe92148d78 \
  --scheme internet-facing \
  --type application
```

出力の `DNSName` をメモしておきます。

```
manabi-quest-alb-510685764.ap-northeast-1.elb.amazonaws.com
```

### ターゲットグループを作成

ALBがリクエストを転送する先（ECSタスク）を定義します。

```bash
aws elbv2 create-target-group \
  --name manabi-quest-tg \
  --protocol HTTP \
  --port 8000 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path /api/health/
```

**ポイント：**

| 設定 | 値 | 理由 |
|------|-----|------|
| `--target-type ip` | ip | Fargate（ECS）はIPタイプが必要 |
| `--health-check-path` | /api/health/ | ALBがECSの死活監視をするURL |

### ヘルスチェック用エンドポイントをDjangoに追加

`/api/health/` がまだDjangoに存在しないので追加します。

```python
# config/urls.py

from django.contrib import admin
from django.urls import path, include
from django.http import JsonResponse  # 追加

# 追加
def health_check(request):
    return JsonResponse({"status": "ok"})

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/health/', health_check),  # 追加
    path('api/', include('players.urls')),
]
```

## STEP2：ACM（SSL証明書）を取得する

### ACMとは？

AWS Certificate Managerの略で、無料でSSL/TLS証明書を発行できるサービスです。

### 証明書をリクエスト

CloudFrontに使う証明書は**us-east-1（バージニア）**リージョンに作成する必要があります。ALBに使う証明書は**ap-northeast-1（東京）**リージョンに作成します。

```bash
# CloudFront用（us-east-1）
aws acm request-certificate \
  --domain-name manabi-quest.com \
  --subject-alternative-names "*.manabi-quest.com" \
  --validation-method DNS \
  --region us-east-1

# ALB用（ap-northeast-1）
aws acm request-certificate \
  --domain-name manabi-quest.com \
  --subject-alternative-names "*.manabi-quest.com" \
  --validation-method DNS \
  --region ap-northeast-1
```

### DNS検証レコードをRoute53に追加

証明書の検証にはDNSレコードの追加が必要です。

```bash
aws acm describe-certificate \
  --certificate-arn <証明書ARN> \
  --region us-east-1 \
  --query "Certificate.DomainValidationOptions"
```

出力された `ResourceRecord` の `Name` と `Value` をRoute53に追加します。

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <ホストゾーンID> \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "_xxxx.manabi-quest.com.",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "_xxxx.acm-validations.aws."}]
      }
    }]
  }'
```

しばらく待つと `ValidationStatus: SUCCESS` になります。

## STEP3：ALBにリスナーを設定する

### HTTPSリスナーを作成

```bash
aws elbv2 create-listener \
  --load-balancer-arn <ALBのARN> \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=<ap-northeast-1の証明書ARN> \
  --default-actions Type=forward,TargetGroupArn=<ターゲットグループARN>
```

### HTTPリスナーを作成（ECSへ転送）

```bash
aws elbv2 create-listener \
  --load-balancer-arn <ALBのARN> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<ターゲットグループARN>
```

:::message
**なぜHTTPをHTTPSリダイレクトにしないのか？**
CloudFront → ALB間の通信はHTTPで行うため、ALBでHTTPSリダイレクトを設定するとCloudFrontからのリクエストが301リダイレクトされてしまいPOSTのボディが失われます。CloudFrontが外部からのHTTPSを終端し、ALBへはHTTPで転送する構成が正しい設定です。
:::

## STEP4：ECSサービスをALBと紐づける

既存のECSサービスを一度削除し、ALB付きで再作成します。

```bash
# タスク数を0にして停止
aws ecs update-service \
  --cluster manabi-quest-cluster \
  --service manabi-quest-api-service \
  --desired-count 0

# サービスを削除
aws ecs delete-service \
  --cluster manabi-quest-cluster \
  --service manabi-quest-api-service

# ALB付きで再作成
aws ecs create-service \
  --cluster manabi-quest-cluster \
  --service-name manabi-quest-api-service \
  --task-definition manabi-quest-api \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0b8186f25fcf9e96c,subnet-0b49c6d3a198bdd07,subnet-00df28d33b60a975f],securityGroups=[sg-0c28383f6d40e3b2c],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=<ターゲットグループARN>,containerName=manabi-quest-api,containerPort=8000"
```

ヘルスチェックが `healthy` になれば成功です。

```bash
aws elbv2 describe-target-health \
  --target-group-arn <ターゲットグループARN>
```

## STEP5：独自ドメインの設定（お名前.com + Route53）

### Route53でホストゾーンを作成

```bash
aws route53 create-hosted-zone \
  --name manabi-quest.com \
  --caller-reference $(date +%s)
```

出力されたネームサーバー（ns-xxx.awsdns-xx.xxx）をお名前.comのネームサーバー設定に登録します。反映まで最大72時間かかります。

## STEP6：CloudFrontに独自ドメイン・証明書を設定

現在のCloudFront設定を取得して更新します。

```bash
# 設定を取得
aws cloudfront get-distribution-config \
  --id <ディストリビューションID> \
  > /tmp/cf-config.json
```

以下の設定を追加します。

```python
import json

with open('/tmp/cf-config.json', 'r') as f:
    data = json.load(f)

config = data['DistributionConfig']

# カスタムドメインを追加
config['Aliases'] = {
    'Quantity': 2,
    'Items': ['manabi-quest.com', 'www.manabi-quest.com']
}

# ACM証明書を設定（us-east-1の証明書）
config['ViewerCertificate'] = {
    'ACMCertificateArn': '<us-east-1の証明書ARN>',
    'SSLSupportMethod': 'sni-only',
    'MinimumProtocolVersion': 'TLSv1.2_2021',
    'Certificate': '<us-east-1の証明書ARN>',
    'CertificateSource': 'acm'
}

# オリジンをALBに変更・HTTP接続に変更
for origin in config['Origins']['Items']:
    if origin['Id'] == 'manabi-quest-api':
        origin['DomainName'] = 'manabi-quest-alb-510685764.ap-northeast-1.elb.amazonaws.com'
        origin['CustomOriginConfig']['OriginProtocolPolicy'] = 'http-only'
        origin['CustomOriginConfig']['HTTPPort'] = 80

with open('/tmp/cf-config-new.json', 'w') as f:
    json.dump(config, f)
```

更新を反映します。

```bash
ETAG=$(aws cloudfront get-distribution-config \
  --id <ディストリビューションID> \
  --query "ETag" --output text)

aws cloudfront update-distribution \
  --id <ディストリビューションID> \
  --distribution-config file:///tmp/cf-config-new.json \
  --if-match $ETAG
```

### CloudFrontのAuthorizationヘッダー転送設定

CloudFrontはデフォルトで `Authorization` ヘッダーを削除してしまいます。DjangoのToken認証を使うため、このヘッダーをECSに転送する設定が必要です。

```python
# /api/*のキャッシュビヘイビアにOriginRequestPolicyを追加
for behavior in config['CacheBehaviors']['Items']:
    if behavior['PathPattern'] == '/api/*':
        # すべてのヘッダーをオリジンに転送するポリシー
        behavior['OriginRequestPolicyId'] = 'b689b0a8-53d0-40ab-baf2-68738e2966ac'
```

:::message
**`b689b0a8-53d0-40ab-baf2-68738e2966ac` とは？**
AWSが提供するマネージドポリシー「AllViewer」のIDです。すべてのヘッダー・クエリ文字列・Cookieをオリジンに転送します。
:::

## STEP7：Route53にAレコードを追加

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id <ホストゾーンID> \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "manabi-quest.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z2FDTNDATAQYW2",
            "DNSName": "d1dymxm4w0e1jw.cloudfront.net",
            "EvaluateTargetHealth": false
          }
        }
      }
    ]
  }'
```

:::message
**`Z2FDTNDATAQYW2` とは？**
CloudFrontのHostedZoneIDです。全リージョン・全ディストリビューションで共通の固定値です。
:::

## つまずいたポイント

### ACM証明書のリージョンが違うとエラーになる

ALBにはALBと同じリージョン（ap-northeast-1）の証明書が必要です。CloudFront用に`us-east-1`で作った証明書をALBに指定するとエラーになります。

```
An error occurred (ValidationError) when calling the CreateListener operation:
Certificate ARN 'arn:aws:acm:us-east-1:...' is not valid
```

**解決策：** ap-northeast-1にも同じドメインの証明書を作成する。

### CloudFrontがAuthorizationヘッダーを削除する

デフォルトのCloudFront設定では `Authorization` ヘッダーがオリジンに転送されず、Djangoのトークン認証が常に `401 Unauthorized` を返していました。

```
detail: "認証情報が含まれていません。"
```

**解決策：** `/api/*` のキャッシュビヘイビアに `OriginRequestPolicy` を設定して、すべてのヘッダーをオリジンに転送する。

### RDSへの接続タイムアウト

ECSタスクを再作成したとき、RDSのセキュリティグループにECSのSG（セキュリティグループ）が許可されていませんでした。

**解決策：** IPアドレスではなくSG同士で許可することで、タスクIPが変わっても自動的に接続できるようにする。

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <RDSのSG ID> \
  --protocol tcp \
  --port 5432 \
  --source-group <ECSのSG ID>
```

## 完成！

`https://manabi-quest.com` にアクセスしてログイン画面が表示され、ゲームがプレイできるようになりました🎉

## 今回の構成まとめ

| AWSサービス | 役割 |
|------------|------|
| Route53 | DNSルーティング（manabi-quest.com → CloudFront） |
| ACM | SSL/TLS証明書（無料） |
| CloudFront | CDN・HTTPS終端・S3/ALBへのルーティング |
| ALB | ECSタスクへのロードバランシング（IP固定化） |
| ECS（Fargate） | DjangoのAPIサーバー |
| RDS（PostgreSQL） | データベース |
| S3 | フロントエンド静的ファイル |

## 次回

次回はStripe連携で月額課金機能を実装する予定です。

## 関連記事

- 【まなびクエスト開発日記】AWSデプロイ構成を理解する（S3・CloudFront・ECS・RDS・Route53）
- 【まなびクエスト開発日記】AWSにRDSを作成してPostgreSQLをデプロイした
- 【まなびクエスト開発日記】ECS（Fargate）でDjangoをAWSにデプロイした