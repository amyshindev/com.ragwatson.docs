---
tags:
  - harness/docs-task
graph-group: docs-task
---

# Maestro — ML DB 4-Layer 구현 작업지시서 (v3)

| 항목 | 내용 |
|------|------|
| 작성 기준 | `개발정의서.md` v1.0 기반 |
| 작업 대상 | `backend/apps/audio` (헥사고날 아키텍처) |
| 목표 | AI 에이전트 학습용 4-Layer 데이터 수집 파이프라인 구축 |
| 아키텍처 참고 | `backend/apps/titanic` (`adapter` / `app` / `dependencies`) |
| 실행 순서 | ORM → schemas → ports → pg repository → interactor → router → orm_registry → Alembic |

> **v2 변경 요약**  
> 비동기 파이프라인 전략, GET 엔드포인트, 인증 연동 계획, 학습 데이터 추출 파이프라인,  
> VisualRating 집계 UseCase, 에러 처리 전략, 체크리스트 검증 항목, AudioFeature AI 추론 필드 추가.
>
> **v3 변경 요약**  
> "How it works" 3단계 파이프라인(Drop Your Sound → AI Aesthetic Analysis → Get Your Artwork) 분석을 반영.  
> 색감·움직임·질감 변환 결과 필드, 루프 영상 메타데이터, 플랫폼별 생성 파라미터,  
> 장르/무드→비주얼 매핑 저장, 비트 싱크 분석 결과, 평가 플랫폼별 루프 품질 점수 추가.

---

## 0. 전제 조건 확인

작업 전 아래를 반드시 확인한다.

- `backend/apps/titanic/` — 헥사고날 레이어·DIP 패턴 참고 (`JamesInteractor`, `get_james_use_case`)
- `backend/apps/orm_registry.py` — `import_all_models()` 에 ML ORM 등록
- `backend/main.py` — `udio_router` 마운트 방식
- `docs/DevOps/Backend/BACKEND_RULES.md` — 레이어·DB 규칙
- `DATABASE_URL` 환경변수 설정 여부 확인
- `PYTHONPATH=apps` 로 실행 (`backend/start-backend.ps1` 참고)

---

## 1. 디렉터리 구조

`titanic` 과 동일한 헥사고날 레이아웃을 사용한다. import 루트는 `udio.*` (`PYTHONPATH=apps`).

```
backend/apps/
└── udio/
    ├── __init__.py
    ├── adapter/
    │   ├── inbound/
    │   │   └── api/
    │   │       ├── __init__.py              # udio_router 집계
    │   │       ├── schemas/                 # Pydantic Create/Read
    │   │       │   ├── audio_features.py
    │   │       │   ├── user_events.py
    │   │       │   ├── generation_logs.py
    │   │       │   ├── visual_ratings.py
    │   │       │   └── training_export.py   # ← v2 추가: 학습 데이터 추출용
    │   │       └── v1/                      # FastAPI 라우터 (레이어별)
    │   │           ├── audio_features_router.py
    │   │           ├── user_events_router.py
    │   │           ├── generation_logs_router.py
    │   │           ├── visual_ratings_router.py
    │   │           └── training_export_router.py  # ← v2 추가
    │   └── outbound/
    │       ├── orm/                         # SQLAlchemy ORM
    │       │   ├── audio_feature_orm.py
    │       │   ├── user_event_orm.py
    │       │   ├── generation_log_orm.py
    │       │   └── visual_rating_orm.py
    │       └── pg/                          # PostgreSQL 구현체
    │           ├── audio_feature_pg_repository.py
    │           ├── user_event_pg_repository.py
    │           ├── generation_log_pg_repository.py
    │           ├── visual_rating_pg_repository.py
    │           └── training_export_pg_repository.py  # ← v2 추가
    ├── app/
    │   ├── db_init.py                       # 선택: ORM side-effect import
    │   ├── ports/
    │   │   ├── input/                       # UseCase (ABC)
    │   │   └── output/                      # Repository (ABC)
    │   └── use_cases/                       # Interactor (commit/rollback)
    └── dependencies/                        # get_*_use_case() DIP 팩토리
        ├── audio_feature.py
        ├── user_event.py
        ├── generation_log.py
        ├── visual_rating.py
        └── training_export.py               # ← v2 추가
```

**데이터 흐름**

```
Router (v1) → Depends(get_*_use_case) → Interactor → Repository(port) → PgRepository → ORM
```

트랜잭션(`commit` / `rollback`)은 **Interactor** 에서 처리한다 (`JamesInteractor` 패턴).

---

## 2. 비동기 파이프라인 전략 ← v2 신규

음악 분석(5~30초)과 영상 생성(1~5분)은 단순 HTTP 요청-응답으로 처리할 수 없다.  
모든 무거운 작업은 **즉시 job_id를 반환하고 백그라운드에서 처리**하는 패턴을 따른다.

### 2-1. 서비스 3단계 파이프라인 (v3 명시)

