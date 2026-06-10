---
title: "【まなびクエスト開発日記】ランダムエンカウント＆バトル画面を実装した"
emoji: "⚔️"
type: "tech"
topics: ["phaser", "javascript", "gamedev"]
published: true
---

# はじめに

小学生向け学習RPG「まなびクエスト」を開発しています。

前回はTiledでマップを作り、フィールド画面を実装しました。

https://zenn.dev/naokisatoojiyan/articles/manabi-quest-field

今回はドラクエ風の**ランダムエンカウント**と**バトル画面**を実装した内容をまとめます。

## 完成イメージ

- フィールドを歩くとランダムでバトルが発生する
- バトル画面で算数の問題が出題される
- 正解→敵にダメージ（クリティカルあり）
- 不正解→敵の反撃（回避あり）
- 敵を倒す→経験値獲得・レベルアップ

---

# STEP6: ランダムエンカウント

## 仕組み

歩数ベースではなく**距離ベース**でエンカウント判定をします。

```
フィールドを移動する
　↓
一定距離（200px）移動するたびに抽選（30%）
　↓
当たり → BattleSceneへ切り替え
```

歩数ベースにすると「キーを押しっぱなし」のときにカウントされないため、距離ベースの方が自然です。

## FieldScene.js（エンカウント部分）

```javascript
create() {
    // ...（省略）

    // エンカウント関連の変数
    this.distanceMoved = 0;        // 移動距離の累計
    this.distanceThreshold = 200;  // 何px移動したら抽選するか
    this.encounterRate = 0.3;      // エンカウント確率（30%）
}

update() {
    const speed = 80;
    let moving = false;

    // ...（移動処理は省略）

    // エンカウント処理
    if (moving) {
        this.distanceMoved += speed / 60; // 1フレームあたりの移動距離を加算

        if (this.distanceMoved >= this.distanceThreshold) {
            this.distanceMoved = 0;

            if (Math.random() < this.encounterRate) {
                this.scene.start('BattleScene');
            }
        }
    }
}
```

### `speed / 60` の意味

```
speed = 80（px/秒）
60fps で動いているので
1フレームあたり 80 ÷ 60 ≈ 1.3px 移動している
```

## main.js への追加

BattleSceneを登録します。

```javascript
import FieldScene from './scenes/FieldScene.js';
import BattleScene from './scenes/BattleScene.js'; // 追加

const config = {
    scene: [FieldScene, BattleScene], // 追加
    // ...
};
```

## zoomについての注意

`main.js` に `zoom: 2` を設定している場合、バトル画面でも2倍になってしまいテキストが画面からはみ出します。

対策として各シーンで `setZoom()` を呼び出します。

```javascript
// FieldScene.js の create() に追加
this.cameras.main.setZoom(2); // フィールドは2倍

// BattleScene.js の create() に追加
this.cameras.main.setZoom(1); // バトルは1倍
```

`main.js` の `zoom: 2` は削除してOKです。

---

# STEP7: バトル画面

## ステータス管理

プレイヤーのステータスは `GameData.js` で一元管理します。

```javascript
// src/GameData.js
const GameData = {
    player: {
        level: 1,
        hp: 10,
        maxHp: 10,
        exp: 0,
        attack: 5,    // 攻撃力（敵へのダメージに影響）
        defense: 5,   // 防御力（受けるダメージを軽減）
        agility: 5,   // 素早さ（回避率に影響）
        luck: 5,      // 運（クリティカル率に影響）
    }
};

export default GameData;
```

### なぜ別ファイルに分けるのか

```
FieldScene.js と BattleScene.js の両方からアクセスする必要があるため
→ どのシーンからでもimportして使える
```

### セキュリティの注意点

JavaScriptの変数はブラウザの開発者ツールから簡単に書き換えられます。

```javascript
// 開発者ツールのConsoleで実行するだけで改ざんできる
GameData.player.attack = 999;
```

対策はバックエンド（Django）側でバリデーションすることです。**「クライアントから来たデータは必ずサーバー側で検証する」** のがWebアプリ開発の鉄則です。

## ステータスの計算式

| ステータス | 使いどころ | 計算式 |
|-----------|-----------|--------|
| attack | 敵へのダメージ | `1 + Math.floor(attack / 3)` |
| defense | 受けるダメージ軽減 | `Math.max(1, 2 - Math.floor(defense / 3))` |
| agility | 回避率 | `agility × 3%` |
| luck | クリティカル率 | `luck × 2%` |

