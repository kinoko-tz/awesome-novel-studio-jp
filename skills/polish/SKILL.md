---
name: polish
description: "Web 小説推敲スキル。6 名の専門エージェントを並列／順次組み合わせで運用してエピソードを順次推敲する。'/polish'、'/lint'、'/polish start'、'/polish EP051'、'/polish {プロジェクト名}'、'/polish {プロジェクト名} EP051' で実行。プロジェクト別の novel-config.md を読み込み、設定文書・ガードレール・カスタム軸を自動適用する。'推敲'、'リント'、'polish'、'lint' のいずれもこのスキルをトリガーする。"
---

# Polish — Web 小説推敲チームオーケストレーター

6 名の専門エージェントを **Fan-out/Fan-in 並列構造** で運用し、
エピソードを順次 12 軸（+ALIVE 4 軸、+プロジェクトカスタム軸）で精密に推敲する自己反復（self-loop）スキルである。

**プロジェクト独立**：`novel-config.md` を読み込み、設定文書・ガードレール・カスタム軸を自動適用する。
他の小説プロジェクトでも novel-config.md を作成するだけで同様に使用可能である。

---

## チーム構成

```
Phase 1 (並列診断 — Agent ツールで 4 個同時呼び出し)
         ┌→ [rule-checker]       ── BANNED, VOICE, TITLE, SILENCE, TRANS ─┐
[エピソード] ├→ [story-analyst]       ── SCENE, LOGIC, UNIFORM (+カスタム)  ─┤
         ├→ [platform-optimizer]  ── HOOK, OPENING, MOBILE, SUMMARY       ─┼→ [診断統合]
         └→ [alive-enhancer]      ── ALIVE-1~4 (反響、沈黙、緊張点、距離感) ─┘

Phase 2 (順次校正)
[診断統合] → [revision-executor] → [校正済みエピソード]

Phase 3 (順次検証)
[校正済みエピソード] → [revision-reviewer] → PASS / REVISE → (REVISE 時 Phase 2 再実行)
```

| エージェント | 役割 | 担当軸 | Phase |
|---------|------|---------|-------|
| rule-checker | ルール違反診断 | BANNED, VOICE, TITLE, SILENCE, TRANS | 1 (並列) |
| story-analyst | 叙事・論理分析 | SCENE, LOGIC, UNIFORM (+カスタム軸) | 1 (並列) |
| platform-optimizer | プラットフォーム最適化 | HOOK, OPENING, MOBILE, SUMMARY | 1 (並列) |
| alive-enhancer | キャラクター生動感 | ALIVE-1~4 | 1 (並列) |
| revision-executor | 校正実行 | 全体統合校正 | 2 (順次) |
| revision-reviewer | 校正検証 | 過剰校正・新規エラー・ガードレール | 3 (順次) |

---

## 実行方法

```
/polish                      ← 既定プロジェクトの次の未完了エピソードから
/polish EP051                ← EP051 から開始
/polish start                ← EP001 から全体開始
/polish {プロジェクト名}      ← 特定プロジェクト指定 (例：/polish 36 億坪)
/polish {プロジェクト名} EP051    ← 特定プロジェクト + 特定エピソード
```

---

## 自己反復ループ (Self-Loop Protocol)

**このスキルの核心：エピソード 1 つを完了したのちに自ら次のエピソードへ進む。**
外部ループは不要である。Claude 自体のエージェンティックループで反復する。

### ループ構造

```
LOOP:
  1. fix_plan.md から次の未完了エピソードを決定
  2. 当該エピソードに対して Phase 1→2→3 を実行
  3. fix_plan.md を更新
  4. learnings.md を更新 (新パターン発見時のみ)
  5. alive-tracker.md を更新 (alive-enhancer レポートに基づく)
  6. 全体完了確認
     - 未完了あり → LOOP 反復 (次のエピソード)
     - 全体完了 → "REVISION_COMPLETE" を出力して終了
       完了時の案内：「設定文書を変更する予定があれば `/rewrite` を実行後 `/polish` を再実行せよ。」
```

### 反復継続規則

