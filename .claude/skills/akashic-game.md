---
description: "Akashic Engineを使ったニコ生ゲーム（ニコニコ新市場）の開発を支援するスキル。ランキングゲーム、マルチプレイゲームの作成、TypeScript/JavaScript対応、game.json設定、エンティティ操作、シーン管理、音声、アニメーション、当たり判定、マルチプレイ同期、3Dゲーム開発（akashic-three/Three.js）など。ユーザーが「ゲームを作りたい」「Akashic」「ニコ生ゲーム」「ニコニコ」「ランキングゲーム」「ブロック崩し」「シューティング」「3Dゲーム」「パターゴルフ」「Three.js」などゲーム開発に関するリクエストをした場合、画像アセットなしで素早くプロトタイプする場合にもこのスキルを使用すること。"
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

### ⚠️ `akashic init` のファイル競合

`akashic init` は `.gitignore` や `README.md` など既存ファイルがあると失敗する。
既存リポジトリ内で新規プロジェクトを作る場合は **サブディレクトリ** を作成してからその中で実行すること:

```bash
mkdir my-game && cd my-game
akashic init -t typescript-shin-ichiba-ranking
```

対話プロンプト（width/height/fps）にはデフォルト値があるので Enter で進められるが、
非対話で実行したい場合は以下のようにパイプする:
```bash
echo -e "\n\n\n" | akashic init -t typescript-shin-ichiba-ranking
```

### 開発フロー (TypeScript)
```bash
npm install        # 依存パッケージ導入（postinstallでbuildも実行される）
npm run build      # TypeScript → JavaScript コンパイル + アセットスキャン
npm run update     # game.json のアセット・ライブラリ情報更新
npm start          # akashic-sandbox起動 (http://localhost:3000/game/)
```

### マルチプレイテスト
```bash
akashic serve                           # http://localhost:3300/
akashic serve --target-service nicolive  # ニコ生環境シミュレート（放送者判別テスト可能）
```

### サンドボックスのブラウザ自動起動を抑制

`akashic sandbox` は内部で `akashic-cli-serve` を使っており、デフォルトでPCのブラウザが自動で開く。Claude Previewなど別のブラウザで確認する場合、PCブラウザは不要なので `-B`（`--no-open-browser`）オプションで抑制できる:

```bash
akashic sandbox -B /path/to/game        # ブラウザを開かずにサーバー起動
akashic sandbox -B -p 3000 /path/to/game # ポート指定と組み合わせ
```

`.claude/launch.json` での設定例:
```json
{
  "runtimeExecutable": "npx",
  "runtimeArgs": ["akashic", "sandbox", "-B", "/path/to/game"],
  "port": 3000
}
```

### Fit to screen（画面サイズ合わせ）

`akashic sandbox` にはゲーム画面をウインドウに合わせて表示する「Fit to screen」をCLIから設定するオプションがない。毎回手動で画面右上の **Display Options → Fit to screen** チェックボックスをONにする必要がある。

Claude Previewで操作する場合は、Display Optionsボタン（imageアイコン）をクリックし、Fit to screenのチェックボックスをクリックする。

---

## 3. ⚠️ TypeScript テンプレートの重要な制約

テンプレートの `tsconfig.json` は `"lib": ["es5"]` が設定されている。
これによりES2015+の配列メソッドや構文が**コンパイルエラー**になる。

### 使えないもの（コンパイルエラーになる）
```typescript
// ❌ for...of（配列に対して）→ TS2488エラー
for (const item of items) { ... }

// ❌ Array.prototype.every / some / find / findIndex
items.every(item => item.done);
items.find(item => item.id === targetId);

// ❌ Array.from, Array.of
Array.from(someIterable);

// ❌ Map, Set, Symbol, Promise
const map = new Map<string, number>();
```

### 代替パターン（必ずこちらを使う）

**⚠️ すべてのコードで `var` と `function()` を使うこと。`let`/`const`/アロー関数 `=>` は使わない。**
これらはコンパイルは通ることがあるが、Akashic Engine の実行環境（ES5）では動作しない場合がある。
統一して `var` + `function()` で書くことで、環境差異による不具合を防ぐ。

