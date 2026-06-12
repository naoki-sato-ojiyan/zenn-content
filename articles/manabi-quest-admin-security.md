---
title: "【まなびクエスト開発日記】本番管理画面の修正とセキュリティ強化"
emoji: "🔐"
type: "tech"
topics: ["aws", "cloudfront", "django", "whitenoise", "security"]
published: true
---

## はじめに

前回まででまなびクエストをAWSに本番公開しました。今回は本番環境で発覚した問題を修正しつつ、セキュリティ強化を行いました。

## 今回やったこと

```
✅ CloudFrontに/admin/*・/static/*のビヘイビア追加
✅ WhiteNoise導入（管理画面のCSS修正）
✅ CSRF設定追加
✅ 本番スーパーユーザー作成（カスタムコマンド）
✅ 機密情報を環境変数化
✅ push前セキュリティチェックの整備
```

## 問題①：本番管理画面がAccess Denied

`https://manabi-quest.com/admin/` にアクセスすると以下のエラーが出ました。

```xml
<Error>
  <Code>AccessDenied</Code>
  <Message>Access Denied</Message>
</Error>
```

### 原因

CloudFrontのルーティング設定が原因でした。

```
/api/*  → ALB（ECS・Django）✅
/*      → S3（デフォルト）

/admin/ → S3に転送 → S3に該当ファイルなし → Access Denied ❌
```

`/admin/` と `/static/` へのリクエストをALB（Django）に転送するビヘイビアが存在しなかったのです。

### 対応：CloudFrontにビヘイビアを追加

まず現在の設定を取得します。

```bash
aws cloudfront get-distribution-config \
  --id （ディストリビューションID） \
  --query "DistributionConfig" \
  --output json > /tmp/cf-config.json
```

Pythonスクリプトで `/admin/*` と `/static/*` のビヘイビアを追加します。

```python
import json

with open('/tmp/cf-config.json', 'r') as f:
    config = json.load(f)

# /api/* のビヘイビアをテンプレートとしてコピー
api_behavior = config['CacheBehaviors']['Items'][0]

admin_behavior = json.loads(json.dumps(api_behavior))
admin_behavior['PathPattern'] = '/admin/*'

static_behavior = json.loads(json.dumps(api_behavior))
static_behavior['PathPattern'] = '/static/*'

config['CacheBehaviors']['Items'].append(admin_behavior)
config['CacheBehaviors']['Items'].append(static_behavior)
config['CacheBehaviors']['Quantity'] = 3

with open('/tmp/cf-config-new.json', 'w') as f:
    json.dump(config, f, indent=2)
```

ETagを取得してCloudFrontを更新します。

```bash
ETAG=$(aws cloudfront get-distribution-config \
  --id （ディストリビューションID） \
  --query "ETag" --output text)

aws cloudfront update-distribution \
  --id （ディストリビューションID） \
  --distribution-config file:///tmp/cf-config-new.json \
  --if-match $ETAG
```

これでCloudFrontのルーティングが以下のようになりました。

```
/api/*    → ALB（ECS・Django）✅
/admin/*  → ALB（ECS・Django）✅
/static/* → ALB（ECS・Django）✅
/*        → S3 ✅
```

## 問題②：管理画面のCSSが当たらない

管理画面は表示されるようになりましたが、CSSが崩れたままでした。

```
GET /static/admin/css/base.css → 404 Not Found
```

### 原因

`DEBUG=False` のとき、DjangoはデフォルトでSTATIC_ROOTを設定しても静的ファイルを配信しません。

### 対応：WhiteNoiseの導入

WhiteNoiseはDjangoが本番環境でも静的ファイルを配信できるようにするライブラリです。

```
本来の構成: Nginx → Django（静的ファイルはNginxが配信）
今回の構成: CloudFront → ALB → ECS（Nginxがない）
　↓
WhiteNoiseでDjangoが直接配信できるようにする
```

`requirements.txt` に追加します。

```
whitenoise==6.9.0
```

`settings.py` の `MIDDLEWARE` に追加します。`SecurityMiddleware` の直後に置くのがルールです。

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',  # 追加
    'django.contrib.sessions.middleware.SessionMiddleware',
    ...
]
```

`settings.py` に `STATIC_ROOT` も追加します。

```python
STATIC_URL = 'static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'  # 追加
```

`Dockerfile` に `collectstatic` を追加します。

```dockerfile
FROM python:3.14-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
COPY .env.prod .env
RUN python manage.py collectstatic --noinput  # 追加
CMD python manage.py create_superuser_prod && \
    python manage.py runserver 0.0.0.0:8000