- **止まらない**：エピソード 1 つを終えたら直ちに次へ進む。
- **ユーザ入力を待たない**：自動で継続する。
- **終了条件**：fix_plan.md の全項目が `[x]` のときのみ停止する。
- **中断時の再開**：`/polish` を再度入力すれば最後の未完了エピソードから継続する。

---

## ワークフロー詳細

### Step 0: 初期化 (最初の 1 回)

```
1. プロジェクト決定
   - $ARGUMENTS からプロジェクト名を抽出 (例："36 億坪")
   - プロジェクト名指定なしの場合：プロジェクトディレクトリを探索し novel-config.md があるディレクトリを自動検出
   - novel-config.md が複数ある場合：ユーザに選択を要請
   - novel-config.md がない場合：references/project-config-template.md を案内後に終了

2. novel-config.md ロードおよび **必須フィールド検証ゲート**
   - {PROJECT_DIR}/novel-config.md を読み込み
   - **即時必須フィールド検証** (1 つでも欠けていればエラー出力後に終了):
     ```
     必須フィールドチェックリスト：
     - [ ] project.target_platform — プラットフォーム名
     - [ ] project.episode_dir — エピソード保存ディレクトリ
     - [ ] project.work_dir — 作業ディレクトリ
     - [ ] 設定文書マッピング.bootstrap — ブートストラップパス (ファイル存在確認)
     - [ ] 設定文書マッピング.character_core — キャラクター核心パス (ファイル存在確認)
     - [ ] 設定文書マッピング.character_detail — キャラクター詳細パス (ファイル存在確認)
     - [ ] EP 範囲別プロットガイド — 最低 1 行存在 (ファイル存在確認)
     ```
     検証失敗時：
     ```
     ❌ novel-config.md 必須フィールド欠落：{欠落フィールドリスト}
     polish スキルを実行するには上記フィールドを記入せよ。
     テンプレート：${CLAUDE_PLUGIN_ROOT}/skills/polish/references/project-config-template.md
     ```
   - **target_platform 許容集合検証**：
     - `project.target_platform` はカクヨム、小説家になろう、アルファポリス、ノベルアップ+、エブリスタ、ノベルピアのいずれかでなければならない
     - polish スキルは novel-config.md の canonical 値のみを使用し自動マッピングしない
     - 非対応値の場合エラー出力後に終了：
       ```
       ❌ 許容されない target_platform：{現在値}
       novel-config.md の project.target_platform を許容プラットフォーム名に修正せよ。
       ```
   - 設定文書パスマッピング抽出 → {CONFIG} 変数として保存
   - 保存ガードレール抽出
   - カスタム軸の存在有無を確認
   - 沈黙パターン例外キャラクター確認

3. 作業ディレクトリ設定
   - novel-config.md の work_dir から fix_plan.md を読み込み
   - なければ生成 (EP001~最終話)
   - learnings.md 読み込み (なければ生成)
   - alive-tracker.md 読み込み (なければ生成)

4. $ARGUMENTS 確認
   - "EP{NNN}" → 当該エピソードから開始
   - "start" → EP001 から
   - なし → fix_plan.md の次の未完了から
```

### Step 0.5: EP 範囲別設定文書決定 (毎エピソード)

対象エピソード番号に応じて参照する設定文書を決定する。
novel-config.md の **「EP 範囲別設定文書」** テーブルから当該 EP のプロット文書とキャラクターシートを選択する。

```
{PLOT_DOC} = novel-config.md の EP 範囲マッピングから該当 EP に合うプロットガイドパス
             詳細プロットガイド列にパスがあり、ファイルが存在すれば詳細ガイドを優先
             (範囲重複発見時は警告出力後に最初の一致行を使用)
{CHAR_CORE} = novel-config.md の共通文書のうち character_core パス
{CHAR_DETAIL} = EP 範囲別設定文書テーブルの詳細キャラクターシート列にパスがあり、ファイルが存在すれば
                詳細キャラクターシートを優先、なければ共通文書の character_detail パス
{BOOTSTRAP} = novel-config.md の共通文書のうち bootstrap パス
{GUIDE} = novel-config.md の補助参照のうち web_novel_guide パス
{VERIFY} = novel-config.md の補助参照のうち verification パス
{PLOT_MACRO} = novel-config.md の補助参照のうち plot_macro パス
{EPISODE_DIR} = novel-config.md の episode_dir パス
{GUARD_RAILS} = novel-config.md の保存ガードレールリスト
{CUSTOM_AXES} = novel-config.md のカスタム診断軸セクション (存在する場合)
{SILENCE_EXCEPT} = novel-config.md の沈黙パターン例外キャラクター (存在する場合)
```

