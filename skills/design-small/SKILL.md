---
name: design-small
description: "Web 小説の小さな設計（25 話単位の詳細設計）を実行するオーケストレーター。詳細キャラクターシート + 詳細プロットフックガイドを生成する。大きな設計文書（ブートストラップ、キャラクターシート、プロットフックガイド）が前提条件。自動リサーチ（domain-researcher サブエージェント）で当該アークの詳細資料を収集した後に設計を進める。「小さな設計」「詳細設計」「25 話設計」「N〜M 話設計」「エピソード別プロット」「話別フック」の要請にこのスキルを使用すること。大きな設計（小説全体）が必要なら design-big を、統合設計や曖昧な要請は design ルーターを、単一領域のみ必要なら bootstrap/character/plot-hook を使用せよ。"
---

# Novel Design Small — 小さな設計オーケストレーター

Web 小説の 25 話単位の詳細設計を実行する。大きな設計の成果物を土台に、domain-researcher サブエージェントがエピソードレベルの具体的な専門知識と実際の事件ディテールを自動でリサーチする。

## 実行モード: エージェントチーム

## エージェント構成

| チームメンバー | エージェントファイル | 役割 | スキル | 出力 |
|------|-------------|------|------|------|
| character-architect | `${CLAUDE_PLUGIN_ROOT}/agents/character-architect.md` | 詳細キャラクター設計 | character (モード B) | 詳細キャラクターシート |
| plot-hook-engineer | `${CLAUDE_PLUGIN_ROOT}/agents/plot-hook-engineer.md` | 詳細プロット/フック設計 | plot-hook (モード B) | 詳細プロットフックガイド |

**domain-researcher はチームメンバーではなくサブエージェント**として、チーム編成（TeamCreate）の前に実行される。

## 前提条件

大きな設計文書 3 種が存在する必要がある:
- `{作品仮題}_ブートストラップ.md`
- `{作品仮題}_キャラクターシート.md`
- `{作品仮題}_プロットフックガイド.md`

## 共有リファレンス

- **genre-dna-framework.md の位置**: `${CLAUDE_PLUGIN_ROOT}/skills/design/references/genre-dna-framework.md` (ルーター配下 — big/small 共用)

## 権限案内

このスキルはチームメンバー（サブエージェント）が `.claude/`、`design/`、`_workspace/` 配下のファイルを Read する。
初回実行時に `Read(//**)` 権限を許可すれば反復承認なしで進行する。

## ワークフロー

### Phase 1: 範囲確認および大きな設計文書の点検

1. 対象話数区間の確認（推奨: 25 話単位。例: 1〜25 話、26〜50 話）
2. 大きな設計文書 3 種の存在確認（novel-config.md があれば `design_dir` から、なければプロジェクトルートと `_workspace/` から探索）:
   - `{DESIGN_DIR}/{作品仮題}_ブートストラップ.md` (または `_workspace/01_*`)
   - `{DESIGN_DIR}/{作品仮題}_キャラクターシート.md` (または `_workspace/02_*`)
   - `{DESIGN_DIR}/{作品仮題}_プロットフックガイド.md` (または `_workspace/03_*`)
   - novel-config.md がない場合は `design/` およびプロジェクトルートを順次探索
   - **欠落時**: ユーザーに大きな設計を先に完了するよう案内し中断
3. 大きな設計文書から該当アークの概要を抽出:
   - アーク題、核心葛藤、主要敵対者
   - 当該区間の核心力量モジュール項目
   - 登場予定キャラクター（VIP、新規キャラクター）
4. **アーク範囲整合性検証**: 要請話数区間が大きな設計のアーク構造と不一致なら、アーク境界をユーザーに案内し区間調整を提案
5. アーク概要をユーザーに確認 → Phase 1.5 へ進行

### Phase 1.5: 自動リサーチ (サブエージェント)

> domain-researcher サブエージェントを呼び出し、当該アークの詳細リサーチを自動で行う。ユーザー待機なしに即座に進行する。

**サブエージェント: domain-researcher**
- subagent_type: `general-purpose`
- リサーチ項目:
  - **R7 専門技術/知識ディテール**: 当該アークで活用される具体的な専門知識、技術、シーンディテール
  - **R8 事件詳細タイムライン**: 当該アーク時間帯の実際の事件詳細（日付、人物、結果、波及効果）

