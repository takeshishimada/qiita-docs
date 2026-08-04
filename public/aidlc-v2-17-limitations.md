---
title: AIで紐解くAWS AI-DLC v2：限界と注意点
tags:
  - AI
  - ClaudeCode
  - AIDLC
  - AI-DLC
private: false
updated_at: '2026-07-30T10:24:56+09:00'
id: 7b7582e2dfac5d942eda
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

> **本記事の位置づけ** — 本記事は、`awslabs/aidlc-workflows` リポジトリの規範ルールおよび利用ガイドを素材として、筆者が AI を活用して読み解き、まとめた解釈です。AWS が公式に発表した方法論ではなく、一次資料の翻訳・要約でもありません。
>
> **シリーズ** — 本記事は [AIで紐解くAI-DLC v2](https://qiita.com/takeshishimada/items/2daa87896110603252ad) シリーズの一部です。
>
> **参照した版** — **Claude Code 実装**を対象に、2026 年 8 月 3 日時点のコミット `046a9a6c`（AIDLC_VERSION 2.5.36、`core/`）を参照しています。Claude Code 以外の実装（Kiro CLI／Kiro IDE／Codex CLI／opencode）は対象外で、記述が異なる場合があります。OSS 実装は更新が続いているため、最新の状態は公式リポジトリをご確認ください。

---

## 概要

AI-DLC v2 の各機構は、ドキュメントや概念図のうえでは整然と動きます。けれど実装（`core/`）に踏み込むと、「助言にとどまり止めない検証」「型はあるがまだ発行されない指示」「コメントが実体に追いついていない箇所」といった段差が見えてきます。これらは欠陥の告発ではなく、速く動いている実装に注記・型・文書が追いつく途中の、いまの限界です。

本記事は導入を検討する立場に向けて、各機構の仕組みそのものは個別記事に委ねたうえで、その「限界・注意点」の側面だけを出典つきで一望します。どこまでを仕様として頼れて、どこからが将来分なのか。煽らず誇張せず、ソースに当たって読み解きます。

## 助言どまりの検証と、止める承認ゲート

「検証ゲートが自動でブロックする」という理解は、実装上は成り立ちません。成果物の中身を見て止められるのは人の承認ゲートだけで、それ以外の「検証」はすべて助言（advisory）です。

### 止めないセンサー

センサー5本は、すべて `default_severity: advisory` です。発火フックは常に exit 0 を返す契約で（`aidlc-sensor-fire.ts`「always exit 0」）、不正な入力もファイル不在も静かに通します。コメント自身が「Blocking semantics defer to the future ralph driver」と書くとおり、ブロック機構はそもそも未実装で、将来のドライバ実装に委ねられています。

判定ロジックも、不確かなものは PASSED と判定します。真理値表（`aidlc-sensor.ts` のコメント）は8行あり（`a`/`0`/`b`/`c`/`d`/`e`/`f`/`default`）、最後の `default` は「到達しないはず（should be unreachable）」と注記された安全網です。実際に到達する7行のうち、FAILED になるのは1行だけ ―「スクリプトが正常終了（exit 0）し、かつ `pass === false`」のとき（`c`）です。タイムアウト超過（`a`）は FAILED ではなく第3の結果 `BUDGET_OVERRIDE` になり、spawn 失敗・ツール不在（exit 127）・不正 JSON・想定外の終了コードは、すべて PASSED と判定されます（「Sensor outcomes are advisory」）。

> センサーが PASSED を返しても「問題なし」とは限りません。「止める根拠を見つけられなかった」に近い状態です。詳しい仕組みは別記事「[センサー](https://qiita.com/takeshishimada/items/5f8dbb62f25c1a09a257)」で扱います。

### 止めないレビュアー

レビュアーは成果物に READY / NOT-READY の判定を返しますが、NOT-READY でもワークフローは止まりません。往復回数の上限に達してもなお NOT-READY なら、未解決の指摘を添えてそのまま承認ゲートへ進みます（「Proceed to approval gate with unresolved findings noted」）。プロトコル自身が「Does not block the workflow — the human always gets final say at the gate」と明記します。学習ゲート（残す価値のある気づきを人が選び取る、助言的な関門）も同じく「advisory and additive — it never blocks the gate」です。つまり機械的な検証（センサー・レビュアー・学習ゲート）はどれも門ではなく助言で、止める判断は最後に人へ集約されます。詳しくは別記事「[レビュアー](https://qiita.com/takeshishimada/items/624d83e946e86e4b1553)」「承認ゲート」で扱います。

### 合否を持たないフェーズ境界検証

監査イベント `PHASE_VERIFIED` は、フェーズ境界をまたぐたびに無条件で発行され、フィールドは `"Phase boundary": "X → Y"` だけです（`aidlc-state.ts`）。合否を持たない監査マーカーであって、「このフェーズは検証に合格した」という判定ではありません。名前から「検証ゲート」を連想すると誤読します。詳細は別記事「[フェーズ境界検証](https://qiita.com/takeshishimada/items/f2f4e426dd542c5b6765)」「状態と監査」で扱います。

CI やパイプラインに「自動で落ちる門」を期待してこの仕組みを置くと、期待は外れます。助言にとどまる検証は通知と監査行を残すだけで、判定を理由に止めることはありません。品質を強制したいなら、止める役割は人の承認ゲート側に設計する前提になります。

### 助言ではないが、門でもないもの

ここで一つ、逆向きの誤読も避けておきます。「すべて助言」と書きましたが、**完了を機械的に拒む仕組みは複数あります**。宣言した成果物がディスクに無い、人がそのターンで操作していない、起動されたはずの協働者の記録が無い、宣言されたレビューが走った記録が無い、作業単位ごとに回るステージで未着手の単位が残っている、コード生成のステージで実装の痕跡が残っていない——いずれの場合も、ステージは完了できません。

ただしこれらは**品質を判定していません**。要求しているのは証拠の存在だけです。レビューの記録を求める仕組みを例に取ると、必要なのは「新しい判定があること」で、判定の中身は問われません。**NOT-READY でも完了できます。**

つまり、中身を見て止められるのは承認ゲートだけ、という構図は変わりません。変わったのは、**証拠なしの完了を拒む前提条件が版を追って増えてきた**ことです。ツールが品質を保証してくれる、と読み替えないよう注意が要ります。

## 型はあるが未発行の指示

機構によっては、型としては定義済みでも、まだ実際には発行されないものがあります。「在る」ことと「動く」ことは別です。

進行エンジンが次の一手として返す指示（directive）は、判別共用体（種類をタグで見分ける型）として**10種**定義されています。このうちエンジンが実際に発行するのは8種です。`load-steering`（ルール本文の配送）/ `run-stage` / `invoke-swarm` / `ask` / `print` / `error` / `done` の7種に、長尺のワークフローをステージ境界で一時停止する終端指示 `parked` が加わります。残る `dispatch-subagent` と `present-gate` の2種は、型としてはあるが発行されません。コメント自身が「`present-gate` and `dispatch-subagent` — arrive in later waves; this handler … cleanly omits those two」と書きます（`aidlc-orchestrate.ts`）。

設計上の「予約」と、未実装は別物です。この2種は将来の波（later waves）に向けた予約スロットで、型に枠を確保しておけば、後で発行を足すときにスキーマを広げずに済みます。読む側としては、型にあるからといって「使える」と前提にしないのが安全です。指示の全体像は別記事「[進行の中核](https://qiita.com/takeshishimada/items/c3ac7c2223e5c7020d82)」で扱います。

## コメントと実体のドリフト

ソースのコメントや散文が、同じ `core/` 内の実体とズレている箇所があります。ここでは公式 docs を片側に使わず、`core/` 内で完結する食い違い（コード対コード、散文対集計）だけを集めました。

| ドリフト | コメント／散文の主張 | 実体（2026 年 8 月 3 日時点） | 出典 |
|---|---|---|---|
| ステージ数 | docstring が一貫して「31 stage files」 | ステージ定義は32本 | `aidlc-graph.ts` の `compileStageGraph` docstring ほか 対 `stages/**` 実数え |
| 全ステージ実行スコープ | enterprise が「全32を実行する唯一のスコープ」 | feature も全32を実行。散文どうしが矛盾 | `scopes/aidlc-enterprise.md` 対 `aidlc-feature.md` 対フロントマター集計 |
| 学習の反映時点 | §13 が「次にステージが走るとき、解決済みのルールと束ねたセンサーが**コンパイルで**読み込まれる」とルールとセンサーをまとめて書く | ルールの本文はステージに入るたびにディスクから読み直される（`readRuleBundle`）。コンパイルを要するのはセンサーの束ねだけ | `stage-protocol.md` §13 対 `aidlc-steering.ts`／`aidlc-orchestrate.ts` |
| `PHASE_VERIFIED` の必須項目 | 監査フォーマットの規約が必須項目を「時刻・フェーズ境界・合否・指摘」と定義 | コードが出すのはフェーズ境界だけ。合否と指摘のフィールドは存在しない | `audit-format.md` 対 `aidlc-state.ts` |
| ルール解決チェーン | 利用ガイドと導入テンプレートが「5層（`org → team → project → phase → stage`）」と記述 | コードは `org → team → project → phase` の4層で、`stage` 層は予約のみ・実体なし | `rules-reading.md` / `templates/onboarding.md` 対 `aidlc-graph.ts`（`SCOPE_PRIORITY`） |

`scopes:` フロントマターを全32本で集計した実数は次のとおりです（参考）。

| scope | 実行するステージ数 |
|---|---|
| enterprise | 32 / 32 |
| feature | 32 / 32 |
| workshop | 25 / 32 |
| mvp | 22 / 32 |
| infra | 13 / 32 |
| security-patch | 10 / 32 |
| poc | 8 / 32 |
| refactor | 8 / 32 |
| bugfix | 7 / 32 |

> これらは「バグ」ではなく、速く動く実装にコメントや概数の散文がズレている典型です。読む側としては、`core/` のコメントや概数表を確定値として鵜呑みにせず、必要なら実体（ファイル数・フロントマター）を数えて確かめるのが安全です。スコープの内訳は別記事「[スコープ](https://qiita.com/takeshishimada/items/c232fb2e994e7b567a5c)」で扱います。

## 注意点の読み方

ここで挙げた限界は、どれも欠陥の告発ではありません。速く動いている実装と、追いつく途中のコメント・型・文書のあいだの、いまの段差です。導入判断の実務に落とすと、次のように読めます。

- 助言にとどまる検証（センサー・レビュアー）に「自動で落とす門」を期待しない。中身を見て止められるのは承認ゲートだけ。ただし証拠なしの完了を拒む前提条件は複数あり、コード生成の失敗は自律モードでも止まる。
- 型としてある機能（`dispatch-subagent` / `present-gate`）を「使える」と前提にしない。
- `core/` のコメントや概数の散文を確定仕様として鵜呑みにせず、必要なら実体を数える。

導入の可否そのものをどう組み立てるかは別記事「[導入判断](https://qiita.com/takeshishimada/items/cef6755e8e23a557f4de)」で、全体像の地図は別記事「[概念マップ](https://qiita.com/takeshishimada/items/6391a320609276d0cfb6)」で扱います。

## 参照元

| ファイル | 内容 |
|---|---|
| [`core/sensors/`](https://github.com/awslabs/aidlc-workflows/tree/046a9a6c/core/sensors) | センサー5本。すべて `default_severity: advisory` |
| [`core/hooks/aidlc-sensor-fire.ts`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/hooks/aidlc-sensor-fire.ts) | 発火フック。「always exit 0」「Blocking … defer to the future ralph driver」 |
| [`core/tools/aidlc-sensor.ts`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/tools/aidlc-sensor.ts) | 判定の真理値表（8行・到達7行）。FAILED は1行のみ、他は PASSED。「Sensor outcomes are advisory」 |
| [`core/aidlc-common/protocols/stage-protocol.md`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/aidlc-common/protocols/stage-protocol.md) | 中身を見て止められるのは承認ゲートのみ、§12a レビュアーは「Does not block」、§13 学習ゲートは「it never blocks the gate」 |
| [`core/tools/aidlc-state.ts`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/tools/aidlc-state.ts) | `PHASE_VERIFIED` をフェーズ境界で無条件発行、合否フィールドなし |
| [`core/tools/aidlc-directive.ts`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/tools/aidlc-directive.ts) | 指示10種の判別共用体（`load-steering` と `parked` を含む） |
| [`core/tools/aidlc-orchestrate.ts`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/tools/aidlc-orchestrate.ts) | 発行8種と、omit する `dispatch-subagent` / `present-gate`（later waves） |
| [`core/tools/aidlc-graph.ts`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/tools/aidlc-graph.ts) | docstring が「31 stage files」（実体32）。`SCOPE_PRIORITY` の4層チェーン（`stage` 層なし） |
| [`core/aidlc-common/stages/`](https://github.com/awslabs/aidlc-workflows/tree/046a9a6c/core/aidlc-common/stages) | ステージ定義の実体32本。各 `scopes:` フロントマターのスコープ別集計 |
| [`core/scopes/aidlc-enterprise.md`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/scopes/aidlc-enterprise.md) / [`aidlc-feature.md`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/scopes/aidlc-feature.md) | 「enterprise が全32実行の唯一」対「feature も全32実行」の散文矛盾 |
| [`core/knowledge/aidlc-shared/rules-reading.md`](https://github.com/awslabs/aidlc-workflows/blob/046a9a6c/core/knowledge/aidlc-shared/rules-reading.md) | 利用ガイドと `core/templates/onboarding.md` の「5層（…→stage）」記述（コードは4層） |

---

## 関連記事

**前の記事**: [状態と監査](https://qiita.com/takeshishimada/items/72234648bb4400cedf53)
**次の記事**: [導入判断](https://qiita.com/takeshishimada/items/cef6755e8e23a557f4de)
**目次**: [AIで紐解くAI-DLC v2](https://qiita.com/takeshishimada/items/2daa87896110603252ad)