以下 Phase 1~3 のエージェントプロンプトでは上記変数を実際の値に置換して渡す。

### Step 1: 対象エピソード決定

fix_plan.md から最上段の未完了 (`[ ]`) 項目を取得する。
**SKIP 厳禁。** 全エピソードを全文精読する。

### Step 1.5: 直前エピソードロード (毎話必須)

対象エピソード (EP{NNN}) の **直前 2 話** を必ず読む：
- `{EPISODE_DIR}/ep{NNN-2}.md` — 直々前話の全文 (存在する場合)
- `{EPISODE_DIR}/ep{NNN-1}.md` — 直前話の全文

**検証目的**：
1. **内容重複**：同一の事件・会話・描写が直前 2 話と重複していないか確認
2. **蓋然性連続性**：直前話の結末 → 現在話の導入が自然か確認
3. **時間・数値クロス検証** (下記プロトコル参照)
4. **キャラクター位置**：直前話で退場した人物が現在話に説明なく登場、またはその逆
5. **感情の流れ**：直前話で怒っていた人物が現在話で突然のんびりしているなど急変がないか

### 時間・数値クロス検証プロトコル (story-analyst 必須)

エピソード間の **時間参照** と **数値** の整合性を構造的に検証する。
単に「数値確認」ではなく、以下 3 段階を必ず実行する。

**1 段階：タイムライン抽出**
対象エピソードと直前 2 話から **すべての時間マーカー** を抽出する：
- 明示的な日付：「10 月 15 日」、「1993 年 3 月」
- 相対時間：「来週」、「明日」、「今週土曜日」、「年末まで」、「1 月に」
- 季節／時期：「春の田植え」、「収穫後」、「冬」
- 経過時間：「2 週間かかった」、「6 か月目」
相対時間は当該シーンの明示的日付を基準として **絶対日付に変換** する。

**2 段階：タイムラインクロス対照**
同じ事件／約束／予告に対する時間参照が直前 2 話で矛盾していないか確認する。

**3 段階：数値クロス対照**
同じ対象に対する数値がエピソード間で一貫しているか確認する：
```yaml
検証_対象:
  面積: ha、エーカー、坪 — 同じ対象の面積が話ごとに異なれば [NUMBER] CRITICAL
  資金: ドル、円 — 同じ取引／借金／売上が話ごとに異なれば [NUMBER] CRITICAL
  収穫量: t/ha、トン、ブッシェル — 同じ収穫の数値が話ごとに異なれば [NUMBER] CRITICAL
  人員: 戸数、人 — 同じ集会／組合の人員が話ごとに異なれば [NUMBER] MAJOR
  算術検証: エピソード内の計算が正しいか [NUMBER] MAJOR

クロス対照_方法:
  1. 対象 EP からすべての数字+単位を抽出
  2. 直前 2 話で同じ対象の数値を探して対照
  3. 不一致発見時に [NUMBER] または [TIMELINE] タグ + 等級付与
  4. どちらが正本か判断が困難なら両方を報告
```

EP001 は直前話なし、EP002 は直前 1 話のみで進める。
直前エピソード情報は Phase 1 の **すべてのエージェントに** 伝達する。

### Step 2: Phase 1 — 並列診断 (Fan-out)

**Agent ツールを使用して 4 個のエージェントを 1 通のメッセージで同時呼び出しする。**

**レポート保存規則**：各エージェントの診断レポートを `_workspace/` にファイルとして保存する。
Phase 2 の revision-executor はレポートファイルを直接 Read してコンテキスト伝達負担を減らす。

