---
title: AIで紐解くAWS AI-DLC v2：成果物の流れ
tags:
  - AI
  - ClaudeCode
  - AIDLC
  - AI-DLC
private: false
updated_at: '2026-08-04T17:17:34+09:00'
id: 46feb553f907f9eedd14
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
> **参照した版** — **Claude Code 実装**を対象に、2026 年 8 月 4 日時点のコミット `c73ee984`（AIDLC_VERSION 2.5.37、`core/`）を参照しています。Claude Code 以外の実装（Kiro CLI／Kiro IDE／Codex CLI／opencode）は対象外で、記述が異なる場合があります。OSS 実装は更新が続いているため、最新の状態は公式リポジトリをご確認ください。

---

## 概要

AI-DLC v2 の各ステージは、版管理された markdown の成果物を1つずつ作ります。意図（intent）から始まり、要件・ストーリー・作業単位・設計を経てコードへ。フェーズが進むほど成果物の抽象度は下がり、解像度が上がっていきます。この段階的な詳細化が、成果物の流れの土台になります。

成果物どうしは、安定IDを振って複写する方式ではつながりません。各ステージは上流の成果物を語彙名で名指しし、実体ファイルはそれを作った生産者のディレクトリに1か所だけ住みます。本記事では、名指しの連結・1成果物＝1生産者の不変条件・生産者ディレクトリへの配置・作業単位ごとのパス解決といった、成果物がどう作られ、どこに置かれ、どう名前でつながるかを読み解きます。

## 抽象度の勾配

AI-DLC v2 の5フェーズは、それぞれ固有の狙いと主要な成果（key outcome）を持ちます。方法論の原則ファイルがこれを表にしています。

| フェーズ | 狙い | 主要な成果（原文） |
| --- | --- | --- |
| 初期化（Initialization） | ブートストラップ | Configured workspace ready for workflow<br>（ワークフロー向けに構成済みのワークスペース） |
| 発想（Ideation） | 構想を検証する | **Approved initiative brief**<br>（承認された構想ブリーフ） |
| 構想（Inception） | 詳細化する | **Detailed execution plan**<br>（詳細な実行計画） |
| 構築（Construction） | 作る | **Working tested code**<br>（動作するテスト済みコード） |
| 運用（Operation） | デプロイ・運用する | Production system with monitoring<br>（監視付きの本番システム） |

「承認された構想 → 詳細な実行計画 → 動作するテスト済みコード」という並びが、そのまま抽象度の勾配になっています。各ステージが実際に作る成果物（フロントマターの `produces`）を順に並べると、勾配が見えてきます。

```mermaid
flowchart TB
  A["intent-capture<br/>intent-statement"] --> B["requirements-analysis<br/>requirements"]
  B --> C["user-stories<br/>stories / personas"]
  C --> D["units-generation<br/>unit-of-work"]
  D --> E["functional-design<br/>business-logic-model ほか"]
  E --> F["code-generation<br/>code-summary ＋ 実コード"]
```

意図が要件になり、要件がストーリーになり、ストーリーが作業単位に分割され、設計を経てコードになる。原則ファイルが掲げる原則3が、これを方法論自身の言葉で宣言しています。各ステージは版管理された markdown を記録として残し、完全な決定記録（complete decision record）をつくる、というものです。各段は前段を素材にして作られます。

## 名指しの連結と唯一の生産者

ステージのフロントマター（ファイル冒頭のメタ情報）には、成果物を扱う2つのフィールドがあります。`produces` はそのステージが作る成果物で、素の名前の配列です（例: `["requirements", "requirements-analysis-questions"]`）。`consumes` は必要とする上流成果物で、`{artifact, required, conditional_on?}` のオブジェクト配列です。

```yaml
produces:
  - requirements
consumes:
  - artifact: intent-statement
    required: false
```

