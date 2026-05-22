# Maestro — ML DB 4-Layer 구현 작업지시서

| 항목 | 내용 |
|------|------|
| 작성 기준 | `개발정의서.md` v1.0 기반 |
| 작업 대상 | `backend/apps/` 내 신규 모듈 및 기존 ERD 확장 |
| 목표 | AI 에이전트 학습용 4-Layer 데이터 수집 파이프라인 구축 |
| 실행 순서 | ORM → db_init → schemas → router → orm_registry 등록 → Alembic |

---

## 0. 전제 조건 확인

작업 전 아래를 반드시 확인한다.

- `backend/apps/main.py` — `domain_intake_router` 등록 방식 참고
- `backend/apps/orm_registry.py` — `import_all_models()` 에 신규 모듈 추가 필요
- `backend/apps/domain_intake/models/` — 기존 ORM 패턴 참고 (FK 관계, Base 상속)
- `DATABASE_URL` 환경변수 설정 여부 확인

---

## 1. 디렉터리 구조 생성

아래 구조를 `backend/apps/` 아래에 생성한다.

```
backend/apps/
└── ml_data/
    ├── __init__.py
    ├── db_init.py
    ├── models/
    │   ├── __init__.py
    │   ├── audio_features.py      # Layer 1
    │   ├── user_events.py         # Layer 2
    │   ├── generation_logs.py     # Layer 3
    │   └── visual_ratings.py      # Layer 4
    ├── schemas/
    │   ├── __init__.py
    │   ├── audio_features.py
    │   ├── user_events.py
    │   ├── generation_logs.py
    │   └── visual_ratings.py
    ├── repository/
    │   ├── __init__.py
    │   ├── audio_features.py
    │   ├── user_events.py
    │   ├── generation_logs.py
    │   └── visual_ratings.py
    ├── service/
    │   ├── __init__.py
    │   └── ml_data_service.py
    └── router.py
```

---

## 2. Layer 1 — `audio_features` 테이블

### 2-1. ORM 모델 (`ml_data/models/audio_features.py`)

```python
import uuid
from datetime import datetime, timezone
from sqlalchemy import Column, String, Float, Integer, ARRAY, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
from backend.apps.db import Base  # 기존 Base import 경로에 맞게 수정


class AudioFeature(Base):
    __tablename__ = "audio_features"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    workspace_id = Column(UUID(as_uuid=True), ForeignKey("studio_workspaces.id"), nullable=True)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)

    # 음악 기본 특성
    bpm = Column(Float, nullable=True)                  # 템포
    energy = Column(Float, nullable=True)               # 에너지 강도 0.0~1.0
    valence = Column(Float, nullable=True)              # 감정 긍정도 0.0~1.0
    danceability = Column(Float, nullable=True)         # 댄스 가능성 0.0~1.0
    spectral_centroid = Column(Float, nullable=True)    # 음색 밝기
    loudness = Column(Float, nullable=True)             # dB 단위 음량
    key = Column(Integer, nullable=True)                # 0=C, 1=C#, ..., 11=B
    mode = Column(Integer, nullable=True)               # 0=minor, 1=major

    # 장르·무드 레이블 (AI 분석 결과)
    genre_primary = Column(String(100), nullable=True)  # "dark ambient", "industrial"
    genre_secondary = Column(String(100), nullable=True)
    mood_tags = Column(ARRAY(String), nullable=True)    # ["melancholic", "intense"]

    # 소스 추적
    source = Column(String(50), nullable=False, default="upload")
    # "upload" | "spotify_link" | "youtube_link"
    source_url = Column(String(500), nullable=True)
    duration_sec = Column(Float, nullable=True)

    created_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))

    # Relationships
    workspace = relationship("StudioWorkspace", back_populates="audio_features", lazy="selectin")
    generation_logs = relationship("GenerationLog", back_populates="audio_feature")
```

> **주의:** `StudioWorkspace` 모델에 `audio_features = relationship(...)` 역참조 추가 필요

### 2-2. Pydantic 스키마 (`ml_data/schemas/audio_features.py`)

