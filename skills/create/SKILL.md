---
name: create
description: "Web 小説エピソード創作オーケストレーター。設定文書（キャラクターシート・プロットガイド・ブートストラップ）に従ってエピソードを順次創作する。'/create'、'/create {プロジェクト名}'、'/create EP051'、'/create {プロジェクト名} EP001-EP010'、'エピソード創作'、'エピソード執筆'、'エピソード作成'、'新規エピソード'、'本文作成'、'エピソードを書く' で実行。novel-config.md の設定文書マッピングとガードレールを適用し、キャラクターの立体性、数値の一貫性、フックの強度、蓋然性を保証する。"
---

# エピソード創作オーケストレーター

設定文書（キャラクターシート・プロットガイド・ブートストラップ）に従ってエピソードを順次創作する 4 エージェントパイプライン。

## アーキテクチャ

```
Phase 1 (並列)              Phase 2 (順次)          Phase 3 (順次)
┌──────────────────┐       ┌──────────────┐       ┌──────────────┐
│ episode-architect│──┐    │  episode-    │       │  quality-    │
│ (設計図抽出)       │  ├──▶│  creator     │──────▶│  verifier    │
└──────────────────┘  │    │  (本文執筆)    │       │  (7 軸検証)  │
┌──────────────────┐  │    └──────────────┘       └──────┬───────┘
│ continuity-bridge│──┘          ◀── REWRITE (max 2) ───┘
│ (連続性収集)       │                     │
└──────────────────┘             PASS ───▶ 次の EP
```

**実行モード**: サブエージェント（パイプラインパターン、lint/revise と同一）
- 理由: 各 Phase が前の Phase 出力に順次依存し、エージェント間のリアルタイム通信が不要

## 前提条件

1. **novel-config.md** が存在する（プロジェクトディレクトリ内）
2. **設定文書** が存在する: ブートストラップ、キャラクターシート（core+detail+dialogue DNA）、プロットガイド
3. **エピソードディレクトリ** が存在する（novel-config.md 内の `episode_dir`）

novel-config.md がなければエラーを出力して終了する。

## 引数パース

| 入力形式 | 解釈 |
|----------|------|
| `/create` | 現在のディレクトリで novel-config.md を探索し、最終 EP の次から |
| `/create 36億坪` | 36億坪プロジェクト、最終 EP の次から |
| `/create EP051` | 現在のプロジェクト、EP051 のみ |
| `/create 36億坪 EP051` | 36億坪プロジェクト、EP051 のみ |
| `/create 36億坪 EP051-EP060` | 36億坪プロジェクト、EP051～EP060 の範囲 |

## 実行フロー

### Step 0: 初期化

0. **novel-config.md 必須フィールド検証ゲート**:
   novel-config.md をロードした直後、以下の必須フィールドが全て存在し空でないことを検証する。
   1 つでも欠落していれば **エラーを出力して即時終了する**。

   ```
   必須フィールドチェックリスト:
   - [ ] project.target_platform — プラットフォーム名
   - [ ] project.episode_dir — エピソード保存ディレクトリ
   - [ ] project.work_dir — 作業ディレクトリ
   - [ ] project.design_dir — 設計文書ディレクトリ
   - [ ] 設定文書マッピング.bootstrap — ブートストラップパス（ファイル存在確認）
   - [ ] 設定文書マッピング.character_core — キャラクター中核パス（ファイル存在確認）
   - [ ] 設定文書マッピング.character_detail — キャラクター詳細パス（ファイル存在確認）
   - [ ] EP 範囲別プロットガイド — 最低 1 行存在（ファイル存在確認）
   ```

   検証失敗時の出力:
   ```
   ❌ novel-config.md 必須フィールドが欠落:
   - {欠落フィールド一覧}

   create スキルを実行するには上記フィールドを記入してください。
   テンプレート: ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md
   ```

