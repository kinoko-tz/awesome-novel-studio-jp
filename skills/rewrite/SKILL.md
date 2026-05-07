---
name: rewrite
description: "エピソード再執筆 (rewrite) スキル。設定文書 (キャラクターシート、プロットガイド、ブートストラップ) の変更に応じて既存エピソードを分析し再執筆する。'/rewrite'、'/revise'、'/rewrite {プロジェクト名}'、'/rewrite EP051'、'/rewrite {プロジェクト名} EP001-EP010' で実行。'エピソード再執筆'、'リバイズ'、'リライト'、'rewrite'、'revise' すべてトリガー。設定文書が変わってエピソードを書き直す必要があるとき、キャラクターの立体性が不足して大幅な修正が必要なときに使用。"
---

# Rewrite — エピソード再執筆チームオーケストレータ

設定文書 (キャラクターシート・プロットガイド・ブートストラップ) の変更に応じて既存エピソードを **分析 → 再執筆 → 検証** する 3-Phase チーム。

**プロジェクト独立**: polish スキルと同じ `novel-config.md` を読み込み、設定文書・ガードレール・カスタム軸を自動適用する。
他の小説プロジェクトでも novel-config.md さえ作成すれば同じように使用可能。

既存 polish (推敲) との違い: 推敲は文章・表現レベルの校正であり、rewrite は **シーン構成・キャラクター行動・プロットビートレベルの再執筆** である。シーンを新しく書いたり、キャラクターのセリフ・行動を根本的に変えたり、プロットビートを再配置する作業である。

---

## チーム構成: 生成-検証 (Producer-Reviewer) パターン

```
Phase 1 (分析 — Agent 2 個並列呼び出し)
         ┌→ [revision-analyst]    ── 設定文書↔エピソード乖離分析 ─┐
[エピソード] ┤                                                       ├→ [乖離レポート + 再執筆設計書]
         └→ [character-sculptor]  ── キャラクター立体性診断           ─┘

Phase 2 (再執筆 — 順次)
[乖離レポート + 再執筆設計書] → [episode-rewriter] → [再執筆されたエピソード]

Phase 3 (検証 — 順次)
[再執筆されたエピソード] → [quality-verifier] → PASS / REWRITE → (REWRITE 時 Phase 2 再実行)
```

| エージェント | 役割 | Phase |
|---------|------|-------|
| revision-analyst | 設定文書↔エピソード乖離分析。プロットビート・数値・時間・資金フロー・カスタム軸の整合性 | 1 (並列) |
| character-sculptor | キャラクター立体性診断。驚愕方法・非言語・内面・関係網・固有緊張点の反映度 | 1 (並列) |
| episode-rewriter | 2 つのレポートを統合してエピソードを再執筆。CLAUDE.md 執筆規則を遵守 | 2 (順次) |
| quality-verifier | 再執筆結果を設定文書・CLAUDE.md 規則と照合検証 (REWRITE モード) | 3 (順次) |

---

## 実行方法

```
/rewrite                    ← デフォルトプロジェクトの次の未完了エピソードから
/rewrite EP042              ← EP042 から開始
/rewrite start              ← EP001 から全体開始
/rewrite EP027-EP076        ← 特定範囲
/rewrite {プロジェクト名}        ← 特定プロジェクト指定 (例: /rewrite 36億坪)
/rewrite {プロジェクト名} EP042  ← 特定プロジェクト + 特定エピソード
```

---

## 自己反復ループ (Self-Loop Protocol)

### ループ構造

