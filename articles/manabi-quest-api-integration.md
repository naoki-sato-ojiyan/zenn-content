---
title: "【まなびクエスト開発日記】フロントエンドとDjango APIを繋げた"
emoji: "🔗"
type: "tech"
topics: ["javascript", "phaser", "django", "drf", "api"]
published: true
---

# はじめに

小学生向け学習RPG「まなびクエスト」を開発しています。

前回はDjango + DRFでバックエンドAPIを構築しました。

https://zenn.dev/naokisatoojiyan/articles/manabi-quest-django-api

今回はフロントエンド（Phaser.js）とバックエンド（Django API）を繋げます。

## 実装した内容

- ログイン・新規登録画面の作成
- ゲーム起動時にAPIからステータスを取得
- バトル終了時にAPIへステータスを保存
- トークンの管理（localStorage）

---

# 全体の流れ

```
ゲーム起動
　↓
LoginScene（ログイン・登録画面）
　↓ ログイン成功 → トークンをlocalStorageに保存
FieldScene（フィールド）← APIからステータスを取得して反映
　↓ エンカウント
BattleScene（バトル）
　↓ バトル終了 → APIにステータスを保存
FieldScene に戻る
```

---

# STEP9: LoginScene.jsを作る

## なぜPhaser.jsではなくHTML inputを使ったか

最初はPhaser.jsのキーボードイベントで入力欄を作りましたが、**日本語（IME）入力に対応できない**問題がありました。

Phaser.jsの`keydown`イベントは1文字ずつ取得する仕組みなので、日本語変換には対応していません。

そこで**HTMLの`<input>`要素をCanvasに重ねる**方法に変更しました。

```
Phaser.jsのCanvas（背景・タイトル・ボタン）
　↑ 重ねる
HTMLの<input>要素（入力欄・日本語OK）
```

この方法のメリットは3つあります。

- 日本語入力ができる
- Chromeのオートフィル機能が使える
- パスワードマネージャーにも対応できる

## LoginScene.js