0.5 **target_platform 許容集合検証ゲート**:
   - novel-config.md の `project.target_platform` は以下の canonical name のいずれかでなければならない:
     - カクヨム、小説家になろう、アルファポリス、ノベルアップ+、エブリスタ、ノベルピア
   - create スキルは novel-config.md に保存された canonical 値のみを使用する。別名の正規化や自動置換は行わない。
   - 非対応値の場合はエラーを出力して即時終了する。

   ```
   ❌ 許可されない target_platform: {現在値}

   許可プラットフォーム:
   - カクヨム
   - 小説家になろう
   - アルファポリス
   - ノベルアップ+
   - エブリスタ
   - ノベルピア

   novel-config.md の project.target_platform を許可プラットフォーム名に修正してから再実行してください。
   ```

1. novel-config.md をパースする:
   ```
   {CONFIG}       ← novel-config.md 全体
   {TARGET_PLATFORM} ← config.target_platform
   {DESIGN_DIR}   ← config.design_dir
   {EPISODE_DIR}  ← config.episode_dir
   {WORK_DIR}     ← config.work_dir
   {GUARD_RAILS}  ← config.guard_rails + config.create_guard_rails (あれば)
   {CUSTOM_AXES}  ← config.custom_axes (あれば)
   {CREATE_CFG}   ← config.create 設定 (あれば、なければデフォルト値)
   ```

2. デフォルト値（`[create]` セクションがない場合）:
   ```
   draft_chars: 6000-10000           # Phase 2 初稿目標（多めに作成）
   final_chars: 5000-8000            # Step 2.5 トリミング後の最終目標
   dialogue_ratio: 40-60%
   max_scenes: 4
   hook_targets:
     opening_intensity: 4
     ending_intensity: 5
   continuity_lookback: 2
   ```

   > **分量戦略: 「あふれるほど書いて削る」**
   > episode-creator は `draft_chars`（6000-10000 字）を目標に初稿を書く。
   > Step 2.5 でオーケストレーターが `final_chars`（5000-8000 字）にトリミングする。
   > 理由: 不足して肉付けすると品質が下がる。十分に書いて不要部分を削除してこそ密度が高まる。

3. EP 範囲決定:
   - 指定 EP があればその範囲を使用する
   - なければ `{EPISODE_DIR}` から最終 EP 番号を探し、次の番号から開始する
   - 終了 EP を指定しなければ **1 話のみ** 創作する（無限ループ防止）

4. EP 範囲に応じた設定文書マッピング:
   - novel-config.md の `ep_range_table` から該当 EP が属する act を確認する
   - **EP 範囲の重複検出**: ep_range_table の全行を走査し、範囲が重なる行があるかを確認する。重複を発見した場合は警告を出力しユーザに確認する
   - 該当 act のプロットガイドを決定する:
     - 詳細プロットガイド列にパスがありファイルが存在すれば → `{PLOT_DOC}` = 詳細プロットガイド
     - なければ → `{PLOT_DOC}` = 大設計プロットガイド
   - 該当 act のキャラクターシート (detail) を決定する:
     - 詳細キャラクターシート列にパスがありファイルが存在すれば → `{CHAR_DETAIL_EP}` = 詳細キャラクターシート
     - なければ → `{CHAR_DETAIL_EP}` = 共通 character_detail
   - **詳細設計文書の有無確認**: 該当 EP 範囲に詳細プロットガイド（小設計成果物）がなければ警告を出力する:
     ```
     ⚠️ EP{NNN} が属する範囲に詳細プロットガイドがありません。
     大設計プロットガイドのみで EP 別ビートを抽出するため、設計図の品質が低下する可能性があります。
     小設計 (design-small) を先に実行することを推奨します。
     それでも進行しますか？
     ```

5. `{WORK_DIR}/_workspace/` ディレクトリを確認する（なければ作成）

6. `{WORK_DIR}/create-plan.md` を生成または更新する:
   ```markdown
   # エピソード創作計画
   ## 対象: EP{start}-EP{end}
   - [ ] EP{NNN} — 未着手
   ...
   ```

### Step 1: Phase 1 — エピソード設計（並列）

