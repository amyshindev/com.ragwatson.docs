# React 프론트엔드 코딩 규칙 (Maestro)

이 문서는 **Cursor / 에이전트**가 `frontend/` 작업 시 따를 React 규칙입니다.  
프로젝트 루트 `.cursor/rules/frontend-react-state.mdc`와 연동되어, 별도 프롬프트 없이도 적용됩니다.

---

## 1. `useState`는 적게 쓴다

- **원칙:** `useState`를 필드마다 여러 개 두지 않는다. 관련 상태는 **하나의 객체**로 묶는다.
- **예외:** 서로 독립·갱신 주기가 다른 상태(예: 모달 open/close만)는 분리해도 된다.
- **지양:** `email`, `username`, `password` 각각 `useState` 3개 → **권장:** `form` 객체 1개 또는 비제어 폼 + `FormData`.

---

## 2. 폼 제출 패턴 (표준)

비동기 제출 후 `e.currentTarget`은 **null**이 될 수 있으므로, `await` 이후 `e.currentTarget.reset()`을 쓰지 않는다.  
`useRef<HTMLFormElement>` 또는 제출 **직전**에 `const form = e.currentTarget`을 변수에 담아 사용한다.

```tsx
const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const form = e.currentTarget
  const formData = new FormData(form)
  const formProps = Object.fromEntries(formData.entries()) as {
    email: string
    username: string
    nickname: string
    password: string
  }

  const { email, username, nickname, password } = formProps
  // 유효성 검사 → API 호출 → 성공 시 form.reset()
}
```

`FormData` + `Object.fromEntries`로 input `name`과 1:1 매핑한다. 필드가 늘어나도 `useState`를 늘리지 않는다.

---

## 3. UI 상태는 객체 하나로 압축

로딩·메시지·에러 등 **한 화면에서 같이 바뀌는 UI 상태**는 분리하지 않고 객체로 관리한다.

### 나쁜 예

```tsx
const [submitting, setSubmitting] = useState(false)
const [error, setError] = useState<string | null>(null)
const [success, setSuccess] = useState<string | null>(null)
```

### 좋은 예

```tsx
type SignupUiState = {
  submitting: boolean
  message: { type: "ok" | "err"; text: string } | null
}

const [ui, setUi] = useState<SignupUiState>({
  submitting: false,
  message: null,
})

// 갱신 예
setUi({ submitting: true, message: null })
setUi({ submitting: false, message: { type: "ok", text: "회원가입이 완료되었습니다." } })
```

필요 시 함수형 업데이트: `setUi((prev) => ({ ...prev, submitting: true }))`

---

## 4. `alert` 대신 인라인 UI

- `alert()` 사용하지 않는다.
- 성공/실패는 폼 아래 배너·`role="status"` 텍스트로 표시한다.

---

## 5. 리팩터링 시 체크리스트

에이전트가 폼/페이지를 수정할 때:

1. `useState`가 3개 이상이면 객체 병합 또는 `FormData` 패턴 검토
2. `onSubmit` async 안에서 `e.currentTarget` 재사용 여부 확인
3. 기존 스타일(`MaestroLightBackdrop`, glass 카드, Tailwind 톤) 유지
4. 요청 범위 밖 리팩터링 금지

---

## 6. Cursor에 붙여넣을 프롬프트 (수동용)

규칙 파일이 적용되지 않을 때만 아래를 복사해 사용한다.

```text
@docs/DevOps/Frontend/REACT_RULES.md 를 따르세요.

React에서 useState는 많이 쓰지 마세요. 다음 코드를 참고해 여러 개의 useState를 하나의 객체 state로 압축하고, 폼은 FormData + Object.fromEntries 패턴으로 바꿔 주세요.

const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const formData = new FormData(e.currentTarget)
  const formProps = Object.fromEntries(formData.entries())
  // ...
}

await 이후 e.currentTarget.reset()은 쓰지 말고 form ref 또는 제출 직전 form 변수를 사용하세요.
```

---

## 7. 참고 경로

| 항목 | 경로 |
|------|------|
| DevOps 인덱스 | `docs/DevOps/README.md` |
| 백엔드 규칙 | `docs/DevOps/Backend/BACKEND_RULES.md` |
| Cursor 규칙 (자동) | `.cursor/rules/frontend-react-state.mdc` |
| 회원가입 예시 | `frontend/app/signup/page.tsx` |
| 로그인 | `frontend/app/login/page.tsx` |
# React 프론트엔드 코딩 규칙 (Maestro)

