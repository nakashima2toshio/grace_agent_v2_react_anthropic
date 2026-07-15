# backend/ ドキュメント整備インデックス

**Version 1.0** | 最終更新: 2026-07-15

`backend/`（GRACE-Support Web API: FastAPI + コアサービス）配下のモジュールについて、
ドキュメント作成対象・出力先・進捗を一覧化した資料。個別モジュールの詳細ドキュメントは
IPO 形式（`.claude/skills/grace-agent-docs/a_class_method_md_format.md`）で作成し、
`backend/app/doc/<module>.md` に配置する。

---

## 1. 対象範囲と方針

- **対象**: `backend/app/` 配下の Python モジュール（FastAPI 起動・API 層・コア層・スキーマ）。
- **フォーマット**:
  - モジュール（クラス/関数）… IPO 形式（`a_class_method_md_format.md`）
  - 単体テスト … SAE 形式（`a_test_md_format.md`、`grace-agent-tests` スキル担当）
- **出力先**: モジュールドキュメントは既存規約 `<package>/doc/<module>.md` に合わせ
  **`backend/app/doc/<module>.md`**。本インデックスは `backend/docs/README.md`。
- **技術スタック表記**: LLM = Anthropic Claude（既定 `claude-sonnet-4-6`）／
  Embedding = Gemini（`gemini-embedding-001`, 3072次元）。

---

## 2. モジュール仕様（IPO 形式）作成対象

| # | ソースファイル | 行数 | クラス/関数 | 役割 | ドキュメント出力先 | 優先度 | 状態 |
|---|---|---:|---:|---|---|:--:|:--:|
| 1 | `backend/app/main.py` | 49 | 0（モジュール構成のみ） | FastAPI 起動・CORS・ルーター結線 | `backend/app/doc/main.md` | ★ | ✅ 作成済 |
| 2 | `backend/app/schemas.py` | 106 | 9 | Pydantic リクエスト/レスポンス/イベント型 | `backend/app/doc/schemas.md` | 高 | 未着手 |
| 3 | `backend/app/api/support.py` | 83 | 4 | `/api/support/*`（query / stream(SSE) / confirm / result） | `backend/app/doc/api_support.md` | 高 | 未着手 |
| 4 | `backend/app/api/meta.py` | 42 | 2 | `/api/verticals`・`/api/health` | `backend/app/doc/api_meta.md` | 中 | 未着手 |
| 5 | `backend/app/core/support_agent.py` | 534 | 5 | ★コア（イベント発行型パイプライン） | `backend/app/doc/core_support_agent.md` | 最高 | 未着手 |
| 6 | `backend/app/core/gates.py` | 371 | 14 | 回答ゲート/強制エスカレ/情報なし検知/救済（純関数群） | `backend/app/doc/core_gates.md` | 高 | 未着手 |
| 7 | `backend/app/core/jobs.py` | 168 | 3 | ジョブ管理（インメモリ） | `backend/app/doc/core_jobs.md` | 中 | 未着手 |
| 8 | `backend/app/core/intervention_bridge.py` | 125 | 2 | HITL ↔ フロント承認の非同期ブリッジ | `backend/app/doc/core_intervention_bridge.md` | 中 | 未着手 |
| 9 | `backend/app/core/verticals.py` | 84 | 2 | VerticalProfile 定義（業界プロファイル） | `backend/app/doc/core_verticals.md` | 中 | 未着手 |

---

## 3. テスト仕様（SAE 形式）で扱うファイル

> 本インデックスの担当外（`grace-agent-tests` スキル・別フォーマット）。参考として掲載。

| ソースファイル | 行数 | 内容 |
|---|---:|---|
| `backend/tests/test_support_agent_core.py` | 214 | CLI とコアの同等性テスト |
| `backend/tests/test_api.py` | 163 | API エンドポイントのテスト |
| `backend/tests/test_intervention_bridge.py` | 105 | HITL ブリッジのテスト |
| `backend/tests/conftest.py` | 119 | pytest フィクスチャ（スタブベース・API キー不要） |

---

## 4. ドキュメント不要（対象外）

いずれも空ファイル（0 行）:

- `backend/__init__.py`
- `backend/app/__init__.py`
- `backend/app/api/__init__.py`
- `backend/app/core/__init__.py`
- `backend/tests/__init__.py`

---

## 5. backend/ 構成（参考）

```
backend/
├── app/
│   ├── main.py                     # FastAPI 起動・CORS（ドキュメント: app/doc/main.md）
│   ├── schemas.py                  # Pydantic: リクエスト/レスポンス/イベント
│   ├── api/
│   │   ├── support.py              # POST /api/support/query, GET /stream(SSE), POST /confirm, GET /result
│   │   └── meta.py                 # GET /api/verticals, GET /api/health
│   ├── core/
│   │   ├── support_agent.py        # ★コアサービス（イベント発行型パイプライン）
│   │   ├── gates.py                # 回答ゲート/強制エスカレ/情報なし検知/救済（純関数）
│   │   ├── intervention_bridge.py  # HITL ↔ フロント承認の非同期ブリッジ
│   │   ├── jobs.py                 # ジョブ管理（インメモリ）
│   │   └── verticals.py            # VerticalProfile 定義
│   └── doc/                        # ← モジュールドキュメント（IPO形式）の出力先
└── tests/                          # pytest（スタブベース・API キー不要）
```

---

## 6. 進行順（推奨）

1. `core/support_agent.py`（最高・コア） → `core_support_agent.md`
2. `schemas.py` → `schemas.md`
3. `api/support.py` → `api_support.md`
4. `core/gates.py` → `core_gates.md`
5. `core/jobs.py` / `core/intervention_bridge.py` / `core/verticals.py` / `api/meta.py`

---

## 7. 変更履歴

| バージョン | 変更内容 |
|-----------|---------|
| 1.0 | 初版作成（backend/ ドキュメント整備の対象一覧・出力先・進捗をまとめたインデックス。`main.py` を作成済としてマーク） |