```
[Step 1] Drop Your Sound
  → POST /api/ml/audio-features (파일 or SoundCloud/YouTube URL 인입)
  → audio_uploads 레코드 생성, job_id 즉시 반환

[Step 2] AI Aesthetic Analysis
  → 백그라운드 워커 실행
  → 1단계: BPM·에너지·무드·장르 신호 추출          → audio_features 기본 필드 저장
  → 2단계: 색감·움직임·질감 화면 언어 변환          → visual_motion_intensity, visual_texture_type 등 저장
  → 3단계: 비트 싱크 분석 (beat_timestamps, highlight 구간) → audio_features 비트 필드 저장
  → 4단계: 장르/무드 → 비주얼 매핑                  → genre_to_visual_mapping, mood_to_color_mapping 저장
  → processing_status: pending → processing → done | failed

[Step 3] Get Your Artwork
  → POST /api/ml/generations (플랫폼·루프 파라미터 포함)
  → 백그라운드 워커: 루프 영상 생성
  → loop_duration_sec, loop_beat_aligned, frame_rate 저장
  → generation_assets에 플랫폼별 파일 저장
  → status: pending → processing → completed | failed
```

### 2-2. 전체 흐름

```
① 클라이언트 → POST /api/ml/audio-features
               body: { user_id, source_type, source_url, ... }

② 서버 → 즉시 응답
          { "id": "uuid", "status": "pending" }

③ 백그라운드 워커 (Celery / ARQ / FastAPI BackgroundTasks)
   → Step 2 AI Aesthetic Analysis 4단계 순차 실행
   → audio_features.processing_status 업데이트: pending → processing → done | failed

④ 클라이언트 → GET /api/ml/audio-features/{id}/status 폴링
               or WebSocket 구독 (선택)
```

### 2-3. 상태값 정의

| 테이블 | 컬럼 | 상태 흐름 |
|--------|------|-----------|
| `audio_uploads` | `processing_status` | `pending` → `processing` → `done` \| `failed` |
| `generation_logs` | `status` | `pending` → `processing` → `completed` \| `failed` |

### 2-4. 백그라운드 워커 선택 기준

| 방식 | 사용 시점 | 비고 |
|------|-----------|------|
| `FastAPI BackgroundTasks` | 분석 시간 < 10초, 단순 작업 | 별도 인프라 불필요 |
| `ARQ` (asyncio 기반) | 분석 시간 < 2분, Redis 있는 경우 | FastAPI와 궁합 좋음 |
| `Celery` | 분석 시간 > 2분, 재시도·스케줄링 필요 | 영상 생성 단계에 권장 |

> MVP 1단계: `BackgroundTasks` → 2단계: `ARQ` → 영상 생성 시: `Celery` 순서로 전환 권장.

### 2-5. 폴링 엔드포인트 예시

```python
@audio_features_router.get("/audio-features/{feature_id}/status")
async def get_audio_feature_status(
    feature_id: uuid.UUID,
    use_case: AudioFeatureUseCase = Depends(get_audio_feature_use_case),
) -> AudioFeatureStatusRead:
    return await use_case.get_status(feature_id)
```

```python
# AudioFeatureStatusRead 스키마
class AudioFeatureStatusRead(BaseModel):
    id: uuid.UUID
    processing_status: str   # pending | processing | done | failed
    error_message: str | None
    created_at: datetime
    inferred_at: datetime | None  # 분석 완료 시각
```

---

## 3. Layer 1 — `audio_features` 테이블

### 3-1. ORM (`udio/adapter/outbound/orm/audio_feature_orm.py`)

- `Base`: `from core.matrix.oracle_database import Base`
- PK: `id` UUID
- FK: `users.id` (필수), `studio_workspaces.id` (선택)
- 음악 특성: `bpm`, `energy`, `valence`, `danceability`, `spectral_centroid`, `loudness`, `key`, `mode`
- 레이블: `genre_primary`, `genre_secondary`, `mood_tags` (ARRAY)
- 소스: `source`, `source_url`, `duration_sec`
- 관계: `generation_logs` ↔ `GenerationLog.audio_feature`

**v2 추가 — AI 추론 결과 컬럼**

음악 분석 후 AI 모델이 예측한 비주얼 파라미터를 같은 테이블에 저장한다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `predicted_visual_style` | `varchar` | AI가 예측한 비주얼 스타일 (`'glitch'`, `'neon'`, `'organic'` 등) |
| `predicted_color_palette` | `ARRAY(String)` | 예측 색상 팔레트 (`['#FF0000', '#1A1A2E']`) |
| `visual_embedding` | `ARRAY(Float)` | 비주얼 스타일 임베딩 벡터 (generation과 유사도 대조용) |
| `model_version` | `varchar` | 추론에 사용한 AI 모델 버전 |
| `inferred_at` | `timestamptz` | AI 추론 완료 시각 |

**v3 추가 — 색감·움직임·질감 변환 결과 컬럼**

"AI Aesthetic Analysis" 2단계: 음악 신호를 화면 언어로 변환한 결과를 저장한다.  
이 값들이 없으면 "어떤 음악 특성이 어떤 비주얼 스타일로 매핑됐는지" 추적·학습이 불가능하다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `visual_motion_intensity` | `Float` | 움직임 강도 (0.0~1.0) — energy·danceability에서 파생 |
| `visual_texture_type` | `varchar` | 질감 유형 (`'grain'`, `'smooth'`, `'glitch'`, `'particle'`) |
| `visual_color_temperature` | `varchar` | 색감 온도 (`'warm'`, `'cool'`, `'neon'`, `'monochrome'`) |
| `visual_rhythm_sync` | `Float` | 비트와 비주얼 싱크 강도 (0.0~1.0) |
| `genre_to_visual_mapping` | `JSONB` | 장르→비주얼 매핑 결과 `{'genre': 'Industrial_Rock', 'mapped_style': 'glitch', 'confidence': 0.92}` |
| `mood_to_color_mapping` | `JSONB` | 무드→색상 매핑 결과 `{'mood': 'dark', 'mapped_palette': ['#1A1A2E'], 'confidence': 0.87}` |

