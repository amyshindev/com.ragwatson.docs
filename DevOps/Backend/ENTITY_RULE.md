 # 엔티티·테이블 설계 규칙 (Maestro)

이 문서는 **모든 DB 테이블**에 적용되는 기본 키 규칙입니다.  
DB 접근 계층은 `docs/DevOps/Backend/BACKEND_RULES.md` §3과 함께 참고합니다.

---

## 1. 기본 키 `id` (필수)

- **모든 테이블**은 시스템 내부용 **자동 증가 정수** 기본 키를 둔다.
- 컬럼명·ORM 필드명을 **`id`로 통일**한다. (`user_id`를 PK 컬럼명으로 쓰지 않는다. 외래 키는 `user_id` 등 도메인 접두사를 쓰면 된다.)
- 타입은 DB·ORM에서 **정수(`int` / `INTEGER`)** 이다. UUID·문자열·복합 기본 키는 별도 합의 없이 사용하지 않는다.

---

## 2. SQLModel 선언 예시 (참조)

시스템 내부용 자동 증감 고유 번호(기본 키)는 아래 패턴을 따른다.

```python
from typing import Optional

from sqlmodel import Field, SQLModel


class ExampleEntity(SQLModel, table=True):
    __tablename__ = "example_entities"

    # 시스템 내부용 자동 증감 고유 번호 (기본 키)
    id: Optional[int] = Field(
        default=None,
        primary_key=True,
        sa_column_kwargs={"name": "id"},  # DB 컬럼명: id
    )
```

- `default=None`은 삽입 시 DB가 시퀀스·`AUTO_INCREMENT`로 값을 채우도록 두는 일반적인 SQLModel 관례다.
- `sa_column_kwargs={"name": "id"}`로 **물리 컬럼명이 반드시 `id`**임을 명시한다 (매핑 클래스 속성명과 DB가 어긋날 때도 동일 규칙을 유지할 수 있다).

---

## 3. SQLAlchemy 2.0 선언 스타일 (현재 코드베이스)

`Mapped` / `mapped_column`을 쓰는 경우, 속성명 `id`가 곧 컬럼명 `id`가 되도록 한다.

```python
from sqlalchemy.orm import Mapped, mapped_column

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
```

컬럼명을 속성과 다르게 둘 필요가 있을 때만 첫 인자로 이름을 넘긴다. **PK는 항시 `id` 컬럼**이어야 한다.

---

## 4. 체크리스트

| 항목 | 요구 |
|------|------|
| PK 컬럼명 | `id` |
| PK 타입 | 정수, 자동 증가 |
| 외래 키 | `{참조_엔티티}_id` 등 (PK와 혼동되지 않게) |

새 테이블·마이그레이션 작성 전에 위 표를 만족하는지 확인한다.
