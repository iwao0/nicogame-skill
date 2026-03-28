---
description: "Akashic Engineを使ったニコ生ゲーム（ニコニコ新市場）の開発を支援するスキル。ランキングゲーム、マルチプレイゲームの作成、TypeScript/JavaScript対応、game.json設定、エンティティ操作、シーン管理、音声、アニメーション、当たり判定、マルチプレイ同期など。"
user_invocable: true
---

# Akashic Engine ニコ生ゲーム開発スキル

あなたはAkashic Engineを使ったニコ生ゲーム（ニコニコ新市場向けゲーム）の開発エキスパートです。
以下の知識に基づいて、ユーザーのゲーム開発を支援してください。

---

## 1. Akashic Engine 概要

Akashic EngineはJavaScriptで動作する2Dゲームエンジンです。ニコニコ生放送上で配信者と視聴者が同時にプレイする「ニコ生ゲーム」の開発に使われます。

- **ライセンス**: フリーソフトウェア、ロイヤリティ不要
- **対応言語**: JavaScript / TypeScript
- **描画**: 2D特化
- **公式ドキュメント**:
  - 入門: https://akashic-games.github.io/tutorial/v3/
  - 逆引きリファレンス: https://akashic-games.github.io/reverse-reference/v3/
  - ニコ生ゲーム開発: https://akashic-games.github.io/shin-ichiba/
  - APIリファレンス: https://akashic-games.github.io/akashic-engine/v3/

---

## 2. 環境セットアップ

### 必要ツール
- Node.js (LTS推奨)
- akashic-cli: `npm install -g @akashic/akashic-cli`

### プロジェクト作成

**JavaScript ランキングゲーム:**
```bash
akashic init -t javascript-shin-ichiba-ranking
```

**TypeScript ランキングゲーム:**
```bash
akashic init -t typescript-shin-ichiba-ranking
```

**TypeScript 汎用:**
```bash
akashic init -t typescript
```

### 開発フロー (TypeScript)
```bash
npm install        # 依存パッケージ導入
npm run build      # TypeScript → JavaScript コンパイル
npm run update     # game.json のアセット・ライブラリ情報更新
npm start          # akashic-sandbox起動 (http://localhost:3000/game/)
```

### マルチプレイテスト
```bash
akashic serve                           # http://localhost:3300/
akashic serve --target-service nicolive  # ニコ生環境シミュレート（放送者判別テスト可能）
```

---

## 3. ニコ生ゲームの種類

### ランキングゲーム
- 個人プレイでスコアを競う
- スコアは `g.game.vars.gameState.score` に0以上の整数を代入（99999以下推奨）
- `g.game.vars.gameState.playThreshold` でランキング対象外閾値設定可能
- ニコ生が自動で順位集計・発表

### マルチプレイゲーム
- 全員で同じゲームを共有（協力・対戦）
- 操作は全プレイヤーに共有され、同一フレーム・同一イベントで決定論的に実行
- 順位表やUI等はゲーム内で自作

---

## 4. game.json 設定

### ニコ生ゲーム設定
```json
{
  "width": 1280,
  "height": 720,
  "fps": 30,
  "main": "./script/main.js",
  "assets": { ... },
  "environment": {
    "nicolive": {
      "supportedModes": ["ranking"],
      "preferredSessionParameters": {
        "totalTimeLimit": 82
      }
    }
  }
}
```

### 対応モード (`supportedModes`)
| モード | 説明 |
|--------|------|
| `"single"` | ひとりプレイ（操作は放送者のみ） |
| `"ranking"` | ランキング（自動終了＋スコア集計） |
| `"multi_admission"` | マルチプレイ（プレイヤー募集あり） |
| `"multi"` | マルチプレイ（募集なし） |

### 技術仕様・制限
- 画面解像度: 1280x720以下（16:9推奨）
- FPS: 1〜60（30または60推奨）
- zipサイズ: 30MB以下
- game.json: 100KB以下
- `totalTimeLimit`: 20〜200の整数（秒）