```
LOOP:
  1. rewrite-plan.md から次の未完了エピソードを決定
  2. 該当エピソードに対して Phase 1→2→3 を実行
  3. rewrite-plan.md を更新
  4. rewrite-log.md を更新 (主要変更を記録)
  5. 全体完了確認
     - 未完了あり → LOOP 反復
     - 全体完了 → "REWRITE_COMPLETE" を出力し、以下の後処理を実行:
       1. **fix_plan.md 部分リセット**: rewrite されたエピソードの polish 状態をリセットする。
          `{WORK_DIR}/fix_plan.md` が存在する場合、rewrite-plan.md から `[x]` で完了した EP リストを抽出して
          fix_plan.md の該当 EP を `[ ] — rewrite 後の再推敲が必要` に変更する。
          fix_plan.md が存在しない場合は rewrite された EP リストで新規作成する。
          ```
          例:
          rewrite-plan.md で EP027〜EP076 が完了
          → fix_plan.md で EP027〜EP076 を [ ] にリセット
          → 残りの EP は既存状態を維持
          ```
       2. **完了案内**:
          ```
          rewrite 完了。`/polish` を実行すると rewrite されたエピソードの文章品質を校正します。
          fix_plan.md で EP{開始}〜EP{終了} が再推敲対象としてリセットされました。
          rewrite → polish の順序を推奨します。
          ```
```

### 反復継続規則

- **止まらない**: エピソードを 1 つ終えたらすぐに次に進む。
- **ユーザ入力を待たない**: 自動で続行する。
- **終了条件**: rewrite-plan.md のすべての項目が `[x]` になったときのみ停止する。
- **中断時の再開**: `/rewrite` を再度打てば最後の未完了エピソードから引き継ぐ。

---

## 設定文書変更影響度の自動分析 (Rewrite Trigger)

`/rewrite` を実行すると Step 0 以降に **設定文書変更分析** を自動実行する。
前回 rewrite または create 時点の設定文書と現在の設定文書を比較し、rewrite 対象 EP を自動決定する。

### 変更分類基準

| 等級 | 変更タイプ | 例 | 影響範囲 |
|------|----------|------|----------|
| **CRITICAL** | キャラクター核心設定の変更 | 主人公の動機/背景/年齢/専門分野、ヴィラン正体、主要関係設定 | 該当キャラクター登場前 EP から全体 |
| **CRITICAL** | 数値/タイムラインの変更 | 資金規模、面積、時間帯、核心日付の変更 | 該当数値の初出 EP から全体 |
| **CRITICAL** | プロットビート再配置 | アーク構造変更、EP 別ビート順序の変更 | 変更されたアーク全体の EP |
| **MAJOR** | キャラクター詳細設定の変更 | ボイステーブル変更、非言語タグ変更、呼称規則変更 | 該当キャラクター登場 EP |
| **MAJOR** | ガードレールの追加/変更 | 新ガードレール項目の追加、既存項目の修正 | 全 EP 再検証 |
| **MINOR** | 文言の微修正 | 設定文書の表現修正、誤字脱字、フォーマット変更 | rewrite 不要 |
| **MINOR** | 補助参照の変更 | verification 文書の更新、ガイド文言の修正 | rewrite 不要 |

### 自動分析プロセス

```
1. 設定文書変更検知:
   - `{DESIGN_DIR}/` 内の設定文書の変更確認 (以下の優先順位):
     1. git repo の場合: `git diff` を使用
     2. git repo でない場合: `{REWRITE_WORK_DIR}/.design-hashes` ファイルと現在ファイルの SHA256 ハッシュを比較
     3. `.design-hashes` ファイルがない場合: すべての設定文書を「変更あり」とみなしてユーザに確認
   - 変更検知後 `.design-hashes` を現在のハッシュに更新
   - 変更がない場合: 「設定文書に変更なし。手動 rewrite モードで進行します。」を案内し Step 1 へ

2. 変更内容の分類:
   - 各変更項目を CRITICAL/MAJOR/MINOR に分類
   - MINOR のみの場合: 「設定文書の変更が軽微なため rewrite が不要です。それでも進行しますか？ 」を確認
   - CRITICAL/MAJOR がある場合: 影響を受ける EP 範囲を自動算出

3. ユーザに分析結果を提示:
   ```
   ## 設定文書変更分析

   ### CRITICAL 変更 (N 件)
   - {変更内容}: EP{開始}-EP{終了} 影響

   ### MAJOR 変更 (N 件)
   - {変更内容}: EP{開始}-EP{終了} 影響

   ### MINOR 変更 (N 件) — rewrite 不要
   - {変更内容}

   ### 推奨 rewrite 範囲: EP{開始}-EP{終了}
   この範囲で進行しますか？  (範囲調整可能)
   ```

4. ユーザ確認後 rewrite-plan.md を生成し Step 1 へ進行
```

