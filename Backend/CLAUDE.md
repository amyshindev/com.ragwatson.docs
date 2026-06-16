---
tags:
  - harness/claude-backend
graph-group: claude-backend
---

# Backend — CLAUDE.md

FastAPI / Python scope for `backend/`. Inherits the repository blueprint from **repository root `CLAUDE.md`** (plain path — vault 그래프에서 Frontend와 연결하지 않음).

> **Precedence:** root `CLAUDE.md` on structure · **this vault folder** (`docs/Backend/`) on backend layout · code tree `backend/apps/{app}/_docs/` on domain detail.

> **Obsidian:** `graph-group: claude-backend` — Frontend vault(`claude-frontend`)와 **wikilink로 연결하지 않음**. API 계약은 코드 경로를 텍스트로만 적습니다.

---

## 0. Vault ↔ Code Sync (Hard Link)

**편집 원본:** `docs/Backend/` 폴더. 코드 harness는 하드 링크로 동기화됩니다.

| 역할 | 경로 |
|------|------|
| **편집 (vault)** | `docs/Backend/CLAUDE.md` |
| **코드 harness** | `backend/_claude/CLAUDE.md` |
| **루트 링크** | `backend/CLAUDE.md` |
| **Harness rules** | `docs/Backend/.cursorrules` ↔ `backend/_claude/.cursorrules` |

```powershell
$root = (Get-Location).Path
$backend = Join-Path $root "docs\Backend"

foreach ($pair in @(
  @("$root\backend\_claude\CLAUDE.md", "$backend\CLAUDE.md"),
  @("$root\backend\CLAUDE.md", "$backend\CLAUDE.md"),
  @("$root\backend\_claude\.cursorrules", "$backend\.cursorrules")
)) {
  if (Test-Path $pair[0]) { Remove-Item $pair[0] -Force }
  cmd /c mklink /H $pair[0] $pair[1]
}

foreach ($name in @(
  "db-rules.md","entity-rules.md","app-rules.md","scaffold-rules.md","auth-rules.md",
  "DOMAIN_APPS_ERD_v4.md","ENTITY_RULE.md"
)) {
  $vault = Join-Path $backend $name
  if (-not (Test-Path $vault)) {
    $src = if ($name -like "DOMAIN*" -or $name -like "ENTITY*") {
      Join-Path $root "backend\_claude\$name"
    } else { Join-Path $root "docs\$name" }
    if (Test-Path $src) { cmd /c mklink /H $vault $src }
  }
}
```

---

## 1. Before You Code

1. Repository root `CLAUDE.md` — blueprint + agent behavior.
2. This folder — backend layout (this file) + [`.cursorrules`](.cursorrules).
3. Vault backend rules (same folder):
   - [`db-rules.md`](db-rules.md)
   - [`entity-rules.md`](entity-rules.md)
   - [`app-rules.md`](app-rules.md)
   - [`scaffold-rules.md`](scaffold-rules.md)
   - [`auth-rules.md`](auth-rules.md)
4. ERD / entity: [`DOMAIN_APPS_ERD_v4.md`](DOMAIN_APPS_ERD_v4.md), [`ENTITY_RULE.md`](ENTITY_RULE.md)
5. Domain app work: code tree `backend/apps/{app}/_docs/CLAUDE.md` (예: `titanic/_docs/CLAUDE.md`).

---

## 2. Runtime Layout

| Item | Rule |
|------|------|
| App root | `backend/apps/` (`PYTHONPATH=apps`) |
| Entry | `backend/main.py` → `uvicorn main:app` |
| Local run | `backend/start-backend.ps1` (port **8000**) |
| Env | `backend/.env` (via `core.matrix` bootstrap) |
| Secrets | Never commit `.env` or API keys |
| Agent harness | `backend/_claude/` (hard link to this vault folder) |

---

## 3. Module Pathing (@backend)

- **Root omission:** Under `backend/apps/`, omit `backend` and `apps` from import paths. Example: `from titanic.adapter.inbound.api import titanic_router`.
- **Core entry:** Shared infrastructure under `backend/core/` imports as `core.*`.
- **Cross-app:** Domain apps do not import each other's adapters; share via `core/` or explicit boundaries.

---

## 4. Sibling Apps under `backend/apps/`

```text
backend/apps/
  titanic/          ← hexagonal ML + character APIs
  audio/            ← ML audio pipeline
  user/             ← auth / signup
  agora/            ← legacy / teaching modules
  board/, gallery/, library/, …
  core/             ← shared config, DB helpers (not a domain app)
```

**Per-app contract:**

