<p align="center">
  <img src="https://img.shields.io/badge/Version-1.2.1-brightgreen.svg" alt="Version">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-blue.svg" alt="License"></a>
  <img src="https://img.shields.io/badge/Claude_Code-Plugin-purple.svg" alt="Claude Code Plugin">
  <img src="https://img.shields.io/badge/Agents-18-orange.svg" alt="18 Agents">
  <img src="https://img.shields.io/badge/Skills-10-green.svg" alt="10 Skills">
  <a href="https://github.com/kinoko-tz/awesome-novel-studio-jp/stargazers"><img src="https://img.shields.io/github/stars/kinoko-tz/awesome-novel-studio-jp?style=social" alt="GitHub Stars"></a>
</p>

# Awesome Novel Studio (日本語版)

**AI Web小説創作ハーネス** — Claude Code Plugin

Claude Code 上で動く Web 小説制作システム。18 体の専門エージェントと 10 個のスキルを組み合わせ、**企画 → 設計 → 執筆 → 推敲 → 再執筆**の全工程を自動化する。

> **本番運用での実証済み** — このワークフローで執筆された Web 小説（韓国市場）が出版社と正式契約を締結。
> 1 日閲覧数 2,500+ / いいね 1,000+ / 購読 300+

> **本リポジトリは [MJbae/awesome-novel-studio](https://github.com/MJbae/awesome-novel-studio) の日本語ローカライズ版**。原典は韓国市場の Web 小説を前提としており、用語・事例・プラットフォームを日本市場（カクヨム・小説家になろう等）向けに調整している。

---

## インストール

### マーケットプレイス経由

#### マーケットプレイスを追加
```shell
/plugin marketplace add kinoko-tz/awesome-novel-studio-jp
```

#### プラグインをインストール
```shell
/plugin install novel-studio@awesome-ai-studio
```

#### セッション再起動で有効化
```shell
/exit
```

---

## クイックスタート

```bash
# 1. 小説の企画案を 3 案生成して 1 案を選ぶ
/propose

# 2. 大設計（ブートストラップ + キャラクターシート + プロットフックガイド）
/design-big

# 3. 小設計（25 話単位の詳細設計）
/design-small

# 4. 各話の執筆
/create

# 5. 推敲
/polish
```

## パイプライン

```
企画 ────── 設計 ─────────────── 執筆 ──── 推敲 ──── 公開
/propose       /design-big              /create   /polish
               /design-small
                    ↕ 設計変更時
                 /rewrite → /polish
```

---

## 長編 AI 小説の三つの壁

2,500 字 1 話を AI で書くのは難しくない。問題は、それを 300 話・75 万字に伸ばしたときに起こる。

| 壁 | 症状 | Awesome Novel Studio の解法 |
|----|------|--------------------------------|
| **キャラクターの立体性崩壊** | 多数の登場人物が持つ固有の性格・話法・内的動機が、回を重ねるごとに薄れて画一化する。 | **キャラクターシート + ボイステーブル** — キャラクターごとの口調・語尾・非言語パレットを設計段階で定義し、執筆・推敲時に `VOICE`・`TITLE`・`ALIVE` 軸で毎話自動検証する。 |
| **ストーリー整合性の破綻** | 広大な世界観の中で積み上げた事件の因果関係や伏線が崩れる。 | **プロットフックガイド + continuity-bridge** — 全体の物語アークを 25 話単位でビート分解し、各話執筆前に `continuity-bridge` が直前 2 話のタイムライン・伏線・キャラクター状態を収集して執筆エージェントに渡す。 |
| **数値・設定の不整合** | 通貨価値、歴史年代、キャラクターの年齢など、作品の骨格を支える数値が辻褄合わなくなる。 | **guard_rails + LOGIC 軸** — `novel-config.md` に絶対ルールを定義し、推敲の `LOGIC` 軸が直前 2 話と数値・タイムラインを相互検証する。 |

---

## コマンド

| コマンド | 説明 | 備考 |
|---------|------|------|
| `/propose` | ジャンル + 投稿プラットフォーム + コンセプトから 3 案生成 | 新規作品の起点 |
| `/design-big` | 作品全体の設計（ブートストラップ・キャラクター・プロット） | 自動リサーチ含む |
| `/design-small` | 25 話単位の詳細設計 | 大設計が前提 |
| `/design` | 設計ルーター（大設計 + 小設計を統合） | 範囲が不明確なときに使う |
| `/bootstrap` | ブートストラップ文書のみ生成 | 世界観・コンセプトだけ欲しいとき |
| `/character` | キャラクターシートのみ生成 | キャラ設計だけ欲しいとき |
| `/plot-hook` | プロットフックガイドのみ生成 | 物語構造だけ欲しいとき |
| `/create` | 各話の順次執筆 | 設計文書ベース |
| `/polish` (= `/lint`) | 16 軸推敲（6 エージェント並列） | 自動連続進行 |
| `/rewrite` (= `/revise`) | 設計変更後の各話再執筆 | 影響範囲を自動算出 |

### 範囲指定

`create` / `polish` / `rewrite` 系コマンドはエピソード範囲指定を受け付ける。

```bash
/create EP051          # EP051 から開始
/create EP001-EP010    # EP001 ～ EP010
/polish start          # EP001 から
/rewrite プロジェクト名 EP001-EP010
```

## ワークフロー

```
/propose                企画案 3 案生成、1 案選択
       ↓
/design-big             ブートストラップ + キャラクターシート + プロットフックガイド
       ↓                → novel-config.md 自動生成
/design-small           25 話単位の詳細設計（任意・推奨）
       ↓
/create                 各話の執筆（自動連続）
       ↓
/polish                 16 軸推敲（自動連続）
       ↓
[設計変更時] /rewrite → /polish   再執筆 → 再推敲
```

## ディレクトリ構造

パイプラインを動かすと以下の構造が生成される。

```
{プロジェクト名}/
├── novel-config.md              # プロジェクト設定（中央ファイル）
├── design/                      # 設計文書
│   ├── {名前}_bootstrap.md      # 世界観・コンセプト・プラットフォーム戦略
│   ├── {名前}_character.md      # キャラクタープロフィール・関係図・対話 DNA
│   └── {名前}_plot-hook.md      # 三幕構成・25 話ビート・フック戦略
├── episode/                     # 各話本文
│   ├── ep001.md
│   └── ...
├── revision/                    # 作業ファイル
│   ├── fix_plan.md              # 推敲進捗トラッカー
│   └── learnings.md             # 推敲中に発見されたパターン
└── _workspace/                  # 一時作業領域
    └── 00_research/             # 自動リサーチ結果
```

## novel-config.md

パイプライン全体の中央設定ファイル。`/design-big` で自動生成され、ユーザがレビュー・編集する。

```yaml
project:
  name: "私の小説"
  target_platform: "カクヨム"
  target_genre: "現代ダンジョン+お仕事小説"
  episode_dir: "episode/"
  work_dir: "revision/"
  design_dir: "design/"

design_documents:
  bootstrap: "design/my-novel_bootstrap.md"
  character_core: "design/my-novel_character.md"
  plot_guide: "design/my-novel_plot-hook.md"

ep_range_table:
  - range: "EP001-EP025"
    label: "第 1 幕：起源"
    plot_guide: "design/my-novel_plot-hook.md"

guard_rails:
  - "主人公のタイムリープ能力は過去の情報の想起のみ可能"

custom_axes:
  EXPERTISE: "専門知識は対話の中で匂わせるのみ。説明はしない"
```

- **ep_range_table**: プロットガイドから自動抽出されたエピソード範囲
- **guard_rails**: 全執筆・推敲段階で強制される絶対ルール
- **custom_axes**: プロジェクト固有の追加推敲基準

## 16 軸推敲システム

| 軸 | 名称 | 説明 |
|----|------|------|
| 1 | BANNED | 禁止表現（時間ジャンプ・クリシェ、メタコメント） |
| 2 | VOICE | キャラクター対話の一貫性（ボイステーブルと照合） |
| 3 | TITLE | 呼称ルール（話者・聞き手・文脈） |
| 4 | SILENCE | 沈黙パターンの過剰使用（1 話につき最大 4 回） |
| 5 | TRANS | 翻訳調・AI トーン・直訳の検出 |
| 6 | SCENE | 場面構造（ビート・葛藤・解決） |
| 7 | LOGIC | 物語論理・タイムライン・数値整合性 |
| 8 | SUMMARY | 場面ごとの存在意義 |
| 9 | UNIFORM | 各話間の一貫性 |
| 10 | HOOK | フック強度（オープニング・中盤・クリフハンガー） |
| 11 | OPENING | 冒頭 200 字以内のフック |
| 12 | MOBILE | モバイル可読性（段落長・対話比率） |
| A1 | ALIVE | エコー対話の解消 |
| A2 | ALIVE | 沈黙→非言語への置換 |
| A3 | ALIVE | 緊張点でのキャラ生命感 |
| A4 | ALIVE | 距離感の管理 |

## エージェント構成

### 設計エージェント (5)
`concept-builder` · `character-architect` · `plot-hook-engineer` · `proposal-generator` · `domain-researcher`

### 執筆エージェント (4)
`episode-architect` · `episode-creator` · `continuity-bridge` · `quality-verifier`

### 推敲エージェント (6)
`rule-checker` · `story-analyst` · `platform-optimizer` · `alive-enhancer` · `revision-executor` · `revision-reviewer`

### 再執筆エージェント (4)
`revision-analyst` · `character-sculptor` · `episode-rewriter` · `quality-verifier`

> `quality-verifier` は執筆と再執筆の両フェーズで共有される（ユニーク数で 18 体）。

## 謝辞

[Minho Hwang (revfactory)](https://github.com/revfactory) 氏に感謝する。氏の [Harness](https://github.com/revfactory/harness) プラグインのおかげで、初期ハーネスのアーキテクチャ立ち上げが容易になった。LinkedIn その他で氏が共有してくれている知見は常に大きなインスピレーション源となっている。

そして本プロジェクトの原典である [MJbae/awesome-novel-studio](https://github.com/MJbae/awesome-novel-studio) の作者 [MJbae](https://github.com/MJbae) 氏に感謝する。本リポジトリはその日本語ローカライズ版である。

## ライセンス

[Apache 2.0](LICENSE)