- プロンプト:
```
あなたは domain-researcher サブエージェントです。
以下のリサーチを実行してください。

1. R7 専門技術/知識ディテール:
   - 専門分野: {専門分野}
   - アーク範囲: {N}〜{M}話
   - 分析項目: 具体的な技術名、手順、用語、リアクション描写に活用するディテール

2. R8 事件詳細タイムライン:
   - 時代: {当該アークの時間帯}
   - 分析項目: 正確な日付、前兆シグナル、波及効果、核心人物、社会的反応

大きな設計文書を参照してください:
- {作品仮題}_ブートストラップ.md (核心力量モジュール、専門分野)
- {作品仮題}_プロットフックガイド.md (当該アーク概要)

出力:
- _workspace/00_research/R7_専門知識_{N}〜{M}話.md
- _workspace/00_research/R8_事件詳細_{N}〜{M}話.md
```

- リサーチ結果は `_workspace/00_research/` に保存
- リサーチ完了後ただちに Phase 2 へ進行

### Phase 2: チーム編成

> ⚠️ **TeamCreate 前の安全点検**:
> 既存チーム（"design-big-team" など）が残存していれば TeamCreate が失敗する。
> "Already leading team" エラー発生時:
> 1. `TeamDelete("{既存チーム名}")` を先に実行する
> 2. TeamDelete 成功後、下記 TeamCreate を進行する

### Phase 2 開始: 事前ロード (TeamCreate 前)

リーダーは TeamCreate 前にチームメンバーへ伝達するコンテキストを事前ロードする。

**Step 2-0a: リサーチファイルのロード**
- R7_CONTENT = Read("_workspace/00_research/R7_専門知識_{N}〜{M}話.md") または "(R7 リサーチ未完了)"
- R8_CONTENT = Read("_workspace/00_research/R8_事件詳細_{N}〜{M}話.md") または "(R8 リサーチ未完了)"

**Step 2-0b: 大きな設計文書のアークセクション抽出**
- BOOTSTRAP_EXCERPT = Phase 1 で抽出したアーク関連セクション（核心力量モジュール、世界観ルール、読者保持転換 — プラットフォーム依存で課金転換または離脱防止転換／書籍化アピール区間）
- CHARACTER_EXCERPT = キャラクターシートから当該アーク登場キャラクター、敵対者、VIP を抽出
- PLOT_EXCERPT = プロットフックガイドから当該アーク概要、カタルシスリズムを抽出

リーダーは TeamCreate 前に `_workspace/00_research/` 内のリサーチ結果ファイルの存在有無を Glob で確認する。存在するファイルのみプロンプトに含める。