```

`collectstatic` はDjangoの各アプリにバラバラに存在する静的ファイルを `STATIC_ROOT` にまとめるコマンドです。

:::message
設定ファイルを変更した場合はDockerのキャッシュを使わず再ビルドします。
```bash
docker build --no-cache --platform linux/amd64 -t manabi-quest-api .
```
:::

## 問題③：管理画面ログイン時にCSRFエラー

CSSは直ったものの、ログインしようとすると以下のエラーが出ました。

```
403 Forbidden
CSRF検証に失敗したため、リクエストは中断されました。
```

### 原因

CloudFront経由のPOSTリクエストに対して、DjangoがCSRFトークンのドメインを信頼していませんでした。

### 対応：CSRF_TRUSTED_ORIGINSの設定

`settings.py` に追加します。

```python
CSRF_TRUSTED_ORIGINS = [
    'https://manabi-quest.com',
    'https://www.manabi-quest.com',
]
```

## 本番スーパーユーザーの作成

本番RDSはプライベートサブネットにあるため、ローカルから直接接続できません。Djangoのカスタム管理コマンドを使って、サーバー起動時に自動でスーパーユーザーを作成します。

```
players/management/
├── __init__.py
└── commands/
    ├── __init__.py
    └── create_superuser_prod.py
```

```python
# players/management/commands/create_superuser_prod.py
from django.core.management.base import BaseCommand
from players.models import User
import os

class Command(BaseCommand):
    help = 'スーパーユーザーを作成する'

    def handle(self, *args, **options):
        email = os.environ.get('SUPERUSER_EMAIL')
        if not email:
            self.stdout.write('SUPERUSER_EMAILが設定されていません。スキップします。')
            return

        password = os.environ.get('SUPERUSER_PASSWORD')
        if not password:
            self.stdout.write('SUPERUSER_PASSWORDが設定されていません。スキップします。')
            return

        if User.objects.filter(email=email).exists():
            user = User.objects.get(email=email)
            user.is_staff = True
            user.is_superuser = True
            user.save()
            self.stdout.write(f'既存ユーザーをスーパーユーザーに昇格: {email}')
        else:
            User.objects.create_superuser(email=email, password=password)
            self.stdout.write(f'スーパーユーザー作成完了: {email}')
```

メールアドレスとパスワードは `.env.prod` の環境変数で管理します。

```bash
SUPERUSER_EMAIL=（メールアドレス）
SUPERUSER_PASSWORD=（パスワード）
```

## セキュリティ強化：push前チェック

GitHubにpushする前に機密情報が含まれていないか確認するコマンドです。

```bash
# バックエンド
grep -rn "メールアドレス\|パスワード\|SECRET\|DB_HOST\|DB_PASSWORD" \
  ~/app/manabi-quest-api \
  --include="*.py" \
  --include="*.txt" \
  --include="*.json" \
  --include="*.yml" \
  --exclude-dir=".git" \
  --exclude-dir="__pycache__" \
  | grep -v ".env"

# フロントエンド
grep -rn "パスワード\|PASSWORD\|SECRET" \
  ~/app/manabi-quest/public \
  --include="*.js" \
  --exclude-dir=".git"
```

以下が出たらpushしないこと：
- 実際のメールアドレス
- 実際のパスワード
- SECRET_KEY
- DB_PASSWORD・DB_HOST
- AWSのアクセスキー

## 再デプロイ手順

```bash
cd ~/app/manabi-quest-api && \
docker build --no-cache --platform linux/amd64 -t manabi-quest-api . && \
aws ecr get-login-password --region ap-northeast-1 | \
docker login --username AWS --password-stdin （ECRのURI） && \
docker tag manabi-quest-api:latest （ECRのURI）:latest && \
docker push （ECRのURI）:latest && \
aws ecs update-service \
  --cluster manabi-quest-cluster \
  --service manabi-quest-api-service \
  --force-new-deployment \
  --region ap-northeast-1
```

## まとめ

| 問題 | 原因 | 対応 |
|------|------|------|
| 管理画面がAccess Denied | CloudFrontが/admin/*をS3に転送 | ビヘイビアを追加 |
| 管理画面のCSSが崩れる | DEBUG=FalseでDjangoが静的ファイルを配信しない | WhiteNoise導入 |
| ログインでCSRFエラー | CloudFrontのドメインが未信頼 | CSRF_TRUSTED_ORIGINS設定 |
| 本番スーパーユーザー作成 | RDSがプライベートサブネット | カスタム管理コマンド作成 |
| 機密情報の漏洩防止 | ハードコードのリスク | 環境変数化・push前チェック |

次回：収益化・アカウント設計の実装（Stripe連携・Family/Userテーブル再設計）