---

## ワークフロー詳細

### Step 0: 初期化 (最初の 1 回)

```
1. プロジェクト決定
   - $ARGUMENTS からプロジェクト名を抽出 (例: "36 億坪")
   - プロジェクト名指定がない場合: プロジェクトディレクトリを探索し novel-config.md があるディレクトリを自動検知
   - novel-config.md が複数ある場合: ユーザに選択を要請
   - novel-config.md がない場合: ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md を案内して終了

2. novel-config.md ロードおよび **必須フィールド検証ゲート**
   - {PROJECT_DIR}/novel-config.md を読み込み
   - **即座に必須フィールド検証** (1 つでも欠落していればエラー出力後に終了):
     ```
     必須フィールドチェックリスト:
     - [ ] project.target_platform — プラットフォーム名
     - [ ] project.episode_dir — エピソード保存ディレクトリ
     - [ ] project.design_dir — 設計文書ディレクトリ (設定文書変更検知に必要)
     - [ ] 設定文書マッピング.bootstrap — ブートストラップパス (ファイル存在確認)
     - [ ] 設定文書マッピング.character_core — キャラクター核心パス (ファイル存在確認)
     - [ ] 設定文書マッピング.character_detail — キャラクター詳細パス (ファイル存在確認)
     - [ ] EP 範囲別プロットガイド — 最低 1 行存在 (ファイル存在確認)
     ```
     検証失敗時:
     ```
     ❌ novel-config.md 必須フィールド欠落: {欠落フィールドリスト}
     rewrite スキルを実行するには上記フィールドを記入してください。
     テンプレート: ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md
     ```
   - **target_platform 許容集合検証**:
     - `project.target_platform` はカクヨム、小説家になろう、アルファポリス、ノベルアップ+、エブリスタ、ノベルピアのうちいずれかでなければならない
     - rewrite スキルは novel-config.md の canonical 値のみ使用し自動マッピングはしない
     - 非対応値の場合エラー出力後に終了:
       ```
       ❌ 許容されない target_platform: {現在値}
       novel-config.md の project.target_platform を許容プラットフォーム名に修正してください。
       ```
   - 設定文書パスマッピングを抽出 → {CONFIG} 変数に保存
   - 保存ガードレール抽出 → {GUARD_RAILS}
   - Rewrite 専用設定抽出:
     - character_dialogue_dna パス → {DIALOGUE_DNA} (なければ character_detail で代替)
     - Rewrite 保存ガードレール → {REWRITE_GUARD_RAILS} (polish ガードレールに追加)
     - rewrite_work_dir → {REWRITE_WORK_DIR} (なければ work_dir を使用)
   - カスタム軸の存在有無を確認 → {CUSTOM_AXES}

3. 必須文書ロード:
   - {WRITING_RULES} — 執筆規則バイブル (novel-config.md の writing_rules、デフォルト値: CLAUDE.md)
   - {CHAR_CORE} — キャラクター核心文書 (novel-config.md の character_core)
   - {CHAR_DETAIL} — キャラクター詳細文書 (novel-config.md の character_detail)
   - {PLOT_MACRO} — 全体プロット構造 (novel-config.md の plot_macro)
   - {BOOTSTRAP} — 世界観・技術・経済規則 (novel-config.md の bootstrap)

4. 作業ディレクトリ設定
   - {REWRITE_WORK_DIR} で rewrite-plan.md を読み込みまたは生成
   - rewrite-log.md を読み込みまたは生成

5. $ARGUMENTS のパース
   - "EP{NNN}" → 該当エピソードから開始
   - "EP{NNN}-EP{MMM}" → 範囲指定
   - "start" → EP001 から
   - なし → rewrite-plan.md の次の未完了から
```

### Step 0.5: EP 範囲別設定文書の決定 (各エピソードごと)

対象エピソード番号に応じて参照する設定文書を決定する。
novel-config.md の **「EP 範囲別設定文書」** テーブルから該当 EP のプロット文書とキャラクターシートを選択。

