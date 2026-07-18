# 依頼プロンプト：agent_support_example.py の React マイグレーション

> 対象プロジェクト: grace_agent_v2_react_anthropic
> （https://github.com/nakashima2toshio/grace_agent_v2_react_anthropic）
> 移行元: nakashima2toshio/anthropic_grace_agent_v2

---

## 0. ゴール（1文で）

CLI ツール `agent_support_example.py`（GRACE-Support：業界特化・自律型サポートエージェント、905行）を、
**バックエンド＝FastAPI による API 化（処理）＋ フロントエンド＝React による Web UI 化（画面）** に移行する。
処理パイプラインのロジックは変更せず、既存の `grace/` パッケージ資産をそのまま再利用する。

## 1. 基本方針（must）

1. **現有資産の有効活用（最優先）**
   - 以下は移植・再利用し、ロジックの書き直しは行わない：
     - `grace/` パッケージ一式：`planner.py` / `executor.py` / `confidence.py`（GroundednessVerifier）/
       `intervention.py`（HITL）/ `tools.py` / `config.py` / `llm_compat.py` / `schemas.py` / `replan.py` / `memory.py`
     - `support_actions.py`（ActionBackend / IdentityVerifier）
     - `agent_support_example.py` 内の純ロジック関数群
       （`VerticalProfile` 定義、`create_intent_classifier`、`create_no_info_judge`、
       `_detect_no_info_answer`、`_should_force_escalate`、`_answer_gate`、
       `_should_rescue_unaffirmed`、`_decide_action`、`_perform_action`、
       `_collect_citations` / `_merge_citations` / `_web_citations`、`SupportResult`）
     - `config.py` / `config.yml`、`services/`、`helper/` のうち必要なもの
2. **プロバイダ方針の維持**（anthropic_grace_agent_v2 の CLAUDE.md に準拠）
   - LLM：Anthropic（既定 `claude-sonnet-4-6`、軽量・意図分類 `claude-haiku-4-5-20251001`）／ `ANTHROPIC_API_KEY`
   - Embedding：Gemini `gemini-embedding-001`（3072次元）／ `GOOGLE_API_KEY`
   - モデル名マッピングの新設禁止。クライアント生成は既存の `llm_compat.create_chat_client` /
     `helper.helper_llm.create_llm_client("anthropic")` を使う。
3. **処理の同等性**：下記 §3 のパイプライン（①〜⑥、④'・④救済・二段判定含む）の判定結果が
   CLI 版と同一になること。UI 化で変えてよいのは「入出力の経路」だけ。
4. 開発ブランチ：`claude/grace-agent-react-migration-jiea79`（変更時は要確認）

## 2. 実施内容の概要

- `run_support_agent()` を UI 非依存の「コアサービス」に分離する
  （現在は `print` / `_banner` で標準出力に密結合。これを**イベントコールバック／ジェネレータ**に変え、
  CLI レンダラと Web ストリーミングの両方から使えるようにする）
- FastAPI で API を新設し、ステップ進捗（①〜⑥）を **SSE または WebSocket** でリアルタイム配信する
- React（TypeScript）でチャット型 UI を作成する
- **HITL CONFIRM の実装変更（本移行の核心）**：
  CLI 版は非対話用に `_AUTO_PROCEED`（自動承認）で潰している。React 版では
  `create_intervention_handler` に渡すレスポンス解決を「フロントエンドの承認待ち」に差し替え、
  ユーザーが画面上で PROCEED / 拒否 を選択できるようにする（既定 dry-run は維持）。

## 3. 処理仕様（CLI 版と同等）

パイプライン（`agent_support_example.py` の実装どおり）：

```
① Plan（planner）
② Execute（executor + tools: 内部RAG → reasoning）
③ Groundedness（GroundednessVerifier：内部回答の裏付け）
④ 回答ゲート（notify_th/confirm_th）＋ エスカレ語による強制エスカレ（二段判定：キーワード候補→軽量LLM意図分類 question/request/incident）
④-救済 出典付き・非「情報なし」・矛盾なし回答の誤エスカレ防止
⑤ Web フォールバック（内部 escalate 時のみ。executor が動的Web検索済みなら再検証のみで重複推論を省略）
④' 「情報なし回答」検知ゲート（定型句候補→軽量LLM の二段判定）
⑥ Action（本人確認 → HITL CONFIRM → ActionBackend 実行。既定ドライラン）
```

