# 백엔드 — 운영 및 관리 가이드

---

## 1. 인프라 정보

### 1.1 서버 구성

| 환경 | 호스트 | 접근 방법 |
|------|--------|----------|
| **Dev/Prod** | EC2 (배포 시 동적 할당) | `ssh -i heesun.pem ubuntu@<IP>` |
| **로컬 개발** | localhost | 직접 실행 |

### 1.2 Docker Compose 서비스

| 컨테이너 | 이미지 | 포트 | 역할 |
|----------|--------|:----:|------|
| `vibe-crawler-db` | `postgres:15-alpine` | 5432 | 메인 DB |
| `vibe-crawler-migration` | `vibe-migration:latest` | - | Alembic 마이그레이션 (1회 실행) |
| `vibe-crawler-backend` | `vibe-backend:latest` | 6767→8080 | API 서버 + 스케줄러 |

### 1.3 AWS 정보

| 항목 | 값 |
|------|-----|
| AWS Account ID | `850205788233` |
| Region | `ap-northeast-2` (서울) |
| ECR | `vibe-backend`, `vibe-migration` |
| SSH Key | `heesun.pem` |

---

## 2. 빌드 및 배포

```bash
# Docker 이미지 빌드 & ECR Push
./infrastructure/scripts/build-and-push-to-ecr.sh

# EC2 배포 (기존 서버)
./infrastructure/scripts/deploy-to-ec2.sh <EC2_IP>

# 새 EC2 프로비저닝 + 배포
./infrastructure/aws/provision-and-deploy-ecr.sh
```

### 배포 워크플로우

```
코드 수정 → Docker 이미지 빌드 → ECR Push → EC2 배포
→ 자동 마이그레이션 + 헬스체크 → API: http://<IP>:8080/docs
```

---

## 3. 모니터링 및 로그

### 3.1 로그 확인

```bash
# Docker 환경
ssh -i heesun.pem ubuntu@<IP>
cd ~/vibe-deployment
docker-compose logs -f backend

# 로컬 환경
uvicorn shared.main:app --host 0.0.0.0 --port 6767 --reload
```

### 3.2 크롤링 로그 파일 구조 (Workspace 기반)

```
system_storage/
├── workspaces/
│   └── {workspace_id}/
│       └── projects/
│           └── {project_id}/
│               ├── scripts/             # 크롤링 스크립트
│               ├── hotfixes/            # 핫픽스 스크립트
│               ├── mdc/                 # MDC 관련 파일
│               └── sessions/
│                   └── {YYYYMMDD}/
│                       └── {session_id}/
│                           ├── action_snapshots/  # 액션별 PNG 스크린샷
│                           ├── videos/            # webm 녹화 파일
│                           ├── downloads/         # 수집 파일 (CSV 등)
│                           ├── crawling_result.json
│                           └── logs/              # 실행 로그
└── crawling_results/                    # Legacy 경로 (사용 안 함)
```

> 워크스페이스별 디렉토리  
> - MINDK: `workspaces/22222222-2222-2222-2222-222222222222/`  
> - INCRO: `workspaces/11111111-1111-1111-1111-111111111111/`

### 3.3 크롤러 Docker 컨테이너

크롤링 실행 시 세션별 Docker 컨테이너가 생성됩니다.

| 항목 | 값 |
|------|----|
| 컨테이너 명명 | `crawler-{session_id}` |
| 실행 방식 | 형제 컨테이너 (`/var/run/docker.sock` 마운트) |
| 브라우저 | Playwright (Firefox/Chromium) |

### 3.4 스케줄러 Runner 목록

| Runner | 주기 | 설명 |
|--------|------|------|
| `scheduled_job_runner` | Cron 기반 | 프로젝트별 Cron 스케줄에 따라 세션 생성 |
| `pending_sessions_runner` | 15초 | PENDING 세션 픽업 → 크롤러 실행 |
| `pending_jobs_runner` | 15초 | 놓친 스케줄 캐치업 |
| `sync_job_status_runner` | 30초 | 실행 중 Job 상태 동기화 (좀비 감지) |
| `cleanup_old_logs_runner` | 매일 04:00 KST | `log_retention_days` 초과 세션 삭제 |
| `docker_system_cleanup_runner` | 1시간 | 미사용 이미지/캐시 + 고아 컨테이너 정리 |
| `recover_zombie_jobs_runner` | 서버 시작 10초 후 1회 | RUNNING 비정상 Job 복구 |

### 3.4 에러 대응 절차

1. 프론트 세션 목록에서 FAILED 세션 → 로그 탭 확인
2. `docker-compose logs -f backend | grep ERROR`
3. `docker-compose exec db psql -U vibe_user -d vibe_crawling_db`
4. `docker-compose restart backend`
5. `./deploy-to-ec2.sh <IP>` (전체 재배포)

---

## 4. 외부 서비스 연동

### 4.1 Incross i-flow API