```python
from uuid import UUID
from typing import Optional, List
from pydantic import BaseModel, Field


class AudioFeatureCreate(BaseModel):
    workspace_id: Optional[UUID] = None
    bpm: Optional[float] = Field(None, ge=20, le=300)
    energy: Optional[float] = Field(None, ge=0.0, le=1.0)
    valence: Optional[float] = Field(None, ge=0.0, le=1.0)
    danceability: Optional[float] = Field(None, ge=0.0, le=1.0)
    spectral_centroid: Optional[float] = None
    loudness: Optional[float] = None
    key: Optional[int] = Field(None, ge=0, le=11)
    mode: Optional[int] = Field(None, ge=0, le=1)
    genre_primary: Optional[str] = None
    genre_secondary: Optional[str] = None
    mood_tags: Optional[List[str]] = None
    source: str = "upload"
    source_url: Optional[str] = None
    duration_sec: Optional[float] = None


class AudioFeatureRead(AudioFeatureCreate):
    id: UUID
    user_id: UUID
    created_at: str

    class Config:
        from_attributes = True
```

---

## 3. Layer 2 — `user_events` 테이블

### 3-1. ORM 모델 (`ml_data/models/user_events.py`)

```python
import uuid
from datetime import datetime, timezone
from sqlalchemy import Column, String, Integer, DateTime, ForeignKey
from sqlalchemy.dialects.postgresql import UUID, JSONB
from backend.apps.db import Base


class UserEvent(Base):
    __tablename__ = "user_events"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    session_id = Column(String(100), nullable=True)     # 프론트에서 생성한 세션 식별자

    # 이벤트 분류
    event_type = Column(String(50), nullable=False)
    # "visual_select" | "style_edit" | "export" | "skip" | "loop_play" | "gallery_like"

    target_id = Column(UUID(as_uuid=True), nullable=True)   # 대상 비주얼/로그 ID
    target_type = Column(String(50), nullable=True)          # "generation" | "gallery_item"

    # 행동 지표
    dwell_ms = Column(Integer, nullable=True)           # 체류 시간 (ms)
    payload = Column(JSONB, nullable=True)
    # 예시: {"color_temp": "warm", "speed": 1.2, "style": "glitch"}
    # 편집 이벤트일 때 변경 전/후 diff 저장

    created_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))
```

### 3-2. Pydantic 스키마 (`ml_data/schemas/user_events.py`)

```python
from uuid import UUID
from typing import Optional, Any, Dict
from pydantic import BaseModel


class UserEventCreate(BaseModel):
    session_id: Optional[str] = None
    event_type: str
    target_id: Optional[UUID] = None
    target_type: Optional[str] = None
    dwell_ms: Optional[int] = None
    payload: Optional[Dict[str, Any]] = None


class UserEventRead(UserEventCreate):
    id: UUID
    user_id: UUID
    created_at: str

    class Config:
        from_attributes = True
```

---

## 4. Layer 3 — `generation_logs` 테이블

### 4-1. ORM 모델 (`ml_data/models/generation_logs.py`)

```python
import uuid
from datetime import datetime, timezone
from sqlalchemy import Column, String, Float, Integer, DateTime, ForeignKey, Text
from sqlalchemy.dialects.postgresql import UUID, JSONB, ARRAY
from sqlalchemy.orm import relationship
from backend.apps.db import Base


class GenerationLog(Base):
    __tablename__ = "generation_logs"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    workspace_id = Column(UUID(as_uuid=True), ForeignKey("studio_workspaces.id"), nullable=True)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    audio_feature_id = Column(UUID(as_uuid=True), ForeignKey("audio_features.id"), nullable=True)

    # AI 생성 파라미터 (입력)
    prompt_params = Column(JSONB, nullable=True)
    # 예시: {"genre": "industrial", "mood": "dark", "palette": "monochrome", "speed": 1.0}

    model_version = Column(String(100), nullable=True)  # 어떤 AI 모델이 생성했는지
    pipeline_version = Column(String(50), nullable=True)

    # 생성 결과 (출력)
    output_asset_url = Column(Text, nullable=True)       # 생성된 비주얼 URL
    render_ms = Column(Integer, nullable=True)           # 렌더 소요 시간
    quality_score = Column(Float, nullable=True)         # 자동 품질 점수 (CLIP score 등)

    # ML 피처: 장르-스타일 임베딩 벡터 (유사 비주얼 검색에 활용)
    style_vector = Column(ARRAY(Float), nullable=True)

    status = Column(String(30), nullable=False, default="pending")
    # "pending" | "completed" | "failed"
    error_message = Column(Text, nullable=True)

    created_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))
    completed_at = Column(DateTime(timezone=True), nullable=True)

    # Relationships
    audio_feature = relationship("AudioFeature", back_populates="generation_logs")
    visual_ratings = relationship("VisualRating", back_populates="generation_log")
```