| Concern | Location |
|---------|----------|
| Domain rules for agents | `apps/{app}/_docs/CLAUDE.md` |
| HTTP surface | `apps/{app}/adapter/inbound/api/` |
| Use cases | `apps/{app}/app/use_cases/` |
| Ports | `apps/{app}/app/ports/input|output/` |
| DI / FastAPI `Depends` | `apps/{app}/dependencies/` |
| ORM / PG | `apps/{app}/adapter/outbound/` |

`main.py` only **registers routers** and global middleware — no business logic.

---

## 5. Hexagonal Layers (per app)

```text
adapter/inbound/api/v1/*_router.py     → HTTP, Pydantic schemas, Depends
dependencies/*_provider.py             → wire repository + interactor
app/use_cases/*_interactor.py          → orchestration
app/ports/input/*_use_case.py          → inbound port (ABC)
app/ports/output/*_repository.py       → outbound port (ABC)
adapter/outbound/pg/*_pg_repository.py → SQLAlchemy / external I/O
domain/entities/                       → pure domain (no FastAPI / ORM)
```

**Depends rule:** Resolve dependencies in `dependencies/*_provider.py` and router `Depends(...)`. Do not put `Depends` inside interactors.

---

## 6. Database & API (summary)

- Async `AsyncSession`; write paths use `commit` / `rollback` at the route or transaction boundary.
- Request/response: Pydantic `BaseModel`, `response_model` on routes.
- Client errors: `HTTPException` with Korean `detail` where the codebase already does.
- Logging: `logger = logging.getLogger(__name__)`, layer completion logs when matching existing style.

---

## 7. Domain Apps (code tree — no vault wikilink)

| App | Agent doc path (code) |
|-----|------------------------|
| Titanic | `backend/apps/titanic/_docs/CLAUDE.md` |
| Audio | `backend/apps/audio/_docs/CLAUDE.md` |
| *(future)* | `backend/apps/{app}/_docs/CLAUDE.md` |

---

## 8. 머신러닝 데이터 분석 원칙

피처 설계·전처리·모델 입력 전 **척도 수준(level of measurement)** 을 먼저 분류한다. 잘못된 척도 가정(예: nominal에 순서 부여, interval에 비율 해석)은 ML 파이프라인 오류로 이어진다.

### Categorical (범주형)

데이터가 **카테고리**로 묶일 때 사용한다.

| 척도 | 정의 | 예시 |
|------|------|------|
| **nominal** (명목) | 이름을 바탕으로 하는 척도. **순서와 무관**하게 셀 수 있는 정도 | 청팀, 홍팀, 백팀 |
| **ordinal** (서열) | **순서(서열)** 가 있는 척도 | 이길 가능성: 1. 매우 낮음 · 2. 낮음 · 3. 보통 · 4. 높음 · 5. 매우 높음 |

**코딩 시:** nominal → one-hot / label encoding(순서 없음). ordinal → 순서를 보존하는 encoding(순위·구간화) — nominal처럼 무작위 정수 부여 금지.

### Quantitative (양적)

**숫자로 셀 수 있을 때** 사용한다.

| 척도 | 정의 | 예시 |
|------|------|------|
| **interval** (등간) | **기준점(원점) 없이** 일정한 측정 구간을 갖는 데이터 | 11:00~11:05, 15:55~16:00, 온도, pH — “10배 덥다”, “10배 시다”는 성립하지 않음 |
| **ratio** (비율) | **임의의 원점(0)** 을 기준으로 하는 데이터. **배율 비교 가능** | 나이, 돈, 몸무게 — “10배 많다”가 성립 |

**코딩 시:** interval/ratio는 수치 피처로 직접 사용 가능. ratio는 스케일링·비율 파생(증감률 등)에 유리. interval은 절대 0의 의미가 없으므로 “0 없음” 가정 모델·정규화 시 주의.

---

## 9. Checklist

- [ ] Logic sits in the correct layer (router → provider → interactor → repository).
- [ ] Imports use `titanic.*` / `core.*` style — not `backend.apps.*`.
- [ ] Domain `_docs/CLAUDE.md` consulted when editing that app.
- [ ] No secrets in diff; smoke-check new endpoints via `/docs`.
- [ ] ML 피처 척도 분류 확인 (nominal / ordinal / interval / ratio).

## References (this vault folder only)

- Harness: [`.cursorrules`](.cursorrules)
- Rules: [`db-rules.md`](db-rules.md) · [`entity-rules.md`](entity-rules.md) · [`app-rules.md`](app-rules.md)
- ERD: [`DOMAIN_APPS_ERD_v4.md`](DOMAIN_APPS_ERD_v4.md) · [`ENTITY_RULE.md`](ENTITY_RULE.md)