| エージェント | レポート保存パス |
|---------|----------------|
| rule-checker | `{WORK_DIR}/_workspace/07_rule-checker_report_ep{NNN}.md` |
| story-analyst | `{WORK_DIR}/_workspace/07_story-analyst_report_ep{NNN}.md` |
| platform-optimizer | `{WORK_DIR}/_workspace/07_platform-optimizer_report_ep{NNN}.md` |
| alive-enhancer | `{WORK_DIR}/_workspace/07_alive-enhancer_report_ep{NNN}.md` |

```
[同時呼び出し — 1 つの応答に 4 個の Agent ツール呼び出し]

Agent("rule-checker"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md) に対してルール検証 5 軸診断を実行。
           直前 2 話 ({EPISODE_DIR}/ep{NNN-2}.md, {EPISODE_DIR}/ep{NNN-1}.md) も全文精読し、
           呼称・禁止表現・沈黙パターンが直前話と重複または衝突していないか確認。

           ★ 設定文書ロード (novel-config.md 基準):
           - {CHAR_DETAIL} — ボイステーブル (終結語尾・長さ・パターン)、呼称規則表 (話者×聴者)
           - {CHAR_CORE} — 主人公感情表現規則 (クラック文法など)

           沈黙パターン例外キャラクター: {SILENCE_EXCEPT}

           エージェント定義 (rule-checker.md) の軸別チェックリストと等級基準に従い
           ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/12-axes.md 軸 1~5 を参照して診断。
           各軸の grep パターンで機械的検出後に精読で誤検出を除去。
           VOICE は設定文書のボイステーブル、TITLE は呼称規則表と 1:1 対照。
           Rule Check Report 形式で出力。"

Agent("story-analyst"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md) に対して SCENE + LOGIC + UNIFORM の 3 軸診断を実行。
           ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/12-axes.md 軸 6・7・9 基準を参照。
           直前 2 話 ({EPISODE_DIR}/ep{NNN-2}.md, {EPISODE_DIR}/ep{NNN-1}.md) も全文精読。

           ★ 設定文書ロード (novel-config.md 基準 — 数値・時間・設定の正本):
           - {PLOT_DOC} — 当該 EP の確定数値 (面積・資金・収穫量・人員・時間帯)
           - {BOOTSTRAP} — マクロ数値正本
           - {CHAR_CORE} — キャラクター年齢 (生年基準)、固有設定正本
           - {VERIFY} — 検証完了数値 (存在する場合)

           ★★★ LOGIC 軸は必ずサブカテゴリを独立実行 ★★★

           [TIMELINE] 時間整合性 — 3 段階プロトコル (省略不可):
             1 段階: 対象 EP + 直前 2 話からすべての時間マーカーを抽出。相対時間は絶対日付に変換。
             2 段階: 同一の事件／約束に対する時間参照が EP 間で矛盾していないか確認。
             3 段階: 抽出結果を表で出力。「問題なし」でも表は必ず出力。

           [NUMBER] 数値整合性:
             対象 EP からすべての数字+単位を抽出 → 直前 2 話で同じ対象を grep 対照。
             EP 内の算術 (除算・乗算・合計) を直接計算検証。
             抽出結果を表で出力。

           [PLAUSIBILITY] 蓋然性:
             キャラクター反応の矛盾、費用脱漏、時代考証違反を検出。

           {CUSTOM_AXES_PROMPT}

           エージェント定義 (story-analyst.md) の出力形式を正確に従い、
           TIMELINE 抽出表、NUMBER 抽出表、クロス対照結果を必ず含む
           Story Analysis Report を出力せよ。"

Agent("platform-optimizer"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md) に対してプラットフォーム最適化 4 軸+特化診断を実行。
           直前 2 話 ({EPISODE_DIR}/ep{NNN-2}.md, {EPISODE_DIR}/ep{NNN-1}.md) も読む。
           直前 EP 末尾 300 字 → 現在 EP 冒頭 500 字の接続が自然か確認。

           ★ 設定文書ロード (novel-config.md 基準):
           - {PLOT_DOC} — 当該 EP のフック類型・感情強度・ビート構造
           - {PLOT_MACRO} — 主要転換ポイント (存在する場合)
           - {GUIDE} — モバイル最適化原則 (存在する場合)

           ★ プラットフォーム別基準:
           - novel-config.md の target_platform はカクヨム、小説家になろう、アルファポリス、ノベルアップ+、エブリスタ、ノベルピアのいずれかでなければならない
           - novel-config.md の target_platform に該当するプラットフォームガイドを参照
           - _workspace/platform-guide-{platform}.md があれば当該ファイルの基準を適用

           エージェント定義 (platform-optimizer.md) の軸別チェックリストに従い、
           ${CLAUDE_PLUGIN_ROOT}/skills/polish/references/12-axes.md 軸 8・10・11・12 を参照して診断。
           HOOK・OPENING・MOBILE・SUMMARY それぞれを定量測定 (フック強度、対話比率など)。
           HOOK 類型は設定文書の当該 EP フック類型と対照して評価。
           主要転換ポイント該当時はフック強度 4+ 必須。
           Platform Optimization Report 形式で出力。"

Agent("alive-enhancer"):
  prompt: "EP{NNN} ({EPISODE_DIR}/ep{NNN}.md) に対してキャラクター生動感 4 軸診断を実行。
           直前 2 話 ({EPISODE_DIR}/ep{NNN-2}.md, {EPISODE_DIR}/ep{NNN-1}.md) も読み、
           同一の非言語表現が直前話で繰り返し使用されていないか確認。

           ★ 設定文書ロード (novel-config.md 基準):
           - {CHAR_CORE} — 助演別の固有緊張点、主要関係曲線
           - {CHAR_DETAIL} — 非言語タグパレット、呼称規則、関係変化表

           alive-tracker.md から助演別の最後の関係イベントを確認。
           Alive Enhancement Report 形式で出力。"
```