### 4-2. Pydantic 스키마 (`ml_data/schemas/generation_logs.py`)

```python
from uuid import UUID
from typing import Optional, Any, Dict, List
from pydantic import BaseModel


class GenerationLogCreate(BaseModel):
    workspace_id: Optional[UUID] = None
    audio_feature_id: Optional[UUID] = None
    prompt_params: Optional[Dict[str, Any]] = None
    model_version: Optional[str] = None
    pipeline_version: Optional[str] = None
    output_asset_url: Optional[str] = None
    render_ms: Optional[int] = None
    quality_score: Optional[float] = None
    style_vector: Optional[List[float]] = None
    status: str = "pending"
    error_message: Optional[str] = None


class GenerationLogRead(GenerationLogCreate):
    id: UUID
    user_id: UUID
    created_at: str
    completed_at: Optional[str] = None

    class Config:
        from_attributes = True
```

---

## 5. Layer 4 — `visual_ratings` 테이블

### 5-1. ORM 모델 (`ml_data/models/visual_ratings.py`)

```python
import uuid
from datetime import datetime, timezone
from sqlalchemy import Column, String, Integer, Boolean, DateTime, ForeignKey, Text
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import relationship
from backend.apps.db import Base


class VisualRating(Base):
    __tablename__ = "visual_ratings"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    generation_id = Column(UUID(as_uuid=True), ForeignKey("generation_logs.id"), nullable=False)
    rater_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)

    # 정량 평가 (지도학습 레이블)
    aesthetic_score = Column(Integer, nullable=True)        # 심미성 1~5
    genre_match_score = Column(Integer, nullable=True)      # 장르 일치도 1~5
    mood_match_score = Column(Integer, nullable=True)       # 무드 일치도 1~5

    # A/B 비교 결과
    ab_test_id = Column(String(100), nullable=True)         # A/B 실험 식별자
    ab_winner = Column(Boolean, nullable=True)              # 이 항목이 이겼는지

    # 콘텐츠 안전
    flag = Column(String(30), nullable=False, default="ok")
    # "ok" | "inappropriate" | "off_brand" | "low_quality"
    flag_reason = Column(Text, nullable=True)

    # 레이터 유형
    rater_type = Column(String(20), nullable=False, default="user")
    # "user" | "admin" | "auto" (자동 품질 평가)

    created_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))

    # Relationships
    generation_log = relationship("GenerationLog", back_populates="visual_ratings")
```

### 5-2. Pydantic 스키마 (`ml_data/schemas/visual_ratings.py`)

```python
from uuid import UUID
from typing import Optional
from pydantic import BaseModel, Field


class VisualRatingCreate(BaseModel):
    generation_id: UUID
    aesthetic_score: Optional[int] = Field(None, ge=1, le=5)
    genre_match_score: Optional[int] = Field(None, ge=1, le=5)
    mood_match_score: Optional[int] = Field(None, ge=1, le=5)
    ab_test_id: Optional[str] = None
    ab_winner: Optional[bool] = None
    flag: str = "ok"
    flag_reason: Optional[str] = None
    rater_type: str = "user"


class VisualRatingRead(VisualRatingCreate):
    id: UUID
    rater_id: UUID
    created_at: str

    class Config:
        from_attributes = True
```

