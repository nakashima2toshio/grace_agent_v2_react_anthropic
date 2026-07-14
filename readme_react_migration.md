# GRACE-Support React マイグレーション（FastAPI + React Web UI）

CLI ツール `agent_support_example.py`（GRACE-Support: 業界特化・自律型サポートエージェント）を、
**バックエンド＝FastAPI（API 化）＋ フロントエンド＝React（Web UI 化）** に移行した構成の説明。
処理パイプライン（①Plan〜⑥Action、④'・④救済・二段判定）のロジックは CLI 版と同一で、
変更したのは「入出力の経路」だけ。

## 構成

```
grace_agent_v2_react_anthropic/
├── backend/
│   ├── app/
│   │   ├── main.py                     # FastAPI 起動・CORS（ローカル開発専用）
│   │   ├── api/
│   │   │   ├── support.py              # POST /api/support/query（ジョブ起動）
│   │   │   │                           # GET  /api/support/stream/{job_id}（SSE: ステップ進捗）
│   │   │   │                           # POST /api/support/confirm/{job_id}（HITL 応答）
│   │   │   │                           # GET  /api/support/result/{job_id}（結果取得）
│   │   │   └── meta.py                 # GET /api/verticals, GET /api/health
│   │   ├── core/
│   │   │   ├── support_agent.py        # ★ コアサービス（イベント発行型パイプライン）
│   │   │   ├── verticals.py            # VerticalProfile 定義（CLI から移設）
│   │   │   ├── gates.py                # 回答ゲート/強制エスカレ/情報なし検知/救済（純関数）
│   │   │   ├── intervention_bridge.py  # HITL ↔ フロント承認の非同期ブリッジ
│   │   │   └── jobs.py                 # ジョブ管理（インメモリ）
│   │   └── schemas.py                  # Pydantic: リクエスト/レスポンス/イベント
│   └── tests/                          # pytest（スタブベース・API キー不要）
├── frontend/                           # Vite + React + TypeScript
│   └── src/
│       ├── api/client.ts               # API クライアント・SSE 購読
│       ├── state/jobReducer.ts         # SSE イベント → UI 状態（vitest でテスト）
│       └── components/                 # QueryForm / StepTimeline / AnswerCard / ConfirmModal
├── agent_support_example.py            # CLI（コアを呼ぶ薄いラッパ。従来どおり動作）
├── grace/ / support_actions.py / config.py  # 既存資産（そのまま再利用）
└── .github/workflows/ci.yml            # CI（ruff / compileall / backend pytest / frontend build）
```

### コアの分離（処理の同等性）

- `run_support_agent()` の print/`_banner` 密結合を分離し、UI 非依存の
  `backend/app/core/support_agent.py::run_support_agent_core()`（**イベント発行型**）へ移設。
- CLI（`agent_support_example.py`）はイベントを print に変換する薄いラッパとして残した。
  純関数群（`_answer_gate` / `_should_force_escalate` / `_detect_no_info_answer` 等）と
  `PROFILES` は `backend/app/core/gates.py` / `verticals.py` へ移し、
  `agent_support_example` が再エクスポート（tests / eval / grace/step_trace の import 互換維持）。
- 同等性は `backend/tests/test_support_agent_core.py`（CLI とコアが同一の SupportResult を
  返すこと）で固定している。

### HITL CONFIRM（本移行の核心）

- CLI 版は非対話用に自動承認（`_AUTO_PROCEED`）だったが、**Web 側には持ち込まない**。
- ⑥ Action で CONFIRM に達すると `intervention` イベントが SSE で届き、画面にモーダル
  （アクション種別 / 引数 / バックエンド / dry-run / 本人確認結果）が表示される。
- ユーザーが「承認して実行（PROCEED）」/「拒否」を選択 → `POST /api/support/confirm/{job_id}`。
- **タイムアウト（既定 300 秒 = grace config `intervention.default_timeout`）時は安全側**:
  アクションを実行せず有人対応へエスカレーションする。

### 通信方式

- ジョブ起動・HITL 応答: HTTP POST
- ステップ進捗（①〜⑥、④'・④救済、スキップ・`web_reused` 含む）: **SSE**
  （`EventSource`。イベントは常に先頭からリプレイされ、再接続でも取りこぼさない）

## 起動手順（ローカル）

前提: `.env` に `ANTHROPIC_API_KEY`（LLM）と `GOOGLE_API_KEY`（Embedding）、Qdrant 起動済み。

```bash
# 1. Qdrant
docker-compose -f docker-compose/docker-compose.yml up -d

# 2. バックエンド（リポジトリルートで）
uv sync --extra dev
uv run uvicorn backend.app.main:app --reload --port 8000

# 3. フロントエンド（別ターミナル）
cd frontend
npm install
npm run dev        # http://localhost:5173 （/api は 8000 へプロキシ）
```

CLI は従来どおり:

```bash
uv run python agent_support_example.py --vertical ec "返品したい"
```

## テスト

```bash
# バックエンド（新規 backend/tests ＋ 既存 tests/ 全体）
uv run pytest

# フロントエンド
cd frontend
npm test           # vitest（jobReducer）
npm run build      # tsc --noEmit + vite build
```

CI（`.github/workflows/ci.yml`）: `ruff check .`・`compileall`・`pytest backend/tests`・
`frontend（tsc + vitest + build）` がブロッキング、レガシー全体スイートは advisory。

## API 概要

| メソッド | パス | 説明 |
|---|---|---|
| POST | `/api/support/query` | ジョブ起動（query / vertical / dry_run / use_web / do_action / verbose） |
| GET | `/api/support/stream/{job_id}` | SSE。`step`（started/finished/skipped）・`log`・`intervention`・`result`・`error`・`done` |
| POST | `/api/support/confirm/{job_id}` | HITL 応答 `{intervention_id, approve}` |
| GET | `/api/support/result/{job_id}` | ジョブ状態と `SupportResult`（JSON） |
| GET | `/api/verticals` | 業界プロファイル一覧（セレクタ用） |
| GET | `/api/health` | 稼働確認（API キー設定有無を含む） |

`SupportResult` には decision（answer/escalate）・回答本文・出典（内部＋`[Web]`）・
groundedness（支持率と判定可能主張数）・`used_web` / `web_reused`・強制エスカレ理由と
意図分類・情報なし検知・アクション結果（action_type / dry-run / バックエンド名）が含まれる。

## スコープ（本移行で決めたこと）

- 画面は**本エージェントのチャット画面のみ**（既存 Streamlit UI は移行元リポジトリで併存）
- 認証なし・ローカル開発のみ（CORS は Vite dev サーバのみ許可）
- `grace/` はリポジトリ内コピーとして共有（移行元 `anthropic_grace_agent_v2` から取り込み済み）
- ジョブ管理はインメモリ（シングルプロセス前提・永続化なし）