**v3 추가 — 비트 싱크 분석 결과 컬럼**

루프 영상이 BPM에 맞게 끊기려면 정확한 비트 위치와 하이라이트 구간이 필요하다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `beat_timestamps` | `ARRAY(Float)` | 비트 타임스탬프 배열 (초 단위) — 루프 싱크 포인트 |
| `highlight_start_sec` | `Float` | 하이라이트 구간 시작 (에너지 최고점 기준 8초 구간) |
| `highlight_end_sec` | `Float` | 하이라이트 구간 끝 |
| `onset_strength` | `Float` | 어택 강도 — 비주얼 파편화·글리치 강도 결정에 사용 |

> 구현은 `Mapped` / `mapped_column` 스타일. 실제 코드는 저장소 파일 참고.

### 3-2. Pydantic (`udio/adapter/inbound/api/schemas/audio_features.py`)

- `AudioFeatureCreate`: `user_id` (int, 필수), 분석 필드, `source` 기본값 `"upload"`
- `AudioFeatureRead`: `from_attributes=True`, `id`, `created_at` 포함
- `AudioFeatureStatusRead` (v2 추가): `id`, `processing_status`, `error_message`, `inferred_at`
- `AudioFeatureVisualRead` (v3 추가): 비주얼 변환 결과 전체 필드 (`visual_motion_intensity`, `visual_texture_type`, `visual_color_temperature`, `beat_timestamps`, `highlight_start_sec`, `highlight_end_sec`, `genre_to_visual_mapping`, `mood_to_color_mapping`)

### 3-3. UseCase 메서드 (v2 보완)

| 메서드 | 설명 |
|--------|------|
| `ingest(body)` | 분석 데이터 인입, job_id 반환 |
| `get(feature_id)` | 단건 조회 |
| `get_status(feature_id)` | 처리 상태 폴링 |
| `list_by_user(user_id, limit, offset)` | 사용자별 분석 이력 목록 |
| `update_inference_result(feature_id, result)` | AI 추론 완료 후 결과 저장 (워커 호출) |
| `update_visual_mapping(feature_id, mapping)` | 장르/무드→비주얼 매핑 결과 저장 (워커 호출) |
| `update_beat_analysis(feature_id, analysis)` | 비트 싱크 분석 결과 저장 (워커 호출) |

---

## 4. Layer 2 — `user_events` 테이블

### 4-1. ORM (`udio/adapter/outbound/orm/user_event_orm.py`)

- FK: `users.id`
- `event_type`: `"visual_select"` | `"style_edit"` | `"export"` | `"skip"` | `"loop_play"` | `"gallery_like"` 등
- `target_id`, `target_type`, `dwell_ms`, `payload` (JSONB)

### 4-2. Pydantic (`udio/adapter/inbound/api/schemas/user_events.py`)

- `UserEventCreate`: `user_id`, `event_type`, 선택적 `session_id`, `target_*`, `payload`
- `UserEventRead`

### 4-3. UseCase 메서드 (v2 보완)

| 메서드 | 설명 |
|--------|------|
| `log_event(body)` | 이벤트 기록 |
| `list_by_user(user_id, event_type, limit)` | 사용자 행동 이력 조회 |

---

## 5. Layer 3 — `generation_logs` 테이블

### 5-1. ORM (`udio/adapter/outbound/orm/generation_log_orm.py`)

- FK: `users.id`, `studio_workspaces.id`, `audio_features.id`
- 입력: `prompt_params` (JSONB), `model_version`, `pipeline_version`
- 출력: `output_asset_url`, `render_ms`, `quality_score`, `style_vector` (ARRAY)
- 상태: `status` (`pending` | `processing` | `completed` | `failed`), `error_message`, `completed_at`
- 관계: `audio_feature`, `visual_ratings`

**v3 추가 — 플랫폼별 생성 파라미터 컬럼**

`prompt_params` JSONB 하나로 묶으면 플랫폼별 최적 파라미터를 ML로 학습할 수 없다.  
플랫폼 식별자와 규격을 별도 컬럼으로 분리한다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `target_platform` | `varchar` | `'spotify_canvas'` \| `'tiktok'` \| `'shorts'` \| `'universal'` |
| `aspect_ratio` | `varchar` | `'9:16'` \| `'1:1'` \| `'16:9'` |
| `target_duration_sec` | `Float` | 플랫폼별 목표 길이 (Canvas=8s, Reels=15s, Shorts=60s) |

**v3 추가 — 루프 영상 메타데이터 컬럼**

최종 결과물이 루프 영상임을 명시하고, 루프 품질 분석에 필요한 값을 저장한다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `loop_duration_sec` | `Float` | 루프 1사이클 길이 (Spotify Canvas 기준 8초) |
| `loop_beat_aligned` | `Boolean` | 루프가 BPM에 맞게 정렬됐는지 여부 |
| `frame_rate` | `Integer` | FPS (24, 30, 60) |
| `loop_sync_offset_ms` | `Integer` | 루프 시작점 오프셋 (비트 싱크 보정값, ms) |

### 5-2. Pydantic (`udio/adapter/inbound/api/schemas/generation_logs.py`)