이 문서는 **Cursor / 에이전트**가 `frontend/` 작업 시 따를 React 규칙입니다.  
프로젝트 루트 `.cursor/rules/frontend-react-state.mdc`와 연동되어, 별도 프롬프트 없이도 적용됩니다.

---

## 1. `useState`는 적게 쓴다

- **원칙:** `useState`를 필드마다 여러 개 두지 않는다. 관련 상태는 **하나의 객체**로 묶는다.
- **예외:** 서로 독립·갱신 주기가 다른 상태(예: 모달 open/close만)는 분리해도 된다.
- **지양:** `email`, `username`, `password` 각각 `useState` 3개 → **권장:** `form` 객체 1개 또는 비제어 폼 + `FormData`.

---

## 2. 폼 제출 패턴 (표준)

비동기 제출 후 `e.currentTarget`은 **null**이 될 수 있으므로, `await` 이후 `e.currentTarget.reset()`을 쓰지 않는다.  
`useRef<HTMLFormElement>` 또는 제출 **직전**에 `const form = e.currentTarget`을 변수에 담아 사용한다.

```tsx
const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const form = e.currentTarget
  const formData = new FormData(form)
  const formProps = Object.fromEntries(formData.entries()) as {
    email: string
    username: string
    nickname: string
    password: string
  }

  const { email, username, nickname, password } = formProps
  // 유효성 검사 → API 호출 → 성공 시 form.reset()
}
```

`FormData` + `Object.fromEntries`로 input `name`과 1:1 매핑한다. 필드가 늘어나도 `useState`를 늘리지 않는다.

---

## 3. UI 상태는 객체 하나로 압축

로딩·메시지·에러 등 **한 화면에서 같이 바뀌는 UI 상태**는 분리하지 않고 객체로 관리한다.

### 나쁜 예

```tsx
const [submitting, setSubmitting] = useState(false)
const [error, setError] = useState<string | null>(null)
const [success, setSuccess] = useState<string | null>(null)
```

### 좋은 예

```tsx
type SignupUiState = {
  submitting: boolean
  message: { type: "ok" | "err"; text: string } | null
}

const [ui, setUi] = useState<SignupUiState>({
  submitting: false,
  message: null,
})

// 갱신 예
setUi({ submitting: true, message: null })
setUi({ submitting: false, message: { type: "ok", text: "회원가입이 완료되었습니다." } })
```

필요 시 함수형 업데이트: `setUi((prev) => ({ ...prev, submitting: true }))`

---

## 4. `alert` 대신 인라인 UI

- `alert()` 사용하지 않는다.
- 성공/실패는 폼 아래 배너·`role="status"` 텍스트로 표시한다.

---

## 5. 리팩터링 시 체크리스트

에이전트가 폼/페이지를 수정할 때:

1. `useState`가 3개 이상이면 객체 병합 또는 `FormData` 패턴 검토
2. `onSubmit` async 안에서 `e.currentTarget` 재사용 여부 확인
3. 기존 스타일(`MaestroLightBackdrop`, glass 카드, Tailwind 톤) 유지
4. 요청 범위 밖 리팩터링 금지

---

## 6. Cursor에 붙여넣을 프롬프트 (수동용)

규칙 파일이 적용되지 않을 때만 아래를 복사해 사용한다.

```text
@docs/DevOps/Frontend/REACT_RULES.md 를 따르세요.

React에서 useState는 많이 쓰지 마세요. 다음 코드를 참고해 여러 개의 useState를 하나의 객체 state로 압축하고, 폼은 FormData + Object.fromEntries 패턴으로 바꿔 주세요.

const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const formData = new FormData(e.currentTarget)
  const formProps = Object.fromEntries(formData.entries())
  // ...
}

await 이후 e.currentTarget.reset()은 쓰지 말고 form ref 또는 제출 직전 form 변수를 사용하세요.
```

---

## 7. 참고 경로

