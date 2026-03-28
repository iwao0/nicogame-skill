# nicogame-skill

Akashic Engine を使ったニコ生ゲーム開発を支援する **Claude Code カスタムスキル** です。

## これは何？

Claude Code に「ニコ生ゲームを作って」と頼むだけで、Akashic Engine の環境構築からゲームのコーディングまで一気通貫でやってくれるようになるスキルです。

Akashic Engine 特有の制約（`Math.random()` 禁止、`modified()` 呼び出し、TypeScript の `lib: ["es5"]` 制約など）を Claude が理解した上でコードを書いてくれるので、ハマりポイントを回避できます。

## インストール

### 方法 1: リポジトリをクローンして使う

```bash
git clone https://github.com/iwao/nicogame-skill.git
cd nicogame-skill
```

このディレクトリ内で Claude Code を起動すれば、`.claude/skills/akashic-game.md` が自動的に読み込まれます。

### 方法 2: 既存プロジェクトにコピーする

```bash
# 既存プロジェクトのルートで
mkdir -p .claude/skills
cp path/to/nicogame-skill/.claude/skills/akashic-game.md .claude/skills/
```

## 使い方

Claude Code を起動して、ゲーム開発をお願いするだけです。

### 基本的な流れ

```
あなた: ブロック崩しゲームを作って
Claude: （環境構築 → コーディング → ビルド → 起動まで自動で実行）
```

### 使用例

```
# 環境構築から
ニコ生用のランキングゲームを新規作成して

# ゲームの種類を指定
シューティングゲームを作って

# 既存プロジェクトの改修
スコアの計算方法を変更して、コンボボーナスを追加して

# 仕様の質問
マルチプレイの同期ってどうやるの？
```

## スキルがカバーする知識

| カテゴリ | 内容 |
|---------|------|
| 環境構築 | `akashic init`、テンプレート選択、ファイル競合の回避 |
| TypeScript制約 | `lib: ["es5"]` で使えない構文と代替パターン |
| エンティティ | `FilledRect`, `Sprite`, `Label` 等の使い方 |
| シーン管理 | シーン遷移、アセット読み込み（`assetIds` / `assetPaths`） |
| 操作 | タッチ、ドラッグ（`onPointMoveCapture`）、クリック |
| アニメーション | フレーム更新、タイマー、akashic-timeline |
| 当たり判定 | 矩形判定、衝突方向の判定、collision-js |
| テキスト・音声 | `DynamicFont`, `Label`, 音声再生 |
| 乱数 | `g.game.random` と `Math.random()` 禁止の理由 |
| マルチプレイ | ローカルエンティティ、`raiseEvent`、プレイヤー同期 |
| ニコ生固有 | ランキングスコア、放送者判別、プレイヤー名取得 |
| 公開 | `akashic export` によるzip/HTML出力 |

## 前提条件

- [Node.js](https://nodejs.org/) (LTS推奨)
- [Claude Code](https://claude.com/claude-code)
- akashic-cli: `npm install -g @akashic/akashic-cli`

## ライセンス

MIT
