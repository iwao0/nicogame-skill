# ゲームアーキテクチャパターン集

Akashic Engineでアクション・シューティング・弾幕など、大量のエンティティを扱うゲームを作るときの設計パターン。

---

## 目次
1. [レイヤー構成](#1-レイヤー構成)
2. [エンティティ管理パターン（dead フラグ + 逆順ループ）](#2-エンティティ管理パターン)
3. [弾幕パターン（STG向け）](#3-弾幕パターンstg向け)
4. [当たり判定の分離（見た目と判定を別にする）](#4-当たり判定の分離)
5. [コンボ・スコアシステム](#5-コンボスコアシステム)
6. [難易度スケーリング](#6-難易度スケーリング)
7. [自動ショットパターン](#7-自動ショットパターン)

---

## 1. レイヤー構成

大量のエンティティが重なるゲームでは、レイヤーの描画順序が重要。Akashic Engineでは `scene.append()` の順序で前後が決まる（後に追加したものが手前）。

```typescript
// 推奨レイヤー構成（奥から手前の順に追加）
var bgLayer = new g.E({ scene: scene, width: GW, height: GH });
scene.append(bgLayer);          // 背景・星

var bulletLayer = new g.E({ scene: scene, width: GW, height: GH });
scene.append(bulletLayer);       // 敵弾（大量なので自機より奥に）

var entityLayer = new g.E({ scene: scene, width: GW, height: GH });
scene.append(entityLayer);       // 敵・自機

var shotLayer = new g.E({ scene: scene, width: GW, height: GH });
scene.append(shotLayer);         // 自弾（自機より手前で見やすく）

var fxLayer = new g.E({ scene: scene, width: GW, height: GH });
scene.append(fxLayer);           // パーティクル・爆発

var uiLayer = new g.E({ scene: scene, width: GW, height: GH });
scene.append(uiLayer);           // スコア・UI（常に最前面）
```

レイヤー分けの理由:
- 敵弾が自機より奥にあることで、自機の位置が弾に隠れず視認しやすい
- エフェクトがゲームオブジェクトより手前で演出が映える
- UIが常に最前面で情報が読める

画面シェイクを入れる場合は `references/visual-effects.md` の `worldContainer` パターンを参照。

---

## 2. エンティティ管理パターン

大量のオブジェクト（弾、敵、パーティクル）を毎フレーム管理するとき、`dead` フラグと逆順ループの組み合わせが定番パターン。

### なぜこのパターンが必要か

配列の要素を前方からループしながら `splice` すると、インデックスがずれて要素がスキップされる。逆順にループすれば、削除した要素より前の要素にはインデックスの影響がない。

### 基本構造

```typescript
var bullets: {
  e: g.FilledRect;
  x: number; y: number;
  vx: number; vy: number;
  dead: boolean;
}[] = [];
var bulletCount = 0;
var MAX_BULLETS = 600;  // 上限を設けてパフォーマンスを守る

function fireBullet(x: number, y: number, vx: number, vy: number): void {
  if (bulletCount >= MAX_BULLETS) return;  // 上限チェック
  var e = new g.FilledRect({
    scene: scene, cssColor: "#FF6688",
    width: 12, height: 12,
    x: x, y: y,
    anchorX: 0.5, anchorY: 0.5
  });
  bulletLayer.append(e);
  bullets.push({ e: e, x: x, y: y, vx: vx, vy: vy, dead: false });
  bulletCount++;
}

// メインループ内
for (var i = bullets.length - 1; i >= 0; i--) {
  var b = bullets[i];
  if (b.dead) { bullets.splice(i, 1); continue; }

  b.x += b.vx;
  b.y += b.vy;
  b.e.x = b.x;
  b.e.y = b.y;
  b.e.modified();

  // 画面外チェック
  if (b.x < -20 || b.x > GW + 20 || b.y < -20 || b.y > GH + 20) {
    b.dead = true;
    if (!b.e.destroyed()) b.e.destroy();
    bulletCount--;
  }
}
```

### MAX制限が重要な理由

弾幕ゲームでは敵が大量の弾を発射する。上限なしだと数千のエンティティが生成され、描画とGCの負荷で劇的にフレームレートが落ちる。`MAX_BULLETS = 600` 程度を目安に、上限に達したら新しい弾の生成をスキップする。

---

## 3. 弾幕パターン（STG向け）

弾幕ゲームの面白さは多彩な弾パターンにある。以下は基本5パターン。

### 全方位弾（円形）

均等に放射状に広がる。`nWay` を増やすと密度が上がる。`age` でフレームごとに回転させると渦巻きに。

```typescript
function fireCircle(x: number, y: number, nWay: number, speed: number, offset: number): void {
  for (var i = 0; i < nWay; i++) {
    var angle = (i / nWay) * Math.PI * 2 + offset;
    fireBullet(x, y, Math.cos(angle) * speed, Math.sin(angle) * speed);
  }
}
// 回転する全方位弾: offset に enemy.age * 0.1 などを渡す
```

### 自機狙い弾（n-way）

プレイヤーに向かう方向を計算し、その周囲に扇状に展開。

```typescript
function fireAimed(ex: number, ey: number, px: number, py: number, nWay: number, spread: number, speed: number): void {
  var dx = px - ex;
  var dy = py - ey;
  var baseAngle = Math.atan2(dy, dx);
  for (var i = 0; i < nWay; i++) {
    var a = baseAngle - spread / 2 + (i / (nWay - 1)) * spread;
    fireBullet(ex, ey, Math.cos(a) * speed, Math.sin(a) * speed);
  }
}
// 3way狙い弾: fireAimed(ex, ey, playerX, playerY, 3, 0.3, 3.0)
```

### 渦巻き弾（スパイラル）

複数の腕が回転しながら弾を発射。`nArms` で腕の本数、回転速度で密度を制御。

```typescript
function fireSpiral(x: number, y: number, nArms: number, age: number, speed: number): void {
  for (var arm = 0; arm < nArms; arm++) {
    var angle = (arm / nArms) * Math.PI * 2 + age * 0.15;
    fireBullet(x, y, Math.cos(angle) * speed, Math.sin(angle) * speed);
  }
}
```

### ランダム散弾

予測不能な弾。密度を高くすると全方位弾に近づくが、ランダム性が心理的圧迫になる。

```typescript
function fireRandom(x: number, y: number, count: number, speed: number): void {
  for (var i = 0; i < count; i++) {
    var angle = rng.generate() * Math.PI * 2;
    var spd = speed * 0.6 + rng.generate() * speed * 0.8;
    fireBullet(x, y, Math.cos(angle) * spd, Math.sin(angle) * spd);
  }
}
```

### 扇形弾

下方向など特定の方向に扇状に展開。`centerAngle` で方向、`spreadAngle` で扇の幅を制御。

```typescript
function fireFan(x: number, y: number, nWay: number, centerAngle: number, spreadAngle: number, speed: number): void {
  for (var i = 0; i < nWay; i++) {
    var a = centerAngle - spreadAngle / 2 + (i / (nWay - 1)) * spreadAngle;
    fireBullet(x, y, Math.cos(a) * speed, Math.sin(a) * speed);
  }
}
// 下方向に扇形: fireFan(x, y, 7, Math.PI / 2, 0.8, 2.5)
```

### パターンの組み合わせ

敵ごとに `pattern` プロパティを持たせ、`switch` で使い分ける:

```typescript
enemy.bulletTimer++;
if (enemy.bulletTimer >= enemy.bulletInterval) {
  enemy.bulletTimer = 0;
  switch (enemy.pattern) {
    case 0: fireCircle(ex, ey, 12, 2.5, enemy.age * 0.1); break;
    case 1: fireAimed(ex, ey, playerX, playerY, 3, 0.3, 3.0); break;
    case 2: fireSpiral(ex, ey, 3, enemy.age, 2.0); break;
    case 3: fireRandom(ex, ey, 5, 2.5); break;
    case 4: fireFan(ex, ey, 7, Math.PI / 2, 0.8, 2.5); break;
  }
}
```

---

## 4. 当たり判定の分離

弾幕ゲームでは「見た目は大きいが当たり判定は小さい」のが常識。プレイヤーが弾を「避けた感覚」を得るために、判定を見た目より大幅に小さくする。

```typescript
var PLAYER_SIZE = 24;    // 見た目のサイズ
var PLAYER_CORE = 4;     // 当たり判定の半径（とても小さい！）

// 見た目のエンティティ
var playerE = new g.FilledRect({
  scene: scene, cssColor: "#00FFFF",
  width: PLAYER_SIZE, height: PLAYER_SIZE,
  x: playerX, y: playerY,
  anchorX: 0.5, anchorY: 0.5, angle: 45
});

// 当たり判定の可視化（白い小さな点）
var coreE = new g.FilledRect({
  scene: scene, cssColor: "#FFFFFF",
  width: PLAYER_CORE * 2, height: PLAYER_CORE * 2,
  x: playerX, y: playerY,
  anchorX: 0.5, anchorY: 0.5
});

// 衝突判定（円 vs 円）
var dx = bullet.x - playerX;
var dy = bullet.y - playerY;
var dist = Math.sqrt(dx * dx + dy * dy);
if (dist < PLAYER_CORE + BULLET_RADIUS) {
  // 被弾！
}
```

`PLAYER_CORE = 4` は画面上でほぼ点。これにより弾の隙間をギリギリ抜ける爽快感が生まれる。

---

## 5. コンボ・スコアシステム

連続撃破でスコアが跳ね上がる仕組み。プレイヤーに「もっと攻めたい」と思わせる動機付け。

```typescript
var combo = 0;
var maxCombo = 0;

function killEnemy(enemy: any): void {
  combo++;
  if (combo > maxCombo) maxCombo = combo;

  // コンボが高いほどスコア倍率が上がる
  var basePoints = 10 + enemy.maxHp * 5;
  var multiplier = Math.max(1, Math.floor(combo / 3));
  var points = basePoints * multiplier;
  g.game.vars.gameState.score += points;

  spawnScorePopup(enemy.x, enemy.y, points);
}

// コンボリセット条件（例: 敵が画面外に逃げた）
function onEnemyEscaped(): void {
  combo = 0;
}
```

コンボ表示はUIレイヤーに:
```typescript
if (combo >= 3) {
  comboLabel.text = combo + " COMBO!";
} else {
  comboLabel.text = "";
}
comboLabel.invalidate();
```

---

## 6. 難易度スケーリング

時間経過で難易度を上げると、序盤は遊びやすく終盤は手応えが出る。

```typescript
var difficulty = 1.0;

// メインループで更新（10秒ごとに+1）
difficulty = 1.0 + frameCount / (g.game.fps * 10);

// 難易度を各パラメータに反映
var bulletSpeed = 2.0 + difficulty * 0.3;           // 弾速
var bulletCount = 8 + Math.floor(difficulty * 2);    // n-way弾の本数
var spawnInterval = Math.max(20, 60 / difficulty);   // 敵出現間隔
var enemyHP = 3 + Math.floor(difficulty);             // 敵のHP
var bulletInterval = Math.max(8, 40 - Math.floor(difficulty * 3)); // 発射間隔
```

難易度をレベルとして表示するとプレイヤーのモチベーションになる:
```typescript
var level = Math.floor(difficulty);
diffLabel.text = "LV." + level;
diffLabel.invalidate();
```

---

## 7. 自動ショットパターン

「避けることに集中」させるため、自弾は自動発射にするケースが多い。

```typescript
var SHOT_INTERVAL = 4;  // 4フレームに1回 = 30fpsで秒間7.5発
var shotTimer = 0;

// メインループ内
shotTimer++;
if (shotTimer >= SHOT_INTERVAL) {
  shotTimer = 0;
  // メインショット（正面）
  createShot(playerX, playerY - 12, 0, -14);
  // サイドショット（少し広がる、威力低め）
  createShot(playerX - 12, playerY - 8, -0.5, -13);
  createShot(playerX + 12, playerY - 8, 0.5, -13);
}
```

サイドショットを少し外側に広げることで、正面以外の敵にも自然に当たるようになる。