| 항목 | 경로 |
|------|------|
| Cursor 규칙 (자동) | `.cursor/rules/frontend-react-state.mdc` |
| 회원가입 예시 | `frontend/app/signup/page.tsx` |
| 로그인 | `frontend/app/login/page.tsx` |
# React 프론트엔드 코딩 규칙 (Maestro)

이 문서는 **Cursor / 에이전트**가 `frontend/` 작업 시 따를 React 규칙입니다.  
프로젝트 루트 `.cursor/rules/frontend-react-state.mdc`와 연동되어, 별도 프롬프트 없이도 적용됩니다.

---

## 1. `useState`는 적게 쓴다

- **원칙:** `useState`를 필드마다 여러 개 두지 않는다. 관련 상태는 **하나의 객체**로 묶는다.
- **예외:** 서로 독립·갱신 주기가 다른 상태(예: 모달 open/close만)는 분리해도 된다.
- **지양:** `email`, `username`, `password` 각각 `useState` 3개 → **권장:** `form` 객체 1개 또는 비제어 폼 + `FormData`.

---

## 2. 폼 제출 패턴 (표준)

비동기 제출 후 `e.currentTarget`은 **null**이 될 수 있으므로, `await` 이후 `e.currentTarget.reset()`을 쓰지 않는다.  
`useRef<HTMLFormElement>` 또는 제출 **직전**에 `const form = e.currentTarget`을 변수에 담아 사용한다.

```tsx
const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const form = e.currentTarget
  const formData = new FormData(form)
  const formProps = Object.fromEntries(formData.entries()) as {
    email: string
    username: string
    nickname: string
    password: string
  }

  const { email, username, nickname, password } = formProps
  // 유효성 검사 → API 호출 → 성공 시 form.reset()
}
```

`FormData` + `Object.fromEntries`로 input `name`과 1:1 매핑한다. 필드가 늘어나도 `useState`를 늘리지 않는다.

---

## 3. UI 상태는 객체 하나로 압축

로딩·메시지·에러 등 **한 화면에서 같이 바뀌는 UI 상태**는 분리하지 않고 객체로 관리한다.

### 나쁜 예

```tsx
const [submitting, setSubmitting] = useState(false)
const [error, setError] = useState<string | null>(null)
const [success, setSuccess] = useState<string | null>(null)
```

### 좋은 예

```tsx
type SignupUiState = {
  submitting: boolean
  message: { type: "ok" | "err"; text: string } | null
}

const [ui, setUi] = useState<SignupUiState>({
  submitting: false,
  message: null,
})

// 갱신 예
setUi({ submitting: true, message: null })
setUi({ submitting: false, message: { type: "ok", text: "회원가입이 완료되었습니다." } })
```

필요 시 함수형 업데이트: `setUi((prev) => ({ ...prev, submitting: true }))`

---

## 4. `alert` 대신 인라인 UI

- `alert()` 사용하지 않는다.
- 성공/실패는 폼 아래 배너·`role="status"` 텍스트로 표시한다.

---

## 5. 리팩터링 시 체크리스트

에이전트가 폼/페이지를 수정할 때:

1. `useState`가 3개 이상이면 객체 병합 또는 `FormData` 패턴 검토
2. `onSubmit` async 안에서 `e.currentTarget` 재사용 여부 확인
3. 기존 스타일(`MaestroLightBackdrop`, glass 카드, Tailwind 톤) 유지
4. 요청 범위 밖 리팩터링 금지

---

## 6. Cursor에 붙여넣을 프롬프트 (수동용)

규칙 파일이 적용되지 않을 때만 아래를 복사해 사용한다.

```text
@docs/DevOps/Frontend/REACT_RULES.md 를 따르세요.

React에서 useState는 많이 쓰지 마세요. 다음 코드를 참고해 여러 개의 useState를 하나의 객체 state로 압축하고, 폼은 FormData + Object.fromEntries 패턴으로 바꿔 주세요.

const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const formData = new FormData(e.currentTarget)
  const formProps = Object.fromEntries(formData.entries())
  // ...
}

await 이후 e.currentTarget.reset()은 쓰지 말고 form ref 또는 제출 직전 form 변수를 사용하세요.
```

---

## 7. 참고 경로

| 항목 | 경로 |
|------|------|
| Cursor 규칙 (자동) | `.cursor/rules/frontend-react-state.mdc` |
| 회원가입 예시 | `frontend/app/signup/page.tsx` |
| 로그인 | `frontend/app/login/page.tsx` |
# React 프론트엔드 코딩 규칙 (Maestro)