```javascript
export default class LoginScene extends Phaser.Scene {
    constructor() {
        super({ key: 'LoginScene' });
    }

    create() {
        // 背景
        this.add.rectangle(0, 0, this.scale.width, this.scale.height, 0x000022)
            .setOrigin(0, 0);

        // タイトル
        this.add.text(this.scale.width / 2, 60, 'まなびクエスト', {
            fontSize: '36px', fill: '#ffff00'
        }).setOrigin(0.5);

        this.mode = 'login'; // 'login' or 'register'

        this.modeText = this.add.text(this.scale.width / 2, 110, 'ログイン', {
            fontSize: '28px', fill: '#ffffff'
        }).setOrigin(0.5);

        // ラベル
        this.add.text(this.scale.width / 2 - 150, 160, 'メールアドレス:', {
            fontSize: '20px', fill: '#aaaaaa'
        });
        this.add.text(this.scale.width / 2 - 150, 250, 'パスワード:', {
            fontSize: '20px', fill: '#aaaaaa'
        });

        // HTML inputで入力欄を作成（日本語対応のため）
        this.emailInput    = this.createInputField('email',    'メールアドレス',     195);
        this.passwordInput = this.createInputField('password', 'パスワード',         285);

        // キャラクター名（登録モードのみ表示）
        this.characterNameLabel = this.add.text(
            this.scale.width / 2 - 150, 340,
            'キャラクター名:',
            { fontSize: '20px', fill: '#aaaaaa' }
        ).setVisible(false);

        this.characterNameInput = this.createInputField('text', 'キャラクター名（省略可）', 375);
        this.characterNameInput.style.display = 'none';

        // 送信ボタン
        this.submitButton = this.add.text(this.scale.width / 2, 440, '[ ログイン ]', {
            fontSize: '28px', fill: '#44ff44'
        }).setOrigin(0.5).setInteractive();
        this.submitButton.on('pointerdown', () => this.handleSubmit());

        // モード切替テキスト
        this.toggleText = this.add.text(
            this.scale.width / 2, 510,
            '新規登録はこちら',
            { fontSize: '20px', fill: '#4444ff' }
        ).setOrigin(0.5).setInteractive();
        this.toggleText.on('pointerdown', () => this.toggleMode());

        // 結果メッセージ（エラー表示用）
        this.resultText = this.add.text(this.scale.width / 2, 570, '', {
            fontSize: '20px', fill: '#ff4444'
        }).setOrigin(0.5);
    }

    // HTML inputを作成するヘルパー
    createInputField(type, placeholder, top) {
        const input = document.createElement('input');
        input.type = type;
        input.placeholder = placeholder;
        input.style.cssText = `
            position: absolute;
            left: 50%;
            top: ${top}px;
            transform: translateX(-50%);
            width: 300px;
            padding: 8px 12px;
            font-size: 18px;
            background: #333333;
            color: #ffffff;
            border: 1px solid #666666;
            border-radius: 4px;
            outline: none;
            z-index: 10;
        `;
        document.body.appendChild(input);
        return input;
    }

    // ログイン ↔ 新規登録の切り替え
    toggleMode() {
        this.mode = this.mode === 'login' ? 'register' : 'login';
        if (this.mode === 'login') {
            this.modeText.setText('ログイン');
            this.submitButton.setText('[ ログイン ]');
            this.toggleText.setText('新規登録はこちら');
            this.characterNameLabel.setVisible(false);
            this.characterNameInput.style.display = 'none';
        } else {
            this.modeText.setText('新規登録');
            this.submitButton.setText('[ 登録する ]');
            this.toggleText.setText('ログインはこちら');
            this.characterNameLabel.setVisible(true);
            this.characterNameInput.style.display = 'block';
        }
        this.emailInput.value = '';
        this.passwordInput.value = '';
        this.characterNameInput.value = '';
        this.resultText.setText('');
    }

    // ログイン or 登録の送信処理
    async handleSubmit() {
        const email    = this.emailInput.value.trim();
        const password = this.passwordInput.value.trim();

        if (!email || !password) {
            this.resultText.setText('メールアドレスとパスワードを入力してください');
            return;
        }

        const body = { email, password };
        if (this.mode === 'register') {
            // キャラクター名が空なら「ゆうしゃ」をデフォルトに
            body.character_name = this.characterNameInput.value.trim() || 'ゆうしゃ';
        }

        const url = this.mode === 'login'
            ? 'http://localhost:8000/api/login/'
            : 'http://localhost:8000/api/register/';

        try {
            const response = await fetch(url, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(body)
            });

            const data = await response.json();

            if (response.ok) {
                // トークンをlocalStorageに保存
                localStorage.setItem('token', data.token);
                localStorage.setItem('email', data.email);
                if (this.mode === 'register') {
                    localStorage.setItem('character_name', body.character_name);
                }
                // シーン移動前にHTML inputを削除
                this.removeInputFields();
                this.scene.start('FieldScene');
            } else {
                this.resultText.setText(data.error || 'エラーが発生しました');
            }

        } catch (e) {
            this.resultText.setText('サーバーに接続できません');
        }
    }

    // シーン移動前にHTML inputを削除する
    removeInputFields() {
        this.emailInput.remove();
        this.passwordInput.remove();
        this.characterNameInput.remove();
    }
}
```

## main.jsにLoginSceneを追加

```javascript
import FieldScene from './scenes/FieldScene.js';
import BattleScene from './scenes/BattleScene.js';
import LoginScene from './scenes/LoginScene.js'; // 追加

const config = {
    type: Phaser.AUTO,
    width: window.innerWidth,
    height: window.innerHeight,
    scene: [LoginScene, FieldScene, BattleScene], // LoginSceneを先頭に
    physics: {
        default: 'arcade',
        arcade: { gravity: { y: 0 }, debug: false }
    }
};

new Phaser.Game(config);
```

`scene`配列の**先頭に書いたシーンが最初に起動**します。

---

# STEP10: FieldSceneでAPIからステータスを取得

FieldSceneの`create()`の最後に`loadPlayerStatus()`を呼び出します。