```
{PLOT_DOC} = novel-config.md の EP 範囲マッピングで該当 EP に合うプロットガイドのパス
             詳細プロットガイド列にパスがあり、ファイルが存在すれば詳細ガイドを優先
{CHAR_CORE} = novel-config.md の共通文書のうち character_core パス
{CHAR_DETAIL} = EP 範囲別設定文書テーブルの詳細キャラクターシート列にパスがあり、ファイルが存在すれば
                詳細キャラクターシートを優先、なければ共通文書の character_detail パス
{BOOTSTRAP} = novel-config.md の共通文書のうち bootstrap パス
{VERIFY} = novel-config.md の補助参照のうち verification パス (ある場合)
{DIALOGUE_DNA} = novel-config.md の Rewrite 専用設定のうち character_dialogue_dna パス (なければ {CHAR_DETAIL})
{WRITING_RULES} = novel-config.md の writing_rules パス (デフォルト値: CLAUDE.md)
{EPISODE_DIR} = novel-config.md の episode_dir パス
{GUARD_RAILS} = novel-config.md の保存ガードレールリスト
{REWRITE_GUARD_RAILS} = novel-config.md の Rewrite 保存ガードレール (ある場合)
{CUSTOM_AXES} = novel-config.md のカスタム診断軸セクション (ある場合)
```

以降、Phase 1〜3 のエージェントプロンプトでは上記変数を実際の値に置換して渡す。

### Step 1: 対象エピソード決定

rewrite-plan.md から最上段の未完了 (`[ ]`) 項目を取得する。

### Step 2: Phase 1 — 並列分析 (Fan-out)

**Agent ツールを使用して 2 個のエージェントを 1 つのメッセージで同時呼び出しする。**

```
[同時呼び出し — 1 つの応答に 2 個の Agent ツール呼び出し]

Agent("revision-analyst"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md) について設定文書↔エピソード乖離分析を実施。

           必須読込:
           - {EPISODE_DIR}/ep{NNN}.md (対象エピソード全文)
           - {EPISODE_DIR}/ep{NNN-1}.md (直前エピソード — 連続性確認)
           - {WRITING_RULES} (執筆規則)
           - {PLOT_DOC} (該当 EP ビート・確定数値の確認)
           - {BOOTSTRAP} (世界観・数値規則)
           - {CHAR_CORE} (キャラクター核心定義・生年・固有設定)

           ★ カスタム軸検証 (novel-config.md に定義されている場合):
           {CUSTOM_AXES}
           カスタム軸が定義されている場合、該当レイヤーを併せて検証する。

           ★ 時間・数値交差検証 (3 段階プロトコル — 必須):
           [1 段階] 対象 EP + 直前 EP からすべての時間マーカーを抽出。
           明示的日付、相対的時間、経過時間をすべて収集。
           相対的時間は該当シーンの明示的日付を基準に絶対日付に変換。
           [2 段階] 同じ事件/約束/予告に対する時間参照が隣接話で矛盾しないか確認。
           [3 段階] 同一対象の数値 (プロジェクト核心数値) が EP 間で一貫しているか確認。
           EP 内算術 (除算・乗算・合算) が正しいかも検証。
           不一致発見時 [TIMELINE] または [NUMBER] タグ + 等級を付与。

           Revision Analysis Report 形式で出力。
           乖離項目別に [CRITICAL/MAJOR/MINOR] 等級を付与。"

Agent("character-sculptor"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md) についてキャラクター立体性診断を実施。

           必須読込:
           - {EPISODE_DIR}/ep{NNN}.md (対象エピソード全文)
           - {EPISODE_DIR}/ep{NNN-1}.md (直前エピソード — 非言語反復対照用)
           - {CHAR_CORE} (キャラクター核心定義)
           - {CHAR_DETAIL} (ボイステーブル・非言語・驚愕・関係変化表・呼称規則)
           - {WRITING_RULES} (キャラクターボイス・非言語タグ)
           - {REWRITE_WORK_DIR}/alive-tracker.md (関係アーク現況)
           - {DIALOGUE_DNA} (対話 DNA ガイド)

           出力パス: {REWRITE_WORK_DIR}/_workspace/06_character-sculptor_report_EP{NNN}.md

           ★ 強化された診断基準 (7 個の次元):
           1. 非言語が叙述の流れに溶け込んでいるか？  (タグ直結 vs 因果順序)
           2. 非言語の強度が事件の重みに比例するか？  (5 段階: 微弱〜圧倒)
           3. 話内同一非言語 2 回超過の有無
           4. 直前 EP と同一表現の連続反復の有無
           5. 同じ感嘆詞が話あたり 1 回超過の有無
           6. 3 人 + 同時反応時に全員同一感情の有無
           7. 対話のトーンがキャラクターアーク段階に合っているか
           8. 脇役のセリフに個人的脈絡 (経験・記憶・利害関係) が乗っているか

           ★ 対話 DNA 診断 (⑦番次元):
           9. セリフの思考パターンがキャラクター DNA と整合するか？
           10. 同じキャラクターの同じ思考経路が話あたり 3 回以上反復しないか？
           11. セリフの話者を別キャラクターに置換したとき不自然か？  (置換不可性)
           12. 情報伝達のセリフにも個人脈絡が乗っているか？  (機能セリフ検出)

           Character Sculpture Report 形式で出力。
           登場キャラクター別に立体性スコアを付与。"
```

