---
title: "【まなびクエスト開発日記】AWSデプロイ構成を理解する（S3・CloudFront・ECS・RDS・Route53）"
emoji: "☁️"
type: "tech"
topics: ["aws", "s3", "cloudfront", "ecs", "rds"]
published: true
---

## はじめに

まなびクエスト（ドラクエ風の小学生向け学習RPG）のAWSデプロイを進めるにあたって、使用するサービスの構成と仕組みを整理しました。

「AWSって何から理解すればいいの？」という初心者の方にも伝わるよう、図を使いながら丁寧に説明します。

---

## まなびクエストのAWS構成

まなびクエストのデプロイ構成は以下の通りです。

```
フロントエンド → S3 + CloudFront（静的ファイル）
バックエンド　 → ECS（Dockerコンテナ）
DB　　　　　　 → RDS（PostgreSQL）
ドメイン　　　 → Route53
```

図にするとこうなります。

```
【ユーザー（ブラウザ）】
　　↓ URLにアクセス

【Route53】← ドメイン管理（manabi-quest.com）
　　↓
【CloudFront】← 世界中に高速配信するCDN
　　↓                    ↓
【S3】               【ECS】← Djangoが動くコンテナ
フロントエンド         バックエンドAPI
HTML/JS/画像           　↓
                     【RDS】← PostgreSQL DB
                     ユーザー・ステータス
```

---

## 各サービスの役割

### Route53 — ドメイン管理・DNS

**一言で言うと「URLとIPアドレスを繋ぐ電話帳」です。**

インターネット上のサーバーはIPアドレス（`54.230.12.45` のような番号）で動いていますが、人間には覚えられません。Route53は `manabi-quest.com` というURLを入力されたとき、対応するIPアドレスに変換してくれます。

```
ユーザーが「manabi-quest.com」と入力
　　↓
Route53が「それは54.230.12.45だよ」と教える
　　↓
ブラウザが「54.230.12.45」に接続する
```

この仕組みを**DNS（Domain Name System）**と言います。

:::message
Route53という名前は、アメリカの国道Route 66をもじったもので、DNSの標準ポート番号が**53番**であることから来ています。
:::

---

### S3 — フロントエンドのファイル置き場

**一言で言うと「ファイルの倉庫」です。**

| | S3 | RDS |
|---|---|---|
| 種類 | ストレージ（倉庫） | データベース |
| 何を入れる | ファイル（HTML・JS・画像） | 構造化データ（表形式） |
| 使い方 | ファイルをそのまま保存・取得 | SQLで検索・更新 |
| まなびクエストでの用途 | フロントエンドのHTML・JS・画像 | ユーザー情報・ステータス |

まなびクエストでは以下のファイルをS3に置きます。

```
S3バケット: manabi-quest-frontend
　├── index.html
　├── src/
│   ├── main.js
│   ├── GameData.js
│   └── scenes/
│       ├── LoginScene.js
│       ├── FieldScene.js
│       └── BattleScene.js
　└── assets/
　    ├── images/gfx/
　    │   ├── Overworld.png
│    │   └── character.png
　    └── maps/
　        └── field.json
```

S3の**静的ウェブサイトホスティング**機能をONにすると、URLでファイルを公開できます。

---

### CloudFront — CDN配信・HTTPS化

**一言で言うと「よく使うファイルを近くのサーバーに前もって置いておく仕組み」です。**

CloudFrontは世界450箇所以上に**エッジロケーション**（キャッシュサーバー）を持っています。

```
【キャッシュなしの場合】
海外ユーザーがアクセス
　→ 毎回S3（東京）から取得 → 遠いので遅い！

【CloudFrontありの場合】
最初のアクセス時
　→ S3からファイルを取得 → 世界中のエッジにコピーを保存

2回目以降のアクセス
　→ 近くのエッジサーバーから即座に返す → 速い！
```

CloudFrontにはキャッシュ以外にもメリットがあります。

| メリット | 内容 |
|---|---|
| 高速配信 | 世界中のエッジから近い場所で配信 |
| HTTPS化 | 自動でHTTPS対応してくれる |
| S3を守る | S3への直接アクセスをブロックできる |

#### キャッシュの有効期限（TTL）

```
TTL（Time To Live）= キャッシュの有効期限

デフォルト: 24時間
　→ 24時間はエッジサーバーのコピーを使う
　→ ファイルを更新したら手動でキャッシュを「無効化」する必要がある
```

---

### ECS（Elastic Container Service） — Djangoを動かすコンテナ

**一言で言うと「ローカルの `docker-compose up` をAWS上でやってくれるサービス」です。**

| | ローカル（今） | AWS（本番） |
|---|---|---|
| Dockerを動かす場所 | 自分のMac | ECS |
| イメージの保存場所 | ローカル | ECR |
| 設定ファイル | docker-compose.yml | タスク定義（JSON） |
| 起動コマンド | `docker-compose up` | ECSが自動で起動 |

まなびクエストでは**Fargate**という方式を使います。

```
ECS on EC2
　→ 自分でサーバーを用意してその上でコンテナを動かす
　→ サーバー管理が必要（大変）

ECS on Fargate  ← まなびクエストはこっち
　→ サーバー不要！コンテナだけ指定すれば勝手に動く
　→ 「何CPU・何メモリ使うか」だけ指定すればOK
```

まなびクエストでのECSの動き方はこうなります。

```
① ECRからDockerイメージを取得
　　（manabi-quest-apiのイメージ）
　　↓
② コンテナを起動（Djangoが起動）
　　↓
③ CloudFrontからのAPIリクエストを受け付ける
　　（/api/login/ や /api/player/ など）
　　↓
④ RDSのPostgreSQLにアクセスしてデータを読み書き
```

---

### RDS — PostgreSQLデータベース

