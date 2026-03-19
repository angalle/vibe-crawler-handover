# 백엔드 — 이슈 리스트 및 히스토리

---

## 2. 해결된 버그

| # | 증상 | 원인 | 해결 |
|---|------|------|------|
| 1 | Borg 싱글턴에서 `InterfaceError` | `db` 세션을 Borg로 공유 | `db` 키를 Borg 공유에서 제외 |
| 2 | 워크스페이스 전환 시 이전 프로젝트 접근 | 단일 조회 API에서 `workspace_id` 필터 누락 | 쿼리에 workspace_id 조건 추가 |
| 3 | PENDING 세션 무한 누적 | `pending_jobs_runner`와 `pending_sessions_runner` 역할 중복 | `pending_jobs_runner` 제거, 통합 |
| 4 | `FileSystemAdapter` 속성 오류 | 레거시 코드 속성명 미반영 | 속성명 수정 |
| 5 | 로그 자동 정리 작동 안 함 | `get_old_session_directories()`에서 `base_path/RESULTS_DIR`(레거시)를 스캔, 실제 데이터는 `results_base_path`에 저장 | 1줄 수정: `results_dir = self.results_base_path` + 경로 일관성 테스트 3개 추가 |
| 6 | 좀비 크롤러 컨테이너 미정리 | DB에 세션 없는데 Docker 컨테이너만 남음 (compose 외부) | Docker Cleanup Service에 고아 컨테이너 역방향 검사 로직 추가 |

---

## 3. 주요 설계 결정 히스토리

### 3.1 Borg 싱글턴 패턴
- FastAPI `Depends()`로 매 요청마다 서비스 새로 생성 → 메모리 비효율
- Borg Pattern(`__dict__` 공유) 채택, **`db` 세션은 제외**
- 검증: `tests/shared/test_repository_singleton.py`

### 3.2 스케줄러 수동 DI
- APScheduler는 HTTP 요청 컨텍스트 밖 → `Depends()` 사용 불가
- Runner 함수에서 `AsyncSession` 직접 주입
- 검증: `tests/scheduler/test_architecture_verification.py`

### 3.3 PENDING 세션 기반 실행 구조
- 초기 `pending_jobs_runner`(10초) + `pending_sessions_runner`(30초) → 세션 무한 누적
- `pending_sessions_runner`(15초)로 통합

### 3.4 후처리 API 2단계 분리
- 1차(`analyze`) → 성공 시 → 2차(`upload`)
- `post_crawling_api_results` 테이블에 단계별 기록

### 3.5 Docker 소켓 마운트
- `/var/run/docker.sock` 마운트 → 형제 컨테이너 방식 (DinD 대비 오버헤드 적음)

### 3.6 좀비 Job 복구 로직
- 초기 5분 간격 반복 → 장시간 크롤링 오판 문제
- 서버 시작 시 1회 + `sync_job_status_runner`(30초) 상시 동기화

### 3.7 Workspace 멀티테넌트
- `workspaces` 테이블 + 모든 엔티티에 `workspace_id` FK
- 사업자별 데이터 격리, API URL 워크스페이스 단위 관리

### 3.8 AMD64 플랫폼 강제 빌드
- Mac ARM64에서 빌드한 이미지가 EC2(AMD64)에서 동작 안 함
- `--platform linux/amd64`로 강제 빌드

### 3.9 워크스페이스 간 프로젝트 이관
- 이관 시 DB `workspace_id` + `name` 변경 + 파일시스템 `mv`
- FK cascade로 jobs/sessions 등은 별도 수정 불필요
- 실제 사례: MINDK → INCRO 8개 프로젝트 이관 (2026-03-17)

### 3.10 고아 컨테이너 자동 정리
- Docker Cleanup Service에 역방향 검사: `docker ps` → 세션 DB 확인 → 불일치 시 stop+rm
- compose 외부 크롤러 컨테이너(`crawler-*`)도 정리 대상

---

## 4. QA

### 4.1 아키텍처 자동 검증
- `tests/arch_util.py`의 `ArchitectureChecker`
- 5개 도메인 전체: 폴더 구조, Port/Adapter 상속, 네이밍, Borg, DTO, DI
- `python -m pytest tests/ -k "architecture" -v`

### 4.2 시간 기반 테스트 주의
- Cron 스케줄 테스트에서 KST/UTC 간헐적 실패
- 고정 mock 시간 사용 권장