**カスタム軸プロンプト挿入 ({CUSTOM_AXES_PROMPT})**：
novel-config.md に「カスタム診断軸」セクションがあれば、story-analyst プロンプトに当該軸の
正本・検出キーワード・判別基準を追加する。例：
```
[PASTLIFE] 前世設定整合性 (novel-config.md カスタム軸):
  レイヤー 1: 'grep "前世"' → キーワード文から正本との矛盾を検出。
  レイヤー 2: 検出_キーワードの grep → 語り手の能力不整合を検出。
  判別: 語り手の事実陳述=VIOLATION / 戦略的隠蔽=ALLOWED。
  novel-config.md の PASTLIFE セクション全文を参照せよ。
```
カスタム軸がなければこの部分は省略する。

### Step 3: Phase 2 — 校正実行 (Fan-in → 順次)

4 個のレポートが返ってきたら統合して revision-executor を呼び出す：

```
Agent("revision-executor"):
  prompt: "下記 4 個の診断レポートを統合し {EPISODE_DIR}/ep{NNN}.md を校正せよ。
           校正前に直前 2 話を必ず読み、重複内容の除去および連続性確保を確認せよ。

           ★ 設定文書ロード (novel-config.md 基準):
           - {PLOT_DOC} — 当該 EP の確定数値・ビート
           - {BOOTSTRAP} — マクロ数値正本
           - {CHAR_CORE} — 主人公固有設定、年齢 (生年基準)
           - {CHAR_DETAIL} — ボイステーブル・呼称表・非言語タグ

           ★ Phase 1 診断レポート (ファイルから読み込み):
           - {WORK_DIR}/_workspace/07_rule-checker_report_ep{NNN}.md
           - {WORK_DIR}/_workspace/07_story-analyst_report_ep{NNN}.md
           - {WORK_DIR}/_workspace/07_platform-optimizer_report_ep{NNN}.md
           - {WORK_DIR}/_workspace/07_alive-enhancer_report_ep{NNN}.md

           上記 4 ファイルを Read したのち、下記レポートも参照せよ：

           [Rule Check Report — 上記ファイル参照]
           {rule-checker 出力要約}

           [Story Analysis Report — 上記ファイル参照]
           {story-analyst 出力要約}

           [Platform Optimization Report — 上記ファイル参照]
           {platform-optimizer 出力要約}

           [Alive Enhancement Report — 上記ファイル参照]
           {alive-enhancer 出力要約}

           ★★★ 校正優先順位 (エージェント定義参照) ★★★
           1 位: [TIMELINE] CRITICAL、[NUMBER] CRITICAL — 直前 2 話を grep して正本確認後に修正
           2 位: BANNED、LOGIC (蓋然性)、カスタム軸 CRITICAL
           3 位: TITLE、VOICE、SILENCE、TRANS
           4 位: HOOK、OPENING、SCENE、ALIVE、SUMMARY、MOBILE、UNIFORM

           時間・数値校正時の特別規則:
           - 正本確認: novel-config.md の数値クロス検証正本優先順位に従う
           - 波及確認: 数値変更時は同一 EP 内の参照文も併せて修正
           - 算術再検証: 修正後に関連計算が正しいか直接確認
           - 新時間マーカー導入時は直前 2 話のタイムラインと矛盾していないか確認

           保存ガードレール: {GUARD_RAILS}
           [ADJACENT] タグ項目は優先校正対象である。
           校正後 fix_plan.md を更新せよ。"
```