---

## 5. コア概念

### エンティティ (Entity)
シーン上で描画されるオブジェクト。

| エンティティ | 用途 |
|-------------|------|
| `g.FilledRect` | 塗りつぶし矩形 |
| `g.Sprite` | 画像表示 |
| `g.FrameSprite` | フレームアニメーション |
| `g.Label` | テキスト表示 |
| `g.E` | グループ化用コンテナ |
| `g.Pane` | クリッピング付きグループ |

**エンティティ作成の基本:**
```javascript
const rect = new g.FilledRect({
  scene: scene,
  cssColor: "red",
  width: 50,
  height: 50,
  x: 100,
  y: 100
});
scene.append(rect);
```

**主要プロパティ:**
| プロパティ | 型 | 説明 |
|-----------|-----|------|
| `x`, `y` | number | 位置 |
| `width`, `height` | number | サイズ |
| `opacity` | 0.0〜1.0 | 透明度 |
| `scaleX`, `scaleY` | number | 拡大率 |
| `angle` | number | 回転角度（度） |
| `anchorX`, `anchorY` | number | 回転の基準点 (0.0〜1.0) |
| `touchable` | boolean | タッチ可能か |
| `local` | boolean | ローカルエンティティか |
| `hidden` | boolean | 非表示か |

**プロパティ変更後は `modified()` を呼ぶ:**
```javascript
rect.x += 10;
rect.modified();
```

**エンティティ削除:**
```javascript
rect.destroy();
```

### シーン (Scene)
アセット管理・エンティティ管理・ゲームロジックの単位。

```javascript
const scene = new g.Scene({
  game: g.game,
  assetPaths: ["/image/player.png", "/audio/bgm"]
});

scene.onLoad.add(() => {
  // シーンのアセット読み込み完了後の処理
});

g.game.pushScene(scene);
```

**シーン遷移メソッド:**
| メソッド | 説明 |
|---------|------|
| `g.game.pushScene(scene)` | 現在のシーンを保持して新シーンへ |
| `g.game.replaceScene(scene)` | 現在のシーンを破棄して新シーンへ |
| `g.game.popScene()` | pushしたシーンに戻る |

### アセット管理
推奨ディレクトリ: `image/`, `audio/`, `text/`, `script/`, `assets/`

```bash
akashic scan asset  # ファイルを自動検出してgame.jsonに登録
```

**アセット取得:**
```javascript
const img = scene.asset.getImage("/image/player.png");
const audio = scene.asset.getAudio("/audio/se1");
```

---

## 6. 画像表示 (Sprite)

```javascript
const sprite = new g.Sprite({
  scene: scene,
  src: scene.asset.getImage("/image/player.png"),
  x: 100,
  y: 200
});
scene.append(sprite);
```

**部分表示 (スプライトシート):**
```javascript
const sprite = new g.Sprite({
  scene: scene,
  src: scene.asset.getImage("/image/spritesheet.png"),
  width: 64,
  height: 64,
  srcX: 128,
  srcY: 0,
  srcWidth: 64,
  srcHeight: 64
});
```

---

## 7. フレームアニメーション

```javascript
const frameSprite = new g.FrameSprite({
  scene: scene,
  src: scene.asset.getImage("/image/explosion.png"),
  width: 100,
  height: 100,
  srcWidth: 100,
  srcHeight: 100,
  frames: [0, 1, 2, 3, 4, 5],
  loop: true
});
scene.append(frameSprite);
frameSprite.start();
// frameSprite.stop(); で停止
```

---

## 8. アニメーション (プロパティベース)

### フレーム毎更新
```javascript
// シーン単位
scene.onUpdate.add(() => {
  rect.x += 2;
  rect.modified();
});

// エンティティ単位（エンティティ破棄時に自動停止）
rect.onUpdate.add(() => {
  rect.angle += 1;
  rect.modified();
});
```

### タイマー
**重要: `setTimeout`/`setInterval` ではなく `scene.setTimeout`/`scene.setInterval` を使うこと。** ゲームの再生速度に連動するため。

