# 백엔드 — 프로젝트 개요

---

## 1. 프로젝트 목적

**Vibe Crawler Platform 백엔드**는 광고 매체 데이터를 자동 수집·관리·전송하기 위한 **크롤링 관리 API 서버**입니다.

| 목적 | 설명 |
|------|------|
| **크롤링 실행** | Docker/Local 환경에서 Playwright 기반 크롤러 실행 관리 |
| **스케줄링** | Cron 기반 정기 실행, 놓친 실행 자동 캐치업 (APScheduler) |
| **모니터링** | Job 실행 상태, 성공/실패, 로그 관리 |
| **후처리 API** | 수집 데이터를 Incross i-flow API로 분석·업로드 |
| **AI Auto Fix** | 크롤링 실패 시 AI가 스크립트를 자동 분석·수정 |

---

## 2. 기술 스택

| 분류 | 기술 | 버전 |
|------|------|------|
| 언어 | Python | 3.10 |
| 프레임워크 | FastAPI | 0.115.6 |
| ORM | SQLAlchemy (Async) | 2.0.23 |
| DB 마이그레이션 | Alembic | 1.12.1 |
| DB | PostgreSQL | 15 (Alpine) |
| DB 드라이버 | asyncpg | 0.29.0 |
| 스케줄러 | APScheduler | 3.10.4 |
| 브라우저 자동화 | Playwright | 1.48.0 |
| AI | OpenAI | 1.58.1 |
| 벡터 DB | Qdrant Client | 1.12.1 |
| 엑셀 처리 | openpyxl | 3.1.0+ |
| 설정 관리 | pydantic-settings | 2.0.0+ |

---

## 3. 환경 설정

### 3.1 로컬 환경

```bash
cd backend
python3.10 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
playwright install firefox chromium
cp env.sample .env
```

### 3.2 주요 환경변수 (`.env`)

| 변수 | 설명 | 기본값 |
|------|------|--------|
| `DATABASE_URL` | PostgreSQL 연결 URL | `postgresql+asyncpg://vibe_user:vibe_password@localhost:5432/vibe_crawling_db` |
| `ENV` | 실행 환경 (`local`/`dev`/`prod`) | `local` |
| `OPENAI_API_KEY` | OpenAI API 키 | - |
| `PROJECT_ROOT` | 프로젝트 루트 경로 | 자동 감지 |
| `SYSTEM_STORAGE_ROOT` | 스토리지 루트 | Docker: `/app/system_storage` |
| `HOST_SYSTEM_STORAGE_ROOT` | 호스트 스토리지 경로 | Docker 크롤러 호스트 참조용 |
| `INCROSS_API_BASE_URL` | i-flow API URL | `https://dev.i-flow.kr/request/api` |
| `INCROSS_API_MODE` | API 연결 모드 | local: `tunnel`, 나머지: `direct` |
| `MAIL_API_URL` | 메일 알림 API URL | - |
| `CORS_ORIGINS` | CORS 허용 Origin | `*` |

### 3.3 환경별 차이

```
shared/config/
├── base.py     → Settings (공통)
├── local.py    → LocalSettings
├── dev.py      → DevSettings
└── prod.py     → ProdSettings
```

| 설정 | local | dev | prod |
|------|:-----:|:---:|:----:|
| `DEBUG` | `True` | `True` | `False` |
| `INCROSS_API_MODE` | `tunnel` | `direct` | `direct` |
| `INCROSS_API_BASE_URL` | `dev.i-flow.kr` | `dev.i-flow.kr` | `i-flow.kr` |

### 3.4 서버 실행

```bash
# DB 마이그레이션
alembic -c shared/alembic/alembic.ini upgrade head

# 서버 실행
uvicorn shared.main:app --host 0.0.0.0 --port 6767 --reload
```

- **API Docs**: http://localhost:6767/docs
- **Health Check**: http://localhost:6767/health

---

## 4. 서비스 포트

| 서비스 | 로컬 | 운영(Docker) |
|--------|:----:|:----:|
| Backend API | 6767 | 8080 (→6767) |
| PostgreSQL | 5432 | 5432 |