### Step 4: Phase 3 — 校正検証 (順次)

```
Agent("revision-reviewer"):
  prompt: "EP{NNN} の校正結果を検証せよ。
           校正済みの {EPISODE_DIR}/ep{NNN}.md を全文精読せよ。
           直前 2 話も必ず参照せよ。

           ★ 設定文書ロード (novel-config.md 基準):
           - {PLOT_DOC} — 当該 EP の確定数値・ビート
           - {BOOTSTRAP} — マクロ数値正本
           - {CHAR_CORE} — 主人公固有設定正本
           - {CHAR_DETAIL} — ボイステーブル・非言語確認

           [Revision Execution Report]
           {revision-executor 出力}

           ★★★ 7 大検証実行 ★★★

           1. 過剰校正: 自然な表現を機械的に変えて不自然になった箇所
           2. 新規エラー: 校正によって新たに発生した数値不一致、呼称エラー、文脈断絶
           3. ★TIMELINE/NUMBER 最終検証:
              校正済み EP から時間マーカーと数字+単位を再抽出して直前 2 話とクロス対照。
              [TIMELINE] CRITICAL 残存 → REVISE
              [NUMBER] CRITICAL 残存 → REVISE
              校正に伴う新規時間・数値不一致 → REVISE
              算術エラー残存 → REVISE
              検証結果を表形式で出力
           4. カスタム軸最終確認 (novel-config.md にカスタム軸があれば)
           5. 保存ガードレール: {GUARD_RAILS}
           6. フック最終確認
           7. 分量 ±15% 以内

           PASS/REVISE 判定。"
```

- **PASS** → Step 5 へ
- **REVISE** → 修正指示を含めて Step 3 を再実行 (最大 1 回)

### Step 5: 記録および次のエピソードへ

1. fix_plan.md 現況カウント更新
2. learnings.md 更新 (新パターン発見時のみ)
3. alive-tracker.md 更新 (alive-enhancer レポートの関係イベント)
4. **直ちに次のエピソードへ → Step 1 に戻る**

---

## 強制校正ポリシー (Zero-Skip)

- **SKIP 厳禁**：全エピソードを全文精読
- **CLEAN 条件**：4 個レポートの合算違反 0 件 + 改善候補 3 件未満 + フック強度 3+
- **CLEAN 比率モニタリング**：10 話連続 CLEAN 50%+ → learnings に警告記録
- **校正率目標**：90%+

## Back-pressure 規則

- 5 話連続校正 15 件+ → learnings に共通原因分析を記録 (中断はしない)
- REVISE 3 回連続 → learnings に校正基準再検討を記録
- 50 話ごとに校正率を集計

## 保存ガードレール

novel-config.md の「保存ガードレール」セクションからロードする。
校正時にこのリストの要素を絶対に毀損しない。

## 主要転換ポイント

novel-config.md でプロジェクト別の主要転換ポイントを定義する。
platform-optimizer にフラグを伝達する。フック強度 4+ 必須。
