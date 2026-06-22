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
4. 다크모드·테마 작업 시 §6 및 `backend/apps/siliconvalley/_docs/darkmode-spec.md` — **vault wikilink 없음**.
5. API 계약이 필요하면 백엔드 코드 트리의 도메인 doc을 참조 (예: `backend/apps/titanic/_docs/CLAUDE.md`) — **vault wikilink 없음**.

---

## 2. Stack & Layout

| Item | Rule |
|------|------|
| Framework | Next.js 16 (App Router), React 19, TypeScript |
| UI | `components/ui/` (shadcn-style), `components/` for app chrome |
| Routes | `app/**/page.tsx` |
| Shared utils | `lib/` |
| Styles | `app/globals.css`, `styles/globals.css` |
| Theming | `next-themes` · `components/theme-provider.tsx` · Tailwind `dark:` (`@custom-variant dark` in globals) |
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
| Silicon Valley admin | `app/admin/`, `components/siliconvalley/` |
| Titanic (when added) | `app/titanic/` |
| *(future)* | `app/{domain}/` |

Keep page-specific API calls in `lib/` helpers; keep pages thin.

---

## 6. Dark Mode (Theming)

**Canonical spec:** `backend/apps/siliconvalley/_docs/darkmode-spec.md` (plain path — vault wikilink 없음). 아래는 harness 요약; 상세·예제는 spec 원문을 따른다.

### 6.1 기본 방침

| Rule | Value |
|------|-------|
| 기본 테마 | **화이트(`light`)** |
| 사용자 전환 | Topbar 우측 **해/달 토글** (`ThemeToggle`) |
| OS 연동 | `enableSystem={true}` — OS 다크모드 감지 후 사용자가 오버라이드 가능 |
| 구현 방식 | Tailwind `dark:` 클래스 + CSS 시맨틱 변수 병행 |

### 6.2 next-themes · 루트 레이아웃

`pnpm add next-themes`. 래퍼는 `components/theme-provider.tsx`.

```tsx
// app/layout.tsx
import { ThemeProvider } from '@/components/theme-provider'

export default function RootLayout({ children }) {
  return (
    <html lang="ko" suppressHydrationWarning>
      <body>
        <ThemeProvider
          attribute="class"
          defaultTheme="light"
          enableSystem
          disableTransitionOnChange
        >
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

- `suppressHydrationWarning` on `<html>` — hydration 불일치 방지.
- `attribute="class"` — `<html class="dark">` 토글.

### 6.3 Tailwind 다크 variant

Classic config: `darkMode: 'class'`. **이 레포(Tailwind v4)** 는 `app/globals.css`에 이미 선언:

```css
@custom-variant dark (&:is(.dark *));
```

`dark:` 유틸은 `.dark` 조상 아래에서만 적용된다. `'media'`(prefers-color-scheme만) 사용 금지 — 토글 제어 불가.

### 6.4 ThemeToggle

`components/theme-toggle.tsx` (신규 시 spec 패턴 따름). Topbar 우측, Auth 버튼 왼쪽.

```tsx
'use client'
import { useTheme } from 'next-themes'
import { useEffect, useState } from 'react'

export function ThemeToggle() {
  const { theme, setTheme } = useTheme()
  const [mounted, setMounted] = useState(false)
  useEffect(() => setMounted(true), [])
  if (!mounted) return null

  return (
    <button
      type="button"
      onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
      aria-label="테마 전환"
      className="p-2 rounded-md text-gray-500 hover:text-gray-900 dark:text-gray-400 dark:hover:text-gray-100"
    >
      {theme === 'dark' ? <SunIcon /> : <MoonIcon />}
    </button>
  )
}
```

`mounted` 가드 없이 렌더하면 SSR/CSR 테마 불일치가 난다.

### 6.5 CSS 변수 (시맨틱 토큰)

shadcn 토큰(`--background`, `--foreground`, …)은 `app/globals.css` `:root` / `.dark`에 정의. spec 추가 토큰과 병행:

```css
:root {
  --bg-primary:    #ffffff;
  --bg-secondary:  #f9fafb;
  --bg-surface:    #f3f4f6;
  --text-primary:  #111827;
  --text-secondary:#6b7280;
  --text-muted:    #9ca3af;
  --border:        #e5e7eb;
  --border-strong: #d1d5db;
  --accent:        #2563eb;   /* RAGWATSON 브랜드 블루 */
  --accent-hover:  #1d4ed8;
}

