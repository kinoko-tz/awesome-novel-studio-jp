---
name: design-big
description: "Web 小説の大設計 (作品全体) を実行するオーケストレーター。ブートストラップ + キャラクターシート + プロットフックガイドを生成する。自動リサーチ (domain-researcher サブエージェント) で専門分野の資料を収集した後に設計を進める。「大設計」「作品全体設計」「小説設計」「ブートストラップからキャラクター・プロットまで」といった依頼にはこのスキルを使うこと。統合設計 (大 + 小の両方) や曖昧な依頼には design ルーターを、小設計のみが必要なら design-small を、単一領域のみ必要なら bootstrap/character/plot-hook を使うこと。"
---

# Novel Design Big — 大設計オーケストレーター

Web 小説の作品全体設計を実行する。AI がジャンル DNA フレームワークで骨格を立て、domain-researcher サブエージェントが自動リサーチで専門分野の肉付けを行う構造である。

## 実行モード: エージェントチーム

## エージェント構成

| チームメンバー | エージェントファイル | 役割 | スキル | 出力 |
|------|-------------|------|------|------|
| concept-builder | `${CLAUDE_PLUGIN_ROOT}/agents/concept-builder.md` | ブートストラップ設計 | bootstrap | ブートストラップ文書 |
| character-architect | `${CLAUDE_PLUGIN_ROOT}/agents/character-architect.md` | キャラクター設計 | character | キャラクターシート |
| plot-hook-engineer | `${CLAUDE_PLUGIN_ROOT}/agents/plot-hook-engineer.md` | プロット/フック設計 | plot-hook | プロットフックガイド |

**domain-researcher はチームメンバーではなくサブエージェント**であり、Phase 1.5 でチーム構成 (TeamCreate) 前に実行される。

## 共有レファレンス

- **genre-dna-framework.md の位置**: `${CLAUDE_PLUGIN_ROOT}/skills/design/references/genre-dna-framework.md` (ルーター配下 — big/small 共用)

## ワークフロー

### Phase 0: 既存設計の確認 (任意)

> プロジェクトルートに既存の大設計成果物 (`{作品仮題}_ブートストラップ.md`、`{作品仮題}_キャラクターシート.md`、`{作品仮題}_プロットフックガイド.md`) が存在する場合のみ実行する。なければ Phase 1 へ直行する。

1. プロジェクトルートで既存の大設計成果物を Glob で確認する
2. 既存成果物が見つかった場合、ユーザーにモード選択を依頼する:

```
既存の大設計文書が見つかりました:
- {見つかったファイル一覧}

以下から選択してください:
1. **全体再設計** — 既存文書を _workspace/_backup/ にバックアップしてから最初から設計し直す
2. **部分修正** — 特定領域のみ再実行 (例: ブートストラップのみ、キャラクターのみ、プロットのみ)
3. **資料補強後の再設計** — 新規リサーチ資料を追加した後に既存設計をアップグレードする
```

3. モード別の処理:
   - **全体再設計**: 既存成果物を `_workspace/_backup/{timestamp}/` にコピー → Phase 1 から進行
   - **部分修正**: 修正対象のエージェントのみ再実行する。残りの成果物は維持する。Phase 2 で当該エージェントのみチームに含める
   - **資料補強後の再設計**: 既存の `_workspace/00_concept_analysis.md` からコンセプトを復元 → Phase 1 をスキップ → Phase 1.5 から進行

### Phase 1: コンセプト分析と方向性の確立

1. **提案書の自動連携** (強化):
   - プロジェクトルートで `*_提案書.md` パターンを Glob で探索する
   - 見つかったら Read して、提案書の **コンセプト、ジャンル、プラットフォーム、差別化ポイント、ログライン** を自動抽出する
   - 抽出されたプラットフォームは必ず以下の許容プラットフォーム集合で検証する:
     - カクヨム、小説家になろう、アルファポリス、ノベルアップ+、エブリスタ、ノベルピア
   - `Kakuyomu`、`kakuyomu`、`カクヨム` のような表記揺れや別名は canonical name に正規化する
   - 許容されないプラットフォームや曖昧な別名であれば自動反映せず、ユーザーに提案書の修正またはプラットフォーム再選択を依頼する
   - 抽出した情報で Phase 1 のコンセプト分析を事前に埋める (ユーザーに確認を依頼する)
   - 提案書の `_workspace/00_research/` ディレクトリも確認し、既存の R1/R2/R5 リサーチがあれば Phase 1.5 で再利用する
   - 提案書がない場合: ユーザー入力から直接分析する (従来動作)
   ```
   提案書 '{ファイル名}' を発見しました。
   以下のコンセプトで大設計を進めます:
   - ジャンル: {提案書から抽出}
   - コンセプト: {提案書から抽出}
   - プラットフォーム: {提案書から抽出}
   - 差別化: {提案書から抽出}

   修正したい部分があれば教えてください。なければそのまま進行します。
   ```