| 항목 | Dev | Prod |
|------|-----|------|
| Base URL | `https://dev.i-flow.kr/request/api` | `https://i-flow.kr/request/api` |
| 연결 모드 | local: `tunnel`, dev/prod: `direct` | `direct` |

- 활성화 조건: `is_api_transmission_enabled = True` + Workspace `data_api_url` 설정

### 4.2 OpenAI API

- **용도**: AI Auto Fix (크롤링 실패 분석 및 스크립트 수정)
- **키 관리**: `.env` 파일의 `OPENAI_API_KEY`

### 4.3 Mail 알림 API

- **URL**: `MAIL_API_URL`
- **트리거**: Job 완료/실패 시 알림 수신자에게 메일 발송

---

## 5. DB 관리

### 5.1 마이그레이션

```bash
# 상태 확인
alembic -c shared/alembic/alembic.ini current

# 적용
alembic -c shared/alembic/alembic.ini upgrade head

# Docker
docker-compose exec -T backend python -m alembic -c /app/alembic.ini upgrade head
```

### 5.2 백업/복원

```bash
# 백업
docker-compose exec db pg_dump -U vibe_user vibe_crawling_db > backup_$(date +%Y%m%d).sql

# 복원
cat backup.sql | docker-compose exec -T db psql -U vibe_user -d vibe_crawling_db
```

---

## 6. 스토리지 용량 계획

> 기준일: 2026-03-17 (운영서버 96GB 디스크)

### 6.1 세션 로그 구성 요소별 용량

| 구성 요소 | 세션당 용량 | 설명 |
|---|---|---|
| `action_snapshots/` | 2.8~6.7GB | 매 액션마다 PNG 스크린샷 (1000+장) |
| `videos/` | 1.1~2.6GB | webm 녹화 파일 1개 |
| `downloads/` | 15~47MB | 실제 크롤링 결과 (CSV 등) |
| `crawling_result.json` | ~1KB | 메타데이터 |

### 6.2 날짜별 실측 데이터 (2026-03 기준)

프로젝트 수에 따라 일당 용량이 크게 달라집니다.

| 시기 | 활성 프로젝트 수 | 일당 용량 | 비고 |
|---|---|---|---|
| 2/19~3/7 | 5~8개 | **120~140MB/일** | 소규모 운영 |
| 3/8~3/12 | 10~12개 | **1.3~1.4GB/일** | 프로젝트 증가 |
| 3/13~3/15 | 12~15개 | **4.0~4.4GB/일** | 다수 동시 실행 |
| 3/17 (피크) | 15+개 | **16GB/일** | 대량 실행일 |

### 6.3 보관 기간별 예상 사용량

**일당 평균 4GB 기준** (현재 프로젝트 규모):

| 보관 기간 | 세션 로그 용량 | 디스크 사용률 (96GB 기준) |
|---|---|---|
| **10일** (현재) | ~40GB | 42% |
| **15일** | ~60GB | 63% |
| **20일** | ~80GB | **83%** ⚠️ |
| **30일** | ~120GB | **125%** ❌ 디스크 부족 |

**일당 평균 8GB 기준** (프로젝트 20+개로 확대 시):

| 보관 기간 | 세션 로그 용량 | 디스크 사용률 (96GB 기준) |
|---|---|---|
| **10일** | ~80GB | **83%** ⚠️ |
| **15일** | ~120GB | **125%** ❌ |
| **20일** | ~160GB | ❌ 디스크 2배 필요 |

### 6.4 디스크 증량 판단 기준

```
필요 디스크 = (일당 평균 세션 용량) × (보관 기간) + 10GB (OS/Docker/기타)
```

| 시나리오 | 필요 디스크 | 권장 |
|---|---|---|
| 15개 프로젝트 × 10일 | ~50GB | 96GB ✅ 현재 디스크 충분 |
| 15개 프로젝트 × 20일 | ~90GB | 96GB ⚠️ 여유 없음, **128GB 이상** 권장 |
| 20개 프로젝트 × 10일 | ~90GB | 96GB ⚠️, **128GB 이상** 권장 |
| 20개 프로젝트 × 20일 | ~170GB | **200GB 이상** 필요 |

### 6.5 용량 모니터링

```bash
# 운영서버 디스크 확인
df -h /

# 워크스페이스별 용량
du -sh ~/vibe-deployment/system_storage/workspaces/*/

# 날짜별 세션 용량 (전체 프로젝트 합산)
for d in ~/vibe-deployment/system_storage/workspaces/*/projects/*/sessions/*/; do
  echo "$(basename $d) $(du -sm $d 2>/dev/null | cut -f1)MB"
done | awk '{a[$1]+=$2} END{for(d in a) print d, a[d]"MB"}' | sort
```

> **주의**: 디스크 사용률 80% 초과 시 디스크 증량 또는 보관 기간 축소를 검토해야 합니다.
> 디스크 100% 도달 시 크롤링 스크린샷 저장 실패 → 무한루프 발생 위험이 있습니다.