```typescript
// ✅ 通常のforループ（varを使う）
for (var i = 0; i < items.length; i++) {
  var item = items[i];
  // ...
}

// ✅ 全要素チェック（everyの代替）
var allDone = true;
for (var i = 0; i < items.length; i++) {
  if (!items[i].done) { allDone = false; break; }
}

// ✅ 要素検索（findの代替）
var found: any = null;
for (var i = 0; i < items.length; i++) {
  if (items[i].id === targetId) { found = items[i]; break; }
}

// ✅ filter/mapはES5なので使用可能（ただしアロー関数ではなくfunction）
var activeItems = items.filter(function(item: any) { return item.active; });
var names = items.map(function(item: any) { return item.name; });
```

### テンプレートの型定義
テンプレートは独自の `GameMainParameterObject` をインポートする:
```typescript
import { GameMainParameterObject } from "./parameterObject";

export function main(param: GameMainParameterObject): void {
  // ...
}
```

`g.GameMainParameterObject` ではなくこちらを使うこと。

---

## 4. ニコ生ゲームの種類

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

## 5. game.json 設定

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

## 6. コア概念

### エンティティ (Entity)
シーン上で描画されるオブジェクト。

| エンティティ | 用途 |
|-------------|------|
| `g.FilledRect` | 塗りつぶし矩形（画像なしプロトタイプに最適） |
| `g.Sprite` | 画像表示 |
| `g.FrameSprite` | フレームアニメーション |
| `g.Label` | テキスト表示 |
| `g.E` | グループ化用コンテナ |
| `g.Pane` | クリッピング付きグループ |

**エンティティ作成の基本:**
```typescript
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
```typescript
rect.x += 10;
rect.modified();
```

**エンティティ削除:**
```typescript
rect.destroy();
```

### シーン (Scene)
アセット管理・エンティティ管理・ゲームロジックの単位。

アセットの指定方法は2通りある:
```typescript
// assetIds: game.jsonのアセットキー名で指定（テンプレートのデフォルト）
const scene = new g.Scene({
  game: g.game,
  assetIds: ["player", "shot", "se"]
});