### Step 3: Phase 2 — 再執筆実行 (Fan-in → 順次)

2 個のレポートが返ったら統合して episode-rewriter を呼び出す:

```
Agent("episode-rewriter"):
  prompt: "下記 2 個の分析レポートを基に {EPISODE_DIR}/ep{NNN}.md を再執筆。

           必須読込:
           - {EPISODE_DIR}/ep{NNN}.md (現在のエピソード全文)
           - {EPISODE_DIR}/ep{NNN-1}.md (直前エピソード — 連続性 + 非言語/対話反復確認)
           - {EPISODE_DIR}/ep{NNN+1}.md (次エピソード — 連続性、ある場合)
           - {WRITING_RULES} (執筆規則バイブル)
           - {PLOT_DOC} (該当 EP ビート・確定数値)
           - {CHAR_CORE} (キャラクター核心)
           - {CHAR_DETAIL} (ボイステーブル・非言語・驚愕・関係変化表)
           - {BOOTSTRAP} (世界観)
           - {REWRITE_WORK_DIR}/alive-tracker.md (関係アーク現況)
           - {DIALOGUE_DNA} (対話 DNA ガイド)

           ★ Phase 1 分析レポート (ファイルから読込):
           - {REWRITE_WORK_DIR}/_workspace/06_character-sculptor_report_EP{NNN}.md

           [Revision Analysis Report]
           {revision-analyst 出力}

           [Character Sculpture Report — 上記ファイルの内容も参照]
           {character-sculptor 出力}

           ★ キャラクター立体性叙述原則 (必須遵守):
           1. 非言語をタグ直結で書かない。因果の順序 (刺激→身体→行動→セリフ) に従う。
           2. 非言語の強度を事件の重みに比例させる (微弱〜圧倒 5 段階)。
           3. 同じキャラクターの同一非言語は話あたり 2 回以下。直前 EP で使用された表現は変奏する。
           4. 脇役のセリフに個人的脈絡 (経験・記憶・利害関係) を 1 行以上。
           5. 3 人 + 同時反応時、最低 1 人は別の感情で反応。
           6. 対話トーンをキャラクターアーク段階に合わせる (alive-tracker 参照)。

           ★ 対話 DNA 原則 (必須遵守 — {DIALOGUE_DNA} 参照):
           7. セリフ作成 3 段階: ①誰の DNA か → ②どんな状況タイプか → ③置換不可検証。
           8. 同じキャラクターの同じ思考経路は話あたり 2 回まで。3 回目は別経路で変奏。
           9. 情報伝達のセリフにもキャラクター DNA を通す。機能セリフ禁止。
           10. セリフ前後に行動・視線・空間叙述を入れる。台本式の対話羅列禁止。
           11. 沈黙描写は質感がなければならない。「沈黙した」の代わりに空間・感覚・行動で表現。

           ★ 保存ガードレール (絶対毀損禁止):
           {GUARD_RAILS}
           {REWRITE_GUARD_RAILS}

           {WRITING_RULES} のエピソード執筆プロセスに従って再執筆。
           再執筆結果を {EPISODE_DIR}/ep{NNN}.md に保存。
           Rewrite Execution Report 形式で出力。"
```