이 문서는 **Cursor / 에이전트**가 `frontend/` 작업 시 따를 React 규칙입니다.  
프로젝트 루트 `.cursor/rules/frontend-react-state.mdc`와 연동되어, 별도 프롬프트 없이도 적용됩니다.

---

## 1. `useState`는 적게 쓴다

- **원칙:** `useState`를 필드마다 여러 개 두지 않는다. 관련 상태는 **하나의 객체**로 묶는다.
- **예외:** 서로 독립·갱신 주기가 다른 상태(예: 모달 open/close만)는 분리해도 된다.
- **지양:** `email`, `username`, `password` 각각 `useState` 3개 → **권장:** `form` 객체 1개 또는 비제어 폼 + `FormData`.

---

## 2. 폼 제출 패턴 (표준)

비동기 제출 후 `e.currentTarget`은 **null**이 될 수 있으므로, `await` 이후 `e.currentTarget.reset()`을 쓰지 않는다.  
`useRef<HTMLFormElement>` 또는 제출 **직전**에 `const form = e.currentTarget`을 변수에 담아 사용한다.

```tsx
const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const form = e.currentTarget
  const formData = new FormData(form)
  const formProps = Object.fromEntries(formData.entries()) as {
    email: string
    username: string
    nickname: string
    password: string
  }

  const { email, username, nickname, password } = formProps
  // 유효성 검사 → API 호출 → 성공 시 form.reset()
}
```

`FormData` + `Object.fromEntries`로 input `name`과 1:1 매핑한다. 필드가 늘어나도 `useState`를 늘리지 않는다.

---

## 3. UI 상태는 객체 하나로 압축

로딩·메시지·에러 등 **한 화면에서 같이 바뀌는 UI 상태**는 분리하지 않고 객체로 관리한다.

### 나쁜 예

```tsx
const [submitting, setSubmitting] = useState(false)
const [error, setError] = useState<string | null>(null)
const [success, setSuccess] = useState<string | null>(null)
```

### 좋은 예

```tsx
type SignupUiState = {
  submitting: boolean
  message: { type: "ok" | "err"; text: string } | null
}

const [ui, setUi] = useState<SignupUiState>({
  submitting: false,
  message: null,
})

// 갱신 예
setUi({ submitting: true, message: null })
setUi({ submitting: false, message: { type: "ok", text: "회원가입이 완료되었습니다." } })
```

필요 시 함수형 업데이트: `setUi((prev) => ({ ...prev, submitting: true }))`

---

## 4. `alert` 대신 인라인 UI

- `alert()` 사용하지 않는다.
- 성공/실패는 폼 아래 배너·`role="status"` 텍스트로 표시한다.

---

## 5. 리팩터링 시 체크리스트

에이전트가 폼/페이지를 수정할 때:

1. `useState`가 3개 이상이면 객체 병합 또는 `FormData` 패턴 검토
2. `onSubmit` async 안에서 `e.currentTarget` 재사용 여부 확인
3. 기존 스타일(`MaestroLightBackdrop`, glass 카드, Tailwind 톤) 유지
4. 요청 범위 밖 리팩터링 금지

---

## 6. Cursor에 붙여넣을 프롬프트 (수동용)

규칙 파일이 적용되지 않을 때만 아래를 복사해 사용한다.

```text
@docs/DevOps/Frontend/REACT_RULES.md 를 따르세요.

React에서 useState는 많이 쓰지 마세요. 다음 코드를 참고해 여러 개의 useState를 하나의 객체 state로 압축하고, 폼은 FormData + Object.fromEntries 패턴으로 바꿔 주세요.

const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const formData = new FormData(e.currentTarget)
  const formProps = Object.fromEntries(formData.entries())
  // ...
}

await 이후 e.currentTarget.reset()은 쓰지 말고 form ref 또는 제출 직전 form 변수를 사용하세요.
```

---

## 7. 참고 경로

| 항목 | 경로 |
|------|------|
| Cursor 규칙 (자동) | `.cursor/rules/frontend-react-state.mdc` |
| 회원가입 예시 | `frontend/app/signup/page.tsx` |
| 로그인 | `frontend/app/login/page.tsx` |
# React 프론트엔드 코딩 규칙 (Maestro)