2. ユーザー入力の分析 — 小説のコンセプト、職業、ムード、差別化方針を把握する
   - ターゲットプラットフォームが明示されていない場合、投稿プラットフォーム 6 個のうちいずれかを確認する
   - ターゲットプラットフォームが非対応値であれば自動置換せず再選択を依頼する
3. `${CLAUDE_PLUGIN_ROOT}/skills/design/references/genre-dna-framework.md` を読み、ジャンル DNA フレームワークを確認する
4. プロジェクトルートの参考文書を確認する (存在時)
5. **コンセプト方向性の要約** をユーザーに提示して確認する:
   - 主人公の職業/専門分野
   - 物語の起点 (時代、契機)
   - 中核となる差別化ポイント
   - ターゲットプラットフォーム (カクヨム、小説家になろう、アルファポリス、ノベルアップ+、エブリスタ、ノベルピアのうち 1 個)
   - 想定トーン & ムード
   - 類似作品群 2~3 作 (リーダーが genre-dna 差別化戦略 + 業界知見から提案)
6. **`{作品仮題}` の確定** — ユーザーに作品仮題を確認する。この仮題が以後すべてのファイル名のプレフィックスとして用いられる
7. Phase 1 の結果を `_workspace/00_concept_analysis.md` に保存する — 新しい会話を開始した場合でもコンセプトを復元できるようにする。保存内容: 作品仮題、専門分野、物語の起点、差別化ポイント、類似作品群、トーン & ムード
8. ユーザー確認後 → Phase 1.5 へ進行

### Phase 1.5: 自動リサーチ (サブエージェント)

> domain-researcher サブエージェントを呼び出して専門分野リサーチを自動実行する。ユーザーの待機なしで即座に進行する。

**サブエージェント: domain-researcher**
- subagent_type: `general-purpose`
- リサーチ項目:
  - **R3 業界/職業構造**: 当該専門分野の組織構造、キャリアパス、権力階層
  - **R4 事件年表**: 当該分野/時代の主要事件、転換点、業界変化
  - **R5 既存作分析**: 類似ジャンル/題材の既存 Web 小説の分析、差別化可能点
  - **R6 葛藤事例**: 当該分野の実在の葛藤/事件事例、ヴィラン・モチーフ

- プロンプト:
```
あなたは domain-researcher サブエージェントです。
以下のリサーチを実行してください:

1. R3 業界/職業構造:
   - 専門分野: {専門分野}
   - 分析項目: 組織構造、役職体系、キャリアパス、権力関係、中核能力

2. R4 事件年表:
   - 時代: {物語起点の時代}~現在
   - 分析項目: 業界の主要事件、社会的転換点、技術変化、危機と機会

3. R5 既存作分析:
   - ジャンル: {ジャンル}
   - 分析項目: 類似作品リスト、成功要因、読者反応、差別化の空白地帯

4. R6 葛藤事例:
   - 分野: {専門分野}
   - 分析項目: 実在の葛藤/不正/事件、ヴィラン・モチーフとして活用可能なパターン

出力:
- _workspace/00_research/R3_業界構造.md
- _workspace/00_research/R4_事件年表.md
- _workspace/00_research/R5_既存作分析.md
- _workspace/00_research/R6_葛藤事例.md
```

- リサーチ結果は `_workspace/00_research/` に保存する
- リサーチ完了後ただちに Phase 2 へ進行する (ユーザー待機なし)

### Phase 2: チーム構成

リーダーは TeamCreate 前に `_workspace/00_research/` 内のリサーチ結果ファイルの存在有無を Glob で確認する。

