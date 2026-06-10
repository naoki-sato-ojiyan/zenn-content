---
title: "【まなびクエスト開発日記】Tiled + Phaser.jsでフィールド画面を作った"
emoji: "🗺️"
type: "tech"
topics: ["phaser", "javascript", "tiled", "gamedev"]
published: true
---

# はじめに

小学生向け学習RPG「まなびクエスト」を開発しています。

フィールドを歩いて敵に遭遇し、算数・漢字の問題を解いて敵を倒すゲームです。

今回はTiledでマップを作り、Phaser.jsでフィールド画面を実装した内容をまとめます。

## 完成イメージ

- 草原・水タイルが並んだRPGフィールド
- キャラクターが4方向に歩く
- カメラがキャラクターを追いかける

---

# 環境

- Tiled 1.12.2
- Phaser.js 3.60.0
- JavaScript（モジュール形式）
- Live Server（VS Code拡張）

---

# STEP1: Tiledのインストール

```bash
brew install --cask tiled
```

---

# STEP2: 素材の準備

## タイルセットの選定

今回はOpenGameArtの**Zelda-like tilesets and sprites**を使いました。

- ライセンス: **CC0（パブリックドメイン）**
- 商用利用: OK
- クレジット表記: 不要
- URL: https://opengameart.org/content/zelda-like-tilesets-and-sprites

CC0なので商用プロジェクトにも安心して使えます。

## フォルダ構成

```
manabi-quest/
├── public/
│   ├── index.html
│   └── assets/
│       ├── images/
│       │   └── gfx/
│       │       ├── Overworld.png  ← タイルセット
│       │       └── character.png  ← キャラクター
│       └── maps/
│           ├── field.json    ← Phaser.jsが読み込む
│           ├── field.tmx     ← Tiledのプロジェクトファイル
│           └── overworld.tsx ← タイルセット定義
└── src/
    ├── main.js
    └── scenes/
        └── FieldScene.js
```

---

# STEP3: Tiledでマップを作る

## タイルセットの登録

Tiledを起動して「新しいタイルセット...」をクリック。

| 設定項目 | 値 |
|---|---|
| 名前 | overworld |
| 画像 | Overworld.png |
| タイルの幅 | 16px |
| タイルの高さ | 16px |

保存先: `public/assets/maps/overworld.tsx`

## マップの作成

「新しいマップ...」をクリック。

| 設定項目 | 値 |
|---|---|
| マップの幅 | 100タイル |
| マップの高さ | 100タイル |
| タイルの幅 | 16px |
| タイルの高さ | 16px |

タイルセットを読み込んで草・水・木などを自由に配置します。

## JSONエクスポート

`ファイル → エクスポート` で`field.json`として保存。

### ⚠️ 注意: タイルセットの埋め込み

Phaser.jsはデフォルトのエクスポートだと外部参照になってしまいエラーが出ます。

```
External tilesets unsupported. Use Embed Tileset and re-export
```

`field.json`の`tilesets`部分を手動で以下に書き換えてください👇

```json
"tilesets":[
  {
    "columns": 40,
    "firstgid": 1,
    "image": "../images/gfx/Overworld.png",
    "imageheight": 576,
    "imagewidth": 640,
    "margin": 0,
    "name": "overworld",
    "spacing": 0,
    "tilecount": 1440,
    "tileheight": 16,
    "tilewidth": 16
  }
]
```

---

# STEP4: Phaser.jsでマップを表示する

## index.html

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <title>まなびクエスト</title>
    <style>
      * { margin: 0; padding: 0; }
      body {
        background: #000;
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100vh;
      }
    </style>
  </head>
  <body>
    <script src="https://cdn.jsdelivr.net/npm/phaser@3.60.0/dist/phaser.min.js"></script>
    <script src="../src/main.js" type="module"></script>
  </body>
</html>
```

## main.js

```javascript
import FieldScene from './scenes/FieldScene.js';

const config = {
  type: Phaser.AUTO,
  width: window.innerWidth,   // ブラウザの横幅に自動調整
  height: window.innerHeight, // ブラウザの縦幅に自動調整
  zoom: 2,                    // 2倍に拡大（16px → 32px相当）
  scene: [FieldScene],
  physics: {
    default: 'arcade',
    arcade: {
      gravity: { y: 0 }, // 重力なし（トップダウンRPG）
      debug: false
    }
  }
};

new Phaser.Game(config);
```

## FieldScene.js

```javascript
export default class FieldScene extends Phaser.Scene {
  constructor() {
    super({ key: 'FieldScene' });
  }

  preload() {
    this.load.image('overworld', '../public/assets/images/gfx/Overworld.png');
    this.load.tilemapTiledJSON('field', '../public/assets/maps/field.json');

    // 1コマ: 16×32px（17列×8行）
    this.load.spritesheet('character', '../public/assets/images/gfx/character.png', {
      frameWidth: 16,
      frameHeight: 32
    });
  }

