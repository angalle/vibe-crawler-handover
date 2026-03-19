# 프론트엔드 — 아키텍처 및 설계

---

## 1. 프로젝트 구조

```
frontend/src/
├── api/              # Orval 자동 생성 API 훅 (직접 수정 금지)
├── api-config/       # API 클라이언트 설정 (axios instance 등)
├── components/       # 아토믹 디자인 기반 UI 컴포넌트
│   ├── atoms/        #   → 최소 단위 (Button, Input, Badge, Spinner 등)
│   ├── molecules/    #   → atoms 조합 (FormField, FileUpload, StatusBadge 등)
│   ├── organisms/    #   → 복합 UI (EnhancedScriptsList 등)
│   ├── templates/    #   → 페이지 레이아웃 (DashboardLayout, ListLayout 등)
│   └── pages/        #   → 라우트 매핑 페이지 컴포넌트
├── features/         # 도메인별 비즈니스 로직 모듈
│   ├── ai-auto-fix/  #   → AI 자동 수정 관련
│   ├── crawling/     #   → 크롤링 세션 관련
│   ├── dashboard/    #   → 대시보드 관련
│   ├── jobs/         #   → Job 관리 관련
│   ├── projects/     #   → 프로젝트 관련
│   └── ui/           #   → 공통 UI 유틸
├── composables/      # Vue 3 Composition API 재사용 로직
├── stores/           # Pinia 상태 관리
├── router/           # Vue Router 라우팅
├── utils/            # 유틸리티 함수
├── views/            # 라우터 뷰 (pages로 위임)
└── assets/           # 정적 리소스
```

---

## 2. 아토믹 디자인 (Atomic Design)

컴포넌트를 **5단계 계층**으로 분류합니다.

### 2.1 계층 구조

| 계층 | 역할 | 예시 | 규칙 |
|------|------|------|------|
| **atoms** | 최소 단위 HTML 래퍼 | `Button`, `Input`, `Badge`, `Spinner`, `Switch`, `Checkbox`, `Icon` | 자체 상태 없음, props로만 작동 |
| **molecules** | atoms 조합 UI | `FormField`, `FileUploadField`, `CronScheduleField`, `SearchForm`, `Pagination`, `StatusBadge`, `DiffViewer` | atoms만 import, 로컬 상태만 가짐 |
| **organisms** | 비즈니스 로직 포함 UI | `EnhancedScriptsList`, `AIAutoFixActionsPanel` | API 호출/store 접근 가능 |
| **templates** | 페이지 레이아웃 껍데기 | `DashboardLayout`, `ListLayout`, `DetailLayout` | slot 기반, 콘텐츠 독립적 |
| **pages** | 라우트 매핑 페이지 | `ProjectsPage`, `SessionListPage`, `AIAutoFixPage`, `OptionsPage` | 데이터 페칭, 상태 조합 |

### 2.2 의존성 규칙

```
atoms → (독립, 외부 의존 없음)
molecules → atoms만 import
organisms → atoms + molecules + API/Store
templates → slot 기반 (레이아웃만)
pages → 모든 계층 사용 가능
```

### 2.3 새 컴포넌트 작성 기준

- **UI만?** → atoms 또는 molecules
- **API 호출/비즈니스 로직?** → organisms 또는 features
- **레이아웃?** → templates
- **라우트에 연결?** → pages

---

## 3. API 코드 생성 (Orval)

백엔드 OpenAPI 스펙에서 Vue 3 Composition API 기반 API 훅을 자동 생성합니다.

```bash
yarn generate:api    # orval 실행 → src/api/ 에 코드 생성
```

### 3.1 동작 방식

1. 백엔드 `/openapi.json` 스펙 읽기
2. `useQuery` / `useMutation` 훅 자동 생성
3. `src/api/` 디렉토리에 저장

### 3.2 주의사항

- 백엔드 API 변경 시 반드시 `yarn generate:api` 재실행
- 자동 생성 코드 직접 수정 금지 (재생성 시 덮어씌워짐)

---

## 4. 상태 관리

### 4.1 Pinia Stores (클라이언트 상태)

| Store | 역할 |
|-------|------|
| `workspace.ts` | 활성 워크스페이스 선택/전환, localStorage 영속 |
| `ui.ts` | 사이드바, 모달, 토스트 등 UI 상태 |
| `counter.ts` | 데모/카운터 (참고용) |

### 4.2 TanStack Vue Query (서버 상태)

- API 데이터 캐싱 및 자동 갱신
- Orval이 자동 생성한 `useQuery`/`useMutation` 훅 사용

### 4.3 Composables (재사용 로직)

| Composable | 역할 |
|------------|------|
| `useFileUpload` | 파일 업로드 핸들링 (드래그&드롭, 다중 파일) |
| `useFiltering` | 리스트 필터링 (검색, 상태별) |
| `usePagination` | 페이지네이션 상태 관리 |
| `useSorting` | 정렬 상태 관리 |
| `useToast` | 토스트 알림 표시 |

---

## 5. Features 디렉토리

도메인별 비즈니스 로직을 분리한 모듈입니다. pages에서 직접 비즈니스 로직을 작성하지 않고, features로 위임합니다.

| Feature | 역할 |
|---------|------|
| `ai-auto-fix/` | AI 자동 수정 세션 관리 로직 |
| `crawling/` | 크롤링 세션 실행/상태 관리 |
| `dashboard/` | 대시보드 데이터 집계 |
| `jobs/` | Job CRUD/스케줄 관리 |
| `projects/` | 프로젝트 CRUD/필터 |
| `ui/` | 공통 UI 유틸리티 |

---

## 6. 백엔드 API 연동

### 6.1 주요 API 엔드포인트

| 태그 | Prefix | 주요 기능 |
|------|--------|----------|
| `projects` | `/api/v1` | 프로젝트 CRUD, 배치 업데이트, 필터 조회 |
| `categories` | `/api/v1` | 카테고리 CRUD |
| `jobs` | `/api/v1` | Job CRUD, 스케줄 관리 |
| `crawling` | `/api/v1` | 크롤링 실행/중지, 세션 조회, 로그 |
| `ai-auto-fix` | `/api/v1` | AI Auto Fix 시작/상태/취소 |
| `admin` | `/api/v1/admin` | 시스템 설정, 알림, Docker 정리 |
| `workspaces` | `/api/v1` | 워크스페이스 CRUD |
| `enhanced-scripts` | `/api/v1` | 스크립트 개선 |

### 6.2 API 문서

- **Swagger UI**: `http://<HOST>:<PORT>/docs`
- **ReDoc**: `http://<HOST>:<PORT>/redoc`