```
TeamCreate(
  team_name: "design-small-team",
  members: [
    {
      name: "character-architect",
      agent_type: "general-purpose",
      prompt: "あなたは character-architect エージェントです。
        ${CLAUDE_PLUGIN_ROOT}/agents/character-architect.md を読み役割を熟知してください。
        ${CLAUDE_PLUGIN_ROOT}/skills/character/SKILL.md を読みモード B（小さな設計）の手順と出力テンプレートに従ってください。
        ${CLAUDE_PLUGIN_ROOT}/skills/design/references/genre-dna-framework.md を読みジャンル DNA フレームワークを熟知してください。

        ★ 大きな設計の要約 (Read 不要 — リーダーが事前ロード):
        --- ブートストラップ要約 ---
        {BOOTSTRAP_EXCERPT}
        --- キャラクターシート要約 ---
        {CHARACTER_EXCERPT}
        --- プロットフックガイド要約 ---
        {PLOT_EXCERPT}

        ★ 自動リサーチ結果 (Read 不要 — リーダーが事前ロード):
        --- R7 専門知識 ---
        {R7_CONTENT}
        --- R8 事件詳細 ---
        {R8_CONTENT}

        (要約が不十分なら大きな設計文書の原本を Read できるが、権限承認が必要となる場合がある)

        対象話数区間: {N}〜{M}話
        詳細キャラクターシートを作成してください。

        ★ 集団キャラクター名付与ルール:
        - R1/R2/R3/R4、インターン、看護師長など職位キャラクターのうち、
          当該話数区間でセリフが 2 回以上ある人物には必ず日本語名を付与せよ。
        - 「氏名」列には実際の名前を、「関係」列には職位（R3 レジデントなど）を記入せよ。
        - plot-hook-engineer がエピソード別ビートで職位コードではなく名前を使用できるようにする。

        出力: _workspace/04_character-architect_detail.md
        完了後リーダーへ SendMessage で完了を報告してください:
        '詳細キャラクターシート作成完了。_workspace/04_character-architect_detail.md に保存。'
        (リーダーが plot-hook-engineer にハンドオフします。PHE に直接 SendMessage しないでください。)"
    },
    {
      name: "plot-hook-engineer",
      agent_type: "general-purpose",
      prompt: "あなたは plot-hook-engineer エージェントです。
        ${CLAUDE_PLUGIN_ROOT}/agents/plot-hook-engineer.md を読み役割を熟知してください。
        ${CLAUDE_PLUGIN_ROOT}/skills/plot-hook/SKILL.md を読みモード B（小さな設計）の手順と出力テンプレートに従ってください。
        ${CLAUDE_PLUGIN_ROOT}/skills/design/references/genre-dna-framework.md を読みジャンル DNA フレームワークを熟知してください。

        ★ 大きな設計の要約 (Read 不要 — リーダーが事前ロード):
        --- ブートストラップ要約 ---
        {BOOTSTRAP_EXCERPT}
        --- キャラクターシート要約 ---
        {CHARACTER_EXCERPT}
        --- プロットフックガイド要約 ---
        {PLOT_EXCERPT}

        リーダーから SendMessage を受信したら直ちに作業を開始してください。別途確認や待機なしに即座に開始します。
        _workspace/04_character-architect_detail.md を Read して詳細キャラクターシートを熟知した上で、
        詳細プロット/フックガイドを作成してください。

        ★ 自動リサーチ結果 (Read 不要 — リーダーが事前ロード):
        --- R7 専門知識 ---
        {R7_CONTENT}
        --- R8 事件詳細 ---
        {R8_CONTENT}

        (要約が不十分なら大きな設計文書の原本を Read できるが、権限承認が必要となる場合がある)

        対象話数区間: {N}〜{M}話

        ★ 名前使用ルール:
        エピソード別ビート・クリフハンガーに登場する人物は職位コード（R1、R3 など）ではなく、
        詳細キャラクターシートに定義された実際の名前で表記せよ。

        出力: _workspace/05_plot-hook-engineer_detail.md"
    }
  ]
)
```

タスク登録:
```
TaskCreate(tasks: [
  { title: "詳細キャラクターシート作成 ({N}〜{M}話)", assignee: "character-architect" },
  { title: "詳細プロット/フックガイド作成 ({N}〜{M}話)", assignee: "plot-hook-engineer",
    depends_on: ["詳細キャラクターシート作成 ({N}〜{M}話)"] }
])
```

### Phase 3: 小さな設計の実行

**実行方式:** リーダー仲介パイプライン

**Step 3-1**: character-architect の完了待機
- `_workspace/04_character-architect_detail.md` の存在有無を Bash で確認
- 未完了なら 60 秒待機後に再確認（最大 5 回）

**Step 3-2**: リーダーのハンドオフ
character-architect 完了確認後ただちに:
1. リーダーが `_workspace/04_character-architect_detail.md` を Read
2. 核心内容の要約を抽出:
   - 主人公の感情曲線要約
   - 新規キャラクターリストと登場話数
   - 敵対者活動タイムライン
3. リーダーが plot-hook-engineer に直接 SendMessage:
   "character-architect 作業完了。
   詳細キャラクターシート: _workspace/04_character-architect_detail.md
   [核心要約添付]
   ただちに詳細プロット/フックガイドの作成を開始してください。
   出力: _workspace/05_plot-hook-engineer_detail.md"

**Step 3-3**: plot-hook-engineer の完了待機
- `_workspace/05_plot-hook-engineer_detail.md` の存在有無を Bash で確認（最大 5 回、60 秒間隔）
- 2 回確認しても未生成ならリーダーが再度 SendMessage で作業開始を要請
- 4 回確認しても未生成ならエラーハンドリング（PHE 失敗とみなす）

**小さな設計中に矛盾を発見した場合:**
- concept-builder は参加しないため、ブートストラップの修正が必要ならリーダーが直接 `{作品仮題}_ブートストラップ.md` を修正
- 修正履歴を `_workspace/06_bootstrap_amendments.md` に記録
- ブートストラップは依然として source of truth