`requirements-analysis` は `intent-statement`（発想フェーズの `intent-capture` が作った成果物）を**名前で**消費します。`required: false` は「あれば使う、なくても止めない」の意味です。なお、ここまでの2つは成果物の依存を表すフィールドですが、これとは別に順番の依存を表す `requires_stage` というフィールドもあります。この `produces` / `consumes` / `requires_stage` の宣言が、そのまま DAG（ステージを点、依存を矢印で表す、循環しない地図）のエッジになります。コンパイル済みのグラフは成果物をファイルパスではなく**語彙名**で保持し、生産者・消費者を引く関数も名前ベースです。

この連結が成り立つ前提が、**1つの成果物を `produces` するステージはちょうど1つ**という不変条件です。消費側はこの 1:1 を前提に、消費した成果物の生産者を一意に特定します（`producersOf(name)[0]` で確定）。実データでも確かめられ、全ステージの `produces` を集計すると**120個の成果物があり、同じ成果物を複数のステージが作る例は1つもありません**（これとは別に、ステージが条件つきで書く任意の成果物が `optional_produces` に2つあります）。グラフの検査（`validateScope`）は、生産者のいない consumes をエラーとして検出し、この 1:1 を守らせています。

連結の起点は `intent-capture` です。このステージの `consumes` は空配列で、何も消費しません。ここから下流のすべての成果物が、名指しで次々とつながっていきます。

## 生産者ディレクトリへの配置

語彙名だけでは、ファイルを開く側のコンダクター（LLM）は実体にたどり着けません。決定論的なエンジンが run-stage の指示（directive）を組むとき、名前を**実パス**に解決します。解決先は、アクティブな intent の記録ディレクトリ `aidlc/spaces/<space>/intents/<YYMMDD>-<label>/`（以下 `<record>`）の配下です。**消費される成果物は、それを消費するステージではなく、それを作ったステージのディレクトリに住む**。

- 非 per-unit（作業単位ごとに分かれない）の成果物: `<record>/<phase>/<slug>/<name>.md`
- 例: `application-design` が `requirements` を消費しても、パスは自分の下ではなく `<record>/inception/requirements-analysis/requirements.md`（生産者 `requirements-analysis` の下）に解決される

```mermaid
flowchart LR
  R["&lt;record&gt;/inception/<br/>requirements-analysis/requirements.md"]
  US["user-stories"] -.参照.-> R
  AD["application-design"] -.参照.-> R
  CG["code-generation"] -.参照.-> R
```

`requirements` を消費するステージが複数あっても、実体ファイルは生産者ディレクトリの1つだけ。消費側は全員その住所を名指しします。コピーは作られません。

AI-DLC v2 には「成果物に安定IDを振り、フェーズごとに複写（copy-forward）して履歴を残す」といった仕組みは**ありません**。`core/` を `copy-forward` / `stable id` で検索しても、該当する実装は出てきません。連結の実体は複写ではなく、**1か所に住み、語彙名で参照される**という、この物理配置と名指しの組み合わせそのものです。

## 存在しない入力の扱い

ここまでは、消費する成果物が実在する前提で話してきました。ところが**実在しないことが設計として正しい**場合があります。スコープが上流の生産者ステージをスキップするからです。たとえば `security-patch` は32ステージのうち10本しか走らせないので、走らないステージが作るはずだった成果物はディスクに現れません。

それでも下流のステージは、フロントマターでその成果物を `consumes` に宣言しています。宣言は工程の設計であって、今回のスコープで実際に何が作られるかとは別の話だからです。

エンジンは指示を組む瞬間に、宣言された入力を**ディスク上に実在するかどうかで2つに分けます**。

- `consumes` … 実在するものだけ
- `consumes_absent` … 実在しない**必須**の入力。1件ずつ `expected` の真偽が付く

`expected` の意味が要点です。**`true` は「そのステージが今回のスコープの経路に無い」**、つまり不在が設計どおりだという意味です。**`false` は「生産者は経路上にあるのに、ファイルがまだ無い」**、つまり本当の欠落です。コンダクターは前者ではステージ本体に書かれたフォールバック（要件から進める、リバースエンジニアリングで作ったコードの知識ベースを使う、ワークスペースの既存の設定を読む、など）に従い、後者は復旧の手順へ回します。

