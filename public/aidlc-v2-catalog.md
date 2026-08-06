---
title: AIで紐解くAWS AI-DLC v2：カタログ
tags:
  - AI
  - ClaudeCode
  - AIDLC
  - AI-DLC
private: false
updated_at: '2026-08-07T08:57:16+09:00'
id: 8c72bc604819b6d625d8
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

AI-DLC v2 のワークフローは、5つのフェーズに分かれた32のステージからなります。初期化の3ステージはコンダクターが担い、残りは14体のエージェントが役割を分けて受け持ちます。本記事は、その全ステージとエージェントを、番号・フェーズ・担当つきで一覧にした引くためのリファレンスです。

ステージは「フェーズ番号.連番」で識別します（フェーズは登場順に Initialization=0／Ideation=1／Inception=2／Construction=3／Operation=4）。各記事に出てくる `2.8` のような番号は、ここで名前と中身を引けます。どのステージが実際に走るかはスコープ次第で、その仕組みは別記事「[スコープ](https://qiita.com/takeshishimada/items/c232fb2e994e7b567a5c)」で扱います。

## フェーズ

| 番号 | フェーズ | ねらい |
| --- | --- | --- |
| 0 | 初期化（Initialization） | ワークスペースの足場を作り、新規（greenfield）か既存（brownfield）かを判定して状態管理を始める。承認ゲートのない自動の準備工程 |
| 1 | 発想（Ideation） | 何を・なぜ作るかを定め、市場と実現性を確かめ、作る範囲を絞る。ビジネス・戦略寄りの工程 |
| 2 | 構想（Inception） | 要件と設計を固め、アプリを作業単位（Unit）に割り、実行計画（Bolt の並び）まで決める |
| 3 | 構築（Construction） | 設計からコードまでを作業単位ごとに実装し、ビルド・テスト・CI を通す。実際にプロダクトができる工程 |
| 4 | 運用（Operation） | 本番へデプロイし、監視・インシデント対応・性能検証・改善を回す |

## ステージ

全32ステージを番号順に並べます。各ステージが実際に走るかはスコープ次第です（別記事「[スコープ](https://qiita.com/takeshishimada/items/c232fb2e994e7b567a5c)」）。

### 初期化（Initialization）

| 番号 | ステージ | 何をする | 主担当 |
| --- | --- | --- | --- |
| 0.1 | 足場作り（workspace-scaffold） | ワークスペース（`aidlc/spaces/<space>/`）のディレクトリ構造を作る | コンダクター |
| 0.2 | 環境判定（workspace-detection） | 新規か既存かを判定する | コンダクター |
| 0.3 | 状態初期化（state-init） | 状態ファイルを作りワークフロー管理を始める | コンダクター |

### 発想（Ideation）

| 番号 | ステージ | 何をする | 主担当 |
| --- | --- | --- | --- |
| 1.1 | 意図キャプチャ（intent-capture） | 何を・なぜ作るかの意図を捉え、枠組みに整理する | プロダクト |
| 1.2 | 市場調査（market-research） | 競合や市場規模を調べ、勝ち筋と位置づけを定める | プロダクト |
| 1.3 | 実現性評価（feasibility） | 技術的な実現性と制約を確かめる | アーキテクト |
| 1.4 | スコープ定義（scope-definition） | 作る機能の範囲と優先順位を決める | プロダクト |
| 1.5 | チーム編成（team-formation） | 人のチーム編成とモブ計画を決める | デリバリー |
| 1.6 | ラフモックアップ（rough-mockups） | 主要画面・操作のラフ案を作る | デザイン |
| 1.7 | 承認・引き継ぎ（approval-handoff） | 発想の成果を承認し、次フェーズへ引き継ぐ | デリバリー |

### 構想（Inception）

| 番号 | ステージ | 何をする | 主担当 |
| --- | --- | --- | --- |
| 2.1 | リバースエンジニアリング（reverse-engineering） | 既存コードを解析し現状を把握する（brownfield のみ） | デベロッパー |
| 2.2 | 作法の発見（practices-discovery） | チームの開発慣習を調べ、ルールへ昇格する | パイプラインデプロイ |
| 2.3 | 要件分析（requirements-analysis） | 要望を洗い出し、検証できる（曖昧さのない）要件に整理する | プロダクト |
| 2.4 | ユーザーストーリー（user-stories） | ストーリーと受け入れ基準を書く | プロダクト |
| 2.5 | 詳細モックアップ（refined-mockups） | 画面・操作を詳細化する | デザイン |
| 2.6 | アプリケーション設計（application-design） | 構成・ドメインモデルを設計する | アーキテクト |
| 2.7 | 作業単位の生成（units-generation） | 作業単位（Unit）と依存関係を作る | アーキテクト |
| 2.8 | デリバリー計画（delivery-planning） | 実行計画（Bolt の並び）を決める | デリバリー |

### 構築（Construction）

| 番号 | ステージ | 何をする | 主担当 |
| --- | --- | --- | --- |
| 3.1 | 機能設計（functional-design） | 各機能の振る舞いを設計する | アーキテクト |
| 3.2 | 非機能要件（nfr-requirements） | 性能・セキュリティ等の非機能要件を定める | アーキテクト |
| 3.3 | 非機能設計（nfr-design） | 非機能要件を満たす設計を行う | アーキテクト |
| 3.4 | インフラ設計（infrastructure-design） | アプリを載せるクラウド基盤（ネットワーク・計算資源など）を設計する | AWSプラットフォーム |
| 3.5 | コード生成（code-generation） | 設計を、実際に動くコードへ落とす | デベロッパー |
| 3.6 | ビルド・テスト（build-and-test） | コードをビルドし、テストを書いて緑（パス）にする | 品質 |
| 3.7 | CIパイプライン（ci-pipeline） | 変更のたびに自動でビルド・テストする仕組み（CI）を用意する | パイプラインデプロイ |

最初の Bolt（3.1〜3.5 を最小構成で1本通す）の仕組みは、別記事「[ウォーキングスケルトン](https://qiita.com/takeshishimada/items/7a24030b9d8905f379ed)」で扱います。ビルド・テスト（3.6）と CI パイプライン（3.7）は Bolt ごとではなく、全 Bolt 完了後に一度だけ走ります。

### 運用（Operation）

| 番号 | ステージ | 何をする | 主担当 |
| --- | --- | --- | --- |
| 4.1 | デプロイパイプライン（deployment-pipeline） | 本番へ安全に届けるためのデプロイ手順を自動化する | パイプラインデプロイ |
| 4.2 | 環境プロビジョニング（environment-provisioning） | アプリが動く本番／検証環境を構築する | AWSプラットフォーム |
| 4.3 | デプロイ実行（deployment-execution） | 本番へデプロイする | パイプラインデプロイ |
| 4.4 | 可観測性セットアップ（observability-setup） | 監視・ログ・メトリクスを整える | オペレーション |
| 4.5 | インシデント対応（incident-response） | インシデント対応の体制・手順を作る | オペレーション |
| 4.6 | 性能検証（performance-validation） | 本番の性能を検証する | 品質 |
| 4.7 | フィードバック・最適化（feedback-optimization） | 運用フィードバックを集め、改善につなげる | オペレーション |

## エージェント

成果物を作る11体（主担当になり得る9体＋補佐専任2体）に加え、レビューだけをする2体と、スコープを組むコンポーザー1体がいて計14体です。レビュアーは主担当・補佐とは別枠で、特定の12ステージに READY／NOT-READY と指摘を添えます。流れを止めない検証で、最終判断は承認ゲートの人が下します（別記事「[レビュアー](https://qiita.com/takeshishimada/items/624d83e946e86e4b1553)」）。

| 名前 | 専門領域 | 主担当になり得るか |
| --- | --- | --- |
| プロダクト | プロダクト管理・ビジネス分析 | ○ |
| デザイン | UX／UI デザイン・アクセシビリティ | ○ |
| デリバリー | エンジニアリングマネジメント・Bolt 計画 | ○ |
| アーキテクト | ソリューション設計・ドメインモデル・NFR | ○ |
| デベロッパー | コード生成・データモデリング | ○ |
| 品質 | QA・テスト戦略 | ○ |
| AWSプラットフォーム | インフラ設計・クラウド基盤 | ○ |
| パイプラインデプロイ | CI／CD・リリース管理 | ○ |
| オペレーション | SRE・信頼性・可観測性 | ○ |
| DevSecOps | セキュリティ・脅威モデリング | ✕（補佐専任） |
| コンプライアンス | GRC・規制対応・リスク評価 | ✕（補佐専任） |
| アーキテクチャレビュアー | 技術設計のレビュー（独立評価） | ✕（レビュー専任） |
| プロダクトリード | 要件・プロダクトのレビュー（顧客の声） | ✕（レビュー専任） |
| コンポーザー | アダプティブワークフローの構成（実行するステージの組み立て） | ✕（コンポーズ専任） |

## 参照元

| ファイル | 内容 |
| --- | --- |
| [`core/aidlc-common/stages/`](https://github.com/awslabs/aidlc-workflows/tree/c73ee984/core/aidlc-common/stages)（32本） | 全32ステージの定義（フェーズ・slug・依存・主担当） |
| [`tools/aidlc-graph.ts`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/tools/aidlc-graph.ts) | ステージの依存グラフとコンパイル順（番号の根拠） |
| [`aidlc-common/protocols/stage-protocol-recovery.md`](https://github.com/awslabs/aidlc-workflows/blob/c73ee984/core/aidlc-common/protocols/stage-protocol-recovery.md) | フェーズ採番（0.1–0.3 … 4.x）の記載 |
| [`core/agents/`](https://github.com/awslabs/aidlc-workflows/tree/c73ee984/core/agents)（14ファイル） | エージェント14体（成果物を作る11体＋レビュー専任2体＋コンポーザー1体） |

---

## 関連記事

**全体像**: [概念マップ](https://qiita.com/takeshishimada/items/6391a320609276d0cfb6) / [工程とエージェント](https://qiita.com/takeshishimada/items/418d7b9e17192e8add85)
**目次**: [AIで紐解くAI-DLC v2](https://qiita.com/takeshishimada/items/2daa87896110603252ad)