2 つのエージェントを **並列で**（1 メッセージで Agent を 2 回呼び出し）実行する。`run_in_background: true` でバックグラウンド実行する。

> ⚠️ **Phase 1 待機方法 — 絶対禁止／必須遵守**:
> - **禁止**: TaskOutput の使用（チーム名前空間衝突により "No task found" エラーが発生）
> - **禁止**: Bash sleep ポーリング（`sleep N && ls` の繰り返し禁止 — 不要な待機 + 承認要求を誘発）
> - **必須**: バックグラウンドエージェントは完了時にシステムが **自動で通知する**。両エージェントが完了通知を受け取った後に Phase 2 へ進む。

> ⚠️ **エージェント共通パス制限指示**（episode-architect、continuity-bridge のプロンプトに必ず含める）:
> 「以下に明示されたファイルパスのみを読め。design/ ディレクトリを Glob で探索したり、
> 明示されていないファイルを自ら探して読もうと試みるな。
> 必要な全ファイルパスはこのプロンプトに含まれている。」

**Agent 1: episode-architect**
- subagent_type: `general-purpose`
- `run_in_background: true`
- プロンプトに含める情報:
  - エージェント定義ファイルパス → 読むよう指示
  - EP 番号
  - 読むべき設定文書のパス（novel-config.md に定義されたパスを使用）:
    - ブートストラップ: `{CONFIG.bootstrap}` (novel-config.md の bootstrap パス)
    - プロットガイド: `{PLOT_DOC}` (Step 0.4 で ep_range_table から決定されたパス)
    - キャラクターシート core: `{CONFIG.character_core}` (novel-config.md の character_core パス)
    - キャラクターシート detail: `{CHAR_DETAIL_EP}` (ep_range_table 詳細キャラクターシートを優先、なければ共通 character_detail)
  - novel-config.md のガードレール、カスタム軸
  - **目標分量の明示**（必ず含める）:
    ```
    blueprint ヘッダの「目標分量」項目には必ず以下の値を使用する:
    - 初稿目標 (draft_chars): {CREATE_CFG.draft_chars} 字
    プラットフォームの常識（カクヨム 3,000～4,000 字など）に基づく独自推論を絶対に使用しないこと。
    novel-config.md の [create] セクション値がない場合はデフォルト値 8,000～10,000 字を使用する。
    ```
  - パス制限指示（上記共通指示を含める）
  - **許可パスホワイトリスト**（プロンプトに必ず含める）:
    ```
    この作業で Read できるファイルは以下のリストが全てだ。
    リストにないファイルはいかなる理由でも Read するな:
    0. ${CLAUDE_PLUGIN_ROOT}/agents/episode-architect.md (自身のエージェント定義)
    1. {CONFIG.bootstrap}
    2. {PLOT_DOC}
    3. {CONFIG.character_core}
    4. {CHAR_DETAIL_EP}
    5. novel-config.md
    上記 6 ファイル以外の Read 試行はパス制限違反だ。
    ```
  - 出力パス: `{WORK_DIR}/_workspace/01_episode-architect_blueprint_EP{NNN}.md`

**Agent 2: continuity-bridge**
- subagent_type: `general-purpose`
- `run_in_background: true`
- プロンプトに含める情報:
  - エージェント定義ファイルパス → 読むよう指示
  - EP 番号
  - 読むべき文書のパス:
    - 直前 N 話: `{EPISODE_DIR}/ep{N-2}.md`、`{EPISODE_DIR}/ep{N-1}.md`
    - alive-tracker: `{WORK_DIR}/alive-tracker.md` (存在時)
    - delta-tracker: `{DESIGN_DIR}/delta_tracker_*.md` (存在時)
    - verification: `{DESIGN_DIR}/verification_*.md` (存在時)
  - EP001 創作時: 直前エピソードの代わりにブートストラップ初期状態を要約するよう指示
  - パス制限指示（上記共通指示を含める）
  - 出力パス: `{WORK_DIR}/_workspace/02_continuity-bridge_report_EP{NNN}.md`