### Step 4: Phase 3 — 検証 (順次)

```
Agent("quality-verifier"):
  prompt: "EP{NNN} の再執筆結果を検証せよ。(REWRITE モード)

           必須読込:
           - {EPISODE_DIR}/ep{NNN}.md (再執筆されたエピソード全文)
           - {EPISODE_DIR}/ep{NNN-1}.md (直前エピソード)
           - {WRITING_RULES} (執筆規則)
           - {PLOT_DOC} (該当 EP ビート・確定数値)
           - {CHAR_CORE} (キャラクター核心)
           - {CHAR_DETAIL} (ボイステーブル・非言語・驚愕・関係変化表)

           [Rewrite Execution Report]
           {episode-rewriter 出力}

           7 カテゴリ QA 実行:
           1. ★数値一貫性 (3 段階交差検証): すべての時間マーカー抽出 → 相対時間を絶対日付に変換
              → 隣接 EP と同じ事件の時間参照対照 → すべての数値を隣接 EP と対照
              → EP 内算術検証。[TIMELINE]/[NUMBER] タグ残存時 FAIL。
           2. 時間順序
           3. キャラクターボイス
           4. 文体/禁止表現
           5. 蓋然性
           6. フック/ペーシング
           7. ★カスタム軸整合性 — novel-config.md に定義されたカスタム軸の最終確認

           ★ 保存ガードレール確認:
           {GUARD_RAILS}
           {REWRITE_GUARD_RAILS}

           キャラクター立体性最終確認。

           ★ キャラクター叙述品質検証 (必須):
           1. 非言語タグ直結 3 件 + → REWRITE
           2. 同一非言語が話あたり 3 回 + または 3 話連続反復 → REWRITE
           3. 全員同一感情反応シーン 1 件 + → REWRITE
           4. 直前 EP との非言語反復を grep で対照する。

           Verification Report 形式で PASS/REWRITE 判定。"
```

- **PASS** → Step 5 へ
- **REWRITE** → 修正指示を含めて Phase 2 を再実行 (最大 2 回)

### Step 5: 記録および次エピソードへ

1. rewrite-plan.md 更新
   ```
   - [x] EP{NNN} | REWRITTEN | 乖離:{C}c/{M}m | キャラクター:{スコア} | フック:{タイプ}/{強度}
   ```
2. rewrite-log.md に主要変更を記録
3. **5 話完了ごと**: 数値連続性・キャラクターアーク・伏線追跡の交差検証
4. **即座に次エピソードへ → Step 1 に戻る**

---

## rewrite-plan.md 形式

```markdown
# Rewrite Plan — エピソード再執筆計画

## 対象: EP{NNN}〜EP{MMM} (設定文書反映)
## 現況: 0/{総数} 完了

### EP 別状態
- [ ] EP{NNN} | {EP 要約}
- [ ] EP{NNN+1} | {EP 要約}
...
- [x] EP{MMM} | REWRITTEN | 乖離:{C}c/{M}m | キャラクター:{スコア} | フック:{タイプ}/★{N}
```

## rewrite-log.md 形式

```markdown
# Rewrite Log

## EP{NNN}
- [CRITICAL] {変更内容}
- [MAJOR] {変更内容}
- [MINOR] {変更内容}
- キャラクター変更: {詳細}
```

---

## 保存ガードレール

再執筆時にも絶対毀損しない要素。**プロジェクト別 novel-config.md からロードする。**

基本ガードレール (novel-config.md の「保存ガードレール」セクション):
- `{GUARD_RAILS}` — プロジェクトが定義した保存項目

Rewrite 専用ガードレール (novel-config.md の「Rewrite 保存ガードレール」セクション):
- `{REWRITE_GUARD_RAILS}` — 驚愕方法の交差汚染禁止、主人公動機の表現規則など