- `GenerationLogCreate`: `user_id`, 선택적 `workspace_id`, `audio_feature_id`, 생성 파라미터·결과 필드
- `GenerationLogRead`
- `GenerationLogStatusRead` (v2 추가): `id`, `status`, `output_asset_url`, `render_ms`, `error_message`
- `GenerationLogPlatformRead` (v3 추가): `target_platform`, `aspect_ratio`, `target_duration_sec`, `loop_duration_sec`, `loop_beat_aligned`, `frame_rate`, `loop_sync_offset_ms`

### 5-3. UseCase 메서드 (v2 보완)

| 메서드 | 설명 |
|--------|------|
| `log_generation(body)` | 생성 요청 기록, job_id 반환 |
| `get(generation_id)` | 단건 조회 |
| `get_status(generation_id)` | 처리 상태 폴링 |
| `list_by_user(user_id, status, limit, offset)` | 사용자별 생성 이력 목록 |
| `update_result(generation_id, result)` | 생성 완료 후 결과 저장 (워커 호출) |
| `update_loop_meta(generation_id, meta)` | 루프 메타데이터 저장 (워커 호출) |

---

## 6. Layer 4 — `visual_ratings` 테이블

### 6-1. ORM (`udio/adapter/outbound/orm/visual_rating_orm.py`)

- FK: `generation_logs.id`, `users.id` (`rater_id`)
- 점수: `aesthetic_score`, `genre_match_score`, `mood_match_score` (1~5)
- A/B: `ab_test_id`, `ab_winner`
- 안전: `flag`, `flag_reason`, `rater_type` (`user` | `admin` | `auto`)

**v3 추가 — 플랫폼별 평가 + 루프 품질 점수 컬럼**

Spotify Canvas용과 TikTok용은 시청 맥락이 다르므로 평가를 플랫폼별로 분리한다.  
루프 자연스러움·비트 싱크는 플랫폼과 독립적인 영상 품질 지표다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `platform` | `varchar` | 어떤 플랫폼용 비주얼을 평가했는지 (`'spotify_canvas'`, `'tiktok'` 등) |
| `loop_smoothness_score` | `Integer` | 루프 자연스러움 (1~5) — 루프 끊김 없이 매끄러운지 |
| `beat_sync_score` | `Integer` | 비트 싱크 정확도 (1~5) — 비주얼이 BPM에 맞게 반응하는지 |

### 6-2. Pydantic (`udio/adapter/inbound/api/schemas/visual_ratings.py`)

- `VisualRatingCreate`: `rater_id`, `generation_id`, 점수·플래그 필드
- `VisualRatingRead`
- `VisualRatingAvgRead` (v2 추가): `generation_id`, `avg_aesthetic`, `avg_genre_match`, `avg_mood_match`, `total_count`
- `AbTestResultRead` (v2 추가): `ab_test_id`, `winner_generation_id`, `win_count`, `lose_count`
- `VisualRatingPlatformAvgRead` (v3 추가): `generation_id`, `platform`, `avg_loop_smoothness`, `avg_beat_sync`, `total_count`

### 6-3. UseCase 메서드 (v2 보완)

| 메서드 | 설명 |
|--------|------|
| `submit_rating(body)` | 평가 제출 |
| `get_ratings_by_generation(generation_id)` | 특정 생성물의 평가 목록 |
| `get_avg_scores(generation_id)` | aesthetic/genre/mood 평균 점수 반환 |
| `get_ab_test_result(ab_test_id)` | A/B 테스트 승패 집계 |
| `flag_rating(rating_id, flag, reason)` | 관리자 신고 처리 |
| `get_platform_avg(generation_id, platform)` | 플랫폼별 루프 품질 평균 점수 조회 |

---

## 7. Output Port + PG Repository

각 레이어에 **Repository ABC** (`app/ports/output/`) 와 **PgRepository** (`adapter/outbound/pg/`) 를 둔다.

### 예시 — `audio_feature_repository.py` (port)

```python
from abc import ABC, abstractmethod

from udio.adapter.inbound.api.schemas.audio_features import (
    AudioFeatureCreate,
    AudioFeatureStatusRead,
)
from udio.adapter.outbound.orm.audio_feature_orm import AudioFeature


class AudioFeatureRepository(ABC):
    @abstractmethod
    async def create(self, body: AudioFeatureCreate) -> AudioFeature:
        ...

    @abstractmethod
    async def get(self, feature_id: uuid.UUID) -> AudioFeature | None:
        ...

    @abstractmethod
    async def get_status(self, feature_id: uuid.UUID) -> AudioFeatureStatusRead | None:
        ...

    @abstractmethod
    async def list_by_user(
        self, user_id: int, limit: int = 20, offset: int = 0
    ) -> list[AudioFeature]:
        ...

    @abstractmethod
    async def update_inference_result(
        self, feature_id: uuid.UUID, result: dict
    ) -> AudioFeature:
        ...
```

### 예시 — `audio_feature_pg_repository.py`