### Step 2: Phase 2 — エピソード執筆（順次）

Phase 1 完了後に episode-creator を実行する。

**Agent 3: episode-creator**
- subagent_type: `general-purpose`
- プロンプトに含める情報:
  - エージェント定義ファイルパス → 読むよう指示
  - 読むべき文書:
    - `{WORK_DIR}/_workspace/01_episode-architect_blueprint_EP{NNN}.md`
    - `{WORK_DIR}/_workspace/02_continuity-bridge_report_EP{NNN}.md`
    - キャラクターシート core + detail + dialogue DNA
    - novel-config.md のガードレール
  - `{CREATE_CFG}` 設定（目標文字数、対話比率など）
  - 出力パス: `{EPISODE_DIR}/ep{NNN}.md`
  - 詳細な執筆原則が必要なら `references/creation-principles.md` を読むよう指示
  - **Bash 禁止指示**: 「執筆完了後に Bash/python3 スクリプトを実行するな。文字数・対話比率・場面数の集計は quality-verifier が行う。ファイル保存後にセルフチェック（あった／していた／のだった の Grep カウント）を行ってから終了せよ。」
  - パス制限指示（Step 1 の共通指示を含める）

**再執筆モード**（REWRITE 後の再実行時）:
- 追加プロンプト:
  - `{WORK_DIR}/_workspace/04_quality-verifier_verdict_EP{NNN}.md` を読むよう指示
  - 既存の初稿 `{EPISODE_DIR}/ep{NNN}.md` を読むよう指示
  - 「修正指示に従って該当部分のみを修正せよ。全体再執筆禁止。」

### Step 2.5: トリミングゲート（オーケストレーター直接実行）

Phase 2 完了後、Phase 3 に入る前にオーケストレーターが初稿をトリミングする。

**戦略: 「あふれるほど書いて削る」**
episode-creator は 6000-10000 字の初稿を作成する。オーケストレーターが 5000-8000 字にトリミングする。
不足して肉付けするより、十分に書いて削る方が品質が高い。

**測定基準**: `wc -m` でファイル全体の文字数を測定する（Markdown ヘッダ・メタデータを含む）。

1. **測定**: Bash `wc -m {EPISODE_DIR}/ep{NNN}.md` → 文字数を確認
2. **判定**: 最終目標範囲（デフォルト final_chars: 5000-8000）と照合する

   - **上限超過**（8000 字超 — 一般的なケース）:
     - episode-creator をトリミングモードで再呼び出しする
     - プロンプトに以下を含める:
       ```
       トリミングモード:
       - 現在文字数: {N} 字 (wc -m 基準、ヘッダ含む)
       - 目標範囲: {FINAL_MIN}～{FINAL_MAX} 字
       - 超過分: {N - FINAL_MAX} 字
       - 指示: 以下の削除優先順位に従って不要部分を除去し {FINAL_MAX} 字以内にせよ。
         削除優先順位:
         1 位: 語り手の状況整理／要約文（メタ叙述）
         2 位: 同じ感情を別の比喩で繰り返した文
         3 位: プロットビートに寄与しない背景描写
         4 位: 既に行動で示した感情を内面独白で再確認する文
         5 位: 過剰な感覚描写（核心場面でない転換区間）
         注意: プロットビート、台詞、クリフハンガー、核心感覚描写は保存。エンディングは台詞／行動で維持。
       ```
     - トリミング後に再度文字数を測定する。1 回までリトライ、その後 Phase 3 へ進行

   - **範囲内**（5000-8000 字）: Phase 3 へ進行

   - **下限未満**（5000 字未満 — 初稿が不足する異常ケース）:
     - episode-creator を即座に再呼び出しする（Phase 3 進入前の補正）
     - プロンプトに以下を含める:
       ```
       文字数不足補正モード:
       - 現在文字数: {N} 字
       - 目標下限: {FINAL_MIN} 字
       - 指示: 既存場面を拡張するか、不足する描写／対話を補強し {FINAL_MIN} 字以上にせよ。
               新規プロットビートを追加せず、既存場面の密度を高めよ。
       ```
     - 補正後に再度文字数を測定する。2 回まで試行、その後 Phase 3 へ進行