---

## 6. Repository 레이어

각 모델별 CRUD repository를 생성한다. 아래는 `audio_features` 예시이며, 나머지 3개도 동일 패턴 적용.

### `ml_data/repository/audio_features.py`

```python
from uuid import UUID
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from ml_data.models.audio_features import AudioFeature
from ml_data.schemas.audio_features import AudioFeatureCreate


async def create_audio_feature(db: AsyncSession, user_id: UUID, data: AudioFeatureCreate) -> AudioFeature:
    obj = AudioFeature(user_id=user_id, **data.model_dump())
    db.add(obj)
    await db.commit()
    await db.refresh(obj)
    return obj


async def get_by_workspace(db: AsyncSession, workspace_id: UUID) -> list[AudioFeature]:
    result = await db.execute(
        select(AudioFeature).where(AudioFeature.workspace_id == workspace_id)
    )
    return result.scalars().all()


async def get_by_user(db: AsyncSession, user_id: UUID, limit: int = 50) -> list[AudioFeature]:
    result = await db.execute(
        select(AudioFeature)
        .where(AudioFeature.user_id == user_id)
        .order_by(AudioFeature.created_at.desc())
        .limit(limit)
    )
    return result.scalars().all()
```

> **나머지 repository**: `user_events.py`, `generation_logs.py`, `visual_ratings.py` 동일 패턴으로 작성

---

## 7. Service 레이어 (`ml_data/service/ml_data_service.py`)

```python
from uuid import UUID
from sqlalchemy.ext.asyncio import AsyncSession

from ml_data.repository import audio_features as af_repo
from ml_data.repository import user_events as ue_repo
from ml_data.repository import generation_logs as gl_repo
from ml_data.repository import visual_ratings as vr_repo

from ml_data.schemas.audio_features import AudioFeatureCreate
from ml_data.schemas.user_events import UserEventCreate
from ml_data.schemas.generation_logs import GenerationLogCreate
from ml_data.schemas.visual_ratings import VisualRatingCreate


async def ingest_audio_feature(db: AsyncSession, user_id: UUID, data: AudioFeatureCreate):
    return await af_repo.create_audio_feature(db, user_id, data)


async def log_user_event(db: AsyncSession, user_id: UUID, data: UserEventCreate):
    return await ue_repo.create_user_event(db, user_id, data)


async def log_generation(db: AsyncSession, user_id: UUID, data: GenerationLogCreate):
    return await gl_repo.create_generation_log(db, user_id, data)


async def submit_rating(db: AsyncSession, rater_id: UUID, data: VisualRatingCreate):
    return await vr_repo.create_visual_rating(db, rater_id, data)
```

---

## 8. Router (`ml_data/router.py`)

```python
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from db import get_db                              # 기존 db dependency 경로에 맞게 수정
from secom.app.utils.auth import get_current_user  # 기존 인증 유틸 경로에 맞게 수정

from ml_data.service.ml_data_service import (
    ingest_audio_feature,
    log_user_event,
    log_generation,
    submit_rating,
)
from ml_data.schemas.audio_features import AudioFeatureCreate, AudioFeatureRead
from ml_data.schemas.user_events import UserEventCreate, UserEventRead
from ml_data.schemas.generation_logs import GenerationLogCreate, GenerationLogRead
from ml_data.schemas.visual_ratings import VisualRatingCreate, VisualRatingRead

router = APIRouter(prefix="/api/ml", tags=["ml-data"])


@router.post("/audio-features", response_model=AudioFeatureRead)
async def create_audio_feature(
    data: AudioFeatureCreate,
    db: AsyncSession = Depends(get_db),
    current_user=Depends(get_current_user),
):
    return await ingest_audio_feature(db, current_user.id, data)


@router.post("/events", response_model=UserEventRead)
async def create_user_event(
    data: UserEventCreate,
    db: AsyncSession = Depends(get_db),
    current_user=Depends(get_current_user),
):
    return await log_user_event(db, current_user.id, data)


@router.post("/generations", response_model=GenerationLogRead)
async def create_generation_log(
    data: GenerationLogCreate,
    db: AsyncSession = Depends(get_db),
    current_user=Depends(get_current_user),
):
    return await log_generation(db, current_user.id, data)


@router.post("/ratings", response_model=VisualRatingRead)
async def create_visual_rating(
    data: VisualRatingCreate,
    db: AsyncSession = Depends(get_db),
    current_user=Depends(get_current_user),
):
    return await submit_rating(db, current_user.id, data)
```