```python
class AudioFeaturePgRepository(AudioFeatureRepository):
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def create(self, body: AudioFeatureCreate) -> AudioFeature:
        data = body.model_dump()
        user_id = data.pop("user_id")
        row = AudioFeature(user_id=user_id, **data)
        self._session.add(row)
        await self._session.flush()
        await self._session.refresh(row)
        return row

    async def get(self, feature_id: uuid.UUID) -> AudioFeature | None:
        result = await self._session.execute(
            select(AudioFeature).where(AudioFeature.id == feature_id)
        )
        return result.scalar_one_or_none()

    async def list_by_user(
        self, user_id: int, limit: int = 20, offset: int = 0
    ) -> list[AudioFeature]:
        result = await self._session.execute(
            select(AudioFeature)
            .where(AudioFeature.user_id == user_id)
            .order_by(AudioFeature.created_at.desc())
            .limit(limit)
            .offset(offset)
        )
        return list(result.scalars().all())
```

> 나머지 3개 레이어(`user_event`, `generation_log`, `visual_rating`)도 동일 패턴.

---

## 8. Input Port + Interactor (Use Case)

### 예시 — `audio_feature_use_case.py`

```python
class AudioFeatureUseCase(ABC):
    @abstractmethod
    async def ingest(self, body: AudioFeatureCreate) -> AudioFeatureRead:
        ...

    @abstractmethod
    async def get(self, feature_id: uuid.UUID) -> AudioFeatureRead:
        ...

    @abstractmethod
    async def get_status(self, feature_id: uuid.UUID) -> AudioFeatureStatusRead:
        ...

    @abstractmethod
    async def list_by_user(
        self, user_id: int, limit: int, offset: int
    ) -> list[AudioFeatureRead]:
        ...
```

### 예시 — `audio_feature_interactor.py` (v2 에러 처리 보완)

```python
from sqlalchemy.exc import IntegrityError, OperationalError

class AudioFeatureInteractor(AudioFeatureUseCase):
    async def ingest(self, body: AudioFeatureCreate) -> AudioFeatureRead:
        try:
            row = await self._repository.create(body)
            await self._session.commit()
        except IntegrityError as exc:
            # FK 위반, unique 제약 위반
            await self._session.rollback()
            raise HTTPException(status_code=409, detail="Duplicate or invalid FK") from exc
        except OperationalError as exc:
            # DB 연결 오류, 타임아웃
            await self._session.rollback()
            raise HTTPException(status_code=503, detail="DB unavailable") from exc
        except HTTPException:
            await self._session.rollback()
            raise
        except Exception as exc:
            await self._session.rollback()
            raise HTTPException(status_code=500, detail=str(exc)) from exc
        return AudioFeatureRead.model_validate(row)

    async def get(self, feature_id: uuid.UUID) -> AudioFeatureRead:
        row = await self._repository.get(feature_id)
        if row is None:
            raise HTTPException(status_code=404, detail="AudioFeature not found")
        return AudioFeatureRead.model_validate(row)
```

**에러 처리 전략 — 레이어별 공통 원칙**

| 예외 | HTTP 상태 | 처리 위치 |
|------|-----------|-----------|
| `IntegrityError` | 409 Conflict | Interactor |
| `OperationalError` | 503 Service Unavailable | Interactor |
| `HTTPException` | 그대로 전파 | Interactor |
| `Exception` (기타) | 500 Internal Server Error | Interactor |
| 레코드 없음 | 404 Not Found | Interactor (`get` 계열) |
| 중복 평가 | 409 Conflict | `VisualRatingInteractor` 전용 |

**UseCase 메서드 — 레이어별 요약**

| 레이어 | 메서드 목록 |
|--------|------------|
| Layer 1 | `ingest`, `get`, `get_status`, `list_by_user`, `update_inference_result`, `update_visual_mapping` (v3), `update_beat_analysis` (v3) |
| Layer 2 | `log_event`, `list_by_user` |
| Layer 3 | `log_generation`, `get`, `get_status`, `list_by_user`, `update_result`, `update_loop_meta` (v3) |
| Layer 4 | `submit_rating`, `get_ratings_by_generation`, `get_avg_scores`, `get_ab_test_result`, `flag_rating`, `get_platform_avg` (v3) |
| Export | `export_labeled_dataset`, `get_dataset_stats` (v2) |

---

## 9. 학습 데이터 추출 파이프라인 ← v2 신규

수집된 4-Layer 데이터를 실제 ML 학습에 사용할 수 있도록 추출하는 파이프라인이다.  
관리자 전용 엔드포인트로 제공하며, 평가 점수 기준 필터링을 지원한다.

### 9-1. TrainingExportUseCase (port)

```python
class TrainingExportUseCase(ABC):
    @abstractmethod
    async def export_labeled_dataset(
        self,
        min_aesthetic_score: int = 3,
        limit: int = 10000,
        format: str = "jsonl",   # "jsonl" | "csv"
    ) -> list[TrainingRecord]:
        ...

    @abstractmethod
    async def get_dataset_stats(self) -> DatasetStatsRead:
        ...
```

### 9-2. TrainingRecord 스키마 (`schemas/training_export.py`)