両ガードレールを合算してすべての Phase で適用する。

---

## キャラクター立体性叙述原則 (方法論)

非言語タグ表は **パレットの基本色** である。これを叙述に溶かす原則:

### 原則 1: 因果の順序
刺激 → 身体反応 (無意識) → 意識的行動 → セリフ/沈黙。

### 原則 2: 感情強度別の変奏 (5 段階)
| 強度 | 処理原則 |
|------|----------|
| 微弱 (1) | 動作が始まりかけて止まる。視線のみ移動 |
| 注目 (2) | 固有動作の縮小版 |
| 驚き (3) | 固有動作の完全な実行 |
| 衝撃 (4) | 固有動作 + 後続行動 |
| 圧倒 (5) | 固有動作すら出ない。対比で衝撃を伝達 |

### 原則 3: 反復防止
- **話内**: 同じキャラクターの同一非言語 2 回以下。同一感嘆詞 1 回。
- **話間**: 直前 2 話で使用した非言語を grep で確認。3 話連続禁止。

### 原則 4: 対話に生涯を乗せる
脇役のセリフには **自分の経験・記憶・利害関係** が一行乗らなければならない。

### 原則 5: キャラクターアーク反映
直前話までの関係変化を確認し、現在 EP の対話トーンをアーク段階に合わせる。
キャラクター詳細文書 ({CHAR_DETAIL}) の関係変化表と alive-tracker.md を参照する。

### 原則 6: 全員同一反応の禁止
同じ事件に 3 人 + 同時反応時、最低 1 人は別の感情で反応。

---

## 再執筆後ワークフロー連結

### rewrite → polish 再実行推奨

rewrite 完了後、再執筆されたエピソードは **12 軸推敲 (polish) を経ていない状態** である。
rewrite の quality-verifier は 6 カテゴリ QA のみを実施するため、polish の 12 軸 + ALIVE 4 軸診断と範囲が異なる。

**REWRITE_COMPLETE 出力後、次の案内を必ず含める:**

```
## 次のステップ: 推敲 (polish) 再実行推奨

再執筆されたエピソードはシーン・キャラクター・プロットレベルの QA を通過したが、
文章・表現レベルの 12 軸推敲はまだ実施されていません。

再執筆過程で新たな SILENCE パターン、UNIFORM 反復、TRANS (翻訳調) などが
発生する可能性があるため、下記コマンドで推敲を再実行してください:

/polish {プロジェクト名} EP{開始}

再執筆範囲: EP{開始}〜EP{終了}
```

### 再試行回数ポリシー

| スキル | 最大再試行 | 根拠 |
|------|-----------|------|
| **create** | 2 回 | 創作は 0 から始まるため修正幅が大きく、1 回で解決されにくい |
| **polish** | 1 回 | 文章レベルの校正は範囲が狭く 1 回追加で大半解決 |
| **rewrite** | 2 回 | シーン再構成は修正幅が大きく create と同一基準を適用 |

---

## 対話 DNA 原則 (方法論)

非言語が **身体** のパレットなら、対話 DNA は **言葉** のパレットである。
詳細ガイド: `${CLAUDE_PLUGIN_ROOT}/skills/rewrite/references/character-dialogue-dna.md` (方法論)
キャラクター別 DNA プロファイル: novel-config.md の `character_dialogue_dna` パス (プロジェクトデータ)

### セリフ作成 3 段階
1. **誰が話すか** — キャラクター DNA 確認。この人物は情報をどう処理するか？
2. **今どんな状況か** — 良い知らせ？  危機？  葛藤？  → 状況別の思考経路を選択
3. **置換不可検証** — このセリフの話者を別キャラクターに変えても自然か？  そうならば DNA が不足している。

### 変奏規則
- 同じキャラクターの同じ思考経路は話あたり 2 回まで。3 回目は別の経路。
- 情報伝達のセリフにもキャラクター DNA を通す。機能セリフ禁止。
- セリフ前後に行動・視線・空間叙述を入れる。台本式羅列禁止。
- 沈黙描写は質感がなければならない。「沈黙した」の代わりに空間・感覚・行動で表現。