```
TeamCreate(
  team_name: "design-big-team",
  members: [
    {
      name: "concept-builder",
      agent_type: "general-purpose",
      prompt: "あなたは concept-builder エージェントです。
        ${CLAUDE_PLUGIN_ROOT}/agents/concept-builder.md を読んで役割を把握してください。
        ${CLAUDE_PLUGIN_ROOT}/skills/bootstrap/SKILL.md を読んで作業手順と出力テンプレートに従ってください。
        ${CLAUDE_PLUGIN_ROOT}/skills/design/references/genre-dna-framework.md を読んでジャンル DNA フレームワークを把握してください。
        プロジェクトルートの参考文書も読んでください (存在時)。

        ★ 自動リサーチ結果 (存在するファイルのみ Read):
        - _workspace/00_research/R3_業界構造.md → 反映先: 世界観 > 業界組織図、主人公の背景 > キャリアパス
        - _workspace/00_research/R4_事件年表.md → 反映先: 中核能力モジュール (未来知識年表) の具体化
        - _workspace/00_research/R5_既存作分析.md → 反映先: 既存作品比でのポジショニング、セリングポイント差別化
        - _workspace/00_research/R6_葛藤事例.md → 反映先: 世界観 > 社会的文脈、中核能力モジュール > 物語的機能
        (ファイルがなければ genre-dna ベースで進行するが、レポートに『リサーチ資料未反映: [カテゴリ名]』を明記する)

        ユーザーの小説コンセプト: {ユーザー入力の要約}
        ブートストラップ文書を作成し、_workspace/01_concept-builder_bootstrap.md に保存してください。
        作成完了後、以下の情報を SendMessage してください:
        - character-architect へ: (1) 主人公の中核設定 (2) 専門分野の特性 (3) 物語の起点 (4) 世界観の社会構造
        - plot-hook-engineer へ: (1) 中核能力モジュールの要約 (2) 読者保持転換戦略 (プラットフォーム依存 — 課金モデルなら課金転換、無料公開モデルなら離脱防止転換／書籍化アピール区間) (3) 50 話単位アークの骨格 (4) スケール拡大ロードマップ"
    },
    {
      name: "character-architect",
      agent_type: "general-purpose",
      prompt: "あなたは character-architect エージェントです。
        ${CLAUDE_PLUGIN_ROOT}/agents/character-architect.md を読んで役割を把握してください。
        ${CLAUDE_PLUGIN_ROOT}/skills/character/SKILL.md を読んで作業手順と出力テンプレートに従ってください。
        ${CLAUDE_PLUGIN_ROOT}/skills/design/references/genre-dna-framework.md を読んでジャンル DNA フレームワークを把握してください。

        ★ 自動リサーチ結果 (存在するファイルのみ Read):
        - _workspace/00_research/R3_業界構造.md → 反映先: ヴィラン > 役職/所属/権限、協力者 > 業界内での位置
        - _workspace/00_research/R6_葛藤事例.md → 反映先: ヴィラン > 動機と行動パターン、葛藤類型の現実的根拠
        (ファイルがなければ genre-dna キャラクターフレームワークベースで進行するが、レポートに未反映カテゴリを明記する)

        concept-builder から SendMessage を受信したら、
        _workspace/01_concept-builder_bootstrap.md を Read して全体ブートストラップを把握した上で
        大設計のキャラクターシートを作成してください。
        出力: _workspace/02_character-architect_sheet.md
        作成完了後、plot-hook-engineer に以下を SendMessage してください:
        (1) 主人公の中核動機 (2) 敵対者階層の全体像 (3) VIP 協力者リストと登場時点 (4) ロマンスライン設定"
    },
    {
      name: "plot-hook-engineer",
      agent_type: "general-purpose",
      prompt: "あなたは plot-hook-engineer エージェントです。
        ${CLAUDE_PLUGIN_ROOT}/agents/plot-hook-engineer.md を読んで役割を把握してください。
        ${CLAUDE_PLUGIN_ROOT}/skills/plot-hook/SKILL.md を読んで作業手順と出力テンプレートに従ってください。
        ${CLAUDE_PLUGIN_ROOT}/skills/design/references/genre-dna-framework.md を読んでジャンル DNA フレームワークを把握してください。

        ★ 自動リサーチ結果 (存在するファイルのみ Read):
        - _workspace/00_research/R4_事件年表.md → 反映先: 中核能力モジュール活用タイムライン > 実在事件のマッピング
        - _workspace/00_research/R6_葛藤事例.md → 反映先: アーク別の葛藤構造、ヴィラン対決の現実的パターン
        (ファイルがなければ genre-dna 物語公式ベースで進行するが、レポートに未反映カテゴリを明記する)

        concept-builder と character-architect の双方から SendMessage を受信したら、
        _workspace/01_concept-builder_bootstrap.md と _workspace/02_character-architect_sheet.md を
        Read して全内容を把握した上で大設計のプロット/フックガイドを作成してください。
        出力: _workspace/03_plot-hook-engineer_guide.md"
    }
  ]
)
```