```javascript
scene.setTimeout(() => {
  rect.destroy();
}, 3000);

const id = scene.setInterval(() => {
  rect.x += 10;
  rect.modified();
}, 200);

scene.clearInterval(id);
```

### akashic-timeline (トゥイーンライブラリ)
```bash
akashic install @akashic-extension/akashic-timeline
```

```javascript
const tl = require("@akashic-extension/akashic-timeline");
const timeline = new tl.Timeline(scene);

timeline.create(rect)
  .moveTo(300, 200, 1000)     // 1秒かけて(300,200)へ移動
  .scaleTo(2, 2, 500)          // 0.5秒かけて2倍に拡大
  .rotateTo(360, 800);         // 0.8秒かけて360度回転
```

---

## 9. クリック・タッチ操作

エンティティに `touchable: true` を設定すると操作イベントを受け取れる。

```javascript
const button = new g.FilledRect({
  scene: scene,
  cssColor: "blue",
  width: 100,
  height: 50,
  touchable: true
});
scene.append(button);

button.onPointDown.add((ev) => {
  // タッチ開始
  console.log(ev.point.x, ev.point.y);
});

button.onPointMove.add((ev) => {
  // ドラッグ
  button.x += ev.prevDelta.x;
  button.y += ev.prevDelta.y;
  button.modified();
});

button.onPointUp.add((ev) => {
  // タッチ終了
});
```

**シーン全体のイベント（エンティティに関係なく取得）:**
```javascript
scene.onPointDownCapture.add((ev) => {
  // 画面のどこをタッチしても発火
});
```

**イベントの座標:**
| プロパティ | 説明 |
|-----------|------|
| `ev.point.x`, `ev.point.y` | タッチ位置 |
| `ev.startDelta` | PointDown位置からの差分 |
| `ev.prevDelta` | 前回PointMove位置からの差分 |

---

## 10. テキスト表示

```javascript
const font = new g.DynamicFont({
  game: g.game,
  fontFamily: "sans-serif",
  size: 30
});

const label = new g.Label({
  scene: scene,
  font: font,
  text: "Score: 0",
  fontSize: 30,
  textColor: "white",
  x: 10,
  y: 10
});
scene.append(label);

// テキスト更新時は invalidate() を呼ぶ
label.text = "Score: 100";
label.invalidate();
```

**fontFamily選択肢:** `"sans-serif"`, `"serif"`, `"monospace"`

---

## 11. 音声再生

音声ファイルは `.ogg` + `.m4a`（または `.aac`）の2形式ペアが必要。

```javascript
const scene = new g.Scene({
  game: g.game,
  assetPaths: ["/audio/se1", "/audio/bgm"]
});

scene.onLoad.add(() => {
  // 簡易再生
  const seAsset = scene.asset.getAudio("/audio/se1");
  g.game.audio.play(seAsset);

  // 音量付き再生
  const se = g.game.audio.create(seAsset);
  se.changeVolume(0.5);
  se.play();
});
```

**BGM (ループ再生):** game.jsonで `"systemId": "music"` を設定:
```json
{
  "bgm": {
    "type": "audio",
    "path": "audio/bgm",
    "systemId": "music",
    "duration": 8000
  }
}
```

---

## 12. 乱数生成

**重要: `Math.random()` は絶対に使わないこと。** マルチプレイで同期が崩れる。

```javascript
// グローバル乱数（全プレイヤーで同じ結果）
const val = g.game.random.generate();           // 0以上1未満
const intVal = Math.floor(g.game.random.generate() * 10);  // 0〜9の整数

// ローカル乱数（プレイヤーごとに異なる、ローカル処理内でのみ使用可）
const localVal = g.game.localRandom.generate();
```

---

## 13. 当たり判定

### 簡易判定 (g.Collision)
```javascript
// 矩形同士の判定
if (g.Collision.intersect(
  entityA.x, entityA.y, entityA.width, entityA.height,
  entityB.x, entityB.y, entityB.width, entityB.height
)) {
  // 衝突！
}
```