```python
class TrainingRecord(BaseModel):
    audio_feature_id: uuid.UUID
    # ── 음악 특성 (Layer 1 기본)
    bpm: float
    energy: float
    valence: float
    danceability: float
    spectral_centroid: float | None
    loudness: float | None
    genre_primary: str
    mood_tags: list[str]
    # ── 비트 싱크 분석 결과 (v3)
    beat_timestamps: list[float] | None       # 루프 싱크 포인트 배열
    highlight_start_sec: float | None         # 하이라이트 구간 시작
    highlight_end_sec: float | None           # 하이라이트 구간 끝
    onset_strength: float | None              # 어택 강도
    # ── 화면 언어 변환 결과 (v3)
    visual_motion_intensity: float | None     # 움직임 강도 (0.0~1.0)
    visual_texture_type: str | None           # 질감 유형
    visual_color_temperature: str | None      # 색감 온도
    visual_rhythm_sync: float | None          # 비트-비주얼 싱크 강도
    genre_to_visual_mapping: dict | None      # 장르→비주얼 매핑 결과
    mood_to_color_mapping: dict | None        # 무드→색상 매핑 결과
    # ── AI 추론 결과 (v2)
    predicted_visual_style: str | None
    predicted_color_palette: list[str] | None
    visual_embedding: list[float] | None
    # ── 생성 파라미터 (Layer 3)
    prompt_params: dict
    model_version: str
    target_platform: str | None               # 플랫폼 (v3)
    loop_duration_sec: float | None           # 루프 길이 (v3)
    loop_beat_aligned: bool | None            # 비트 정렬 여부 (v3)
    frame_rate: int | None                    # FPS (v3)
    # ── 평가 레이블 (Layer 4)
    aesthetic_score: int
    genre_match_score: int
    mood_match_score: int
    loop_smoothness_score: int | None         # 루프 자연스러움 (v3)
    beat_sync_score: int | None               # 비트 싱크 정확도 (v3)
    platform_rated: str | None               # 평가된 플랫폼 (v3)
    ab_winner: bool | None

class DatasetStatsRead(BaseModel):
    total_audio_features: int
    total_generation_logs: int
    total_ratings: int
    avg_aesthetic_score: float
    avg_loop_smoothness: float               # v3 추가
    avg_beat_sync: float                     # v3 추가
    labeled_count: int                       # aesthetic_score >= 3인 레코드 수
    platform_breakdown: dict                 # v3 추가: 플랫폼별 생성 건수
```

### 9-3. 엔드포인트

```python
@training_export_router.get(
    "/export/training-set",
    response_model=list[TrainingRecord],
    dependencies=[Depends(require_admin)],   # 관리자 전용
)
async def export_training_set(
    min_aesthetic_score: int = 3,
    limit: int = 10000,
    format: str = "jsonl",
    use_case: TrainingExportUseCase = Depends(get_training_export_use_case),
) -> list[TrainingRecord]:
    return await use_case.export_labeled_dataset(min_aesthetic_score, limit, format)

@training_export_router.get(
    "/export/stats",
    response_model=DatasetStatsRead,
    dependencies=[Depends(require_admin)],
)
async def get_dataset_stats(
    use_case: TrainingExportUseCase = Depends(get_training_export_use_case),
) -> DatasetStatsRead:
    return await use_case.get_dataset_stats()
```

### 9-4. PG Repository 핵심 쿼리

```python
# audio_features + generation_logs + visual_ratings 3-way JOIN (v3 필드 포함)
stmt = (
    select(AudioFeature, GenerationLog, VisualRating)
    .join(GenerationLog, GenerationLog.audio_feature_id == AudioFeature.id)
    .join(VisualRating, VisualRating.generation_id == GenerationLog.id)
    .where(VisualRating.aesthetic_score >= min_aesthetic_score)
    .where(AudioFeature.processing_status == "done")        # 분석 완료된 것만
    .where(GenerationLog.status == "completed")             # 생성 완료된 것만
    .order_by(VisualRating.created_at.desc())
    .limit(limit)
)

# 플랫폼별 평균 루프 품질 집계 (v3)
platform_stmt = (
    select(
        VisualRating.platform,
        func.avg(VisualRating.loop_smoothness_score).label("avg_loop_smoothness"),
        func.avg(VisualRating.beat_sync_score).label("avg_beat_sync"),
        func.count(VisualRating.id).label("total_count"),
    )
    .where(VisualRating.generation_id == generation_id)
    .group_by(VisualRating.platform)
)

# 데이터셋 통계 — 플랫폼별 생성 건수 포함 (v3)
platform_breakdown_stmt = (
    select(
        GenerationLog.target_platform,
        func.count(GenerationLog.id).label("count"),
    )
    .group_by(GenerationLog.target_platform)
)
```

---

## 10. 인증 연동 계획 ← v2 신규

현재는 body에 `user_id`를 직접 포함하는 방식이다. 아래 계획에 따라 단계적으로 전환한다.

### 10-1. 현재 (MVP 1단계)

```python
# body에 user_id 포함 — 개발·테스트 편의용
class AudioFeatureCreate(BaseModel):
    user_id: int   # ← 인증 연동 후 제거 예정
    bpm: float
    ...
```

### 10-2. 전환 계획 (user JWT 미들웨어 연동 후)

```python
# 1. 라우터: Depends(get_current_user) 추가
@audio_features_router.post("/audio-features")
async def post_audio_feature(
    body: AudioFeatureCreate,
    current_user: UserRecord = Depends(get_current_user),   # ← 추가
    use_case: AudioFeatureUseCase = Depends(get_audio_feature_use_case),
) -> AudioFeatureRead:
    return await use_case.ingest(body, user_id=current_user.id)  # user_id 주입

# 2. body에서 user_id 필드 제거
class AudioFeatureCreate(BaseModel):
    # user_id 제거됨
    bpm: float
    ...

# 3. 관리자 전용 엔드포인트 보호
@training_export_router.get(
    "/export/training-set",
    dependencies=[Depends(require_admin)],  # role = 'admin' 검증
)
```