**成果物保存:**

| チームメンバー | 出力パス |
|------|----------|
| character-architect | `_workspace/04_character-architect_detail.md` |
| plot-hook-engineer | `_workspace/05_plot-hook-engineer_detail.md` |

### Phase 4: 統合検証および整理

1. 各チームメンバーの成果物を Read で収集
2. **一貫性検証チェックリスト:**
   - [ ] 詳細キャラクターの登場話数が詳細プロットのエピソード配置と一致するか
   - [ ] 主人公の感情曲線が単調でないか
   - [ ] 新規キャラクターが 3 名以内か（序盤の過剰登場防止）
   - [ ] VIP 遭遇イベントが 10〜15 話間隔で配置されているか
   - [ ] 敵対者の活動がカタルシスリズムと連動するか
   - [ ] 専門知識活用シーンがリサーチ資料の実際の技術/事例に基づくか
   - [ ] 核心力量モジュール活用シーンの日付/人物/結果が事件詳細資料と一致するか
   - 矛盾発見時: 大きな設計文書を基準に修正

3. 最終成果物を `{DESIGN_DIR}` にコピー（novel-config.md のパスと一致させる）:

| 中間成果物 | 最終パス |
|-----------|----------|
| `_workspace/04_*_detail.md` | `{DESIGN_DIR}/{作品仮題}_詳細キャラクターシート_{N}〜{M}話.md` |
| `_workspace/05_*_detail.md` | `{DESIGN_DIR}/{作品仮題}_詳細プロットフックガイド_{N}〜{M}話.md` |

   > **パス一貫性原則**: novel-config.md の `design_dir` と実際のファイル位置を必ず一致させる。

4. **novel-config.md 自動アップデート** (小さな設計の成果物をダウンストリームスキルに連結):

   novel-config.md を Read した後、下記フィールドを自動で追加/更新する:

   ```
   アップデート項目:
   a) EP 範囲別の設定文書テーブルの該当行に詳細文書のパスを追加:
      - 詳細プロットガイド列: {DESIGN_DIR}/{作品仮題}_詳細プロットフックガイド_{N}〜{M}話.md
      - 詳細キャラクターシート列: {DESIGN_DIR}/{作品仮題}_詳細キャラクターシート_{N}〜{M}話.md
      (既存行の EP 範囲が一致すれば該当列のみ更新、範囲がなければ新規行追加)
   b) 共通文書の character_detail は変更しない（大きな設計のキャラクターシートを維持）。
      EP 範囲別の詳細キャラクターシートがあれば create/polish が ep_range_table から優先参照する。
   c) R7/R8 リサーチ結果参照パス（補助参照セクション — EP 範囲別配列）:
      既存の research_r7/r8 項目があれば**配列に追加**（上書きしない）:
      - research_r7:
        - { range: "EP{N}〜EP{M}", path: "_workspace/00_research/R7_専門知識_{N}〜{M}話.md" }
      - research_r8:
        - { range: "EP{N}〜EP{M}", path: "_workspace/00_research/R8_事件詳細_{N}〜{M}話.md" }
      同一 EP 範囲の既存項目があれば該当項目のみ上書きする。
   ```

   novel-config.md を Read した後ただちにアップデートを適用する（非対話型）。
   アップデート完了後、変更履歴をユーザーに出力する（事前確認要請なし）。

5. **チームメンバー終了要請**
   - SendMessage(to: "character-architect", message: "作業完了。終了してください。")
   - SendMessage(to: "plot-hook-engineer", message: "作業完了。終了してください。")
   - 5 秒待機（チームメンバー終了時間の確保）

6. **TeamDelete("design-small-team")** — チーム即時解散
   ⚠️ チームメンバー終了 SendMessage 後ただちに実行する。ユーザー応答待機なし。
   TeamDelete 失敗時:
   - 10 秒待機後 1 回再試行
   - 再失敗時はエラーを無視し次ステップへ進行
     (次回 TeamCreate 時に Phase 2 安全点検で既存チームを削除)

7. `_workspace/` ディレクトリの保存（事後検証用）

