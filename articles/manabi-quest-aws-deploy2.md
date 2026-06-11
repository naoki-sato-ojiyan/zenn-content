---
title: "【まなびクエスト開発日記】ECS（Fargate）でDjangoをAWSにデプロイした"
emoji: "🚀"
type: "tech"
topics: ["aws", "ecs", "fargate", "django", "docker"]
published: true
---

## はじめに

前回の記事でRDS（PostgreSQL）を作成しました。今回はいよいよ**ECS（Fargate）を使ってDjangoのDockerコンテナをAWSにデプロイ**します。

ECSは最初に権限エラーがいくつか出て手間取りましたが、一つひとつ解決していけば問題なく構築できました。同じようにつまずいた方の参考になれば幸いです。

---

## 今回やること

```
① ECRにDockerイメージをpush
② ECSクラスター作成
③ タスク定義作成
④ ECSサービス作成・起動
⑤ RDSのセキュリティグループ設定（ECS→RDS接続許可）
```

---

## AWS CLIの設定

まずターミナルからAWSを操作できるようにAWS CLIをインストールします。

```bash
brew install awscli
aws --version
# aws-cli/2.35.2 Python/3.14.5 Darwin/...
```

インストール後、IAMユーザーの認証情報を設定します。

```bash
aws configure
```

4つ入力します。

| 質問 | 入力値 |
|---|---|
| AWS Access Key ID | IAMユーザーのアクセスキー |
| AWS Secret Access Key | シークレットアクセスキー |
| Default region name | `ap-northeast-1` |
| Default output format | `json` |

設定確認はこのコマンドで行います。

```bash
aws sts get-caller-identity
```

以下のようなJSONが返れば成功です。

```json
{
    "UserId": "XXXXXXXXXXXXXXXXXXXX",
    "Account": "575589967783",
    "Arn": "arn:aws:iam::575589967783:user/manabi-quest-dev"
}
```

---

## ① ECRにDockerイメージをpush

**ECR（Elastic Container Registry）** はAWSのDockerイメージ置き場です。DockerHubのAWS版と考えるとわかりやすいです。

### ECRリポジトリの作成

```bash
aws ecr create-repository \
  --repository-name manabi-quest-api \
  --region ap-northeast-1
```

成功するとリポジトリURIが返ってきます。

```
repositoryUri: 575589967783.dkr.ecr.ap-northeast-1.amazonaws.com/manabi-quest-api
```

### DockerイメージをECRにpush

3ステップで完了します。

```bash
# ① ECRにログイン
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin \
  575589967783.dkr.ecr.ap-northeast-1.amazonaws.com

# ② Dockerイメージをビルド（Apple SiliconはAMD64を指定）
cd ~/app/manabi-quest-api
docker build --platform linux/amd64 -t manabi-quest-api .

# ③ タグをつけてpush
docker tag manabi-quest-api:latest \
  575589967783.dkr.ecr.ap-northeast-1.amazonaws.com/manabi-quest-api:latest

docker push \
  575589967783.dkr.ecr.ap-northeast-1.amazonaws.com/manabi-quest-api:latest
```

pushが完了したら確認します。

```bash
aws ecr list-images \
  --repository-name manabi-quest-api \
  --region ap-northeast-1
```

`imageTag: latest` が表示されれば成功です。

:::message
Apple Silicon（M1/M2/M3）のMacでビルドする場合は `--platform linux/amd64` が必要です。指定しないとECS（x86_64）上で動かないイメージができてしまいます。
:::

---

## ② ECSクラスターの作成

**ECSクラスター**はコンテナをまとめて管理する「箱」です。

```
クラスター（manabi-quest-cluster）
　└── サービス（Djangoを常時起動させる設定）
　　　　└── タスク（実際に動いているDjangoコンテナ）
```

AWSコンソールでECSを開き、「クラスターの作成」をクリックします。

| 項目 | 設定値 |
|---|---|
| クラスター名 | `manabi-quest-cluster` |
| インフラストラクチャ | `AWS Fargate（サーバーレス）` |

:::message alert
**ECSサービスリンクロールのエラーが出た場合**

初回ECS使用時は `AWSServiceRoleForECS` というロールが必要です。以下のコマンドで作成できます。

```bash
aws iam create-service-linked-role --aws-service-name ecs.amazonaws.com
```

「has been taken」というエラーが出た場合はすでに存在しているので問題ありません。
:::

---

## ③ タスク定義の作成

**タスク定義**はコンテナの「設計図」です。どのDockerイメージを使うか、CPUやメモリはいくつか、環境変数は何かを定義します。

ECSコンソールの「タスク定義」→「新しいタスク定義の作成」をクリックします。

### 基本設定

