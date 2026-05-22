# 設定文書マッピングガイド — Polish エージェント用

推敲エージェントが参照する設定文書は、**プロジェクト別の novel-config.md** に定義する。
このファイルは novel-config.md を作成する際に参考にするガイドである。

---

## マッピング構造

各小説プロジェクトの `novel-config.md` に、以下の構造を記入する。

### 1. 共通文書（すべての EP に適用）

| 文書キー | 用途 | エージェント |
|---------|------|---------|
| character_core | キャラクターの核となる定義、固有設定、能力マトリクス | 全体 |
| character_detail | ボイス表、呼称表、非言語タグ、関係変化 | rule-checker, alive-enhancer, revision-* |
| bootstrap | マクロ数値、世界観ルール、時間軸 | story-analyst, revision-* |

### 2. EP 範囲別プロットガイド

小説の幕（Act）構造に従って EP 範囲を分割し、各範囲に該当するプロットガイドのパスを指定する。

```
EP001~EP026 → plot-hook-guide_act1.md
EP027~EP076 → plot-hook-guide_act2.md
EP077~EP150 → plot-hook-guide_act3.md
```

推敲スキルは対象 EP 番号を見て、自動的に該当するプロットガイドを選択する。

### 3. 補助参照（任意）

| 文書キー | 用途 | エージェント |
|---------|------|---------|
| web_novel_guide | モバイル最適化の原則 | platform-optimizer |
| verification | 検証完了した数値の記録 | story-analyst |
| plot_macro | 核心となる転換ポイントのマクロ | platform-optimizer |

---

## 数値クロスチェックの正本優先順位

数値の不一致を発見した際にどの文書を正本とするかを novel-config.md に明記する。
一般的な優先順位は以下のとおりである。

1. EP 別プロットガイド（最も具体的）
2. ブートストラップ（マクロ数値）
3. 検証記録（すでに確認済みの数値）
4. 直前のエピソード（叙事の連続性）

---

## 新規プロジェクトの設定方法

1. プロジェクトルートに `novel-config.md` を作成する
2. `${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md` を参照してセクションを記入する
3. `/polish {プロジェクト名}` で実行する