### 10-3. 전환 시 수정 범위

| 대상 | 수정 내용 |
|------|-----------|
| `schemas/*.py` | `user_id`, `rater_id` 필드 제거 |
| `router/*.py` | `Depends(get_current_user)` 추가 |
| `interactor/*.py` | `user_id` 파라미터를 외부 주입으로 변경 |
| `dependencies/*.py` | `require_admin` 의존성 추가 |

---

## 11. Dependencies (DIP)

라우터는 구현체를 직접 import 하지 않는다. `titanic/dependencies/james.py` 와 동일 패턴.

### `udio/dependencies/audio_feature.py`

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from core.matrix.oracle_database import get_db
from udio.adapter.outbound.pg.audio_feature_pg_repository import AudioFeaturePgRepository
from udio.app.ports.input.audio_feature_use_case import AudioFeatureUseCase
from udio.app.ports.output.audio_feature_repository import AudioFeatureRepository
from udio.app.use_cases.audio_feature_interactor import AudioFeatureInteractor


def get_audio_feature_use_case(
    db: AsyncSession = Depends(get_db),
) -> AudioFeatureUseCase:
    repository: AudioFeatureRepository = AudioFeaturePgRepository(session=db)
    return AudioFeatureInteractor(session=db, repository=repository)
```

---

## 12. Router (`adapter/inbound/api/`)

### 레이어별 v1 라우터 — 전체 엔드포인트

```python
audio_features_router = APIRouter(prefix="/api/ml", tags=["ml-data"])

# ── Layer 1: audio features ──────────────────────────────────────────
@audio_features_router.post("/audio-features", response_model=AudioFeatureRead)
async def post_audio_feature(...): ...

@audio_features_router.get("/audio-features/{feature_id}", response_model=AudioFeatureRead)
async def get_audio_feature(...): ...

@audio_features_router.get("/audio-features/{feature_id}/status", response_model=AudioFeatureStatusRead)
async def get_audio_feature_status(...): ...

@audio_features_router.get("/audio-features", response_model=list[AudioFeatureRead])
async def list_audio_features(user_id: int, limit: int = 20, offset: int = 0, ...): ...

# ── Layer 2: user events ─────────────────────────────────────────────
@user_events_router.post("/events", response_model=UserEventRead)
async def post_user_event(...): ...

@user_events_router.get("/events", response_model=list[UserEventRead])
async def list_user_events(user_id: int, event_type: str | None = None, ...): ...

# ── Layer 3: generation logs ─────────────────────────────────────────
@generation_logs_router.post("/generations", response_model=GenerationLogRead)
async def post_generation_log(...): ...

@generation_logs_router.get("/generations/{generation_id}", response_model=GenerationLogRead)
async def get_generation_log(...): ...

@generation_logs_router.get("/generations/{generation_id}/status", response_model=GenerationLogStatusRead)
async def get_generation_log_status(...): ...

@generation_logs_router.get("/generations", response_model=list[GenerationLogRead])
async def list_generation_logs(user_id: int, status: str | None = None, ...): ...

# ── Layer 4: visual ratings ──────────────────────────────────────────
@visual_ratings_router.post("/ratings", response_model=VisualRatingRead)
async def post_visual_rating(...): ...

@visual_ratings_router.get("/ratings", response_model=list[VisualRatingRead])
async def list_ratings(generation_id: uuid.UUID, ...): ...

@visual_ratings_router.get("/ratings/avg", response_model=VisualRatingAvgRead)
async def get_avg_scores(generation_id: uuid.UUID, ...): ...

@visual_ratings_router.get("/ratings/ab-test/{ab_test_id}", response_model=AbTestResultRead)
async def get_ab_test_result(...): ...
```

### 집계 — `adapter/inbound/api/__init__.py`

```python
udio_router = APIRouter()
udio_router.include_router(audio_features_router)
udio_router.include_router(user_events_router)
udio_router.include_router(generation_logs_router)
udio_router.include_router(visual_ratings_router)
udio_router.include_router(training_export_router)   # ← v2 추가
```

---

## 13. 기존 파일 수정 사항

### 13-1. `orm_registry.py`

```python
from udio.adapter.outbound.orm.audio_feature_orm import AudioFeature      # noqa
from udio.adapter.outbound.orm.user_event_orm import UserEvent            # noqa
from udio.adapter.outbound.orm.generation_log_orm import GenerationLog    # noqa
from udio.adapter.outbound.orm.visual_rating_orm import VisualRating      # noqa
```

### 13-2. `main.py`

```python
from udio.adapter.inbound.api import udio_router

app.include_router(udio_router)
```

> ML 테이블은 Alembic 마이그레이션으로 관리한다. 별도 `create_ml_tables` startup 호출은 필수 아님.

---

## 14. Alembic 마이그레이션

```bash
# backend/ 디렉터리에서 실행
alembic revision --autogenerate -m "add_udio_4_layers"
alembic upgrade head

