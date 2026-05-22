# 設定文書マッピング — Rewrite エージェント用

rewrite エージェントが参照する設定文書は、プロジェクトごとの `novel-config.md` に定義する。
polish スキルと同一の設定ファイルを共有し、rewrite 専用キーが追加される。

共通マッピングガイド: `${CLAUDE_PLUGIN_ROOT}/skills/polish/references/design-document-map.md` を参照。
プロジェクト設定の書き方: `${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md` を参照。

---

## Rewrite 専用設定キー

novel-config.md の共通文書（character_core, character_detail, bootstrap）に加えて、
rewrite スキルは以下の専用キーを追加で使用する。

### 1. writing_rules（共通文書セクション）

| 文書キー | 既定値 | 用途 | 参照エージェント |
|---------|--------|------|-------------|
| writing_rules | CLAUDE.md | 執筆規則バイブル — 文体、視点、エピソード構成原則 | revision-analyst, episode-rewriter, quality-verifier |

novel-config.md にこのキーがない場合は、プロジェクトルートの `CLAUDE.md` を既定値として使用する。
プロジェクトごとに執筆規則が異なる場合は、別ファイルのパスを指定する。

### 2. character_dialogue_dna（Rewrite 専用設定セクション）

| 文書キー | 既定値 | 用途 | 参照エージェント |
|---------|--------|------|-------------|
| character_dialogue_dna | （character_detail で代替） | キャラクター別対話 DNA — 思考パターン、情報処理、説得方式、状況別の変奏 | character-sculptor, episode-rewriter |

このファイルがない場合は、character_detail のみで対話診断を実施する。
対話 DNA を別途管理すると、キャラクターのセリフの交換不可能性の検証が強化される。

### 3. rewrite_work_dir（Rewrite 専用設定セクション）

| キー | 既定値 | 用途 |
|----|--------|------|
| rewrite_work_dir | （polish の work_dir を使用） | rewrite-plan.md, rewrite-log.md の保存場所 |

polish と rewrite の作業ファイルを分離したい場合は、別パスを指定する。
同じディレクトリを共有してもファイル名が異なる（fix_plan.md と rewrite-plan.md）ため、衝突しない。

### 4. Rewrite 保存ガードレール（Rewrite 専用設定セクション）

polish の「保存ガードレール」に追加される rewrite 専用ガードレール。
再執筆時のキャラクターの驚愕の仕方の交差汚染防止、主人公の動機表現の規則などを定義する。

---

## エージェント別文書参照

### revision-analyst
| 参照目的 | 文書キー |
|----------|---------|
| プロットビート・確定数値 | plot_by_ep（EP 範囲別） |
| 世界観・数値規則 | bootstrap |
| キャラクター核心定義 | character_core |
| 執筆規則 | writing_rules |
| カスタム軸 | custom_axis（ある場合） |

### character-sculptor
| 参照目的 | 文書キー |
|----------|---------|
| ボイス表・非言語・関係変化 | character_detail |
| キャラクター核心定義 | character_core |
| 対話 DNA | character_dialogue_dna（ない場合は character_detail） |
| 関係アークの現況 | alive-tracker.md（work_dir） |

### episode-rewriter
| 参照目的 | 文書キー |
|----------|---------|
| 執筆規則 | writing_rules |
| プロットビート・確定数値 | plot_by_ep |
| キャラクター核心 | character_core |
| ボイス表・非言語 | character_detail |
| 世界観 | bootstrap |
| 対話 DNA | character_dialogue_dna |
| 関係アークの現況 | alive-tracker.md（work_dir） |

### quality-verifier（REWRITE モード）
| 参照目的 | 文書キー |
|----------|---------|
| 執筆規則 | writing_rules |
| プロットビート・確定数値 | plot_by_ep |
| キャラクター核心 | character_core |
| ボイス表・非言語 | character_detail |
