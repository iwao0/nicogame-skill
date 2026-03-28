# 演出パターン集 (画像なし、g.FilledRect のみ)

Akashic Engineでは画像アセットなしでも、`g.FilledRect` の組み合わせでリッチな演出が作れる。
このリファレンスでは、プロトタイプでも「気持ちいい」と感じるレベルのゲームジュースを実装するパターンを解説する。

---

## 目次
1. [レイヤー構成](#1-レイヤー構成)
2. [パーティクルシステム](#2-パーティクルシステム)
3. [画面シェイク](#3-画面シェイク)
4. [衝撃波リング](#4-衝撃波リング)
5. [軌跡エフェクト](#5-軌跡エフェクト)
6. [土煙・着弾エフェクト](#6-土煙着弾エフェクト)
7. [スコアポップアップ](#7-スコアポップアップ)
8. [ヒビ・ダメージ表現](#8-ヒビダメージ表現)
9. [組み合わせのコツ](#9-組み合わせのコツ)

---

## 1. レイヤー構成

演出を適切に重ねるには、レイヤー分けが重要。特に画面シェイクを実装する場合、UIが一緒に揺れると見づらくなる。

```typescript
// ゲーム全体を包むコンテナ（シェイク対象）
var worldContainer = new g.E({ scene: scene, width: GW, height: GH });
scene.append(worldContainer);

// 背景・ゲームオブジェクト（worldContainerの中）
var gameLayer = new g.E({ scene: scene, width: GW, height: GH });
worldContainer.append(gameLayer);

// エフェクト層（worldContainerの中 = シェイクする）
var fxLayer = new g.E({ scene: scene, width: GW, height: GH });
worldContainer.append(fxLayer);

// UI層（sceneに直接追加 = シェイクしない）
var uiLayer = new g.E({ scene: scene, width: GW, height: GH });
scene.append(uiLayer);
```

背景はworldContainerより少し大きめに作る（+40pxなど）と、シェイク時に端が見切れない。

---

## 2. パーティクルシステム

破壊・爆発・着弾などあらゆるエフェクトの基盤。小さな `g.FilledRect` を大量に生成し、速度・重力・フェードで動かす。

### データ構造

```typescript
var particles: {
  e: g.FilledRect;
  vx: number; vy: number;
  life: number; maxLife: number;
  spin: number;
}[] = [];
```

### 生成関数

```typescript
function spawnParticles(x: number, y: number, color: string, count: number, spreadW: number, spreadH: number): void {
  for (var i = 0; i < count; i++) {
    var size = 3 + Math.floor(g.game.random.generate() * 14);
    var angle = g.game.random.generate() * Math.PI * 2;
    var speed = 6 + g.game.random.generate() * 18;
    var px = x + (g.game.random.generate() - 0.5) * spreadW;
    var py = y + (g.game.random.generate() - 0.5) * spreadH;
    var life = 30 + Math.floor(g.game.random.generate() * 35);
    var p = new g.FilledRect({
      scene: scene, cssColor: color,
      width: size, height: size,
      x: px, y: py,
      anchorX: 0.5, anchorY: 0.5,
      angle: g.game.random.generate() * 360
    });
    fxLayer.append(p);
    particles.push({
      e: p,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed - 8,  // 上方向バイアス
      life: life, maxLife: life,
      spin: (g.game.random.generate() - 0.5) * 20
    });
  }
}
```

### 更新（メインループ内）

```typescript
for (var pi = particles.length - 1; pi >= 0; pi--) {
  var p = particles[pi];
  p.life--;
  p.vy += 0.5;      // 重力
  p.vx *= 0.98;      // 空気抵抗
  p.e.x += p.vx;
  p.e.y += p.vy;
  p.e.opacity = Math.max(0, p.life / p.maxLife);
  p.e.angle += p.spin;
  p.e.modified();
  if (p.life <= 0) {
    if (!p.e.destroyed()) p.e.destroy();
    particles.splice(pi, 1);
  }
}
```

**逆順ループ（`pi--`）** で回すのは、`splice` で要素を削除してもインデックスがずれないため。

### 破壊エフェクトの参考値
| シーン | count | speed | life | 重力 |
|--------|-------|-------|------|------|
| 小物破壊 | 10-15 | 4-12 | 20-40 | 0.3 |
| 大物破壊 | 25-35 | 6-18 | 30-65 | 0.5 |
| 敵破壊 | 35+ | 8-20 | 30-50 | 0.5 |
| 白フラッシュ（追加） | 5 | 2-6 | 10-20 | 0.2 |

---

## 3. 画面シェイク

衝撃を体感させる最もコスパの良い演出。

```typescript
var shakeTime = 0;
var shakeIntensity = 0;

function startShake(intensity: number, duration: number): void {
  shakeIntensity = intensity;
  shakeTime = duration;
}

// メインループ内
if (shakeTime > 0) {
  shakeTime--;
  var sx = (g.game.random.generate() - 0.5) * shakeIntensity * 2;
  var sy = (g.game.random.generate() - 0.5) * shakeIntensity * 2;
  worldContainer.x = sx;
  worldContainer.y = sy;
  worldContainer.modified();
  shakeIntensity *= 0.85;  // 減衰
} else {
  if (worldContainer.x !== 0 || worldContainer.y !== 0) {
    worldContainer.x = 0;
    worldContainer.y = 0;
    worldContainer.modified();
  }
}
```

### シェイク強度の目安
| イベント | intensity | duration |
|----------|-----------|----------|
| 発射 | 5-7 | 6-10 |
| 着弾 | 3-6 | 5-8 |
| 大破壊 | 4-5 | 6 |
| 倒壊 | 2-3 | 4 |

---

## 4. 衝撃波リング

発射・着弾時に広がる白い輪。`g.FilledRect` を正方形にして拡大アニメーションで疑似的にリングを表現する。

```typescript
function spawnShockwave(x: number, y: number, maxRadius: number, color: string): void {
  var ring = new g.FilledRect({
    scene: scene, cssColor: color,
    width: 10, height: 10,
    x: x, y: y,
    anchorX: 0.5, anchorY: 0.5,
    opacity: 0.8
  });
  fxLayer.append(ring);
  var age = 0;
  var dur = 15;
  ring.onUpdate.add(function() {
    age++;
    var progress = age / dur;
    var r = maxRadius * progress;
    ring.width = r * 2;
    ring.height = r * 2;
    ring.opacity = 0.8 * (1 - progress);
    ring.modified();
    if (age >= dur) ring.destroy();
  });
}
```

- 発射: `spawnShockwave(x, y, 80, "#FFAA00")`
- 着弾: `spawnShockwave(x, y, 60, "#FFFFFF")`
- 敵破壊: `spawnShockwave(x, y, 50, "#66FF66")`

---

## 5. 軌跡エフェクト

飛行する物体の後ろに残像を残すことで、スピード感とダイナミクスを演出。

```typescript
var trailInterval = scene.setInterval(function() {
  if (!body || !body.entity || body.entity.destroyed()) {
    scene.clearInterval(trailInterval);
    return;
  }
  var te = body.entity;
  var trail = new g.FilledRect({
    scene: scene, cssColor: "#FF8866",
    width: 8, height: 8,
    x: te.x, y: te.y,
    anchorX: 0.5, anchorY: 0.5,
    opacity: 0.6
  });
  fxLayer.append(trail);
  var tLife = 12;
  trail.onUpdate.add(function() {
    tLife--;
    trail.opacity = tLife / 12 * 0.6;
    trail.scaleX = tLife / 12;
    trail.scaleY = tLife / 12;
    trail.modified();
    if (tLife <= 0) trail.destroy();
  });
}, 50);  // 50ms間隔で軌跡を生成
```

---

## 6. 土煙・着弾エフェクト

地面付近への着弾時に上方向へ扇状に広がるパーティクル。色を地面に合わせる。

```typescript
function spawnDustCloud(x: number, y: number): void {
  var colors = ["#A0866A", "#8B7355", "#C4A882", "#D2B48C"];
  for (var i = 0; i < 18; i++) {
    var size = 8 + Math.floor(g.game.random.generate() * 20);
    var angle = -Math.PI * 0.2 - g.game.random.generate() * Math.PI * 0.6;  // 上方向に扇状
    var speed = 3 + g.game.random.generate() * 10;
    var life = 25 + Math.floor(g.game.random.generate() * 25);
    var col = colors[Math.floor(g.game.random.generate() * colors.length)];
    var p = new g.FilledRect({
      scene: scene, cssColor: col,
      width: size, height: size,
      x: x + (g.game.random.generate() - 0.5) * 30, y: y,
      anchorX: 0.5, anchorY: 0.5,
      opacity: 0.7
    });
    fxLayer.append(p);
    particles.push({
      e: p,
      vx: Math.cos(angle) * speed + (g.game.random.generate() - 0.5) * 4,
      vy: Math.sin(angle) * speed,
      life: life, maxLife: life,
      spin: (g.game.random.generate() - 0.5) * 5
    });
  }
}
```

---

## 7. スコアポップアップ

得点が入った位置から浮き上がって消えるラベル。

```typescript
function spawnScorePopup(x: number, y: number, pts: number): void {
  var label = new g.Label({
    scene: scene, font: font,
    text: "+" + pts,
    fontSize: 32, textColor: "#FF6600",
    x: x - 25, y: y - 40
  });
  fxLayer.append(label);
  var life = 40;
  label.onUpdate.add(function() {
    life--;
    label.y -= 1.8;
    label.opacity = Math.min(1, life / 20);
    // 登場時にポップアップ感
    if (life > 30) {
      label.scaleX = 1 + (40 - life) * 0.03;
      label.scaleY = label.scaleX;
    }
    label.modified();
    if (life <= 0) label.destroy();
  });
}
```

---

## 8. ヒビ・ダメージ表現

ブロックのHP減少に応じて、色を暗くしつつヒビ線を追加。物理的な「もろさ」を視覚的に伝える。

```typescript
function showCrack(entity: g.E, w: number, h: number): void {
  // 縦ヒビ
  var cx = Math.floor(g.game.random.generate() * (w - 6)) + 3;
  entity.append(new g.FilledRect({
    scene: scene, cssColor: "rgba(0,0,0,0.3)",
    width: 2,
    height: Math.floor(h * 0.4 + g.game.random.generate() * h * 0.3),
    x: cx,
    y: Math.floor(g.game.random.generate() * h * 0.3)
  }));
  // 横ヒビ
  var cy = Math.floor(g.game.random.generate() * (h - 6)) + 3;
  entity.append(new g.FilledRect({
    scene: scene, cssColor: "rgba(0,0,0,0.25)",
    width: Math.floor(w * 0.3 + g.game.random.generate() * w * 0.3),
    height: 2,
    x: Math.floor(g.game.random.generate() * w * 0.3),
    y: cy
  }));
}
```

---

## 9. 組み合わせのコツ

一つの演出だけでは物足りない。**複数の演出を同時に重ねる**ことで、格段にインパクトが増す。

### 発射時
```typescript
spawnLaunchBurst(x, y);      // 放射状パーティクル
spawnShockwave(x, y, 80, "#FFAA00");  // 衝撃波
startShake(6, 8);              // 画面シェイク
seAsset.play();                // 効果音
// + 軌跡エフェクトを開始
```

### 着弾時
```typescript
spawnShockwave(x, y, 60, "#FFFFFF");  // 白い衝撃波
startShake(3 + velocity * 0.5, 6);    // 速度に応じたシェイク
if (nearGround) spawnDustCloud(x, groundY);  // 地面付近なら土煙
```

### 破壊時
```typescript
spawnParticles(x, y, blockColor, 25, w, h);  // 破片
spawnParticles(x, y, "#FFFFFF", 5, w * 0.5, h * 0.5);  // 白フラッシュ
spawnScorePopup(x, y, points);  // スコア表示
seAsset.play();
if (isEnemy) {
  spawnShockwave(x, y, 50, "#66FF66");  // ボス破壊時の追加衝撃波
  startShake(4, 6);
}
```

**ポイント**: エフェクトの量は最初は多めに作って、うるさければ減らす方が良い。足りない演出を後から足すよりも、多すぎる演出を削る方が調整しやすい。