이 문서는 **Cursor / 에이전트**가 `frontend/` 작업 시 따를 React 규칙입니다.  
프로젝트 루트 `.cursor/rules/frontend-react-state.mdc`와 연동되어, 별도 프롬프트 없이도 적용됩니다.

---

## 1. `useState`는 적게 쓴다

- **원칙:** `useState`를 필드마다 여러 개 두지 않는다. 관련 상태는 **하나의 객체**로 묶는다.
- **예외:** 서로 독립·갱신 주기가 다른 상태(예: 모달 open/close만)는 분리해도 된다.
- **지양:** `email`, `username`, `password` 각각 `useState` 3개 → **권장:** `form` 객체 1개 또는 비제어 폼 + `FormData`.

---

## 2. 폼 제출 패턴 (표준)

비동기 제출 후 `e.currentTarget`은 **null**이 될 수 있으므로, `await` 이후 `e.currentTarget.reset()`을 쓰지 않는다.  
`useRef<HTMLFormElement>` 또는 제출 **직전**에 `const form = e.currentTarget`을 변수에 담아 사용한다.

```tsx
const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const form = e.currentTarget
  const formData = new FormData(form)
  const formProps = Object.fromEntries(formData.entries()) as {
    email: string
    username: string
    nickname: string
    password: string
  }

  const { email, username, nickname, password } = formProps
  // 유효성 검사 → API 호출 → 성공 시 form.reset()
}
```

`FormData` + `Object.fromEntries`로 input `name`과 1:1 매핑한다. 필드가 늘어나도 `useState`를 늘리지 않는다.

---

## 3. UI 상태는 객체 하나로 압축

로딩·메시지·에러 등 **한 화면에서 같이 바뀌는 UI 상태**는 분리하지 않고 객체로 관리한다.

### 나쁜 예

```tsx
const [submitting, setSubmitting] = useState(false)
const [error, setError] = useState<string | null>(null)
const [success, setSuccess] = useState<string | null>(null)
```

### 좋은 예

```tsx
type SignupUiState = {
  submitting: boolean
  message: { type: "ok" | "err"; text: string } | null
}

const [ui, setUi] = useState<SignupUiState>({
  submitting: false,
  message: null,
})

// 갱신 예
setUi({ submitting: true, message: null })
setUi({ submitting: false, message: { type: "ok", text: "회원가입이 완료되었습니다." } })
```

필요 시 함수형 업데이트: `setUi((prev) => ({ ...prev, submitting: true }))`

---

## 4. `alert` 대신 인라인 UI

- `alert()` 사용하지 않는다.
- 성공/실패는 폼 아래 배너·`role="status"` 텍스트로 표시한다.

---

## 5. 리팩터링 시 체크리스트

에이전트가 폼/페이지를 수정할 때:

1. `useState`가 3개 이상이면 객체 병합 또는 `FormData` 패턴 검토
2. `onSubmit` async 안에서 `e.currentTarget` 재사용 여부 확인
3. 기존 스타일(`MaestroLightBackdrop`, glass 카드, Tailwind 톤) 유지
4. 요청 범위 밖 리팩터링 금지

---

## 6. Cursor에 붙여넣을 프롬프트 (수동용)

규칙 파일이 적용되지 않을 때만 아래를 복사해 사용한다.

```text
@docs/DevOps/Frontend/REACT_RULES.md 를 따르세요.

React에서 useState는 많이 쓰지 마세요. 다음 코드를 참고해 여러 개의 useState를 하나의 객체 state로 압축하고, 폼은 FormData + Object.fromEntries 패턴으로 바꿔 주세요.

const handleSignup = async (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault()
  const formData = new FormData(e.currentTarget)
  const formProps = Object.fromEntries(formData.entries())
  // ...
}

await 이후 e.currentTarget.reset()은 쓰지 말고 form ref 또는 제출 직전 form 변수를 사용하세요.
```

---

## 7. 참고 경로

| 항목 | 경로 |
|------|------|
| Cursor 규칙 (자동) | `.cursor/rules/frontend-react-state.mdc` |
| 회원가입 예시 | `frontend/app/signup/page.tsx` |
| 로그인 | `frontend/app/login/page.tsx` |