| 項目 | 設定値 |
|---|---|
| タスク定義ファミリー | `manabi-quest-api` |
| 起動タイプ | `AWS Fargate` |
| CPU | `0.25 vCPU` |
| メモリ | `0.5 GB` |
| タスク実行ロール | `ecsTaskExecutionRole`（新規作成） |

### コンテナの設定

| 項目 | 設定値 |
|---|---|
| 名前 | `manabi-quest-api` |
| イメージURI | `575589967783.dkr.ecr.ap-northeast-1.amazonaws.com/manabi-quest-api:latest` |
| コンテナポート | `8000` |
| プロトコル | `TCP` |
| アプリケーションプロトコル | `HTTP` |

### 環境変数の設定

「一括編集」でまとめて入力します。

```
DEBUG=False
SECRET_KEY=（本番用のSECRET_KEY）
DB_HOST=manabi-quest-db.cf00uqi8kwtd.ap-northeast-1.rds.amazonaws.com
DB_NAME=manabi_quest
DB_USER=postgres
DB_PASSWORD=（RDS作成時に設定したパスワード）
DB_PORT=5432
ALLOWED_HOSTS=*
```

:::message
**アプリケーションプロトコルをHTTPにする理由**

ECS内部の通信はHTTPで問題ありません。HTTPS化はCloudFrontが担当するため、ECS→RDS間はVPC内のプライベート通信になりHTTPで十分安全です。これを「SSLターミネーション」といいます。
:::

---

## ④ ECSサービスの作成

**ECSサービス**はタスクを常時起動させておく仕組みです。タスクが落ちると自動で再起動してくれます。

クラスター（`manabi-quest-cluster`）を開き「サービス」→「作成」をクリックします。

| 項目 | 設定値 |
|---|---|
| タスク定義ファミリー | `manabi-quest-api` |
| サービス名 | `manabi-quest-api-service` |
| 必要なタスク数 | `1` |

ネットワーキングはデフォルトVPC・デフォルトセキュリティグループのままでOKです。

作成完了後、以下の状態になれば成功です。

```
サービス: Active: 1
タスク: Running: 1
```

---

## ⑤ RDSのセキュリティグループ設定

現状はECS（Django）とRDS（PostgreSQL）がセキュリティグループで遮断されています。ECSからRDSへの接続を許可する設定を追加します。

AWSコンソールでVPCを開き、「セキュリティグループ」→ `manabi-quest-rds-sg` を選択します。

「インバウンドルール」→「インバウンドのルールを編集」をクリックし、新しいルールを追加します。

| 項目 | 設定値 |
|---|---|
| タイプ | `PostgreSQL` |
| ソース | `カスタム` → `sg-03c45f465a3b6c5c8`（ECSのdefaultセキュリティグループ） |

「ルールを保存」をクリックして完了です。

```
設定後の接続フロー:

インターネット
　↓
ECS（Django・ポート8000）
　↓ ✅ 接続許可済み（ポート5432）
RDS（PostgreSQL）
```

---

## 今回の構成まとめ

| サービス | 役割 | 状態 |
|---|---|---|
| ECR | Dockerイメージ保管 | ✅ 完了 |
| ECSクラスター | コンテナの管理単位 | ✅ 完了 |
| タスク定義 | コンテナの設計図 | ✅ 完了 |
| ECSサービス | コンテナの常時起動 | ✅ 完了 |
| RDS | PostgreSQL | ✅ 接続許可済み |

---

## つまずいたポイント

### IAM権限エラーが多発した

IAMユーザーに以下のポリシーが必要でした。

- `AmazonECS_FullAccess`
- `AmazonEC2ContainerRegistryFullAccess`
- `AWSCloudFormationFullAccess`
- `IAMReadOnlyAccess`

1ユーザーに付けられるポリシーの上限は10個なので、不要なポリシーは整理が必要です。

### ecsTaskExecutionRoleがなかった

ECSタスクがECRからイメージを取得するために必要なIAMロールです。IAMコンソールで手動作成しました。

| 項目 | 設定値 |
|---|---|
| ロール名 | `ecsTaskExecutionRole` |
| 信頼されたエンティティ | `ecs-tasks.amazonaws.com` |
| ポリシー | `AmazonECSTaskExecutionRolePolicy` |

---

## 次回

次回はS3バケットを作成してフロントエンド（HTML・JS・画像）をアップロードし、CloudFrontで配信できるようにします！

---

## 関連記事

- [【まなびクエスト開発日記】AWSデプロイ構成を理解する（S3・CloudFront・ECS・RDS・Route53）](https://zenn.dev/)
- [【まなびクエスト開発日記】AWSにRDSを作成してPostgreSQLをデプロイした](https://zenn.dev/)