タスク登録:
```
TaskCreate(tasks: [
  { title: "ブートストラップ文書の作成", assignee: "concept-builder" },
  { title: "キャラクターシート作成 (大設計)", assignee: "character-architect",
    depends_on: ["ブートストラップ文書の作成"] },
  { title: "プロット/フックガイド作成 (大設計)", assignee: "plot-hook-engineer",
    depends_on: ["ブートストラップ文書の作成", "キャラクターシート作成 (大設計)"] }
])
```

### Phase 3: 大設計の実行

**実行方式:** パイプライン + 部分並列

1. concept-builder がブートストラップ文書を作成する
2. concept-builder → character-architect、plot-hook-engineer に SendMessage (中核設定の共有)
3. character-architect がキャラクターシートを作成する (concept-builder の設定をベースに)
4. character-architect → plot-hook-engineer に SendMessage (敵対者階層、VIP リスト)
5. plot-hook-engineer がプロット/フックガイドを作成する (ブートストラップ + キャラクターシートをベースに)

**チームメンバー間の通信ルール:**
- concept-builder はブートストラップ完成時に双方のメンバーへ中核設定を SendMessage する
- character-architect はキャラクターシート完成時に plot-hook-engineer へ SendMessage する
- 設定矛盾を発見した場合、該当メンバーへ直接 SendMessage で調整を依頼する
- 各メンバーはファイル保存完了時にリーダーへ通知する

**成果物の保存:**

| チームメンバー | 出力パス |
|------|----------|
| concept-builder | `_workspace/01_concept-builder_bootstrap.md` |
| character-architect | `_workspace/02_character-architect_sheet.md` |
| plot-hook-engineer | `_workspace/03_plot-hook-engineer_guide.md` |

**リーダーモニタリング:**
- TaskGet で全体の進捗率を確認する
- メンバー遊休時の自動通知を受信する
- 特定メンバーが詰まった場合、SendMessage で介入する

### Phase 4: 統合検証と整理

1. 全メンバーの作業完了を待機する (TaskGet で状態確認)
2. 各メンバーの成果物を Read で収集する
3. **整合性検証チェックリスト** (ブートストラップ文書が source of truth):
   - [ ] 主人公の職業/年齢/物語起点がブートストラップ↔キャラクターシートで一致しているか
   - [ ] 主人公の過去/背景設定がブートストラップ↔キャラクターシートで一致しているか
   - [ ] 中核能力モジュール (ブートストラップ) の項目がプロットガイドの活用タイムラインに反映されているか
   - [ ] キャラクターシートの敵対者名/アークがプロットガイドのアーク別敵対者と一致しているか
   - [ ] VIP 協力者の登場時点 (キャラクターシート) がプロットガイドのタイムラインと整合しているか
   - [ ] ロマンスラインの初対面時点がキャラクターシート↔プロットガイドで一致しているか
   - [ ] 読者保持転換戦略 (ブートストラップ — プラットフォーム依存で課金転換または離脱防止転換／書籍化アピール区間) がプロットガイドの 25 話/50 話配置と整合しているか
   - [ ] リサーチ資料の中核情報がブートストラップの中核能力モジュールに反映されているか
   - [ ] 敵対者の役職/行動が業界構造リサーチと整合しているか
   - [ ] 実在の葛藤事例から着想を得たアークが最低 1 個以上あるか
   - 矛盾を発見した場合: ブートストラップを基準に他文書を修正し、修正内容を結果報告に含める

4. 最終成果物を `{DESIGN_DIR}` にコピーする (novel-config.md のパスと一致させる):

| 中間成果物 | 最終パス |
|-----------|----------|
| `_workspace/01_*_bootstrap.md` | `{DESIGN_DIR}/{作品仮題}_ブートストラップ.md` |
| `_workspace/02_*_sheet.md` | `{DESIGN_DIR}/{作品仮題}_キャラクターシート.md` |
| `_workspace/03_*_guide.md` | `{DESIGN_DIR}/{作品仮題}_プロットフックガイド.md` |

   > **パス整合性原則**: novel-config.md の設定文書マッピングパスと実際のファイル位置は必ず一致させなければならない。
   > Phase 5 で config に `design/{作品仮題}_*.md` として記録するため、ここでも `design/` 配下に保存する。
   > `{DESIGN_DIR}` ディレクトリがなければ生成する (デフォルト値: `design/`)。