### CLI 入力 → API/UI パラメータ対応

| CLI | API / UI |
|---|---|
| `query`（位置引数） | チャット入力テキスト |
| `--vertical {gov\|saas\|ec}` | 画面のプロファイル切替（セレクタ） |
| `--dry-run` / `--no-dry-run`（既定 dry-run） | 設定トグル（既定 ON） |
| `-v`（詳細ログ） | ステップトレースパネルの展開表示 |

### 出力（`SupportResult` → API レスポンス）

`decision (answer/escalate)`、回答本文、出典リスト（内部＋`[Web]`）、groundedness スコアと supported/total、
`used_web` / `web_reused`、強制エスカレ理由・意図分類結果、情報なし検知結果、アクション実行結果
（action_type / dry-run / バックエンド名）を JSON で返し、ステップ進捗はストリームで逐次配信する。

## 4. TODO

### 4.1 準備 TODO（migration 前）

- [ ] P0: 移行先リポジトリ `grace_agent_v2_react_anthropic` の状態確認（空か／初期構成があるか）と clone
- [ ] P0: `agent_support_example.py` → `grace/`・`support_actions.py`・`config` の依存グラフを精読し、
      再利用ファイル一覧（コピー対象）を確定する
- [ ] P0: `run_support_agent()` の print 依存箇所（`_banner`・`_render`・途中経過 print）を洗い出し、
      イベント化ポイント（step_started / step_finished / intervention_required / result）を設計する
- [ ] P0: HITL の割り込み点（`intervention.py` のハンドラ IF）を確認し、
      「非同期承認待ち」に差し替え可能か検証する（ここが最大のリスク）
- [ ] P1: 技術スタックの確定（→ 不足情報 §6 参照。推奨: Vite + React + TypeScript / FastAPI / SSE＋確認POST）
- [ ] P1: API 契約（OpenAPI）ドラフト作成（下記 4.2 のエンドポイント案をベースに）
- [ ] P1: 実行前提の確認：`.env`（ANTHROPIC_API_KEY / GOOGLE_API_KEY）、Qdrant 起動、
      vertical 用コレクション（`*_anthropic`）投入手順の README 化
- [ ] P2: 既存 pytest 資産のうちコアロジックに対するテストを移行対象として選定

### 4.2 処理・ディレクトリ・ファイル構成 TODO

**提案ディレクトリ構成（モノレポ）：**

```
grace_agent_v2_react_anthropic/
├── backend/
│   ├── app/
│   │   ├── main.py                  # FastAPI 起動・CORS
│   │   ├── api/
│   │   │   ├── support.py           # POST /api/support/query（ジョブ起動）
│   │   │   │                        # GET  /api/support/stream/{job_id}（SSE: ステップ進捗）
│   │   │   │                        # POST /api/support/confirm/{intervention_id}（HITL応答）
│   │   │   └── meta.py              # GET /api/verticals, GET /api/health
│   │   ├── core/
│   │   │   ├── support_agent.py     # run_support_agent のUI非依存版（イベント発行型）
│   │   │   ├── verticals.py         # VerticalProfile 定義（CLI から移設）
│   │   │   ├── gates.py             # 回答ゲート/強制エスカレ/情報なし検知/救済ロジック
│   │   │   └── intervention_bridge.py  # HITL ↔ フロント承認の非同期ブリッジ
│   │   └── schemas/                 # Pydantic: リクエスト/レスポンス/イベント
│   ├── grace/                       # 移行元から再利用（コピー or パッケージ化 → 要確認）
│   ├── support_actions.py
│   ├── config.py / config.yml
│   ├── tests/
│   └── requirements.txt (or pyproject.toml)
├── frontend/
│   ├── src/
│   │   ├── api/                     # APIクライアント・SSE購読フック
│   │   ├── components/              # 下記 4.3 の画面部品
│   │   ├── pages/ (or routes/)
│   │   ├── types/                   # SupportResult 等の型（OpenAPI から生成推奨）
│   │   └── App.tsx
│   ├── package.json
│   └── vite.config.ts
├── docker-compose.yml               # Qdrant ＋ backend ＋ frontend
└── README.md
```