### Step 3: Phase 3 — 品質検証（順次）

**Agent 4: quality-verifier**（CREATE モード）
- subagent_type: `general-purpose`
- プロンプトに含める情報:
  - エージェント定義ファイルパス → 読むよう指示
  - CREATE モードで動作するよう指示（創作品質検証 — 以下の 8 軸を適用）:
    ```
    CREATE モード検証 8 軸（構造的整合性中心 — polish の文体・表現レベル校正と区別）:
    1. PLOT_BEAT — 設計図のプロットビートがエピソードに全て反映されているか
    2. TIMELINE — 時間マーカー・日付・年齢が設定文書および直前エピソードと一致するか
    3. NUMBER — 核心数値（面積・資金・収穫量など）が設定文書と一致するか
    4. GUARDRAIL — novel-config.md の保存ガードレールに違反していないか
    5. CONTINUITY — continuity-bridge レポートの未解決伏線／関係状態が反映されているか
    6. HOOK — オープニング／エンディングのフック強度が設定された目標値以上か
    7. CHAR_VOICE — 登場人物の台詞がキャラクターボイステーブル／対話 DNA と一貫しているか
    8. CUSTOM — novel-config.md のカスタム軸違反がないか
    ```
    > **polish との関係**: create の quality-verifier は構造的整合性（プロット・数値・ガードレール）を検証する。
    > polish は文章品質（禁止表現・沈黙パターン・翻訳調・モバイル可読性・キャラクター生動感）を校正する。
    > したがって create で PASS したエピソードも polish を経て初めて最終品質に到達する。
  - **再検証指示**（REWRITE 後の再実行時に必ず含める）:
    「既存の verdict ファイルがあっても絶対に読むな。エピソード原文ファイルを最初に読み、
     全検証軸を最初から実行せよ。結果を verdict ファイルに新規上書きせよ。」
  - パス制限指示（Step 1 の共通指示を含める）
  - 読むべき文書:
    - `{EPISODE_DIR}/ep{NNN}.md`（創作されたエピソード）
    - `{WORK_DIR}/_workspace/01_episode-architect_blueprint_EP{NNN}.md`
    - `{WORK_DIR}/_workspace/02_continuity-bridge_report_EP{NNN}.md`
    - 設定文書（ブートストラップ、キャラクターシート、プロットガイド）
    - novel-config.md（ガードレール、数値検証優先順位、カスタム軸）
  - 出力パス: `{WORK_DIR}/_workspace/04_quality-verifier_verdict_EP{NNN}.md`

### Step 4: 判定分岐

verdict ファイルを読み判定を確認する。

**PASS 判定時 — オーケストレーター交差検証（必須）:**
> 二重安全装置: quality-verifier が直接 Grep カウントを行ったとしても、
> オーケストレーターは独立して再確認する。両レイヤーが同一結果を出してこそ PASS が確定する。

verdict が PASS の場合、オーケストレーターが核心ガードレールを直接交差検証する:
1. Grep `あった` in `{EPISODE_DIR}/ep{NNN}.md` → 5 回以下を確認
2. Grep `していた` in `{EPISODE_DIR}/ep{NNN}.md` → 3 回以下を確認
3. Bash: `wc -m {EPISODE_DIR}/ep{NNN}.md` → 目標文字数範囲（デフォルト final_chars: 5000-8000）を確認

3 つのうち 1 つでも違反した場合、違反種別に応じて分岐する:

**経路 A — オーケストレーター直接 Edit（MINOR 違反）**:
以下の条件を **全て** 満たす場合、オーケストレーターが直接修正する:
- 文字数違反ではない（文字数の不足／超過は経路 B）
- 違反項目の **超過分が 3 件以下**（例: 'あった' 7 回 → 2 件超過）
- 違反が **単一表現のテキスト置換** で解消可能