### 高度な判定 (collision-js)
```bash
akashic install @akashic-extension/collision-js
```

```javascript
const co = require("@akashic-extension/collision-js");

const box1 = {
  position: { x: 300, y: 200 },
  halfExtend: { x: 128, y: 96 },
  angle: 0
};
const box2 = {
  position: { x: 100, y: 100 },
  halfExtend: { x: 64, y: 48 },
  angle: 0
};
const hit = co.boxToBox(box1, box2);
// 他: co.segmentToBox(), AABB, 円, 線分, 点, ポリゴン対応
```

---

## 14. マルチプレイ開発

### 基本原理
Akashic Engineのマルチプレイは「操作の共有」で実現。全プレイヤーが同じスクリプトを同じフレーム数・同じイベントで実行し、決定論的に同期する。

### プレイヤー識別
```javascript
scene.onPointDownCapture.add((ev) => {
  const playerId = ev.player.id;
  // プレイヤーごとの処理
});
```

### Join / Leave
```javascript
let broadcasterId = null;

g.game.onJoin.add((ev) => {
  broadcasterId = ev.player.id;  // ニコ生では最初のjoinが放送者
});

g.game.onLeave.add((ev) => {
  // プレイヤー離脱処理
});

// join済みプレイヤー一覧
const playerIds = g.game.joinedPlayerIds;

// 自分のID（サーバ実行時はundefined）
const myId = g.game.selfId;
```

### ローカルエンティティとローカルイベント

ローカルエンティティは各プレイヤーのUI表示用。タッチイベントはローカルイベントになる。

```javascript
// ローカルエンティティ（プレイヤー個別UI）
const myButton = new g.FilledRect({
  scene: scene,
  local: true,
  cssColor: "green",
  width: 80,
  height: 40,
  touchable: true
});
scene.append(myButton);

// ローカルイベントから全体通知
myButton.onPointUp.add((ev) => {
  g.game.raiseEvent(new g.MessageEvent({ action: "attack", color: "green" }));
});

// 全プレイヤーでメッセージ受信
scene.onMessage.add((msg) => {
  const data = msg.data;
  // data.action, data.color を使ってグローバル処理
});
```

**ローカル処理の制約:**
- 非ローカルエンティティの作成・変更 **禁止**
- シーン遷移 **禁止**
- `g.game.random` の使用 **禁止**
- `g.game.raiseEvent()` の呼び出し **可能**
- ローカルエンティティの操作 **可能**
- `g.game.localRandom` の使用 **可能**

---

## 15. ニコ生固有機能

### プレイヤー名取得
```bash
akashic install @akashic-extension/resolve-player-info
```

game.jsonに追加:
```json
{
  "environment": {
    "external": {
      "coeLimited": "0"
    }
  }
}
```

```javascript
const { resolvePlayerInfo } = require("@akashic-extension/resolve-player-info");

const nameTable = {};

g.game.onPlayerInfo.add((ev) => {
  if (ev.player.name != null) {
    nameTable[ev.player.id] = ev.player.name;
  }
});

// ローカル処理内で呼び出す
resolvePlayerInfo({ raises: true, limitSeconds: 15 });
```

### 放送者判別
```javascript
let broadcasterId = null;

g.game.onJoin.add((ev) => {
  broadcasterId = ev.player.id;
});

scene.onPointDownCapture.add((ev) => {
  if (ev.player.id === broadcasterId) {
    // 放送者のみの操作
  }
  if (g.game.selfId === broadcasterId) {
    // 放送者の画面でのみ実行（ローカル処理）
  }
});
```

### ランキングゲームのセッションパラメータ
実行時に `g.MessageEvent` で通知される:

| パラメータ | 説明 |
|-----------|------|
| `service` | 常に `"nicolive"` |
| `mode` | 起動モード |
| `totalTimeLimit` | 制限時間（秒） |
| `randomSeed` | 共通乱数シード（ランキング時のみ） |
| `difficulty` | 難易度 (1〜10) |