- [ ] P0: `run_support_agent` を `backend/app/core/support_agent.py` へ移設し、
      print/_banner をイベントコールバックに置換（CLI 版はこのコアを呼ぶ薄いラッパとして残す）
- [ ] P0: HITL ブリッジ実装：⑥で CONFIRM が必要になったら `intervention_required` イベントを
      ストリームに流し、`/confirm` の応答が来るまで await（タイムアウト時は安全側＝escalate）
- [ ] P0: ジョブ管理（インメモリで可）：query 受付 → job_id 発行 → ステップイベント蓄積 → 完了で SupportResult
- [ ] P1: `grace/` の取り込み方式決定・実施（コピー／git submodule／pip パッケージ化 → 要確認）
- [ ] P1: エラーハンドリング（APIキー未設定・Qdrant 未起動・LLM タイムアウト）をイベントとして配信
- [ ] P1: pytest：コアの同等性テスト（CLI 版と同一 query で decision/citations が一致すること）
- [ ] P2: docker-compose 統合、CI（ruff + pytest + frontend lint/build）

### 4.3 画面作成 TODO（React）

- [ ] P0: **チャット画面**：質問入力、vertical セレクタ（gov/saas/ec）、dry-run トグル、送信
- [ ] P0: **ステップトレース表示**：①〜⑥（＋④'・④救済）をタイムライン表示。
      実行中／完了／スキップ（例：⑤の重複Web省略時は web_reused を明示）をステータス表示し、
      SSE で逐次更新する
- [ ] P0: **回答カード**：decision バッジ（answer=緑／escalate=赤）、回答本文、
      出典リスト（内部出典と `[Web] タイトル（URL）` を区別表示）、
      groundedness スコア（supported/total）表示
- [ ] P0: **HITL CONFIRM モーダル**：アクション内容（action_type / args / バックエンド名 / dry-run 有無）、
      本人確認ステップの表示、承認／拒否ボタン。承認待ち中はストリームを保留状態で表示
- [ ] P1: エスカレ理由の表示（強制エスカレ語ヒット＋意図分類結果、情報なし検知、しきい値未達の別）
- [ ] P1: 実行結果表示（ActionResult、ドライラン明示）、エラー／ローディング状態、リトライ
- [ ] P2: 履歴（セッション内の過去質問一覧）、詳細ログビュー（CLI `-v` 相当）
- [ ] P2: レスポンシブ対応・ダークテーマ

## 5. 受け入れ条件

1. 代表 5 ケースで CLI 版と同一の decision / 出典 / アクション判定になること：
   - `"パスワードを忘れました"`（既定）
   - `--vertical gov "住民票の写しの取り方は？"`
   - `--vertical ec "返品したい"`（本人確認→CONFIRM→ドライラン）
   - `--vertical saas "サービスが落ちています"`（障害→escalate）
   - 範囲外質問（④' 情報なし検知→escalate）
2. ⑥ の CONFIRM が画面承認なしに実行されないこと（自動承認 `_AUTO_PROCEED` の Web 側持ち込み禁止）
3. backend pytest・frontend build/lint が CI で green

## 6. 【要確認】着手前に回答が必要な項目

（本プロンプトの利用者への質問。回答をこのセクションに追記してから着手すること）

1. 「react」の意味の確認 → 本プロンプトは **React.js（Web UI）＋API化** と解釈して記述
2. 移行先リポジトリの現状（空／初期化済み）とアクセス権付与
3. フロント技術スタック（Vite+React+TS を推奨。Next.js 必要性、UI ライブラリ、状態管理）
4. 通信方式（SSE＋確認POST を推奨。WebSocket 必要性）
5. HITL 承認待ちのタイムアウトと既定挙動（推奨：タイムアウト＝escalate に倒す）
6. 画面スコープ（本エージェントのチャット画面のみか、既存 Streamlit の他ページも対象か）
7. 既存 anthropic_grace_agent_v2 側の Streamlit UI の扱い（併存 or 廃止）
8. `grace/` の共有方式（コピー／submodule／パッケージ化）
9. 認証・デプロイ要件（ローカル開発のみか）
10. テスト・CI の水準