// assetPaths: ファイルパスで指定
const scene = new g.Scene({
  game: g.game,
  assetPaths: ["/image/player.png", "/audio/se"]
});
```

```typescript
scene.onLoad.add(() => {
  // シーンのアセット読み込み完了後の処理
  // assetIdsを使った場合: scene.asset.getImageById("player")
  // assetPathsを使った場合: scene.asset.getImage("/image/player.png")
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

**アセット取得（2つのAPI）:**
```typescript
// assetIdsで指定した場合 → getXxxById()
const img = scene.asset.getImageById("player");
const audio = scene.asset.getAudioById("se");

// assetPathsで指定した場合 → getXxx()
const img = scene.asset.getImage("/image/player.png");
const audio = scene.asset.getAudio("/audio/se1");
```

---

## 7. 画像なしプロトタイピング

画像アセットを用意せずに `g.FilledRect` だけでゲームを素早く作るパターン。
ブロック崩し、シューティング、パズル等のプロトタイプに有効。

```typescript
// 背景
const bg = new g.FilledRect({
  scene, cssColor: "#1a1a2e", width: g.game.width, height: g.game.height
});
scene.append(bg);

// プレイヤー（パドル等）
const paddle = new g.FilledRect({
  scene, cssColor: "#ecf0f1",
  width: 160, height: 20,
  x: (g.game.width - 160) / 2, y: g.game.height - 60
});
scene.append(paddle);

// 敵やブロックを大量生成
const COLORS = ["#e74c3c", "#e67e22", "#f1c40f", "#2ecc71", "#3498db"];
for (let row = 0; row < 5; row++) {
  for (let col = 0; col < 10; col++) {
    const block = new g.FilledRect({
      scene, cssColor: COLORS[row],
      width: 110, height: 30,
      x: col * 118 + 50, y: row * 38 + 80
    });
    scene.append(block);
  }
}
```

テンプレートに含まれるSEアセット（`"se"`）はそのまま使える。
不要なアセット（`"player"`, `"shot"`）は `assetIds` から除外すればよい。

---

## 8. 画像表示 (Sprite)

```typescript
const sprite = new g.Sprite({
  scene: scene,
  src: scene.asset.getImageById("player"),
  x: 100,
  y: 200
});
scene.append(sprite);
```

**部分表示 (スプライトシート):**
```typescript
const sprite = new g.Sprite({
  scene: scene,
  src: scene.asset.getImageById("spritesheet"),
  width: 64,
  height: 64,
  srcX: 128,
  srcY: 0,
  srcWidth: 64,
  srcHeight: 64
});
```

---

## 9. フレームアニメーション

```typescript
const frameSprite = new g.FrameSprite({
  scene: scene,
  src: scene.asset.getImageById("explosion"),
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

## 10. アニメーション (プロパティベース)

### フレーム毎更新
```typescript
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
`setTimeout`/`setInterval` ではなく `scene.setTimeout`/`scene.setInterval` を使うこと。ゲームの再生速度に連動し、マルチプレイでの同期にも必要。

```typescript
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

```typescript
const tl = require("@akashic-extension/akashic-timeline");
const timeline = new tl.Timeline(scene);

timeline.create(rect)
  .moveTo(300, 200, 1000)     // 1秒かけて(300,200)へ移動
  .scaleTo(2, 2, 500)          // 0.5秒かけて2倍に拡大
  .rotateTo(360, 800);         // 0.8秒かけて360度回転
```

---

## 11. クリック・タッチ操作

エンティティに `touchable: true` を設定すると操作イベントを受け取れる。

```typescript
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
```typescript
// タッチ開始
scene.onPointDownCapture.add((ev) => {
  // ev.point.x, ev.point.y で画面上のタッチ位置が取れる
});

// ドラッグ移動（パドル操作等に使う）
scene.onPointMoveCapture.add((ev) => {
  paddle.x += ev.prevDelta.x;
  paddle.modified();
});
```

**イベントの座標:**
| プロパティ | 説明 |
|-----------|------|
| `ev.point.x`, `ev.point.y` | タッチ位置（onPointDownCapture時は画面座標） |
| `ev.startDelta` | PointDown位置からの累積差分 |
| `ev.prevDelta` | 前回PointMove位置からの差分（ドラッグ操作に最適） |

---

## 12. テキスト表示

```typescript
const font = new g.DynamicFont({
  game: g.game,
  fontFamily: "sans-serif",
  size: 48
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

## 13. 音声再生

音声ファイルは `.ogg` + `.m4a`（または `.aac`）の2形式ペアが必要。

```typescript
const scene = new g.Scene({
  game: g.game,
  assetIds: ["se", "bgm"]
});

scene.onLoad.add(() => {
  // SE再生
  const seAsset = scene.asset.getAudioById("se");
  seAsset.play();

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

## 14. 乱数生成

`Math.random()` は絶対に使わないこと。マルチプレイでは全プレイヤーが同じ乱数列を共有することでゲーム状態を同期しているため、`Math.random()` を使うと各プレイヤーで異なる結果になり同期が崩れる。

```typescript
// グローバル乱数（全プレイヤーで同じ結果）
const val = g.game.random.generate();           // 0以上1未満
const intVal = Math.floor(g.game.random.generate() * 10);  // 0〜9の整数

// ローカル乱数（プレイヤーごとに異なる、ローカル処理内でのみ使用可）
const localVal = g.game.localRandom.generate();
```

---

## 15. 当たり判定

### 簡易矩形判定 (g.Collision)
```typescript
if (g.Collision.intersect(
  entityA.x, entityA.y, entityA.width, entityA.height,
  entityB.x, entityB.y, entityB.width, entityB.height
)) {
  // 衝突！
}
```

### 衝突方向の判定パターン
ブロック崩し等で「どの面から衝突したか」を判定する場合:
```typescript
// オーバーラップ量から衝突方向を判定
const overlapLeft = ball.x + ballW - block.x;
const overlapRight = block.x + block.width - ball.x;
const overlapTop = ball.y + ballH - block.y;
const overlapBottom = block.y + block.height - ball.y;

const minOverlapX = Math.min(overlapLeft, overlapRight);
const minOverlapY = Math.min(overlapTop, overlapBottom);

if (minOverlapX < minOverlapY) {
  ballVx = -ballVx;  // 左右から衝突 → X反転
} else {
  ballVy = -ballVy;  // 上下から衝突 → Y反転
}
```

### 高度な判定 (collision-js)
```bash
akashic install @akashic-extension/collision-js
```

```typescript
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

## 16. マルチプレイ開発

### 基本原理
Akashic Engineのマルチプレイは「操作の共有」で実現。全プレイヤーが同じスクリプトを同じフレーム数・同じイベントで実行し、決定論的に同期する。

### プレイヤー識別
```typescript
scene.onPointDownCapture.add((ev) => {
  const playerId = ev.player.id;
  // プレイヤーごとの処理
});
```

### Join / Leave
```typescript
let broadcasterId: string | null = null;

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

```typescript
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

## 17. ニコ生固有機能

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

```typescript
const { resolvePlayerInfo } = require("@akashic-extension/resolve-player-info");

const nameTable: { [id: string]: string } = {};

g.game.onPlayerInfo.add((ev) => {
  if (ev.player.name != null) {
    nameTable[ev.player.id] = ev.player.name;
  }
});

// ローカル処理内で呼び出す
resolvePlayerInfo({ raises: true, limitSeconds: 15 });
```

### 放送者判別
```typescript
let broadcasterId: string | null = null;

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

制限時間の10秒前には演出を完了させること（ネットワーク遅延考慮）。

---

## 18. 拡張ライブラリ

```bash
akashic install <パッケージ名>     # インストール
akashic uninstall <パッケージ名>   # アンインストール
```

| ライブラリ | 用途 |
|-----------|------|
| `@akashic-extension/akashic-timeline` | トゥイーンアニメーション |
| `@akashic-extension/akashic-box2d` | 2D物理演算（アングリーバード、ピンボール、物理パズル等） |
| `@akashic-extension/akashic-keyboard-plugin` | キーボード入力（テトリス、アクション等） |
| `@akashic-extension/akashic-label` | 高機能テキスト表示 |
| `@akashic-extension/collision-js` | 高度な当たり判定 |
| `@akashic-extension/resolve-player-info` | プレイヤー名取得 |

**非Akashicパッケージの利用条件:**
- Node.jsコアモジュール（`fs`, `http`等）を使わない
- `Math.random()` を使わない
- DOM APIを使わない

### Box2D 物理演算を使う場合

物理演算が必要なゲーム（アングリーバード風、ピンボール、物理パズル、倒壊系など）を作る場合は、**必ず `references/box2d-physics.md` を読むこと**。以下の致命的な落とし穴がある:

1. **`b2BodyDef` / `b2FixtureDef` は `new` が必須** — `new` なしだと `Cannot set properties of undefined` エラー
2. **インパルス/力に `box2d.vec2()` を使うと値が1/50になる** — `new b2Vec2()` を直接使うこと
3. **`step()` 中に `removeBody()` するとクラッシュ** — 削除キューパターンを使うこと
4. **衝突の連続検出でブロックが一瞬で壊れる** — クールダウンシステムが必要

### キーボード入力を使う場合

テトリスやアクションゲームなどキーボード操作が必要なゲームでは `@akashic-extension/akashic-keyboard-plugin` を使う。

**⚠️ `game.json` の `operationPlugins` には登録しない** — モジュールのエクスポート形式が合わず `isSupported` エラーになる。必ずコードで登録すること。

```typescript
// main関数内、scene.onLoad の外で登録
var kbMod = require("@akashic-extension/akashic-keyboard-plugin");
g.game.operationPluginManager.register(kbMod.KeyboardOperationPlugin, 1);

// シーンがアクティブになったら開始
scene.onStateChange.add(function(state) {
  if (state === "active") {
    g.game.operationPluginManager.start(1);
  } else if (state === "deactive") {
    g.game.operationPluginManager.stop(1);
  }
});

// scene.onLoad 内でキーイベントを受け取る
scene.onOperation.add(function(ev) {
  if (ev.code !== 1) return;  // キーボードプラグインの識別コード
  if (!ev.data) return;
  var data = ev.data as any;
  if (data.type !== "keydown") return;  // "keydown" または "keyup"

  var key = data.key as string;  // KeyboardEvent.key の値
  if (key === "ArrowLeft") { /* 左移動 */ }
  else if (key === "ArrowRight") { /* 右移動 */ }
  else if (key === "ArrowUp") { /* 回転等 */ }
  else if (key === "ArrowDown") { /* 下移動 */ }
  else if (key === " ") { /* スペースキー */ }
  // data.code も使える（"KeyA", "Space" 等の物理キー名）
});
```

**重要なポイント:**
- `ev.data` はオブジェクト（`{type, key, code, shiftKey, altKey, ctrlKey, metaKey}`）で、配列ではない
- `ev.code` はプラグインの識別コード（ここでは `1`）。`data.code` はキーボードの物理キー名
- `data.key` は `KeyboardEvent.key` の値（`"ArrowLeft"`, `"a"`, `" "` など）
- `data.code` は `KeyboardEvent.code` の値（`"ArrowLeft"`, `"KeyA"`, `"Space"` など）

### `require()` の型定義

`@akashic-extension/*` パッケージを `require()` で読み込む場合、テンプレートに `typings/require.d.ts` が存在しないとコンパイルエラー（TS2591）になる。

```typescript
// typings/require.d.ts を作成
declare function require(moduleName: string): any;
```

`tsconfig.json` の `include` に `"typings/require.d.ts"` を追加すること。

---

## 19. sandbox.config.js（開発時の警告制御）

`akashic-sandbox` の実行時警告は `sandbox.config.js` で制御できる。プロジェクトルートに配置する。

```javascript
module.exports = {
  "warn": {
    "es6": false,           // ES6オブジェクト使用の警告を無効化
    "useMathRandom": false, // Math.random()使用の警告を無効化
    "useDate": false,       // Dateオブジェクト使用の警告を無効化
    "drawOutOfCanvas": false,         // 描画範囲外の警告を無効化
    "drawDestinationEmpty": false,    // 空描画先の警告を無効化
    "createNonIntegerSurface": false  // 非整数サーフェスの警告を無効化
  }
};
```

**よくある使い方:**
- Three.js等の外部ライブラリが内部で `Math.random()` を呼ぶ場合 → `"useMathRandom": false`
- ES2020の `lib` を使うプロジェクト（akashic-three等）→ `"es6": false`
- 全項目のデフォルトは `true`（警告ON）。必要なものだけ `false` にする

---

## 20. @iwao0/akashic-three（3Dゲーム開発）

`@iwao0/akashic-three` はThree.jsをAkashic Engine上で使うためのラッパーライブラリ。3Dゲームの開発が可能になる。

### セットアップ

```bash
cd my-game
npm install @iwao0/akashic-three
```

### 必須の設定変更

**1. game.json — `renderers` を `"webgl"` に設定:**
```json
{
  "renderers": "webgl"
}
```

**2. tsconfig.json — `lib` を `ES2020` + `DOM` に、`GPUTexture.d.ts` を追加:**
```json
{
  "compilerOptions": {
    "lib": ["ES2020", "DOM"],
    "target": "ES6"
  },
  "files": [
    "node_modules/@akashic/akashic-engine/index.runtime.d.ts",
    "node_modules/@iwao0/akashic-three/GPUTexture.d.ts"
  ]
}
```

**⚠️ akashic-three使用時はES5制約が適用されない。** `let`/`const`/アロー関数/`for...of` 等を自由に使ってよい。`lib: ["ES2020", "DOM"]` が必要なため、通常のES5テンプレートとは異なる。

**3. sandbox.config.js — 警告を抑制:**
```javascript
module.exports = {
  "warn": {
    "es6": false,
    "useMathRandom": false
  }
};
```

Three.jsは内部で `Math.random()` を使うため、`useMathRandom: false` が必要。

### 基本的な使い方

```typescript
import * as THREE from "three";
import { Sprite3D } from "@iwao0/akashic-three";
import { GameMainParameterObject } from "./parameterObject";

export function main(param: GameMainParameterObject): void {
  const scene = new g.Scene({
    game: g.game,
    assetIds: ["se"],
    assetPaths: [
      ...Sprite3D.getAssetPaths()
    ]
  });

  scene.onLoad.add(() => {
    const GW = g.game.width;
    const GH = g.game.height;

    const sprite3d = new Sprite3D({
      scene: scene,
      width: GW,
      height: GH
    });

    // WebGL非対応環境のフォールバック
    if (!sprite3d.isSupported) {
      const image = sprite3d.getWarningImage();
      const warning = new g.Sprite({
        scene: scene,
        src: image,
        x: GW / 2,
        y: GH - 60,
        anchorX: 0.5,
        anchorY: 1
      });
      scene.append(warning);
      return;
    }

    scene.append(sprite3d);

    // Three.jsのレンダラーを取得
    const renderer = sprite3d.renderer as THREE.WebGLRenderer;
    const scene3d = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(45, GW / GH, 0.1, 200);

    // メインループで描画
    scene.onUpdate.add(() => {
      renderer.render(scene3d, camera);
    });
  });

  g.game.pushScene(scene);
}
```

### 3Dゲームでのカメラ制御パターン

カメラが対象をリアルタイムに追従すると不自然になる。lerp（線形補間）で滑らかに追従させるのが効果的:

```typescript
// カメラの現在位置を保持
let camX = 0, camZ = 0;

function updateCamera(lerpSpeed: number): void {
  // 目標位置を計算
  const goalX = targetX - dirX * 6;
  const goalZ = targetZ - dirZ * 6;

  // 補間で滑らかに追従（距離が大きいほど速く追従）
  const dx = goalX - camX;
  const dz = goalZ - camZ;
  const dist = Math.sqrt(dx * dx + dz * dz);
  const lerp = Math.min(1, lerpSpeed + dist * 0.01);

  camX += dx * lerp;
  camZ += dz * lerp;

  camera.position.set(camX, 6, camZ);
  camera.lookAt(targetX, 0, targetZ);
}

// 使い分け:
// - ボール転がり中: updateCamera(0.03)  ← ゆっくり追従
// - 停止後（操作待ち）: updateCamera(0.15) ← 素早く追いつく
// - ホール切り替え: camX = goalX; camZ = goalZ; ← 即座に配置
```

### 2DのUI要素との共存

3D描画は `Sprite3D` エンティティ上で行われる。Akashicの2Dエンティティ（`g.Label`、`g.FilledRect` 等）は `Sprite3D` の上に重ねて配置できるため、スコアやタイマー等のUIは通常の2Dエンティティで作る:

```typescript
scene.append(sprite3d);  // 3D描画レイヤー

// UIは3Dの上に重ねる
const scoreLabel = new g.Label({ scene, font, text: "SCORE: 0", ... });
scene.append(scoreLabel);
```

---

## 21. デバッグログによるゲームテスト

スクリーンショットだけでは物理演算やゲームロジックの問題を正確に把握できない。ボールの軌道が「左に曲がった」ように見えても、実際には右に曲がって手前で止まっていた、ということがある。数値データに基づく分析が圧倒的に正確で効率的。

### ログ出力の設計

ゲームの重要なポイントに `console.log` を仕込み、入力値・物理パラメータ・座標を出力する:

```typescript
// ユーザー入力時
console.log("[TAP-POWER] gaugeValue=" + gaugeValue.toFixed(3));
console.log("[TAP-ACCURACY] value=" + accuracyValue.toFixed(3) + " angle=" + (angle * 180 / Math.PI).toFixed(1) + "deg");

// ショット発射時（全パラメータをまとめて出力）
console.log("[SHOOT] power=" + power.toFixed(3) + " angle=" + angle.toFixed(3) + " vx=" + vx.toFixed(3) + " vz=" + vz.toFixed(3));

// 物理更新（毎フレーム）
console.log("[BALL] x=" + x.toFixed(2) + " z=" + z.toFixed(2) + " speed=" + speed.toFixed(4) + " distToTarget=" + dist.toFixed(2) + " surface=" + surface);
```

### ログの接頭辞ルール

- `[TAP-POWER]`, `[TAP-ACCURACY]` — ユーザー入力イベント
- `[SHOOT]`, `[FIRE]` — アクション発生時の全パラメータ
- `[BALL]`, `[PLAYER]` — 毎フレーム物理状態

接頭辞でフィルタできるので、問題分析が容易になる。

### Claude Previewでのログ取得

`preview_console_logs` は直近100件しか返さないため、毎フレームログがあるとユーザー入力のログが流れてしまう。全ログを確保するには `preview_eval` でキャプチャを注入する:

```javascript
// ページロード後に注入
window.__allLogs = [];
var origLog = console.log;
console.log = function() {
  var msg = Array.prototype.slice.call(arguments).join(' ');
  window.__allLogs.push(msg);
  origLog.apply(console, arguments);
};
```

ログ取得時はフィルタして読む:
```javascript
var logs = window.__allLogs;
var tapShoot = logs.filter(function(l) { return l.indexOf('[TAP') === 0 || l.indexOf('[SHOOT') === 0; });
var ballFirst5 = logs.filter(function(l) { return l.indexOf('[BALL') === 0; }).slice(0, 5);
```

### ログ分析の例

以下のようなデータが取れたら:
```
[TAP-POWER] gaugeValue=0.750 (power will be 0.375)
[TAP-ACCURACY] value=-0.550 angle=-4.7deg
[SHOOT] power=0.375 vx=-0.031 vz=-0.374
[BALL] x=-0.92 z=-11.08 speed=0.005 distToHole=1.30 surface=green
```

- パワー0.375で12ユニット先のホールに1.30まで近づいた → パワーバランス適切
- 方向ずれ-4.7度でx=-0.92 → 許容範囲
- グリーン上で停止 → サーフェス摩擦が機能

**デバッグログは開発完了後に削除すること。**

---

## 22. 物理パラメータのバランス調整

ゲーム内の物理シミュレーション（ボール転がり、弾速、移動速度など）のバランスは、事前に数式で確認してからコードに入れると手戻りが少ない。

### 摩擦による減速モデル

毎フレーム速度に摩擦係数をかける場合（`speed *= friction`）、理論上の最大移動距離は等比級数の和で求まる:

```
最大移動距離 = 初速 / (1 - 摩擦係数)
```

例: パッティングゲームでの計算
| 摩擦係数 | 初速0.5の移動距離 | 用途 |
|---------|------------------|------|
| 0.988 (グリーン) | 0.5 / 0.012 = 41.7 | よく転がる |
| 0.982 (フェアウェイ) | 0.5 / 0.018 = 27.8 | 適度に減速 |
| 0.960 (ラフ) | 0.5 / 0.040 = 12.5 | 大きく減速 |

この公式で「ゲージ50%でホールに届くか」「100%で大幅オーバーするか」を事前チェックできる。

### バランス設計の手順

1. **目標距離を決める**: 最短ホール12ユニット、最長25ユニット
2. **適正パワーを逆算**: ゲージ50%で最短ホールに届く → `0.5 * MAX_POWER / (1 - friction) = 12`
3. **MAX_POWERを決定**: `MAX_POWER = 12 * (1 - friction) / 0.5`
4. **100%パワーで検算**: 最長ホールに届くか確認

### 角度パラメータの目安

方向ずれの倍率は小さめにする。パッティングで最大22.9度のずれは大きすぎる（ボールがコースアウトする）。目安:
- パッティング: 最大8〜10度（倍率 0.15程度）
- ドライバーショット: 最大15〜20度

---

## 23. ゲームの公開

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

## 24. ランキングゲームの基本テンプレート (TypeScript)

**⚠️ このテンプレートはすべて `var` と `function()` で書かれている。`let`/`const`/`=>` は使わないこと。**

```typescript
import { GameMainParameterObject } from "./parameterObject";

export function main(param: GameMainParameterObject): void {
  var scene = new g.Scene({
    game: g.game,
    assetIds: ["se"]  // 使用するアセットのみ列挙
  });

  var time = 60;
  if (param.sessionParameter.totalTimeLimit) {
    time = param.sessionParameter.totalTimeLimit;
  }
  g.game.vars.gameState = { score: 0 };

  scene.onLoad.add(function () {
    var seAudioAsset = scene.asset.getAudioById("se");

    var font = new g.DynamicFont({
      game: g.game,
      fontFamily: "sans-serif",
      size: 48
    });

    var scoreLabel = new g.Label({
      scene: scene, font: font, text: "SCORE: 0",
      fontSize: 28, textColor: "white",
      x: 20, y: 10
    });
    scene.append(scoreLabel);

    var timeLabel = new g.Label({
      scene: scene, font: font, text: "TIME: " + Math.ceil(time),
      fontSize: 28, textColor: "white",
      x: g.game.width - 200, y: 10
    });
    scene.append(timeLabel);

    var gameOver = false;

    // ゲームロジック
    scene.onPointDownCapture.add(function () {
      if (gameOver) return;
      g.game.vars.gameState.score += 10;
      scoreLabel.text = "SCORE: " + g.game.vars.gameState.score;
      scoreLabel.invalidate();
      seAudioAsset.play();
    });

    // メインループ
    scene.onUpdate.add(function () {
      if (gameOver) return;
      time -= 1 / g.game.fps;
      if (time <= 0) {
        time = 0;
        gameOver = true;
      }
      timeLabel.text = "TIME: " + Math.ceil(time);
      timeLabel.invalidate();
    });
  });

  g.game.pushScene(scene);
}
```

### ⚠️ main関数のエクスポート形式

テンプレートの `_bootstrap.ts` は `import { main } from "./main"` でメイン関数を読み込む。
そのため **必ず `export function main()` を使うこと**。`module.exports = main` は使わない:

```typescript
// ✅ 正しい: named export（テンプレートの _bootstrap.ts と互換）
export function main(param: GameMainParameterObject): void { ... }

// ❌ 間違い: module.exports（_bootstrap.tsのimportと非互換でTS2306エラー）
// module.exports = main;
```

`require()` で外部パッケージを読み込む場合は `export function main()` の中で使う:
```typescript
export function main(param: GameMainParameterObject): void {
  var b2 = require("@akashic-extension/akashic-box2d");
  // ... b2 を使ったコード
}
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
10. **TypeScript: `var` と `function()` だけを使う** — `let`/`const`/`=>`/`for...of`/`.every()`/`.find()` は使わない（`lib: ["es5"]`の制約。セクション3参照）
11. **`akashic init` は既存ファイルと競合する** → サブディレクトリで実行する
12. **`require()` でコンパイルエラーが出たら** → `typings/require.d.ts` を作成し tsconfig.json に追加
13. **Box2D使用時は `references/box2d-physics.md` を必ず参照** — 致命的な落とし穴が複数ある
14. **キーボードプラグインは `game.json` に登録しない** — コードで `register()` + `onStateChange` で `start()` すること
15. **大量エンティティは逆順ループで削除** — `for (i = arr.length - 1; i >= 0; i--)` + `splice(i, 1)`、上限 `MAX` も設ける
16. **`module.exports` は使わない** — テンプレートの `_bootstrap.ts` は `import { main } from "./main"` でnamed importする。`module.exports = main` だとTS2306エラーになる。必ず `export function main()` を使う
17. **`require()` は `export function main()` の中で呼ぶ** — トップレベルで `var b2 = require(...)` と書くと、`export function main()` と共存できずモジュール解決エラーになる場合がある
18. **外部ライブラリが `Math.random()` を使う場合** → `sandbox.config.js` で `"useMathRandom": false` を設定する（セクション19参照）
19. **akashic-three使用時はES5制約は適用外** — `lib: ["ES2020", "DOM"]`、`"renderers": "webgl"`、`GPUTexture.d.ts` の設定が必要（セクション20参照）
20. **3Dゲームのカメラは即座追従しない** — lerp（線形補間）でゆっくり追従させ、操作待ちに戻ったら速く追いつかせる
21. **ゲームテストにはデバッグログを使う** — スクリーンショットの目視では物理演算の問題を正確に把握できない。`console.log` で数値データを出力し分析する（セクション21参照）
22. **物理パラメータは数式で事前検証する** — `最大移動距離 = 初速 / (1 - 摩擦係数)` でバランスを確認してからコードに入れる（セクション22参照）
23. **`akashic sandbox -B` でブラウザの自動起動を抑制** — Claude Previewで確認する場合、PCブラウザは不要。`launch.json` にも `-B` を設定する
24. **Fit to screenはCLIオプションがない** — 毎回手動で Display Options → Fit to screen をチェックする必要がある

---

## 演出・ゲームジュース

画像アセットなしでもリッチなゲーム体験を作るための演出パターンを `references/visual-effects.md` にまとめている。
以下の演出が必要な場合に参照すること:

- **パーティクルシステム**: 破壊・爆発・着弾の破片エフェクト
- **画面シェイク**: worldContainerを揺らし、UIレイヤーは揺らさない構成
- **衝撃波リング**: 拡大＋フェードアウトする矩形で疑似リング
- **軌跡エフェクト**: 飛行物体の残像
- **土煙**: 地面付近の着弾で扇状に広がるパーティクル
- **スコアポップアップ**: 浮き上がって消えるラベル
- **ヒビ・ダメージ色変化**: ブロックのHP視覚フィードバック

**ポイント**: 演出は重ねるほど効果的。発射時なら「放射状パーティクル + 衝撃波 + 画面シェイク + SE」を同時に発火させる。

---

## ゲームアーキテクチャパターン

弾幕STG、アクション、大量エンティティを扱うゲームの設計パターンを `references/game-patterns.md` にまとめている。
以下のパターンが必要な場合に参照すること:

- **レイヤー構成**: bgLayer → bulletLayer → entityLayer → shotLayer → fxLayer → uiLayer の描画順序
- **エンティティ管理**: `dead` フラグ + 逆順ループ `splice` + `MAX_BULLETS` 上限
- **弾幕パターン**: 全方位弾、自機狙い、渦巻き、ランダム散弾、扇形弾
- **当たり判定の分離**: 見た目サイズ（24px）と判定サイズ（4px）を別にして弾幕ゲーの爽快感を出す
- **コンボシステム**: 連続撃破でスコア倍率アップ
- **難易度スケーリング**: 時間経過で弾速・弾数・敵HP・出現間隔を変動
- **自動ショット**: プレイヤーが避けることに集中できる設計

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