```javascript
import GameData from '../GameData.js'; // 追加

export default class FieldScene extends Phaser.Scene {

    create() {
        // ...（既存のコード）

        // 追加：APIからステータスを取得
        this.loadPlayerStatus();
    }

    // 追加：APIからステータスを取得するメソッド
    async loadPlayerStatus() {
        const token = localStorage.getItem('token');

        // トークンがなければログイン画面へ
        if (!token) {
            this.scene.start('LoginScene');
            return;
        }

        try {
            const response = await fetch('http://localhost:8000/api/player/', {
                method: 'GET',
                headers: {
                    'Authorization': `Token ${token}`, // 認証トークンを送る
                    'Content-Type': 'application/json'
                }
            });

            if (response.ok) {
                // APIから取得したデータをGameDataに反映
                const data = await response.json();
                GameData.player.level   = data.level;
                GameData.player.hp      = data.hp;
                GameData.player.maxHp   = data.max_hp;
                GameData.player.exp     = data.exp;
                GameData.player.attack  = data.attack;
                GameData.player.defense = data.defense;
                GameData.player.agility = data.agility;
                GameData.player.luck    = data.luck;

            } else if (response.status === 404) {
                // プレイヤーデータがない → 初回作成
                await this.createPlayerStatus(token);
            } else {
                // 認証エラーなどはログイン画面へ
                this.scene.start('LoginScene');
            }

        } catch (e) {
            console.error('API接続エラー:', e);
        }
    }

    // 追加：プレイヤーデータを初回作成するメソッド
    async createPlayerStatus(token) {
        const p = GameData.player;
        const characterName = localStorage.getItem('character_name') || 'ゆうしゃ';

        await fetch('http://localhost:8000/api/player/create/', {
            method: 'POST',
            headers: {
                'Authorization': `Token ${token}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                character_name: characterName,
                level:   p.level,
                hp:      p.hp,
                max_hp:  p.maxHp,
                exp:     p.exp,
                attack:  p.attack,
                defense: p.defense,
                agility: p.agility,
                luck:    p.luck
            })
        });
    }
}
```

### Authorizationヘッダーとは

```javascript
'Authorization': `Token ${token}`
```

DjangoのTokenAuthenticationは、リクエストヘッダーに`Token <トークン文字列>`の形式で認証情報を送る必要があります。これがないと401（Unauthorized）エラーになります。

---

# STEP11: BattleSceneでAPIにステータスを保存

バトル終了時（勝利・ゲームオーバー）にステータスをAPIへ保存します。

```javascript
// 追加：ステータスをAPIに保存するメソッド
async savePlayerStatus() {
    const token = localStorage.getItem('token');
    if (!token) return;

    const p = GameData.player;

    await fetch('http://localhost:8000/api/player/', {
        method: 'PUT', // 更新はPUT
        headers: {
            'Authorization': `Token ${token}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            level:   p.level,
            hp:      p.hp,
            max_hp:  p.maxHp,
            exp:     p.exp,
            attack:  p.attack,
            defense: p.defense,
            agility: p.agility,
            luck:    p.luck
        })
    });
}
```

呼び出しはバトル終了の2か所に追加します。

```javascript
// 敵を倒したとき
this.time.delayedCall(2500, () => {
    this.savePlayerStatus(); // 追加
    this.scene.start('FieldScene');
});

// ゲームオーバーのとき
this.time.delayedCall(2000, () => {
    GameData.player.hp = GameData.player.maxHp;
    this.savePlayerStatus(); // 追加
    this.scene.start('FieldScene');
});
```

### HTTPメソッドの使い分け

| メソッド | 使う場面 |
|---------|---------|
| GET | データを**取得**する（フィールド起動時） |
| POST | データを**新規作成**する（初回登録時） |
| PUT | データを**更新**する（バトル終了時） |

---

# 動作確認

Django管理画面（`http://localhost:8000/admin/`）でステータスの変化を確認できます。

管理画面を使うにはスーパーユーザーが必要です。

```bash
docker-compose exec web python manage.py createsuperuser
```

`players/admin.py`にモデルを登録しておきます。

```python
from django.contrib import admin
from .models import User, PlayerStatus

admin.site.register(User)
admin.site.register(PlayerStatus)
```

バトルでレベルアップした後に管理画面をリロードすると、DBの値が更新されていました。

---

# ハマったポイント

## 1. HTML inputがシーン移動後も残る

HTML `<input>`はPhaser.jsのシーンと無関係にDOMに存在します。`scene.start()`でシーンを切り替えても自動で消えません。

```javascript
// ❌ これだけではHTML inputが残ったまま
this.scene.start('FieldScene');

// ✅ 削除してからシーン移動する
this.removeInputFields();
this.scene.start('FieldScene');
```

## 2. APIのURLをハードコーディングしている

今回は開発環境なので`http://localhost:8000`を直書きしています。

AWSデプロイ時はURLが変わるため、`src/config.js`で管理する予定です。

```javascript
// src/config.js（予定）
const API_BASE_URL = 'http://localhost:8000'; // 開発用
// const API_BASE_URL = 'https://api.manabi-quest.com'; // 本番用

export default API_BASE_URL;
```

URLの直書きは現場ではNGです。環境変数や設定ファイルで管理するのが慣習です。

---

# まとめ

フロントエンドとバックエンドを繋げることで、ステータスがDBに永続化されました。

ログアウトしてもレベルやステータスが保持されます。

次回はいよいよ**AWSデプロイ**です！S3 + CloudFront + ECS + RDSの構成で本番環境を作ります。

---

# ソースコード

https://github.com/naoki-sato-ojiyan/manabi-quest

https://github.com/naoki-sato-ojiyan/manabi-quest-api