**一言で言うと「クラウド上のPostgreSQL」です。**

ローカルの `docker-compose` 内のdbコンテナをAWS管理のDBに置き換えたイメージです。

まなびクエストのDBには2つのテーブルがあります。

#### USERSテーブル（ログイン情報）

| カラム | 内容 | 例 |
|---|---|---|
| `id` | ユーザーの番号 | 1, 2, 3... |
| `email` | ログインに使うメアド | nakky@gmail.com |
| `password` | ハッシュ化されたパスワード | pbkdf2_sha256$... |
| `character_name` | キャラクター名 | ゆうしゃ |
| `created_at` | 登録日時 | 2026-06-01 10:00 |

#### PLAYER_STATUSテーブル（ゲームのステータス）

| カラム | 内容 | 初期値 |
|---|---|---|
| `user_id` | どのユーザーか（USERSと紐付け） | — |
| `level` | レベル | 1 |
| `hp` | 現在HP | 10 |
| `max_hp` | 最大HP | 10 |
| `exp` | 経験値 | 0 |
| `attack` | 攻撃力 | 5 |
| `defense` | 防御力 | 5 |
| `agility` | 素早さ | 5 |
| `luck` | 運 | 5 |

2つのテーブルは**1対1の関係**です。

```
1人のユーザー = 1つのステータス

user_id = 3 のユーザー
　→ player_statusのuser_id = 3 の行がそのステータス
```

---

## ユーザーがアクセスしてゲームが始まるまでの流れ

### ① ゲームが表示されるまで

```
URLを入力
　↓
Route53がIPアドレスを調べる
　↓
CloudFrontがリクエストを受け取る
　↓
S3からHTML・JS・画像を返す
　↓
ブラウザでPhaser.jsが起動
　↓
LoginSceneが表示される
```

### ② ゲームが表示される仕組み（ブラウザ内）

```
① S3からindex.htmlを受け取る
　↓
② index.htmlがJSファイルを読み込む
　（main.js・LoginScene.js など）
　↓
③ Phaser.jsが起動
　（main.jsのconfigを読んで初期化）
　↓
④ LoginSceneが最初に表示
　（scene配列の先頭がLoginSceneなので）
　↓
⑤ ログイン成功 → FieldSceneへ
　（フィールドマップ・キャラが表示）
```

```javascript
// main.jsのconfigでシーン順を指定
const config = {
  scene: [LoginScene, FieldScene, BattleScene], // 先頭が最初に起動
  ...
};
new Phaser.Game(config); // ← ここでPhaser起動！
```

### ③ ログイン・データ取得（API通信）

```
Phaser.jsが起動したらAPIを呼び出す
　↓
CloudFront経由でECS（Django）へ
　↓
RDSからユーザー情報・ステータスを取得
　↓
JSONで返ってくる
　↓
GameData.playerにステータスが反映
　↓
ゲーム開始！
```

---

## ゲーム中のステータスの使われ方

### バトル中の計算式

| ステータス | バトルでの使われ方 | 計算式 |
|---|---|---|
| `attack` | 自分の攻撃ダメージ | `1 + Math.floor(attack / 3)` |
| `defense` | 受けるダメージを減らす | `Math.max(1, 2 - Math.floor(defense / 3))` |
| `agility` | 敵の攻撃を回避する確率 | `agility × 3%` |
| `luck` | クリティカルヒットの確率 | `luck × 2%` |

### レベルアップ時の変化

```
expが level × 10 を超えたとき
　→ level + 1
　→ maxHp + 3
　→ attack・defense・agility・luck 全部 + 1
　→ HP全回復（hp = maxHp）
```

### セーブのタイミング

```
バトル勝利       → PUT /api/player/ でステータス保存
ゲームオーバー   → PUT /api/player/ でステータス保存（HPを初期値に戻す）
フィールド歩き中 → まだ保存しない
```

:::message alert
フィールド歩き中はまだDBに保存していません。ブラウザを閉じるとその分は失われます。「セーブ・ロード機能」は今後の課題です。
:::

---

## 月額費用の見込み

| サービス | 月額費用 | 備考 |
|---|---|---|
| RDS | 約$15〜25 | db.t3.micro（最小） |
| ECS | 約$10〜20 | Fargate 0.25vCPU/0.5GB |
| ECR | 約$1 | 500MB以内 |
| S3 | ほぼ無料 | 静的ファイルのみ |
| CloudFront | 約$1〜2 | 1TB/月まで無料枠あり |
| Route53 | 約$0.5 | ホストゾーン料金 |
| **合計** | **約$27〜48/月** | **≒ 4,000〜7,000円/月** |

### 開発中のコストを抑える方法

```
使うときだけ起動 → 月500〜1,000円くらい

RDSを止める
　→ スナップショット → 削除 → 必要時に復元

ECSタスクを0にする
　→ タスク数を0にスケールダウン
```

---

## まとめ

| サービス | 担当 | 一言まとめ |
|---|---|---|
| Route53 | ドメイン管理 | URLをIPアドレスに変換する電話帳 |
| CloudFront | CDN配信 | 近くのサーバーから高速配信 |
| S3 | フロントエンド | HTML・JS・画像の倉庫 |
| ECS | バックエンド | Djangoコンテナを動かす場所 |
| ECR | Dockerイメージ保存 | コンテナイメージの置き場 |
| RDS | DB | PostgreSQLのクラウド版 |

次回は実際にAWSコンソールでRDS・ECS・S3・CloudFrontを順番に構築していきます！

---

## 関連記事

- [【完全初心者】プログラミング未経験からDjango+Docker+PostgreSQLでToDoアプリを作るまで](https://zenn.dev/)
- [【まなびクエスト開発日記】Django + DRFでバックエンドAPIを構築した](https://zenn.dev/)