# 프론트엔드 — 프로젝트 개요

---

## 1. 프로젝트 목적

**Vibe Crawler Platform 프론트엔드**는 크롤링 작업을 등록·모니터링·관리하기 위한 **웹 대시보드**입니다.

| 목적 | 설명 |
|------|------|
| **프로젝트 관리** | 광고주/캠페인/매체별 크롤링 프로젝트 등록·수정 |
| **Job 관리** | Cron 스케줄 설정, 스크립트 업로드·관리 |
| **모니터링** | 크롤링 세션 상태, 로그, 스냅샷 실시간 확인 |
| **AI Auto Fix** | 실패 세션 자동 분석·수정 트리거 및 결과 확인 |
| **Admin** | 시스템 설정, 알림 수신자, Docker 정리 등 관리 |

---

## 2. 기술 스택

| 분류 | 기술 | 버전 |
|------|------|------|
| 프레임워크 | Vue | 3.5 |
| 언어 | TypeScript | 5.9 |
| 빌드 도구 | Vite | 7.1 |
| CSS | TailwindCSS | 3.4 |
| 상태 관리 | Pinia | 3.0 |
| 서버 상태 | TanStack Vue Query | 5.90 |
| API 코드 생성 | Orval | 7.17 |
| 라우팅 | Vue Router | 4.6 |
| Node.js | - | v20.18+ |

---

## 3. 환경 설정

### 3.1 로컬 환경

```bash
cd frontend

# Node.js 버전 확인 (v20.18+ 또는 v22.12+)
node -v

# 의존성 설치
yarn install

# 환경변수
cp env.sample .env
# VITE_API_BASE_URL=http://localhost:8080 설정

# 개발 서버 실행
yarn dev
```

- **Frontend**: http://localhost:5173

### 3.2 주요 환경변수

| 변수 | 설명 | 기본값 |
|------|------|--------|
| `VITE_API_BASE_URL` | 백엔드 API 주소 | `http://localhost:8080` |

---

## 4. 서비스 포트

| 서비스 | 로컬 | 운영(Docker) |
|--------|:----:|:----:|
| Frontend | 5173 | 80 / 443 |

---

## 5. 주요 페이지 구성

| 기능 | 설명 |
|------|------|
| 프로젝트 관리 | CRUD, 배치 업데이트, CSV 필터 조회 |
| 카테고리 관리 | CRUD |
| Job 관리 | CRUD, Cron 스케줄, 스크립트 업로드 |
| 크롤링 모니터링 | 세션 실행/중지, 상태, 로그/스냅샷 조회 |
| AI Auto Fix | 시작/상태/취소, 수정 스크립트 적용 |
| 스크립트 개선 | 개선 트리거/미리보기/적용 |
| Admin 패널 | 시스템 설정, 알림, 옵션, Docker 정리 |
| 워크스페이스 전환 | 멀티테넌트 워크스페이스 선택 |

### 5.1 라우트 구조

| Path | Page | 설명 |
|------|------|------|
| `/` | `DashboardPage` | 대시보드 (메인) |
| `/projects` | `ProjectsPage` | 프로젝트 목록 |
| `/projects/new` | `ProjectCreatePage` | 프로젝트 생성 |
| `/projects/:id` | `ProjectDetailPage` | 프로젝트 상세 (Job 관리 포함) |
| `/projects/:projectId/sessions` | `SessionListPage` | 세션 목록/상세 |
| `/logs` | `LogViewerPage` | 로그 뷰어 |
| `/scripts` | `ScriptsPage` | 스크립트 관리 |
| `/options` | `OptionsPage` | 시스템 설정/Admin |
| `/ai-auto-fix/:sessionId` | `AIAutoFixPage` | AI 자동 수정 상세 |
| `/mdc-generator` | `MDCGeneratorPage` | MDC 생성기 |
| `/mdc-generator/:id` | `MDCDetailPage` | MDC 상세 |