  create() {
    // マップ作成
    const map = this.make.tilemap({ key: 'field' });
    const tileset = map.addTilesetImage('overworld', 'overworld');
    const layer = map.createLayer('草', tileset, 0, 0);

    // キャラクター配置（マップ中央）
    this.player = this.physics.add.sprite(
      map.widthInPixels / 2,
      map.heightInPixels / 2,
      'character', 0
    );

    // 歩きアニメーション
    this.anims.create({ key: 'walk-down',  frames: this.anims.generateFrameNumbers('character', { start: 0,  end: 3  }), frameRate: 8, repeat: -1 });
    this.anims.create({ key: 'walk-right', frames: this.anims.generateFrameNumbers('character', { start: 17, end: 20 }), frameRate: 8, repeat: -1 });
    this.anims.create({ key: 'walk-up',    frames: this.anims.generateFrameNumbers('character', { start: 34, end: 37 }), frameRate: 8, repeat: -1 });
    this.anims.create({ key: 'walk-left',  frames: this.anims.generateFrameNumbers('character', { start: 51, end: 54 }), frameRate: 8, repeat: -1 });

    // 停止アニメーション
    this.anims.create({ key: 'idle-down',  frames: [{ key: 'character', frame: 0  }], frameRate: 1 });
    this.anims.create({ key: 'idle-right', frames: [{ key: 'character', frame: 17 }], frameRate: 1 });
    this.anims.create({ key: 'idle-up',    frames: [{ key: 'character', frame: 34 }], frameRate: 1 });
    this.anims.create({ key: 'idle-left',  frames: [{ key: 'character', frame: 51 }], frameRate: 1 });

    this.player.anims.play('idle-down');
    this.cursors = this.input.keyboard.createCursorKeys();
    this.lastDirection = 'down';

    // カメラ追従
    this.cameras.main.startFollow(this.player);
    this.cameras.main.setBounds(0, 0, map.widthInPixels, map.heightInPixels);

    // 移動範囲をマップに制限
    this.physics.world.setBounds(0, 0, map.widthInPixels, map.heightInPixels);
    this.player.setCollideWorldBounds(true);
  }

  update() {
    const speed = 80;
    let moving = false;

    if (this.cursors.left.isDown) {
      this.player.setVelocityX(-speed);
      this.player.setVelocityY(0);
      this.player.anims.play('walk-left', true);
      this.lastDirection = 'left';
      moving = true;
    } else if (this.cursors.right.isDown) {
      this.player.setVelocityX(speed);
      this.player.setVelocityY(0);
      this.player.anims.play('walk-right', true);
      this.lastDirection = 'right';
      moving = true;
    } else if (this.cursors.up.isDown) {
      this.player.setVelocityY(-speed);
      this.player.setVelocityX(0);
      this.player.anims.play('walk-up', true);
      this.lastDirection = 'up';
      moving = true;
    } else if (this.cursors.down.isDown) {
      this.player.setVelocityY(speed);
      this.player.setVelocityX(0);
      this.player.anims.play('walk-down', true);
      this.lastDirection = 'down';
      moving = true;
    }

    if (!moving) {
      this.player.setVelocityX(0);
      this.player.setVelocityY(0);
      this.player.anims.play(`idle-${this.lastDirection}`, true);
    }
  }
}
```

---

# STEP5: スプライトシートのコマ割りを調べる方法

スプライトシートのコマ番号を間違えるとアニメーションがおかしくなります。Pythonで事前に確認する方法が便利です。

```bash
pip3 install pillow --break-system-packages
```

```python
python3 -c "
from PIL import Image
img = Image.open('character.png')
w, h = img.size
print('サイズ:', w, 'x', h)

for fw in [16, 17, 32, 34]:
    for fh in [32, 34]:
        if w % fw == 0 and h % fh == 0:
            cols = w // fw
            rows = h // fh
            print(f'frameWidth:{fw} frameHeight:{fh} → {cols}列 × {rows}行 = {cols*rows}コマ')
"
```

実行結果:
```
サイズ: 272 x 256
frameWidth:16 frameHeight:32 → 17列 × 8行 = 136コマ
frameWidth:17 frameHeight:32 → 16列 × 8行 = 128コマ
frameWidth:34 frameHeight:32 → 8列 × 8行 = 64コマ
```

今回の素材は`frameWidth:16, frameHeight:32`（17列×8行）が正解でした。

行ごとのコマ番号は`列数 × 行番号`で計算できます。

```
行1（下向き）:  0  × 17 =  0〜3
行2（右向き）:  1  × 17 = 17〜20
行3（上向き）:  2  × 17 = 34〜37
行4（左向き）:  3  × 17 = 51〜54
```

---

# ハマったポイント

## 1. タイルセットの外部参照問題

Tiledのデフォルトエクスポートは外部参照形式になるため、Phaser.jsでエラーになります。`field.json`の`tilesets`を手動編集する必要があります。

## 2. スプライトシートのコマ番号

`frameWidth`の値によってコマ番号が変わります。Pythonで事前に確認するのが確実です。

## 3. Tiledのレイヤー名

レイヤー名を日本語にした場合（例: `草`）、Phaser.jsのコードでも同じ名前を使う必要があります。

```javascript
// レイヤー名はTiledと一致させる
const layer = map.createLayer('草', tileset, 0, 0);
```

---

# まとめ

TiledとPhaser.jsを組み合わせることで、本格的なRPGフィールドを実装できました。

次回はランダムエンカウントとバトル画面を実装します！

---

# ソースコード

https://github.com/naoki-sato-ojiyan/manabi-quest