`required: false` の任意の入力が実在しない場合は、どちらにも載らず黙って落ちます。無くてもよい入力が無いことは、欠落ではないからです。

この分け方があると、**コンダクターは読めないパスを渡されません**。不在は失敗した読み取りではなく、データとして届きます。復旧の手順にも「生産者がスコープ外なら、再実行という選択肢は無い。フォールバックで進め、欠けた成果物の中身を発明するな」という分岐があります。フォールバックを備えた9つのステージ本体も、各入力を「あれば読む」形で書かれており、無かった場合に何を代わりに使うかが明記されています。

## 必ず作るとは限らない成果物

`produces` に並ぶ名前は「このステージが作るもの」ですが、**作業単位によっては要らないもの**があります。UI を持たない作業単位に画面部品の設計は不要ですし、インフラを共有しない作業単位に共有インフラの設計も要りません。

これを2つの仕組みで扱います。

### 任意の成果物

`optional_produces` は、**条件が揃ったときだけ書く成果物**を並べるフィールドです。現在2つあります。機能設計の `frontend-components`（作業単位が UI を持つときだけ）と、インフラ設計の `shared-infrastructure`（作業単位がインフラを共有するときだけ）です。

`produces` から外れているので、**書かれていなくてもステージは完了できます**。バックエンドだけの作業単位が、中身の無い「該当なし」ファイルを置いて形だけ揃える必要はありません。書く場合の置き場所はエンジンが指示に載せるので、作るときの手順は変わりません。

### 作業単位の種類による絞り込み

もう一段細かい仕組みがあります。作業単位ごとに **`kind`**（種類）を付けると、構築フェーズの設計4ステージが**その種類に必要な成果物だけ**を要求するようになります。

ステージ側は `produces_kinds` というフィールドで、どの成果物がどの種類に効くかを宣言します。機能設計の例で言えば、業務ロジックのモデルは `service`／`ui`／`library` に効き、画面部品は `ui` にだけ効く、という具合です。

種類を付けなければ、あるいはその成果物が `produces_kinds` に載っていなければ、全部を要求します。ある作業単位について必要な成果物が1つも残らない場合、そのステージはその作業単位には適用されないものとして扱われます。

## 作業単位ごとのパス解決

構築フェーズの一部のステージは、作業単位（Unit of Work）ごとに1回ずつ走ります。per-unit かどうかの判定の拠り所は、ノードのフロントマターの **`for_each: unit-of-work`** です。該当するのは5ステージ（`nfr-requirements` / `nfr-design` / `functional-design` / `infrastructure-design` / `code-generation`）で、前節の設計4ステージに `code-generation` を加えたものです。これらの成果物は、`construction/<unit>/` を挟んだパスに解決されます。

```
<record>/construction/<unit>/<slug>/<name>.md
```

`<unit>` というセグメントは、構造化された `produces` / `consumes` には一切現れません。フロントマターの配列はどこまでも素の名前で、作業単位のセグメントはエンジンがパスを組むときに差し込まれます。

**この差し込みを、エンジンは作業単位ごとに回しながら行います。** コンパイル済みの作業単位 DAG があれば、エンジンは問い合わせのたびに**まだ成果物が揃っていない最初の作業単位**を選び、その**実名**を produces / consumes / memory のパスに差し込んだ run-stage を1つ発行します（解決済みの名前は `directive.unit` にも載ります）。「どこまで終わったか」の台帳は状態の追加フィールドではなく、ディスク上の成果物の有無で判定されるため、エンジンは読み取り専用のままです。リテラルのプレースホルダ `{unit-name}` が発行されるのは、作業単位 DAG が無いスコープやコンパイル前などの**フォールバックの場合だけ**です。なおこのプレースホルダが残ったままのパスは、前節の `consumes_absent` には載りません。実在するかどうかを判定できないためです。

