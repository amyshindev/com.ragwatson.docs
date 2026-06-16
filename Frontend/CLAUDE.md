---
tags:
  - harness/claude-frontend
graph-group: claude-frontend
---

# Frontend — CLAUDE.md

Next.js / React scope for `frontend/`. Inherits the repository blueprint from **repository root `CLAUDE.md`** (plain path — vault 그래프에서 Backend와 연결하지 않음).

> **Precedence:** root `CLAUDE.md` on agent behavior · [`REACT_RULES.md`](REACT_RULES.md) on React patterns · this file on **frontend layout and API integration**.

> **Obsidian:** `graph-group: claude-frontend` — Backend vault(`claude-backend`)와 **wikilink로 연결하지 않음**. API 경로·스키마는 아래 텍스트로만 기술합니다.

---

## 0. Vault ↔ Code Sync (Hard Link)

**편집 원본:** `docs/Frontend/` 폴더. 코드 harness는 하드 링크로 동기화됩니다.

| 역할 | 경로 |
|------|------|
| **편집 (vault)** | `docs/Frontend/CLAUDE.md` |
| **코드 harness** | `frontend/_claude/CLAUDE.md` |
| **루트 링크** | `frontend/CLAUDE.md` |
| **Harness rules** | `docs/Frontend/.cursorrules` ↔ `frontend/_claude/.cursorrules` |
| **React rules** | `docs/Frontend/REACT_RULES.md` ↔ `frontend/_claude/REACT_RULES.md` |

```powershell
$root = (Get-Location).Path
$frontend = Join-Path $root "docs\Frontend"

foreach ($pair in @(
  @("$root\frontend\_claude\CLAUDE.md", "$frontend\CLAUDE.md"),
  @("$root\frontend\CLAUDE.md", "$frontend\CLAUDE.md"),
  @("$root\frontend\_claude\.cursorrules", "$frontend\.cursorrules"),
  @("$root\frontend\_claude\REACT_RULES.md", "$frontend\REACT_RULES.md")
)) {
  if (Test-Path $pair[0]) { Remove-Item $pair[0] -Force }
  cmd /c mklink /H $pair[0] $pair[1]
}
```

---

## 1. Before You Code

1. Repository root `CLAUDE.md` — blueprint + agent behavior.
2. This folder — layout (this file) + [`.cursorrules`](.cursorrules).
3. [`REACT_RULES.md`](REACT_RULES.md) — **mandatory** for UI work.
4. API 계약이 필요하면 백엔드 코드 트리의 도메인 doc을 참조 (예: `backend/apps/titanic/_docs/CLAUDE.md`) — **vault wikilink 없음**.

---

## 2. Stack & Layout

| Item | Rule |
|------|------|
| Framework | Next.js 16 (App Router), React 19, TypeScript |
| UI | `components/ui/` (shadcn-style), `components/` for app chrome |
| Routes | `app/**/page.tsx` |
| Shared utils | `lib/` |
| Styles | `app/globals.css`, `styles/globals.css` |
| Package manager | `pnpm` (see `pnpm-lock.yaml`) |

```text
frontend/
  app/              ← routes (e.g. app/signup/, landing)
  components/       ← UI + site chrome
  lib/              ← API clients, helpers
  hooks/
  _claude/          ← hard link to docs/Frontend/ harness files
```

---

## 3. React Rules (summary)

Full rules: [`REACT_RULES.md`](REACT_RULES.md) — do not duplicate here.

Mandatory patterns:

- Prefer **one object state** over many `useState` fields for related UI.
- Forms: `FormData` + `Object.fromEntries`; capture `e.currentTarget` before `await`.
- No `alert()` — inline banners / `role="status"`.
- API errors surfaced in UI state, not swallowed.

---

## 4. Backend Integration (HTTP only — no vault link)

| Item | Rule |
|------|------|
| API base | `http://127.0.0.1:8000` (local) |
| Titanic routes | `/titanic/*` (no `/api` prefix on titanic router) |
| Client modules | `lib/*-api.ts` per domain when the pattern exists |
| Types | Mirror backend Pydantic shapes in TypeScript interfaces |

Example Titanic endpoints:

- `GET /titanic/smith/myself` — captain intro
- `POST /titanic/smith/chat` — body `{ "message": "..." }`, response `{ "text": "..." }`

---

## 5. Domain Pages

| Product area | Typical frontend path |
|--------------|------------------------|
| Landing / Maestro | `app/page.tsx`, `components/landing/` |
| User signup | `app/signup/` |
| Titanic (when added) | `app/titanic/` |
| *(future)* | `app/{domain}/` |

Keep page-specific API calls in `lib/` helpers; keep pages thin.

---

## 6. Checklist

- [ ] [`REACT_RULES.md`](REACT_RULES.md) patterns followed (state, forms, errors).
- [ ] No unnecessary `useState` proliferation.
- [ ] API paths and response shapes match backend OpenAPI / domain docs in code tree.
- [ ] Request-scoped diff only.

## References (this vault folder only)

- Harness: [`.cursorrules`](.cursorrules)
- React: [`REACT_RULES.md`](REACT_RULES.md)