---

## 9. db_init (`ml_data/db_init.py`)

```python
from sqlalchemy.ext.asyncio import AsyncEngine
from db import Base

# 모든 모델 import (테이블 메타데이터 등록)
from ml_data.models.audio_features import AudioFeature
from ml_data.models.user_events import UserEvent
from ml_data.models.generation_logs import GenerationLog
from ml_data.models.visual_ratings import VisualRating


async def create_ml_tables(engine: AsyncEngine):
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

---

## 10. 기존 파일 수정 사항

### 10-1. `orm_registry.py` — 신규 모델 등록

```python
# 기존 import 목록 아래에 추가
from ml_data.models.audio_features import AudioFeature      # noqa
from ml_data.models.user_events import UserEvent            # noqa
from ml_data.models.generation_logs import GenerationLog    # noqa
from ml_data.models.visual_ratings import VisualRating      # noqa
```

### 10-2. `main.py` — 라우터 및 db_init 등록

```python
# 기존 router import 아래에 추가
from ml_data.router import router as ml_data_router
from ml_data.db_init import create_ml_tables

# app.include_router 목록에 추가
app.include_router(ml_data_router)

# lifespan 또는 startup 이벤트에 추가
@app.on_event("startup")
async def startup():
    # ... 기존 db_init 호출 유지 ...
    await create_ml_tables(engine)
```

### 10-3. `StudioWorkspace` 모델 — 역참조 추가

```python
# studio_workspaces ORM 파일에 추가
audio_features = relationship("AudioFeature", back_populates="workspace", lazy="selectin")
```

---

## 11. Alembic 마이그레이션

```bash
# backend/ 디렉터리에서 실행
alembic revision --autogenerate -m "add_ml_data_4_layers"
alembic upgrade head
```

> 마이그레이션 전에 `alembic/env.py`에서 `import_all_models()` 가 `ml_data` 모델을 포함하는지 확인한다.

---

## 12. 완료 체크리스트

- [ ] `ml_data/` 디렉터리 구조 생성
- [ ] Layer 1: `AudioFeature` ORM + 스키마 + repository
- [ ] Layer 2: `UserEvent` ORM + 스키마 + repository
- [ ] Layer 3: `GenerationLog` ORM + 스키마 + repository
- [ ] Layer 4: `VisualRating` ORM + 스키마 + repository
- [ ] `ml_data_service.py` 서비스 레이어
- [ ] `ml_data/router.py` 라우터 (4개 POST 엔드포인트)
- [ ] `orm_registry.py` 신규 모델 4개 등록
- [ ] `main.py` 라우터 마운트 + startup db_init 추가
- [ ] `StudioWorkspace` 역참조 추가
- [ ] Alembic 마이그레이션 실행 및 테이블 생성 확인
- [ ] `/docs` (Swagger)에서 `/api/ml/*` 엔드포인트 4개 노출 확인

---

## 13. 신규 API 엔드포인트 요약

| 메서드 | 경로 | 설명 | 인증 |
|--------|------|------|------|
| POST | `/api/ml/audio-features` | Layer 1 음악 분석 데이터 인입 | 필요 |
| POST | `/api/ml/events` | Layer 2 사용자 행동 이벤트 로그 | 필요 |
| POST | `/api/ml/generations` | Layer 3 AI 생성 결과 기록 | 필요 |
| POST | `/api/ml/ratings` | Layer 4 비주얼 평점·레이블 제출 | 필요 |

---

*본 작업지시서는 `개발정의서.md` v1.0 기준이며, 기존 `domain_intake` 패턴(ORM→repository→service→router)을 동일하게 적용한다.*