per-unit かどうかは、消費するステージではなく**所有者（生産者）の属性**で決まります。`functional-design`（per-unit）が `units-generation`（非 per-unit）の作った `unit-of-work` を消費するときは、所有者が per-unit でないため接頭辞は付かず `<record>/inception/units-generation/unit-of-work.md` に解決されます。住所はいつも所有者が決めるので、作業単位をまたいでも連結は壊れません。

なお、各ステージの `.md` が持つ `outputs:` フロントマターは、実行時には参照されません。エンジンは `outputs:` を読まず、`produces[]` の名前を記録ディレクトリへ解決します。`outputs:` はパス契約ではなくドキュメントだ、と core 自身が明記しています。解説ドキュメント（`docs/` 配下）に残る旧 `aidlc-docs/...` 形式の配置説明と食い違う場合は、実装を正としてください。

### 巡る順番

設計4ステージを作業単位ごとに回すとき、既定の順番は**ステージ単位**です。機能設計を全作業単位ぶん済ませ、次に非機能要件を全作業単位ぶん、というように進みます。

これを**作業単位単位**に切り替えられます。状態ファイルに `Construction Iteration: unit-major` を記録すると、1つの作業単位について設計4ステージを続けて仕上げ、それから次の作業単位へ移ります。デリバリー計画が「作業単位を先に固めるべき」と判断したときに記録されます。人があとから切り替えることもできます。

承認ゲートの数と仕組みは変わりません。**変わるのは提示される時期**で、作業単位単位では設計がひととおり終わったところで4つのゲートが続けて現れます。フィールドを設定しなければ、既定のステージ単位のままです。