8. ユーザーへ結果要約 + 次の 25 話区間の小さな設計を案内:
   ```
   ## 小さな設計完了 ({N}〜{M}話)

   成果物:
   - {DESIGN_DIR}/{作品仮題}_詳細キャラクターシート_{N}〜{M}話.md
   - {DESIGN_DIR}/{作品仮題}_詳細プロットフックガイド_{N}〜{M}話.md

   次の区間の小さな設計を進めますか？
   → 次の範囲: {M+1}〜{M+25}話
   「小さな設計 {M+1}〜{M+25}話」とおっしゃればすぐに開始します。
   ```

## R7/R8 リサーチ結果の活用経路

domain-researcher が生成する R7（専門知識）/R8（事件詳細）リサーチ結果は**詳細設計文書に溶け込む方式**で活用される。
ダウンストリームスキル（create/polish/rewrite）から直接参照することはない。

```
活用フロー:
R7 → character-architect の詳細キャラクターシートに反映 (専門知識活用シーンのディテール)
R7 → plot-hook-engineer の詳細プロットに反映 (技術的ディテールのエピソード配置)
R8 → character-architect の敵対者活動に反映 (実際の事件に基づくタイムライン)
R8 → plot-hook-engineer のエピソード別事件に反映 (正確な日付/前兆シグナル)

最終成果物 (詳細キャラクターシート、詳細プロットフックガイド) にリサーチが内在化されるため、
create/polish/rewrite は成果物を読むだけでリサーチ内容が自動で反映される。

参考パス (novel-config.md 補助参照):
- research_r7: _workspace/00_research/R7_専門知識_{N}〜{M}話.md
- research_r8: _workspace/00_research/R8_事件詳細_{N}〜{M}話.md
→ 必要に応じて手動照会可能
```

## エラーハンドリング

| 状況 | 戦略 |
|------|------|
| 大きな設計文書 3 種のうち一部欠落 | Phase 1 で検知。欠落文書なしには小さな設計不可 → ユーザーに大きな設計の先行完了を案内 |
| 大きな設計文書のフォーマットが異なる | Phase 1 で核心セクション（アーク構造、核心力量モジュール、敵対者階層）の存在有無を確認。核心セクション不在時は補完を要請 |
| 要請話数区間がアーク構造と不一致 | アーク境界をユーザーに案内し区間調整を提案 |
| domain-researcher 失敗 | 1 回再試行。再失敗時は大きな設計文書 + genre-dna に基づいて進行し、レポートに「自動リサーチ未反映」を明記 |
| character-architect 失敗 | 1 回再試行。再失敗時はリーダーが大きな設計のキャラクターシートに基づき基本詳細シートを生成 |
| plot-hook-engineer 失敗 | 1 回再試行。再失敗時は大きな設計プロットガイド + 詳細キャラクターシートで基本詳細プロットを生成 |
| チームメンバー間の通信遅延 | リーダーが中間でファイルを Read し手動で情報伝達 |
| リサーチファイル Read 失敗 | リーダーが「(リサーチ未完了)」テキストをプロンプトに埋め込む。チームメンバーは大きな設計の要約に基づき進行 |

## データフロー

```
[ユーザー] → 小さな設計要請 (話数区間)
    ↓
Phase 1: 範囲確認 + 大きな設計文書の点検
    ↓
Phase 1.5: domain-researcher サブエージェント → _workspace/00_research/ (R7、R8)
    ↓ (ユーザー待機なし)
Phase 2: TeamCreate("design-small-team") — リサーチ結果を含む
    ↓
Phase 3: character-architect → plot-hook-engineer
    ↓
Phase 4: 統合検証 → 成果物 2 種
    ↓
次のアーク案内
```

## テストシナリオ

### 正常フロー
1. ユーザーが「1〜50 話の小さな設計をしてくれ」と要請
2. Phase 1 で大きな設計文書 3 種を確認、アーク概要を抽出
3. Phase 1.5 で domain-researcher が R7（専門知識）+ R8（事件詳細）を自動リサーチ
4. Phase 2 でチーム編成（リサーチ結果を含む）
5. Phase 3 で CA→PHE の順序で小さな設計を完成
6. Phase 4 で検証後、成果物 2 種 + 次のアーク案内

### エラーフロー
1. Phase 1 で `{作品仮題}_プロットフックガイド.md` の欠落を検知
2. ユーザーに「プロットフックガイドがありません。大きな設計（design-big）を先に完了してください。」と案内
3. 小さな設計を中断
