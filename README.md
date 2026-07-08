# public-memo

# Agentic OS構築ガイド: Fable 5を本物の従業員にする方法（2026年7月版）

## Introduction

あなたは人類史上最も強力なモデルを一般利用可能にしているのに、まだ1回ずつプロンプトを入力して使っています。このガイドはその問題を解決します。

2026年7月現在の状況：

- **Fable 5**は数時間無人稼働可能で、自律的にサブエージェントを呼び出し、1晩で1週間分の仕事をこなせます。
- しかし高コスト（$10/M input, $50/M output）、スコープを制限しないと過剰出力、間違いを人間以上に説得力を持って主張します。
- カジュアルに使うと「高価な印象的な間違い生成機」ですが、**システム内に組み込めば1日3ドルで雇える最も近い存在**になります。

このガイドはその**システム**を実際に構築します。哲学ではなく、具体的なファイル構成・順序・チェックポイント付きです。

### 完成後に手に入るもの

- モデルが交渉できないCLAUDE.md（BUILD 1）
- Fable 5が決定を下し、安価モデルが実行、独立検証者が採点、bashスクリプトが最終判断する日常ループ（BUILD 2-3）
- スキルごとの自動信頼度台帳（BUILD 4）
- 完了した仕事が永続的に再検証されるGoalsディレクトリ（BUILD 5）
- 自動執行される予算管理（BUILD 6）
- 任意の追加ループ（Quorum, Ratchet, Sparring, Compost）（BUILD 7）
- Cronスケジュール、アラーム対応runbook、30日間の段階的信頼獲得スケジュール

**対象者**: リポジトリとターミナル、Claude Codeサブスク or APIキーを持っている人。  
**読み方**: 順番に読みながら実際に構築。各BUILDは10〜20分程度。合計実作業時間約2時間。

### 3つの基本原則

1. **Laws, not tips** — 全てのルールは数字・never・検証コマンドで表現。
2. **Nothing grades its own homework** — Planner, Worker, Verifier, Gateは別。
3. **Nothing that passed once goes unwatched** — 完了した仕事は永続的不変条件として再検証。

-----

**BUILD 0: Configure the Engine（Anthropic公式設定）**

Anthropic公式ドキュメントに基づく設定。

#### 重要な5つの事実

- `max_tokens`は思考＋出力のハードリミット。高effort時は大きく設定（64k推奨）。
- `xhigh`は長時間agenticタスク専用。
- RefusalsはHTTP 200で返る（stop_reason: “refusal”を確認）。
- 「show your thinking」系プロンプトは避ける（reasoning_extraction拒否トリガー）。
- ターン長は長い。非同期処理前提。

#### 公式推奨プロンプト（そのまま使用）

（Anti-overplanning, Anti-gold-plating, Grounded progress claims, など詳細プロンプト群）

**CHECK 0**: 全てのclaude -p呼び出しでmax_tokens明示、refusalチェック、thinking表示禁止。

**Part 2（BUILD 1〜BUILD 3）**

-----

## BUILD 1: The Constitution（CLAUDE.md）

**目的**: Fableは「laws」に従い、tipsを最適化してしまうため、全てを厳格なルール化。

- 150行以内
- 数字・never・検証コマンド中心
- 「think step by step」禁止
- 事前定義ペルソナ禁止

**CLAUDE.md作成例**（実際のファイル内容は投稿を参照し、各自調整）:

```markdown
# CLAUDE.md - System Constitution

## Core Laws
1. Never output more than X tokens without explicit approval.
...
```

**CHECK 1**: `wc -l CLAUDE.md` が15行未満になるよう厳格化。各行が「80%遵守して成功主張可能か？」を検証。

-----

## BUILD 2: Walls and Gate

**目的**: 爆発範囲を事前宣言し、bashスクリプトが最終判断。

**作成ファイル**:

- `loop/contract.md`
- `loop/guardrails/verify.sh`（リポジトリ状態検証スクリプト）

**CHECK 2**: `./loop/guardrails/verify.sh` が正常終了（0）すること。失敗したらここを先に修正。

-----

## BUILD 3: The Heartbeat（日常ループ）

**目的**: Fableは高コストなので「決定のみ担当（10-20%トークン）」、実行は安価モデル、検証は別モデル。

**主なファイル**:

- `loop/triage.md`（振り分け）
- `loop/conductor.md`（Fable 5用指揮者）
- `loop/workers/implement.md`
- `loop/workers/verify.md`
- `loop/loop.sh`（メインスクリプト）

**exitコードの意味**:

- 0: quiet/done
- 1: cap
- 2: reroute
- 3: budget

**CHECK 3**: `./loop.sh`を手動実行。静かなリポジトリはほぼ0円で終了、作業が必要ならwork-order.json生成。

**Part 3（BUILD 4〜BUILD 8 + 締めくくり）**  
（最終パートです）

-----

## BUILD 4: The Trust Ledger

**目的**: スキルごとの信頼度を自動管理（20回以上・95%成功で自動化）。

- `loop/scripts/trust-log.sh`
- 各スキルごとに `loop/skills/<name>/SKILL.md` 作成（frontmatter + neverリスト + done-when）

**CHECK 4**: デモでpass/failを繰り返し、tierが自動変更されることを確認。

-----

## BUILD 5: Standing Goals + Goal Ledger

**目的**: 一度成功した仕事が腐らないよう、毎日再検証。

- `goals/<name>.md`（各完了作業ごとに1ファイル）
- Predicateはシェルコマンド（exit 0 = 正常）

**作成スクリプト**: `loop/verify-goals.sh`

**CHECK 5**: 意図的に成功/失敗するgoalを追加して検証。

-----

## BUILD 6: The Budget

**目的**: 予算が自動執行される仕組み。

- `loop/scripts/log-cost.sh`
- `loop/scripts/cost-check.sh`

**CHECK 6**: 1週間運用後にレポートが予算内に収まることを確認。

-----

## BUILD 7: Optional Loops（必要に応じて追加）

- **Quorum**: 複数廉価モデルで投票（Fable起動の閾値）
- **Ratchet**: 単調改善保証
- **Sparring**: Builder vs Breaker
- **Compost**: 失敗から法則を生成（週1、人間承認必須）

**CHECK 7**: 各ループの導入条件をメモ。

-----

## BUILD 8: Ops

- `Makefile`
- Cron設定
- Runbook（アラーム対応手順）
- 30日間信頼獲得スケジュール（段階的に監督を減らす）

### The Rules（まとめ）

- Laws, not tips
- Nothing grades its own homework
- Nothing that passed once goes unwatched
- など（投稿内の全ルール一覧）

### Closing

30日後には：

- 退屈な作業を無人実行
- Goalsディレクトリによる永続検証
- 2つの台帳による真実の可視化
- システム自身が改善を提案する週次Compost

**最初の一歩**: 今夜からBUILD 2のverify.sh → BUILD 3の初回tick → 1つのstanding goal作成。

-----

**Disclaimer**: 実際の運用は自己責任。コスト・安全性・API制限を十分考慮してください。