**制限時間の10秒前には演出を完了させること（ネットワーク遅延考慮）。**

---

## 16. 拡張ライブラリ

```bash
akashic install <パッケージ名>     # インストール
akashic uninstall <パッケージ名>   # アンインストール
```

| ライブラリ | 用途 |
|-----------|------|
| `@akashic-extension/akashic-timeline` | トゥイーンアニメーション |
| `@akashic-extension/akashic-box2d` | 2D物理演算 |
| `@akashic-extension/akashic-label` | 高機能テキスト表示 |
| `@akashic-extension/collision-js` | 高度な当たり判定 |
| `@akashic-extension/resolve-player-info` | プレイヤー名取得 |

**非Akashicパッケージの利用条件:**
- Node.jsコアモジュール（`fs`, `http`等）を使わない
- `Math.random()` を使わない
- DOM APIを使わない

---

## 17. ゲームの公開

### HTML5出力
```bash
akashic export html --output game.zip
```

### ニコ生ゲーム出力
```bash
akashic export zip --nicolive --output game.zip
```

投稿先: https://game.nicovideo.jp/atsumaru/ から投稿

---

## 18. よくあるパターン

### ランキングゲームの基本テンプレート (TypeScript)
```typescript
function main(param: g.GameMainParameterObject): void {
  const scene = new g.Scene({
    game: g.game,
    assetPaths: ["/image/player.png", "/audio/se1"]
  });

  scene.onLoad.add(() => {
    let score = 0;
    let timeLimit = 60;

    // セッションパラメータ受信
    if (param.sessionParameter && param.sessionParameter.totalTimeLimit) {
      timeLimit = param.sessionParameter.totalTimeLimit;
    }

    const font = new g.DynamicFont({
      game: g.game,
      fontFamily: "sans-serif",
      size: 48
    });

    const scoreLabel = new g.Label({
      scene, font, text: "Score: 0",
      fontSize: 48, textColor: "white",
      x: 10, y: 10
    });
    scene.append(scoreLabel);

    // ゲームロジック
    scene.onPointDownCapture.add(() => {
      score += 10;
      scoreLabel.text = `Score: ${score}`;
      scoreLabel.invalidate();
      g.game.vars.gameState = { score };
    });

    // 毎フレーム更新
    scene.onUpdate.add(() => {
      // ゲームロジック更新
    });
  });

  g.game.pushScene(scene);
}

export = main;
```

---

## 開発時の重要な注意事項

1. **`Math.random()` は使わない** → `g.game.random.generate()` を使う
2. **`setTimeout`/`setInterval` は使わない** → `scene.setTimeout`/`scene.setInterval` を使う
3. **DOM APIは使わない** (canvas直接操作、document、window等)
4. **プロパティ変更後は `modified()` を呼ぶ** (位置、サイズ、角度等)
5. **テキスト変更後は `invalidate()` を呼ぶ**
6. **ローカル処理内でグローバルな状態を変更しない**
7. **音声は .ogg + .m4a の2形式を用意する**
8. **制限時間の10秒前には結果表示を完了させる**
9. **zipサイズ30MB以下、画面サイズ1280x720以下**

---

## ドキュメント参照先

ユーザーの質問に対して、より詳細な情報が必要な場合は以下のURLからWebFetchで取得してください:

- チュートリアル一覧: https://akashic-games.github.io/tutorial/v3/
- 逆引きリファレンス: https://akashic-games.github.io/reverse-reference/v3/
- ニコ生ゲーム開発: https://akashic-games.github.io/shin-ichiba/
- API リファレンス: https://akashic-games.github.io/akashic-engine/v3/
- 開発ツール: https://akashic-games.github.io/reference/tool/
- game.json仕様: https://akashic-games.github.io/reference/manifest/
- サンプルデモ: https://akashic-games.github.io/demo/
- Tips: https://akashic-games.github.io/tips/
- デザインガイドライン: https://akashic-games.github.io/shin-ichiba/design-guidelines.html
