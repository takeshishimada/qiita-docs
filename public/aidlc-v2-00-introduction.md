---
title: AIで紐解くAWS AI-DLC v2：はじめに
tags:
  - AI
  - AI-DLC
  - AIDLC
  - ClaudeCode
private: false
updated_at: '2026-08-04T17:14:40+09:00'
id: 2daa87896110603252ad
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

:::note info
**参照版を変更しました（2026 年 8 月 4 日）** — 本連載の基準を **2.5.36**（コミット `046a9a6c`）から **2.5.37**（コミット `c73ee984`）へ切り替えました。
:::

---

## 概要

AI-DLC v2 は、アイデアから動くシステムまでの開発ライフサイクルを、AI と人で構造化して進める方法論です。AWS Labs が `awslabs/aidlc-workflows` として OSS で公開しており、その本体は、5つのフェーズと32のステージ、それを担う14体のエージェント、進行を制御する決定論的なエンジン、成果物の中身を見て止められる承認ゲート、そして一度の是正を次に活かす学習ループから成ります。

本連載は、これを**実装から読み解く**試みです。解説ドキュメントではなく、`core/` のソースコードと規範ルールを根拠に、「実際にどう動くか」を一つずつ確かめながら全体像へ近づきます。この記事では、連載が何を・誰のために扱い、どの順で読めるのかを案内します。

## AI-DLC v2 とは

AI にコードを書かせること自体は、もう特別ではありません。難しいのは、それをプロジェクトの規模で破綻させずに続けることです。前の判断が次のプロンプトで忘れられ、なぜその設計にしたのかが残らず、モデルが勝手に先へ進んでしまう。AI-DLC v2 は、この問題を「構造の不在」と捉え、開発の流れに型を与えることで制御しようとします。

具体的には、ワークフローを**初期化・発想・構想・構築・運用**の5フェーズ32ステージに区切り、各ステージを専門のエージェントが担います。次にどのステージへ進むかは決定論的なエンジンが決め、AI には任せません。初期化を除く各ステージの終わりには、承認ゲートがあります。人が「承認」と言うまで先へは進みません。人が入れた是正は、学習ループを通じて恒久的なルールになり、以後の判断に効きます。こうした仕組みはどれも、「AI の速度を活かしつつ、人が主導権を保つ」状態を狙っています。

本連載が対象にするのは、この方法論の **Claude Code 実装**です。AI-DLC v2 の本体は `core/` というハーネス中立のディレクトリに一度だけ書かれ、そこから Claude Code・Kiro CLI・Kiro IDE・Codex CLI・opencode の5種類の配布物が生成されます。Claude Code 以外の実装は記述が異なる場合があるため対象外とします。

## 想定する読者

**AI-DLC v2 をまだ知らない開発者**と、**導入を検討するリードやアーキテクト、EM** を想定しています。前者には仕組みの全体像を、後者には内部の作り込みと導入判断の材料を届けます。

前提として、Claude Code そのものは既知として書きます。AI-DLC v2 に固有の語や仕組みは、初出で短く説明するか、深掘りの記事へリンクで送ります。なお本連載は、他の開発手法との優劣比較はしません。構造化された開発を前提に置き、その中身を読み解くことに徹します。

## 出典と方針

事実の根拠は、一次資料である `core/` のソースコードです。解説ドキュメント（`docs/` 配下）は実装と食い違うことがあるため、使いません。バージョン履歴（`CHANGELOG.md`）も、その版の時点の記録なので参照範囲には入れません。数値や担当は実装で裏を取り、確証が持てないものは推測と区別して正直に示します。

参照するのは 2026 年 8 月 4 日時点のコミット `c73ee984`（AIDLC_VERSION 2.5.37）です。版番号ではなく日付とコミットで固定しているのは、この実装が短い周期で更新され続けていて、版番号だけでは指し先が一意に決まらないからです。たとえばエージェントの数や監査イベントの数のように、更新のたびに動く数値があります。引用するときは参照したコミットを添え、最新の状態は公式リポジトリで確認できるようにしています。

## この連載の読み方

番号順に読むと、設計の動機から全体像、開発の工程、進行の仕組み、規律と検証、学習、状態の管理、そして運用と導入判断へと、関心の自然な流れでたどれます。気になるテーマから拾い読みもできます。各記事は独立して読めるように書き、深掘りは相互リンクでつなぎます。

