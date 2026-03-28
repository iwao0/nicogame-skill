# Box2D 物理演算ガイド (@akashic-extension/akashic-box2d)

Akashic Engineで2D物理演算を使うための完全ガイド。
アングリーバード風、ピンボール、物理パズルなど、剛体シミュレーションが必要なゲームで参照すること。

---

## 目次
1. [セットアップ](#1-セットアップ)
2. [初期化](#2-初期化)
3. [剛体の作成（⚠️ 最重要）](#3-剛体の作成)
4. [力とインパルス（⚠️ 最重要）](#4-力とインパルス)
5. [World Locking と削除キュー](#5-world-locking-と削除キュー)
6. [衝突検知とダメージシステム](#6-衝突検知とダメージシステム)
7. [よくあるエラーと対処法](#7-よくあるエラーと対処法)
8. [完全なパターン例](#8-完全なパターン例)

---

## 1. セットアップ

### インストール
```bash
akashic install @akashic-extension/akashic-box2d
```

### require.d.ts の追加

`require()` がコンパイルエラーになる場合、型定義ファイルが必要:

**`typings/require.d.ts` を作成:**
```typescript
declare function require(moduleName: string): any;
```

**`tsconfig.json` の `include` に追加:**
```json
{
  "include": [
    "src/**/*.ts",
    "typings/console.d.ts",
    "typings/require.d.ts"
  ]
}
```

テンプレートによっては `typings/require.d.ts` が最初から存在しないため、なければ作成すること。

---

## 2. 初期化

**⚠️ `require()` は必ず `export function main()` の中で呼ぶこと。**
トップレベルに書くと `module.exports` と混同され、`_bootstrap.ts` の `import { main }` と非互換になる。

```typescript
import { GameMainParameterObject } from "./parameterObject";

export function main(param: GameMainParameterObject): void {
  // ✅ require は main() の中で呼ぶ
  var b2Mod = require("@akashic-extension/akashic-box2d");

  var scene = new g.Scene({ game: g.game });
  scene.onLoad.add(function () {
    // Box2Dワールド生成
    var box2d = new b2Mod.Box2D({
      gravity: [0, 9.8],  // 下方向の重力
      scale: 50,           // ピクセル→メートル変換係数（SCALE）
      sleep: true          // 静止したボディをスリープさせる（パフォーマンス向上）
    });

    // Box2DWeb の内部モジュールへのアクセス（b2BodyDef等に必要）
    var b2Web = b2Mod.Box2DWeb;

    // ... ゲームロジック ...
  });
  g.game.pushScene(scene);
}
```

`scale` は「何ピクセルが1メートルか」を表す。SCALE=50 なら 50px = 1m。

---

## 3. 剛体の作成

### ⚠️ 最大の落とし穴: `new` キーワード

`b2BodyDef` と `b2FixtureDef` は **必ず `new` で生成する**。`new` なしで呼ぶと `Cannot set properties of undefined` エラーになる。

```typescript
// ❌ これはエラーになる！
var bd = b2Web.Dynamics.b2BodyDef();

// ✅ 必ず new を使う
var bd = new b2Web.Dynamics.b2BodyDef();
var fd = new b2Web.Dynamics.b2FixtureDef();
```

### ボディタイプ
| 値 | 定数名 | 説明 |
|----|--------|------|
| 0 | Static | 動かない（地面、壁） |
| 1 | Kinematic | コードで動かす（プラットフォーム） |
| 2 | Dynamic | 物理で動く（キャラ、ブロック） |

### ヘルパー関数パターン

毎回 `new b2BodyDef()` を書くのは冗長なので、ヘルパーで包むのが実践的:

```typescript
function mkBodyDef(type: number, userData: string, bullet?: boolean): any {
  var bd = new b2Web.Dynamics.b2BodyDef();
  bd.type = type;
  bd.userData = userData;  // 衝突検知で識別に使う文字列
  if (bullet) bd.bullet = true;  // 高速物体の貫通防止
  bd.angularDamping = 0.3;  // 回転減衰
  bd.linearDamping = 0.1;   // 移動減衰
  return bd;
}

function mkFixDef(shape: any, density: number, friction: number, restitution: number): any {
  var fd = new b2Web.Dynamics.b2FixtureDef();
  fd.shape = shape;
  fd.density = density;      // 密度（重さ）
  fd.friction = friction;     // 摩擦
  fd.restitution = restitution;  // 反発係数（0=吸収、1=完全反射）
  return fd;
}
```

### ボディの生成

```typescript
// 矩形ボディ
var entity = new g.FilledRect({
  scene: scene, cssColor: "#DEB887",
  width: 30, height: 80,
  x: 500, y: 400,
  anchorX: 0.5, anchorY: 0.5  // Box2Dは中心基準なのでanchorを0.5にする
});
gameLayer.append(entity);

var ebody = box2d.createBody(
  entity,
  mkBodyDef(2, "block_0"),  // Dynamic, userData="block_0"
  mkFixDef(box2d.createRectShape(30, 80), 2.5, 0.5, 0.15)
);

// 円形ボディ
var circleEntity = new g.FilledRect({
  scene: scene, cssColor: "#FF4444",
  width: 40, height: 40,
  x: 200, y: 300,
  anchorX: 0.5, anchorY: 0.5
});
gameLayer.append(circleEntity);

var circleBody = box2d.createBody(
  circleEntity,
  mkBodyDef(2, "ball", true),  // bullet=trueで高速時の貫通を防ぐ
  mkFixDef(box2d.createCircleShape(40), 1.5, 0.3, 0.35)
  // createCircleShapeの引数は直径（ピクセル）
);
```

### 静的ボディ（地面・壁）

```typescript
// 地面
var groundE = new g.FilledRect({
  scene: scene, cssColor: "transparent",
  width: GW, height: 40,
  x: GW / 2, y: GROUND_Y + 20
});
gameLayer.append(groundE);
box2d.createBody(
  groundE,
  mkBodyDef(0, "ground"),  // Static
  mkFixDef(box2d.createRectShape(GW, 40), 0, 0.6, 0.2)
);
```

---

## 4. 力とインパルス

### ⚠️ `box2d.vec2()` vs `new b2Vec2()` — 単位変換の罠

これは見落としやすいが非常に重要なポイント:

- **`box2d.vec2(x, y)`** はピクセル値を受け取り、**内部でSCALEで割って**メートル単位に変換する。位置の設定には適切。
- **`ApplyImpulse()` / `ApplyForce()`** はメートル単位の値をそのまま受け取る。`box2d.vec2()` を使うと値が1/SCALE（= 1/50）になってしまい、ほぼ動かない。

```typescript
// ❌ これだとインパルスが1/50になり、鳥がほぼ動かない！
var imp = box2d.vec2(-dragDx * LAUNCH_POWER, -dragDy * LAUNCH_POWER);
body.b2Body.ApplyImpulse(imp, body.b2Body.GetWorldCenter());

// ✅ 直接 b2Vec2 を使う（SCALE除算なし）
var b2Vec2 = b2Web.Common.Math.b2Vec2;
var imp = new b2Vec2(-dragDx * LAUNCH_POWER, -dragDy * LAUNCH_POWER);
body.b2Body.ApplyImpulse(imp, body.b2Body.GetWorldCenter());
```

**覚え方**: 位置 → `box2d.vec2()`、力/インパルス → `new b2Vec2()`

---

## 5. World Locking と削除キュー

Box2Dは `step()` の実行中（衝突コールバック含む）にボディを追加・削除できない。
`step()` 中に `removeBody()` を呼ぶとクラッシュする。

**解決策: 削除キューパターン**

```typescript
var removeQueue: any[] = [];

// メインループ
scene.onUpdate.add(function() {
  // 1. 先にキューを処理してからstepを呼ぶ
  for (var r = 0; r < removeQueue.length; r++) {
    var eb = removeQueue[r];
    if (eb && eb.entity && !eb.entity.destroyed()) {
      eb.entity.destroy();
    }
    try { box2d.removeBody(eb); } catch (_e) { /* 安全に無視 */ }
  }
  removeQueue = [];

  // 2. 物理ステップ
  box2d.step(1 / g.game.fps);

  // 3. 衝突処理（ここでremoveQueueに追加する）
  processCollisions();
});

// ブロック破壊時
function killBlock(block: any): void {
  block.dead = true;
  removeQueue.push(block.ebody);  // すぐ消さない、キューに入れる
}
```

---

## 6. 衝突検知とダメージシステム

### GetContactList による衝突走査

```typescript
var hitCooldown: { [key: string]: number } = {};

function processCollisions(): void {
  var contact = box2d.world.GetContactList();
  while (contact) {
    if (contact.IsTouching()) {
      var bA = contact.GetFixtureA().GetBody();
      var bB = contact.GetFixtureB().GetBody();
      var udA: string = bA.GetUserData() || "";
      var udB: string = bB.GetUserData() || "";

      // 相対速度でダメージ判定
      var vA = bA.GetLinearVelocity();
      var vB = bB.GetLinearVelocity();
      var dvx = vA.x - vB.x, dvy = vA.y - vB.y;
      var relVel = Math.sqrt(dvx * dvx + dvy * dvy);

      // クールダウン（同じペアの連続衝突を防ぐ）
      var pairKey = udA < udB ? udA + "|" + udB : udB + "|" + udA;
      if (hitCooldown[pairKey] && hitCooldown[pairKey] > 0) {
        contact = contact.GetNext();
        continue;
      }

      // 閾値を超えたらダメージ処理
      if (relVel > 3.0) {
        var damage = relVel > 8 ? 3 : (relVel > 5 ? 2 : 1);
        // ... ダメージ適用 ...
        hitCooldown[pairKey] = 8;  // 8フレームのクールダウン
      }
    }
    contact = contact.GetNext();
  }
}
```

### クールダウンの更新（メインループ内）

```typescript
// 毎フレーム減算
for (var key in hitCooldown) {
  if (hitCooldown.hasOwnProperty(key)) {
    hitCooldown[key]--;
    if (hitCooldown[key] <= 0) delete hitCooldown[key];
  }
}
```

### なぜクールダウンが必要か

Box2Dでは衝突がフレームをまたいで連続検出される。クールダウンなしだと、ブロックに触れた瞬間に何十回もダメージが入り、硬いはずのブロックが一瞬で壊れてしまう。これでは「壊す気持ちよさ」が損なわれる。

### ダメージカラー変化

HPの残量に応じて見た目を変えると、プレイヤーに「もう少しで壊れる」というフィードバックが伝わる:

```typescript
var ratio = block.hp / block.maxHp;
var entity = block.ebody.entity as g.FilledRect;
if (ratio < 0.3) {
  entity.cssColor = "#A08050";  // 大ダメージ色
} else if (ratio < 0.7) {
  entity.cssColor = "#C4A470";  // 中ダメージ色
}
entity.modified();
```

### ヒビ演出

```typescript
function showCrack(entity: g.E, w: number, h: number): void {
  var cx = Math.floor(g.game.random.generate() * (w - 6)) + 3;
  entity.append(new g.FilledRect({
    scene: scene, cssColor: "rgba(0,0,0,0.3)",
    width: 2,
    height: Math.floor(h * 0.4 + g.game.random.generate() * h * 0.3),
    x: cx,
    y: Math.floor(g.game.random.generate() * h * 0.3)
  }));
}
```

---

## 7. よくあるエラーと対処法

| エラー | 原因 | 対処 |
|--------|------|------|
| `Cannot set properties of undefined (setting 'type')` | `b2BodyDef()` を `new` なしで呼んだ | `new b2Web.Dynamics.b2BodyDef()` にする |
| 物体がほぼ動かない（インパルスが効かない） | `box2d.vec2()` で値が1/SCALE | `new b2Vec2()` を使う |
| `step()` 中にクラッシュ | 衝突コールバック内で `removeBody()` | 削除キューパターンを使う |
| ブロックが一瞬で消える | 衝突が連続検出されてダメージ過多 | クールダウンシステムを入れる |
| `require is not defined` (TS2591) | `typings/require.d.ts` がない | ファイル作成 + tsconfig.json に追加 |
| ボディが静止しない | `sleep: false` / damping不足 | `sleep: true` + `angularDamping`, `linearDamping` 設定 |

---

## 8. 完全なパターン例

### スリングショット（引っ張り発射）

```typescript
var SLING_X = 200;
var SLING_Y = GROUND_Y - 80;
var MAX_DRAG = 120;
var LAUNCH_POWER = 0.25;
var dragging = false;
var dragDx = 0, dragDy = 0;

scene.onPointDownCapture.add(function(ev) {
  if (launched) return;
  var dx = ev.point.x - SLING_X;
  var dy = ev.point.y - SLING_Y;
  if (Math.sqrt(dx * dx + dy * dy) < 70) {
    dragging = true;
    dragDx = 0;
    dragDy = 0;
  }
});

scene.onPointMoveCapture.add(function(ev) {
  if (!dragging || !birdEntity) return;
  dragDx += ev.prevDelta.x;
  dragDy += ev.prevDelta.y;
  // ドラッグ距離を制限
  var dist = Math.sqrt(dragDx * dragDx + dragDy * dragDy);
  if (dist > MAX_DRAG) {
    var r = MAX_DRAG / dist;
    dragDx *= r;
    dragDy *= r;
  }
  birdEntity.x = SLING_X + dragDx;
  birdEntity.y = SLING_Y + dragDy;
  birdEntity.modified();
});

scene.onPointUpCapture.add(function() {
  if (!dragging) return;
  dragging = false;
  var dist = Math.sqrt(dragDx * dragDx + dragDy * dragDy);
  if (dist < 15) return;  // 最小距離未満はキャンセル

  // ボディ生成
  var bd = mkBodyDef(2, "bird", true);
  var fd = mkFixDef(box2d.createCircleShape(BIRD_R * 2), 1.5, 0.3, 0.35);
  var birdBody = box2d.createBody(birdEntity, bd, fd);

  // ⚠️ 直接 b2Vec2 でインパルス（box2d.vec2は使わない！）
  var b2Vec2 = b2Web.Common.Math.b2Vec2;
  var imp = new b2Vec2(-dragDx * LAUNCH_POWER, -dragDy * LAUNCH_POWER);
  birdBody.b2Body.ApplyImpulse(imp, birdBody.b2Body.GetWorldCenter());

  launched = true;
});
```

### 物理パラメータの目安

| パラメータ | 推奨範囲 | 補足 |
|-----------|---------|------|
| `gravity` | 9.8 ~ 15.0 | 9.8がリアル、高いとゲーム的 |
| `scale` | 50 | 一般的な値。変える理由は少ない |
| `density` (動的) | 1.0 ~ 5.0 | 大きいほど重い |
| `friction` | 0.2 ~ 0.8 | 0=ツルツル、1=ザラザラ |
| `restitution` | 0.1 ~ 0.5 | 0=跳ねない、1=完全バウンド |
| `LAUNCH_POWER` | 0.15 ~ 0.35 | 画面サイズとSCALEに応じて調整 |
| HP (ブロック) | 3 ~ 7 | 少なすぎると一撃で壊れて気持ちよくない |
| 衝突閾値 (鳥) | 2.0 ~ 4.0 | 低すぎると触れただけで壊れる |
| 衝突閾値 (ブロック同士) | 4.0 ~ 6.0 | 鳥より高く設定 |