5. メンバーへ終了要請 (SendMessage)
6. **TeamDelete("design-big-team")** — チーム解散
   > 以後 design-small 実行時のチーム衝突を防止する。必ず実行する。
7. `_workspace/` ディレクトリは保持する (事後検証用)
8. ユーザーへ結果サマリーを報告する

### Phase 5: novel-config.md ドラフトの自動生成

大設計完了後、創作/推敲/再執筆スキルが利用する `novel-config.md` のドラフトを自動生成する。
ユーザーが手動で作成する必要はなく、設計成果物のパスと構造を分析してドラフトを作成する。

1. プロットフックガイドからアーク構造 (1 幕/2 幕/3 幕) を抽出して EP 範囲テーブルを自動構成する
2. ブートストラップから保存ガードレール候補を抽出する (世界観ルール、中核設定)
3. キャラクターシートで対話 DNA セクションの存在有無を確認する
4. `${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md` を参照して形式を合わせる
5. **target_platform 検証ゲート**:
   - Phase 1 で確定したプラットフォームが投稿プラットフォーム canonical name 6 個のいずれかに該当するか再検証する
   - 非対応値であれば `novel-config.md` を生成せずユーザー修正を依頼する
6. **カクヨムプリセットの流し込み** (Phase 1 でカクヨムプリセットを適用した場合のみ):
   - `${CLAUDE_PLUGIN_ROOT}/skills/design/references/platform-preset-kakuyomu.md` と project-config-template.md の「11. カクヨムプリセット」セクションを参照する
   - 自動生成する novel-config.md に以下を流し込む:
     - ターゲット読者の上書き（偏差値 60 以上の高校・大学生／編集者層が好む感性）
     - KPI の上書き（書籍化スカウト／コンテスト評価。課金転換なし）
     - custom_axes: FORESHADOW / DESPAIR / EMOTION_SWING / HAPPYEND / DUAL_LAYER
     - 保存ガードレール: 各章末の「希望の灯」省略不可、ラスト 80% 最強化の成長曲線厳守
   - あわせてプロットフックガイドに二段の絶望 (50%)・再起の階段 (50〜80%)・伏線の設置→回収対応表が反映されているか整合確認する

生成パス: `{プロジェクトルート}/novel-config.md`

```markdown
# novel-config.md (自動生成ドラフト — レビュー後に修正可)

## プロジェクト基本情報
project:
  name: "{作品仮題}"
  target_platform: "{Phase 1 で確認したプラットフォーム}"
  target_genre: "{Phase 1 で確認したジャンル}"
  episode_dir: "episode/"
  work_dir: "revision/"
  design_dir: "design/"

## 設定文書マッピング
### 共通文書
| 文書キー | パス | 用途 |
|---------|------|------|
| character_core | {DESIGN_DIR}/{作品仮題}_キャラクターシート.md | キャラクター中核定義 |
| character_detail | {DESIGN_DIR}/{作品仮題}_キャラクターシート.md | ボイステーブル、非言語タグ |
| dialogue_dna | {DESIGN_DIR}/{作品仮題}_キャラクターシート.md#dialogue-dna | Dialogue DNA (台詞固有性) — キャラクターシート内のセクション |
| bootstrap | {DESIGN_DIR}/{作品仮題}_ブートストラップ.md | 世界観、マクロ数値 |
| writing_rules | CLAUDE.md | 執筆ルール |

### EP 範囲別の設定文書
| EP 範囲 | レーベル | プロットガイドのパス | 詳細プロットガイド (任意) | 詳細キャラクターシート (任意) |
|---------|--------|----------------|----------------------|----------------------|
| {アーク 1 範囲} | {アーク 1 レーベル} | {DESIGN_DIR}/{作品仮題}_プロットフックガイド.md | | |
| {アーク 2 範囲} | {アーク 2 レーベル} | {DESIGN_DIR}/{作品仮題}_プロットフックガイド.md | | |
| {アーク 3 範囲} | {アーク 3 レーベル} | {DESIGN_DIR}/{作品仮題}_プロットフックガイド.md | | |

## 保存ガードレール
{ブートストラップから抽出した中核保存項目}

## 数値クロス検証の正本優先順位
1. plot_by_ep — EP 別の確定数値
2. bootstrap — マクロ数値
3. verification — 検証完了済み数値
4. 直前エピソード — 物語連続性
```