| # | 記事 | この記事で分かること |
| --- | --- | --- |
| 01 | [設計思想](https://qiita.com/takeshishimada/items/4c8c4ae93b4184588ee6) | 承認ゲート・決定論的なエンジン・学習ループという三つの柱 |
| 02 | [概念マップ](https://qiita.com/takeshishimada/items/6391a320609276d0cfb6) | 5つの観点で見る全体像 |
| 03 | [工程とエージェント](https://qiita.com/takeshishimada/items/418d7b9e17192e8add85) | 5フェーズ32ステージと、それを担う14体のエージェント |
| 04 | [進行の中核](https://qiita.com/takeshishimada/items/c3ac7c2223e5c7020d82) | 決定論的なエンジンと、それを実行するコンダクターの分離。問い合わせと報告を往復する制御ループ |
| 05 | [スコープ](https://qiita.com/takeshishimada/items/c232fb2e994e7b567a5c) | 9種のスコープが、実行するステージを絞り込む |
| 06 | [深さ](https://qiita.com/takeshishimada/items/f2246466b9e3bdef570b) | 同じ工程をどこまで作り込むか、3段階の深さ |
| 07 | [成果物の流れ](https://qiita.com/takeshishimada/items/46feb553f907f9eedd14) | intent からコードへ、成果物が段階的に詳しくなっていく流れ |
| 08 | [ウォーキングスケルトン](https://qiita.com/takeshishimada/items/7a24030b9d8905f379ed) | 最初の Bolt（構築の実行単位）で端から端まで通し、土台を先に確かめる |
| 09 | [ブラウンフィールド](https://qiita.com/takeshishimada/items/0a22742c273797429aee) | 既存コードベースを起点にする工程と安全策 |
| 10 | [承認ゲート](https://qiita.com/takeshishimada/items/cd6827700443c9987fd7) | 成果物の中身を見て止められるのはここだけ。差し戻しと「現状で承認」 |
| 11 | [センサー](https://qiita.com/takeshishimada/items/5f8dbb62f25c1a09a257) | 保存のたびに自動で走る、進行を止めない助言チェック |
| 12 | [レビュアー](https://qiita.com/takeshishimada/items/624d83e946e86e4b1553) | 専任2体が READY／NOT-READY を返す、進行を止めない品質判定 |
| 13 | [フェーズ境界検証](https://qiita.com/takeshishimada/items/f2f4e426dd542c5b6765) | フェーズの継ぎ目で、上流から下流へのつながり（トレーサビリティ）を点検する |
| 14 | [ルールとナレッジ](https://qiita.com/takeshishimada/items/33f3b2b401d4d3c1c266) | 守るルール（push）と参照するナレッジ（pull）の違い |
| 15 | [学習ループ](https://qiita.com/takeshishimada/items/dd7f3d034ee2c137cff5) | 一度の是正を恒久ルールに変え、以後の判断に効かせる |
| 16 | [状態と監査](https://qiita.com/takeshishimada/items/72234648bb4400cedf53) | 進捗を記録する状態ファイルと、追記専用の監査ログ |
| 17 | [限界と注意点](https://qiita.com/takeshishimada/items/7b7582e2dfac5d942eda) | 実装に降りて初めて見える、いまの限界と版依存 |
| 18 | [導入判断](https://qiita.com/takeshishimada/items/cef6755e8e23a557f4de) | 案件タイプとチームから見た、入れるべきかの判断軸 |

はじめは設計思想（01）と概念マップ（02）で全体の見取り図をつかみ、そこから関心に応じて各記事へ降りるのがおすすめです。

このほかに、後半の任意の深掘りとして4本を用意しています。構築を並列で回す「[並列実行](https://qiita.com/takeshishimada/items/d179ca1bde4b047adf6f)」、エージェント同士の協働のかたちを扱う「[協働のトポロジー](https://qiita.com/takeshishimada/items/47e11209aed038250b7a)」、工程を外から足す「[プラグイン機構](https://qiita.com/takeshishimada/items/0e2966be3c9295941509)」、エージェントがどのモデルで動くかを決める「[エージェント階層](https://qiita.com/takeshishimada/items/6d7e524276eb7cd72052)」です。本編を読み終えたあと、関心のあるものだけ拾えば十分です。

また、全32ステージを番号・主担当つきで引ける「[カタログ](https://qiita.com/takeshishimada/items/8c72bc604819b6d625d8)」を、随時参照用のリファレンスとして置いています。

## 参照元

| ファイル | 内容 |
| --- | --- |
| [`awslabs/aidlc-workflows`](https://github.com/awslabs/aidlc-workflows/tree/c73ee984) | 一次資料のリポジトリ（本連載は `core/` の Claude Code 実装を対象に、2026 年 8 月 4 日時点のコミットを参照） |

---

## 関連記事

**次の記事**: [設計思想](https://qiita.com/takeshishimada/items/4c8c4ae93b4184588ee6)