.dark {
  --bg-primary:    #0a0a0a;
  --bg-secondary:  #111111;
  --bg-surface:    #1a1a1a;
  --text-primary:  #ededed;
  --text-secondary:#a1a1aa;
  --text-muted:    #71717a;
  --border:        #27272a;
  --border-strong: #3f3f46;
  --accent:        #3b82f6;
  --accent-hover:  #60a5fa;
}
```

신규 페이지는 하드코딩 hex보다 `bg-background` / `text-foreground` 또는 위 `--bg-*` 변수 우선.

### 6.6 컴포넌트 `dark:` 패턴 (통일)

```tsx
// 배경
<div className="bg-white dark:bg-[#0a0a0a]">
<div className="bg-gray-50 dark:bg-[#111111]">   {/* 사이드바·패널 */}

// 텍스트
<p className="text-gray-900 dark:text-gray-100">
<p className="text-gray-500 dark:text-gray-400">
<p className="text-gray-400 dark:text-gray-500">

// 보더
<div className="border border-gray-200 dark:border-gray-800">
<div className="border-b border-gray-100 dark:border-gray-900">

// 코드 블록
<pre className="bg-gray-100 dark:bg-gray-900 text-gray-800 dark:text-gray-200">

// accent
<a className="text-blue-600 dark:text-blue-400">
<button className="bg-blue-600 hover:bg-blue-700 dark:bg-blue-500 dark:hover:bg-blue-400">
```

`components/ui/*` shadcn 프리미티브는 이미 `dark:` 내장 — 커스텀 chrome(Topbar·사이드바·패널)에 위 패턴 적용.

### 6.7 레이아웃 chrome

**Topbar** (`components/site-header.tsx` 등):

```tsx
<header className="border-b border-gray-100 dark:border-gray-800 bg-white dark:bg-[#0a0a0a] sticky top-0 z-50">
  <div className="flex items-center px-6 h-12">
  {/* … */}
    <div className="ml-auto flex items-center gap-4">
      <ThemeToggle />
      <AuthButton />
    </div>
  </div>
</header>
```

**좌측 사이드바:** `bg-gray-50 dark:bg-[#111111]`, `border-r border-gray-100 dark:border-gray-800`. active `text-blue-600 dark:text-blue-400`, 일반 `text-gray-500 dark:text-gray-400`.

**우측 패널:** `bg-gray-50 dark:bg-[#111111]`, `border-l border-gray-100 dark:border-gray-800`.

### 6.8 이미지

밝은 이미지가 다크에서 튀면 `dark:opacity-80 dark:brightness-90`. 테마별 asset이 다르면 `useTheme()`로 `logo-dark.svg` / `logo-light.svg` 분기.

### 6.9 Agent 한 줄 지시

> `next-themes`로 `defaultTheme='light'`, `attribute='class'`. Tailwind는 class 기반 dark variant. Topbar 우측 `ThemeToggle`. 모든 색상은 `bg-white dark:bg-[#0a0a0a]` 패턴 또는 globals 시맨틱 변수. 사이드바·우측패널·본문·코드블록 전부 `dark:` 대응.

---

## 7. Checklist

- [ ] [`REACT_RULES.md`](REACT_RULES.md) patterns followed (state, forms, errors).
- [ ] No unnecessary `useState` proliferation.
- [ ] API paths and response shapes match backend OpenAPI / domain docs in code tree.
- [ ] Dark mode: `defaultTheme="light"`, `ThemeToggle` in topbar, every custom surface has `dark:` pairs (§6).
- [ ] Request-scoped diff only.

## References (this vault folder only)

- Harness: [`.cursorrules`](.cursorrules)
- React: [`REACT_RULES.md`](REACT_RULES.md)
- Dark mode spec (code tree): `backend/apps/siliconvalley/_docs/darkmode-spec.md`