直接 Edit 手順:
1. Grep で違反表現の全出現位置を把握する（output_mode: "content"、-n: true）
2. 超過分に該当する出現を選択する（エピソード後半部 → 前半部の順で選択）
3. 各出現に対して **前後 2 文を Read で確認** して文脈を把握した後に置換する
4. 置換戦略（保守的 — 時制保存優先）:
   - 'あった' → '～だった'（過去維持、最も安全）または '～である'（現在文脈の場合のみ）
   - 'していた' → '～した'（単純過去転換、時制エラー最小）
   - 'のだった' → '～だった'、'～というわけだった'
   - **時制を過去→現在に変更する必要がある場合は経路 B に分類** する（文脈依存的修正）
5. 修正後に再度 Grep でカウントが上限以内かを確認する
6. 確認完了時は PASS 経路を継続する（verdict ファイルに「[オーケストレーター直接修正: {項目} {元値}→{修正値}]」を追記）

**経路 B — REWRITE 手順（MAJOR 違反）**:
以下のいずれかに該当する場合は既存の REWRITE 手順に従う:
- 文字数違反（構造的修正が必要）
- 超過分が 4 件以上（広範囲修正が必要）
- 時制変更が必要な置換（文脈依存的修正）
- 単純置換で解消不能（文脈依存的修正が必要）

REWRITE 手順:
- 「⚠️ quality-verifier の PASS 判定が原文と不一致。オーケストレーターが REWRITE で再判定します。」
- 既存の REWRITE 手順（Step 4 後半部）で進行する

交差検証通過時:
1. `create-plan.md` で該当 EP を `[x]` に更新し要約を追加する
2. 次の EP へ移動する（Step 1 から）

**REWRITE:**
1. 再試行回数を確認する（該当 EP の累積 REWRITE 回数）
2. **既存の verdict ファイルを削除する**: Bash `rm -f {WORK_DIR}/_workspace/04_quality-verifier_verdict_EP{NNN}.md`
   > verdict ファイルが残っていると quality-verifier が再検証なしに前回結果を再利用する。必ず削除する。
3. **2 回未満**: Step 2 へ戻る（再執筆モード、修正指示を含める）
4. **2 回以上**: ユーザに通知し該当 EP を `[△]`（部分通過）として表示、次の EP へ移動

### Step 5: 自己ループ

対象範囲の全 EP が処理されるまで Step 1～4 を繰り返す。

完了時:
1. `create-plan.md` を最終更新する
2. **`.design-hashes` ベースライン生成**: `{WORK_DIR}/.design-hashes` ファイルを生成または更新する。
   設定文書（ブートストラップ、キャラクターシート、プロットガイドなど `{DESIGN_DIR}/` 内の全設定文書）の SHA256 ハッシュを記録する。
   このファイルは rewrite スキルが設定文書変更を検知するベースラインとして使用する。
   ```
   .design-hashes 形式:
   {ファイルパス}\t{SHA256ハッシュ}\t{生成時刻}
   ```
   > **目的**: rewrite 初回実行時に「全設定文書が変更された」と誤検知することを防止する。
   > create が完了した時点の設定文書状態がベースラインとなる。
3. 結果要約出力:
   ```
   ## 創作完了要約
   - 総エピソード: N 話
   - PASS: X 話
   - 部分通過 (△): Y 話
   - REWRITE 発生: Z 回

   ### 次のステップ
   - 文章品質向上: `/polish` — 12 軸診断で文体・フック・数値・キャラクター生動感を校正
   - 設定変更反映: `/rewrite` — 設定文書変更時に影響を受けるエピソードを再執筆
   - ⚠️ 設定を変更する予定があれば `/rewrite` → `/polish` の順で実行してください
   ```

## エラーハンドリング