## BattleScene.js

```javascript
import GameData from '../GameData.js';

export default class BattleScene extends Phaser.Scene {
    constructor() {
        super({ key: 'BattleScene' });
    }

    create() {
        this.add.rectangle(0, 0, this.scale.width, this.scale.height, 0x000000)
            .setOrigin(0, 0);

        // 敵データ
        this.enemy = {
            name: 'スライム',
            maxHP: 3,
            hp: 3,
            level: 1,
            baseExp: 10,
        };

        this.question = this.generateQuestion();
        this.inputText = '';
        this.battleEnded = false;

        this.cameras.main.setZoom(1);

        // 表示
        this.enemyText = this.add.text(
            this.scale.width / 2, 60,
            `${this.enemy.name}  HP: ${this.enemy.hp}`,
            { fontSize: '28px', fill: '#ffffff' }
        ).setOrigin(0.5);

        this.playerText = this.add.text(
            this.scale.width / 2, 110,
            this.getPlayerStatusText(),
            { fontSize: '24px', fill: '#aaffaa' }
        ).setOrigin(0.5);

        this.questionText = this.add.text(
            this.scale.width / 2, 200,
            `もんだい: ${this.question.text}`,
            { fontSize: '32px', fill: '#ffff00' }
        ).setOrigin(0.5);

        this.inputDisplay = this.add.text(
            this.scale.width / 2, 270,
            'こたえ: _',
            { fontSize: '32px', fill: '#ffffff' }
        ).setOrigin(0.5);

        this.resultText = this.add.text(
            this.scale.width / 2, 350,
            '',
            { fontSize: '28px', fill: '#ffffff' }
        ).setOrigin(0.5);

        this.add.text(
            this.scale.width / 2, this.scale.height - 40,
            '数字を入力してEnterで回答',
            { fontSize: '20px', fill: '#aaaaaa' }
        ).setOrigin(0.5);

        this.input.keyboard.on('keydown', this.handleInput, this);
    }

    getPlayerStatusText() {
        const p = GameData.player;
        return `Lv.${p.level}  HP: ${p.hp}/${p.maxHp}  攻:${p.attack}  防:${p.defense}  素:${p.agility}  運:${p.luck}  EXP:${p.exp}`;
    }

    generateQuestion() {
        const a = Phaser.Math.Between(1, 9);
        const b = Phaser.Math.Between(1, 9);
        return { text: `${a} + ${b} = ?`, answer: a + b };
    }

    handleInput(event) {
        if (this.battleEnded) return;

        if (event.key >= '0' && event.key <= '9') {
            this.inputText += event.key;
            this.inputDisplay.setText(`こたえ: ${this.inputText}`);
        }
        if (event.key === 'Backspace') {
            this.inputText = this.inputText.slice(0, -1);
            this.inputDisplay.setText(`こたえ: ${this.inputText || '_'}`);
        }
        if (event.key === 'Enter' && this.inputText !== '') {
            this.checkAnswer();
        }
    }

    checkAnswer() {
        const playerAnswer = parseInt(this.inputText);

        if (playerAnswer === this.question.answer) {
            // クリティカル判定
            const isCritical = Math.random() < GameData.player.luck * 0.02;
            let damage = 1 + Math.floor(GameData.player.attack / 3);
            if (isCritical) damage *= 2;

            this.enemy.hp -= damage;
            this.enemyText.setText(`${this.enemy.name}  HP: ${Math.max(0, this.enemy.hp)}`);

            if (isCritical) {
                this.resultText.setStyle({ fill: '#ffaa00' });
                this.resultText.setText(`クリティカル！${this.enemy.name}に${damage}ダメージ！`);
            } else {
                this.resultText.setStyle({ fill: '#44ff44' });
                this.resultText.setText(`せいかい！${this.enemy.name}に${damage}ダメージ！`);
            }

            if (this.enemy.hp <= 0) {
                const expGain = Math.floor(this.enemy.baseExp * this.enemy.level * 0.4);
                const levelUp = this.gainExp(expGain);
                this.battleEnded = true;
                this.input.keyboard.off('keydown', this.handleInput, this);

                if (levelUp) {
                    this.resultText.setStyle({ fill: '#ffff00' });
                    this.resultText.setText(
                        `${this.enemy.name}をたおした！\n${expGain}EXP獲得！\nレベルアップ！Lv.${GameData.player.level}になった！`
                    );
                } else {
                    this.resultText.setStyle({ fill: '#44ff44' });
                    this.resultText.setText(`${this.enemy.name}をたおした！\n${expGain}EXP獲得！`);
                }

                this.playerText.setText(this.getPlayerStatusText());
                this.time.delayedCall(2500, () => { this.scene.start('FieldScene'); });
                return;
            }

            this.question = this.generateQuestion();
            this.questionText.setText(`もんだい: ${this.question.text}`);

        } else {
            // 回避判定
            const dodged = Math.random() < GameData.player.agility * 0.03;

            if (dodged) {
                this.resultText.setStyle({ fill: '#44aaff' });
                this.resultText.setText('ちがう！でも攻撃をかわした！');
            } else {
                const damage = Math.max(1, 2 - Math.floor(GameData.player.defense / 3));
                GameData.player.hp -= damage;
                this.playerText.setText(this.getPlayerStatusText());
                this.resultText.setStyle({ fill: '#ff4444' });
                this.resultText.setText(`ちがう！${this.enemy.name}の反撃！${damage}ダメージ！`);

                if (GameData.player.hp <= 0) {
                    this.battleEnded = true;
                    this.resultText.setText('やられた…ゲームオーバー');
                    this.input.keyboard.off('keydown', this.handleInput, this);
                    this.time.delayedCall(2000, () => {
                        GameData.player.hp = GameData.player.maxHp;
                        this.scene.start('FieldScene');
                    });
                    return;
                }
            }
        }

        this.inputText = '';
        this.inputDisplay.setText('こたえ: _');
    }

    // STEP8で使用：経験値・レベルアップ処理
    gainExp(amount) {
        const p = GameData.player;
        p.exp += amount;

        const requiredExp = p.level * 10; // 必要EXP = レベル × 10

        if (p.exp >= requiredExp) {
            p.exp -= requiredExp; // 余りを持ち越す
            p.level++;
            p.maxHp   += 3;
            p.attack  += 1;
            p.defense += 1;
            p.agility += 1;
            p.luck    += 1;
            p.hp = p.maxHp; // HP全回復
            return true;
        }
        return false;
    }
}
```