> 作業単位ごとに run-stage を発行するあいだ、ステージの承認ゲートは全作業単位が揃うまで抑止されます。最後の作業単位の成果物が出そろった再入で初めて、ステージ単一の承認ゲートが1回提示されます。早期承認の拒否を含むこの挙動は、別記事「[承認ゲート](https://qiita.com/takeshishimada/items/cd6827700443c9987fd7)」で扱います。

## リバースエンジニアリングの例外

「消費される成果物は生産者のディレクトリに住む」という規約には、ただ1つの例外があります。リバースエンジニアリングが作る9つの成果物だけは、記録ディレクトリではなく**空間レベルの codekb** `aidlc/spaces/<space>/codekb/<repo>/` に住みます。記録ディレクトリは1つの intent に紐づくのに対し、codekb は intent をまたいで永続するコードベースの知識ベースだからです。既存コードを下敷きにするときだけ消費される条件付き連結（`conditional_on: brownfield`）を含め、ブラウンフィールド固有の成果物群は別記事「[ブラウンフィールド](https://qiita.com/takeshishimada/items/0a22742c273797429aee)」で扱います。

## ここまでの連結と、その先の検証

ここまで組んだ連なり（名指しの連結と生産者ディレクトリへの配置）は、フェーズの境界で「鎖」として検証されます。意図 → 要件 → ストーリー → 設計 → コード → テストが切れ目なくつながっているか。ただし、その**検証**自体は別記事「[フェーズ境界検証](https://qiita.com/takeshishimada/items/f2f4e426dd542c5b6765)」で扱います。本記事が集中したのは、その**素材**、すなわち成果物がどう作られ、どこに置かれ、どう参照されるかでした。素材が正しく連結されているからこそ、境界での検証が成り立ちます。

名前からパスへの解決は、すべて決定論的なエンジンの仕事で、コンダクター（LLM）に再導出させません。`aidlc-orchestrate.ts` のコメントは、パスの組み立ては教科書的なツールの仕事であり、それを LLM に回せば設計の趣旨そのものが反転する、と書いています。本記事が扱ったのはその解決の**結果**、つまり何がどこに住むかの規約で、解決の機構そのもの（問い合わせ・指示・報告）は別記事「[進行の中核](https://qiita.com/takeshishimada/items/c3ac7c2223e5c7020d82)」で扱います。連結を保つ働きは保存のたびにもあり、上流成果物が本文で実際に参照されているかを助言として拾う仕組みは別記事「[センサー](https://qiita.com/takeshishimada/items/5f8dbb62f25c1a09a257)」で、成果物の作成・更新の記録は別記事「[状態と監査](https://qiita.com/takeshishimada/items/72234648bb4400cedf53)」で扱います。

## 参照元

| ファイル | 内容 |
| --- | --- |
| [`core/knowledge/aidlc-shared/ai-dlc-principles.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/knowledge/aidlc-shared/ai-dlc-principles.md) | 5フェーズの狙いと主要な成果。原則3「Traceable artifacts」 |
| [`core/tools/aidlc-orchestrate.ts`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/tools/aidlc-orchestrate.ts) | 成果物パスの解決。`resolveArtifactPath`（住所は所有者の下）・`resolveConsumePath`（生産者は 1:1 で `[0]` 確定）・`isPerUnit`／`{unit-name}` の差し込み・`buildRunStageDirective`（発行時に名前→パス） |
| [`core/tools/aidlc-graph.ts`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/tools/aidlc-graph.ts) | グラフは成果物を語彙名で保持。`producersOf`／`consumersOf`（名前で引く）・`validateScope`（生産者のいない consumes をエラー化し 1:1 を守らせる） |
| [`core/aidlc-common/protocols/stage-definition.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/aidlc-common/protocols/stage-definition.md) | 発行時に消費物を実在で二分する規約。`consumes_absent` の `expected` 真偽の定義、任意の入力が不在なら黙って落とす旨 |
| [`core/tools/aidlc-directive.ts`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/tools/aidlc-directive.ts) | 指示の凍結契約。`consumes_absent?` の型（`{path, expected}` の配列）とランタイム検証 |
| [`core/aidlc-common/protocols/stage-protocol-recovery.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/aidlc-common/protocols/stage-protocol-recovery.md) | 復旧の手順。生産者がスコープ外なら再実行という選択肢は無く、フォールバックで進め成果物の中身を発明しない分岐 |
| [`core/aidlc-common/stages/inception/requirements-analysis.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/aidlc-common/stages/inception/requirements-analysis.md) | `produces`／`consumes`（`conditional_on: brownfield` 付き）の実例 |
| [`core/aidlc-common/stages/ideation/intent-capture.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/aidlc-common/stages/ideation/intent-capture.md) | 連結の起点。`consumes: []`（何も消費しない） |
| [`core/aidlc-common/stages/construction/functional-design.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/aidlc-common/stages/construction/functional-design.md) | per-unit ステージ（`for_each: unit-of-work`）の実例。`optional_produces`（`frontend-components`）と `produces_kinds`（成果物ごとに効く種類の対応表）の実例でもある |
| [`core/tools/aidlc-stage-schema.ts`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/tools/aidlc-stage-schema.ts) | フロントマターの検証。`optional_produces`／`produces_kinds` を含む省略可能フィールドの定義 |
| [`core/tools/aidlc-state.ts`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/tools/aidlc-state.ts) | `Construction Iteration` の記録（`unit-major` のときだけ作業単位単位の巡回になり、それ以外はステージ単位） |
| [`core/aidlc-common/stages/construction/code-generation.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/aidlc-common/stages/construction/code-generation.md) | per-unit の終端。`outputs:` が記録ディレクトリ相対で engine-resolved である旨を明記 |
| [`core/sensors/aidlc-upstream-coverage.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/sensors/aidlc-upstream-coverage.md) | `consumes:` の上流成果物が出力本文で参照されているかを保存ごとに検査 |

---

## 関連記事

**前の記事**: [深さ](https://qiita.com/takeshishimada/items/f2246466b9e3bdef570b)
**次の記事**: [ウォーキングスケルトン](https://qiita.com/takeshishimada/items/7a24030b9d8905f379ed)
**目次**: [AIで紐解くAI-DLC v2](https://qiita.com/takeshishimada/items/2daa87896110603252ad)