| エラー種別 | 戦略 |
|----------|------|
| エージェント失敗 | 1 回再試行 → 失敗時は該当 EP を `[!]` で表示し次へ |
| 設定文書欠落 | ユーザに警告後、利用可能な文書で進行。プロットガイド欠落時は終了 |
| 直前 EP なし (EP001) | continuity-bridge がブートストラップ初期状態を要約 |
| novel-config.md 欠落 | エラー出力後に終了 |
| ガードレール衝突 | episode-creator が `[ガードレール迂回]` タグ → auditor が該当部分を別途評価 |
| Phase 1 片方のみ失敗 | 成功したレポートで Phase 2 進行、欠落を明示 |

## novel-config.md 拡張 — [create] セクション

novel-config.md に以下のセクションを追加すれば創作設定をカスタマイズできる。なければデフォルト値を使用する。

```markdown
## create 設定
draft_chars: 6000-10000          # Phase 2 初稿目標（多めに作成）
final_chars: 5000-8000           # Step 2.5 トリミング後の最終目標
dialogue_ratio: 40-60%           # 対話比率（デフォルト: 40-60%）
max_scenes: 4                    # 最大場面数（デフォルト: 4）
hook_targets:
  opening_intensity: 4           # 1-5、オープニングフック強度（デフォルト: 4）
  ending_intensity: 5            # 1-5、エンディングクリフハンガー強度（デフォルト: 5）
continuity_lookback: 2           # 直前参照話数（デフォルト: 2）
point_scenes_per_ep: 2-3         # ポイント場面数（デフォルト: 2-3）
dead_zone_threshold: 3500        # ポイント場面なしの最大許容文字数（デフォルト: 3500）

## create 事前予防 (shift-left)
# create は lint と異なり、問題を事後検出ではなく事前予防する。
# 以下項目を episode-architect の設計図に事前に含めて episode-creator が遵守するようにする:
timeline_lock: true              # エピソード別の絶対日付／年齢を事前確定（デフォルト: true）
voice_quickref: true             # 登場人物別ボイスクイックレフカードを事前抽出（デフォルト: true）
nonverbal_memory: true           # 直前 2 話の非言語反復メモリを収集（デフォルト: true）
echo_dialogue_filter: true       # こだま対話防止ルールを適用（デフォルト: true）
cider_gogumo_balance: true       # サイダー／ごくもバランス追跡（デフォルト: true）

## create ガードレール
# lint ガードレールに追加で適用する創作専用ガードレール
# 例:
# - 主人公が 1 話で専門分野外の知識を披露してはならない
# - タイムリープ／人生やり直しの事実を 3 幕以前に他者に露呈してはならない
```

## テストシナリオ

### 正常フロー
1. `/create 36億坪 EP051` を実行
2. novel-config.md パース → EP051 は Act2 範囲
3. Phase 1: episode-architect (Act2 プロットガイド + ブートストラップ + キャラクターシート) ‖ continuity-bridge (EP049、EP050) を並列
4. Phase 2: episode-creator が設計図 + 連続性 + キャラクター DNA で執筆 → `36億坪/episodes/ep051.md`
5. Phase 3: quality-verifier が 7 軸検証 → PASS
6. create-plan.md を更新 `[x] EP051`、EP052 へ移動

### エラーフロー — REWRITE
1. Phase 3 で REWRITE 判定（TIMELINE CRITICAL: 日付不一致）
2. verdict の修正指示を episode-creator に渡す
3. episode-creator が該当部分のみを修正
4. Phase 3 再実行 → PASS
5. create-plan.md 更新、次の EP へ移動

### エラーフロー — 最初のエピソード
1. `/create 36億坪 EP001` を実行
2. continuity-bridge: 直前エピソードなし → ブートストラップ初期状態を要約
3. 以降は正常フローと同一

### エラーフロー — 範囲創作
1. `/create 36億坪 EP051-EP055` を実行
2. EP051 PASS → EP052 REWRITE (1 次) → REWRITE (2 次) → 部分通過 (△) → EP053 PASS → ...
3. 最終要約: PASS 4 話、部分通過 1 話