5. ユーザーにドラフトのレビューを依頼する:
```
novel-config.md のドラフトを生成しました。
保存ガードレールと EP 範囲テーブルをレビューし、必要に応じてカスタム軸を追加してください。
```

### 大設計完了時の小設計案内

```
## 大設計が完了しました。

### 生成された文書
- design/{作品仮題}_ブートストラップ.md
- design/{作品仮題}_キャラクターシート.md
- design/{作品仮題}_プロットフックガイド.md
- novel-config.md (ドラフト — レビュー後に修正可)

### 次のステップ: 小設計 (25 話単位の詳細設計)

⚠️ **小設計を飛ばしていきなり `/create` を実行すると、EP 別プロットビートがないためエピソード品質が大きく低下します。**
大設計のプロットフックガイドはアーク単位の概要しか含まないため、episode-architect が EP 別の設計図を抽出することは困難です。

小設計を進める場合は `design-small` スキルを使用してください。
小設計でも domain-researcher が当該アークの詳細リサーチを自動実行します。
小設計完了時には novel-config.md の EP 範囲テーブルに詳細プロットガイドのパスが自動追加されます。
```

## エラーハンドリング

| 状況 | 戦略 |
|------|------|
| domain-researcher 失敗 | 1 回再試行。再失敗時は genre-dna ベースでチーム構成を進行し、レポートに「自動リサーチ未反映」を明記 |
| concept-builder 失敗 | 中核設定であるため必ず再試行。再失敗時はリーダーが直接ブートストラップのドラフトを作成 |
| character-architect 失敗 | 1 回再試行。再失敗時はリーダーが genre-dna ベースの基本キャラクターシートを生成 |
| plot-hook-engineer 失敗 | 1 回再試行。再失敗時はブートストラップ + キャラクターシートのみで基本プロットガイドを生成 |
| 設定矛盾の発見 | ブートストラップ文書を基準 (source of truth) に他文書を修正 |
| メンバー間通信の遅延 | リーダーが間に入ってファイルを Read し、手動で情報を伝達 |
| リサーチ結果の品質不足 | genre-dna フレームワークベースで補強、レポートに明記 |
| 提案書/入力のプラットフォームが非対応値 | 自動マッピングせず投稿プラットフォーム 6 個から再選択を依頼 |

## データフロー

```
[ユーザー] → 小説コンセプト
    ↓
Phase 0: 既存設計の確認 (任意)
    ↓
Phase 1: コンセプト分析 → _workspace/00_concept_analysis.md
    ↓ (提案書存在時に自動ロード)
Phase 1.5: domain-researcher サブエージェント → _workspace/00_research/ (R3~R6)
    ↓ (ユーザー待機なし)
Phase 2: TeamCreate("design-big-team") — リサーチ結果を含む
    ↓
Phase 3: concept-builder → character-architect → plot-hook-engineer
    ↓
Phase 4: 統合検証 → 成果物 3 種
    ↓
小設計案内
```

## テストシナリオ

### 正常フロー
1. ユーザーが「人生やり直しの法医学専門医」コンセプトを提供
2. Phase 1 でコンセプト分析、作品仮題を確定 (「法医学専門医」)
3. Phase 1.5 で domain-researcher が R3~R6 の自動リサーチを実行
4. Phase 2 でチーム構成 (リサーチ結果をプロンプトに含む)
5. Phase 3 で CB→CA→PHE の順に大設計を完成
6. Phase 4 で整合性検証後、成果物 3 種 + 小設計案内

### 提案書連携フロー
1. プロジェクトルートに `法医学専門医_提案書.md` が存在
2. Phase 1 で提案書をロードしてコンセプトを自動把握
3. ユーザー確認後、Phase 1.5 へ即時進行
4. 以降は正常フローと同じ

### エラーフロー
1. Phase 1.5 で domain-researcher が失敗
2. リーダーが 1 回再試行 → 再失敗
3. genre-dna フレームワークベースで Phase 2 を進行、レポートに「自動リサーチ未反映」を明記
4. Phase 3 で plot-hook-engineer がエラーで停止
5. リーダーが遊休通知を受信 → SendMessage で状態確認 → 1 回再試行
6. 再試行失敗時はリーダーがブートストラップ + キャラクターシートをベースに基本プロットガイドを直接作成
7. 最終レポートに「plot-hook-engineer 自動生成 — 手動レビュー推奨」を明記