# 롤백 확인 (반드시 검증)
alembic downgrade -1
alembic upgrade head
```

> `alembic/env.py` 에서 `import_all_models()` 가 ML ORM 을 포함하는지 확인한다.  
> 기존 마이그레이션: `alembic/versions/f6af73a4f087_add_udio_4_layers.py`

---

## 15. 완료 체크리스트 (v2 보완)

### 구조 확인
- [ ] `udio/` 헥사고날 디렉터리 구조 (`adapter` / `app` / `dependencies`)
- [ ] Layer 1: `audio_feature_orm` + schema + port + pg + interactor + router
- [ ] Layer 2: `user_event_*`
- [ ] Layer 3: `generation_log_*`
- [ ] Layer 4: `visual_rating_*`
- [ ] `training_export_*` (v2 추가)
- [ ] `dependencies/get_*_use_case` 5개 (DIP, training_export 포함)
- [ ] `adapter/inbound/api/__init__.py` → `udio_router` 집계 (5개 라우터)
- [ ] `orm_registry.py` ORM 4개 등록
- [ ] `main.py` 에 `udio_router` 마운트

### 데이터 흐름 검증
- [ ] `audio_features → generation_logs` FK 연결 통합 테스트
- [ ] `generation_logs → visual_ratings` FK 연결 통합 테스트
- [ ] `audio_features → generation_logs → visual_ratings` 3-way JOIN 쿼리 확인
- [ ] `visual_ratings.aesthetic_score` 1~5 범위 Pydantic validator 동작 확인
- [ ] `user_events.payload` JSONB 직렬화/역직렬화 확인
- [ ] `generation_logs.style_vector` ARRAY 타입 저장 확인
- [ ] `audio_features.visual_embedding` ARRAY 타입 저장 확인 (v2)
- [ ] `audio_features.beat_timestamps` ARRAY 저장 및 루프 싱크 포인트 활용 확인 (v3)
- [ ] `audio_features.genre_to_visual_mapping` JSONB 저장 확인 (v3)
- [ ] `audio_features.visual_motion_intensity`, `visual_texture_type`, `visual_color_temperature` 저장 확인 (v3)
- [ ] `generation_logs.target_platform`, `loop_duration_sec`, `loop_beat_aligned` 저장 확인 (v3)
- [ ] `visual_ratings.platform`, `loop_smoothness_score`, `beat_sync_score` 저장 확인 (v3)
- [ ] 플랫폼별 평균 루프 품질 쿼리(`get_platform_avg`) 동작 확인 (v3)
- [ ] `ab_test_id` 기반 A/B 결과 집계 쿼리 확인

### 비동기·에러 처리 검증
- [ ] POST 엔드포인트가 즉시 `{ id, status: "pending" }` 반환하는지 확인
- [ ] 백그라운드 워커가 `processing_status` / `status` 를 올바르게 업데이트하는지 확인
- [ ] GET `/status` 폴링 엔드포인트 동작 확인
- [ ] `IntegrityError` → 409 응답 확인
- [ ] `OperationalError` → 503 응답 확인
- [ ] 동시 요청 시 `AsyncSession` 격리 확인

### Alembic 검증
- [ ] `alembic upgrade head` 성공 및 테이블 생성 확인
- [ ] `alembic downgrade -1` 롤백 확인 ← v2 추가 (운영 장애 방지)
- [ ] `alembic upgrade head` 재적용 확인

### API 노출 확인
- [ ] `/docs` (Swagger)에서 아래 엔드포인트 전체 노출 확인

---

## 16. 신규 API 엔드포인트 요약 (v3 전체)

### Layer 1 — audio features

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/ml/audio-features` | 음악 분석 데이터 인입 (job_id 반환) |
| GET | `/api/ml/audio-features/{id}` | 단건 조회 |
| GET | `/api/ml/audio-features/{id}/status` | 처리 상태 폴링 |
| GET | `/api/ml/audio-features?user_id=&limit=&offset=` | 사용자별 분석 이력 목록 |

### Layer 2 — user events

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/ml/events` | 사용자 행동 이벤트 로그 |
| GET | `/api/ml/events?user_id=&event_type=` | 사용자 이벤트 이력 조회 |

### Layer 3 — generation logs

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/ml/generations` | AI 생성 요청 기록 (job_id 반환) |
| GET | `/api/ml/generations/{id}` | 단건 조회 |
| GET | `/api/ml/generations/{id}/status` | 처리 상태 폴링 |
| GET | `/api/ml/generations?user_id=&status=` | 사용자별 생성 이력 목록 |

### Layer 4 — visual ratings

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/ml/ratings` | 비주얼 평점 제출 |
| GET | `/api/ml/ratings?generation_id=` | 특정 생성물 평가 목록 |
| GET | `/api/ml/ratings/avg?generation_id=` | 평균 점수 조회 |
| GET | `/api/ml/ratings/ab-test/{ab_test_id}` | A/B 테스트 결과 집계 |
| GET | `/api/ml/ratings/platform-avg?generation_id=&platform=` | 플랫폼별 루프 품질 평균 점수 (v3) |

### 학습 데이터 추출 (관리자 전용)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/api/ml/export/training-set` | 학습 데이터셋 추출 (JSONL/CSV) |
| GET | `/api/ml/export/stats` | 데이터셋 통계 조회 |

> 요청 본문에 `user_id` / `rater_id` 를 포함한다 (현재 인증 미연동).  
> 인증 연동 계획은 **10번 섹션** 참고.

---

*본 작업지시서는 `개발정의서.md` v1.0 기준이며, `titanic` 헥사고날 패턴(adapter → app ports/use_cases → dependencies)을 따른다.*  
*v3은 "How it works" 3단계 파이프라인(Drop Your Sound → AI Aesthetic Analysis → Get Your Artwork)을 DB 설계에 완전 반영한 버전이다.*