---

# STEP8: 経験値・レベルアップ

## 経験値の計算式

```
獲得EXP = 基本EXP × 敵レベル × 調整値（0.4）

例：
スライム（baseExp:10、Lv1）→ 10 × 1 × 0.4 = 4EXP
スライム（baseExp:10、Lv2）→ 10 × 2 × 0.4 = 8EXP
ゴブリン（baseExp:15、Lv1）→ 15 × 1 × 0.4 = 6EXP
```

調整値を掛けることで、高レベルの敵からもらいすぎるのを防ぎます。調整値はバランス調整のタイミングで変更予定です。

## レベルアップ時のステータス上昇

```
maxHp   +3
attack  +1
defense +1
agility +1
luck    +1
hp      全回復（maxHpに設定）
```

## 必要EXPの計算式

```
必要EXP = 現在レベル × 10
Lv1→2：10EXP
Lv2→3：20EXP
Lv3→4：30EXP
```

レベルが上がるほど必要EXPが増えるシンプルな設計です。

---

# ハマったポイント

## 1. アニメーションの重複登録

`this.scene.start('FieldScene')` でフィールドに戻ったとき、`create()` が再度呼ばれてアニメーションが重複登録されてエラーになります。

```javascript
// 対策：登録済みかチェックしてからcreateする
if (!this.anims.exists('walk-down')) {
    this.anims.create({ key: 'walk-down', ... });
    // ...
}
```

## 2. バトル終了後のキー入力

`battleEnded` フラグを使ってバトル終了後のキー入力を無効化します。

```javascript
handleInput(event) {
    if (this.battleEnded) return; // バトル終了後は何もしない
    // ...
}
```

---

# まとめ

Phaser.jsのシーン切り替えを使うことで、フィールドとバトルをスムーズに行き来できるようになりました。

ステータス管理を `GameData.js` に分離することで、複数シーンからアクセスしやすい設計になっています。

次回はDjango + DRFでバックエンドAPIを構築し、ステータスをDBに保存する実装を紹介します！

---

# ソースコード

https://github.com/naoki-sato-ojiyan/manabi-quest