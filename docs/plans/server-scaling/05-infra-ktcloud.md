# KT Cloud 인프라 및 실행 계획

> AWS 대신 KT Cloud(G1/G2) 환경에서 Worker 분리 구성 시 참고

---

## KT Cloud 리소스

| 리소스 | 스펙 | 용도 | 예상 월 비용 |
|---|---|---|---|
| **Master VM** | m2.large (2vCPU, 4GB) | API + 스케줄러 + DB | 기존 유지 |
| **Worker VM × 1** | m2.xlarge (4vCPU, 8GB) | 크롤러 실행 전용 | ~₩70,000 |
| **NAS (NFS)** | Standard | 공유 스토리지 | ~₩30,000/100GB |
| **방화벽 정책** | Zone 내부 통신 | 서버 간 통신 | 무료 |

> KT Cloud는 G1(Classic) / G2(Enterprise) 존에 따라 콘솔 및 서비스명이 다릅니다.

---

## 디스크 용량 해결

| 항목 | 현재 | NAS 전환 후 |
|---|---|---|
| 용량 제한 | 100GB (Block Storage 고정) | **최대 10TB** (NAS 볼륨) |
| 서버 간 공유 | ❌ 불가 | ✅ NFS 마운트 |
| 자동 백업 | 수동 | NAS 스냅샷 기능 |
| 비용 | Block Storage ₩10,000/100GB | NAS ₩30,000/100GB |

---

## 네트워크 구성

```mermaid
graph TD
    subgraph ZONE["KT Cloud Zone (Default Network)"]
        subgraph PUB["Public Tier"]
            MASTER["Master VM<br/>포트포워딩: 80,443,8080(외부)<br/>5432(내부만)"]
        end
        subgraph PRIV["Private Tier"]
            WORKER["Worker VM-1<br/>방화벽: 6767(Master만)"]
        end
        NAS["NAS 볼륨<br/>NFS 마운트 (Zone 내부)"]
    end

    MASTER <-->|6767| WORKER
    MASTER <-->|NFS| NAS
    WORKER <-->|NFS| NAS
```

### AWS vs KT Cloud 네트워크 매핑

| AWS | KT Cloud | 비고 |
|---|---|---|
| VPC | Zone (Default Network) | KT Cloud는 Zone 단위로 네트워크 격리 |
| Public Subnet | Public Tier + 포트포워딩 | 공인 IP는 포트포워딩으로 연결 |
| Private Subnet | Private Tier | 내부 IP만 사용 |
| Security Group | 방화벽 정책 | 콘솔에서 포트별 인/아웃바운드 설정 |
| EFS | NAS (NFS) | KT Cloud NAS 서비스 사용 |
| EBS | Block Storage | 가상서버에 추가 디스크 연결 |

---

## 단계별 실행 계획

### Phase 1: 준비 (코드 변경 없이 즉시 가능)

- [ ] Master VM 스펙 변경 (콘솔 → 서버 중지 → 스펙 변경 → 재시작)
- [ ] Block Storage 추가 연결 (100GB → 200GB)
- [ ] `max_concurrent_jobs` 조정 (3 → 5)
- [ ] 소요: **1일**, 코드 변경 없음

### Phase 2: 공유 스토리지 전환

- [ ] KT Cloud NAS 신청 + NFS 마운트 (Master)
- [ ] 기존 system_storage 데이터 NAS로 마이그레이션
- [ ] Docker Compose `volumes` NAS 경로로 변경
- [ ] 소요: **2~3일**

```bash
# KT Cloud NAS 마운트 예시
sudo mount -t nfs <NAS_IP>:/vol/<볼륨명> /mnt/system_storage
# /etc/fstab에 영구 마운트 추가
echo "<NAS_IP>:/vol/<볼륨명> /mnt/system_storage nfs defaults 0 0" | sudo tee -a /etc/fstab
```

### Phase 3: 세션 선점 락 구현

- [ ] `acquire_session_lock()` 메서드 구현
- [ ] `CheckAndExecutePendingSessionsService`에 선점 로직 적용
- [ ] 유닛테스트 작성 (경쟁 조건 시뮬레이션)
- [ ] 소요: **2~3일**

### Phase 4: Worker 서버 배포

- [ ] Worker VM 생성 (콘솔 → 가상서버 → 생성)
- [ ] 방화벽 정책 설정: Master↔Worker 간 6767, 5432 포트 허용
- [ ] NAS 마운트 (Worker에도 동일 NAS 볼륨 연결)
- [ ] Worker Agent 서비스 구현 + 배포
- [ ] Master↔Worker 통신 테스트
- [ ] 소요: **3~5일**

### Phase 5: 운영 전환 및 검증

- [ ] 멀티 서버 통합 테스트
- [ ] 배치 선점 경합 테스트
- [ ] Worker 장애 시뮬레이션
- [ ] 운영 환경 롤아웃
- [ ] 소요: **2~3일**

---

## 리스크 및 고려사항

| 리스크 | 영향 | 완화 방안 |
|---|---|---|
| NAS 네트워크 레이턴시 | 대용량 스크린샷 저장 속도 저하 | NAS를 동일 Zone에 구성, 로컬 캐시 후 비동기 동기화 |
| Worker 장애 시 세션 유실 | RUNNING 세션이 FAILED로 전환되지 않을 수 있음 | 기존 `sync_job_status_runner`로 감지 (30초 주기) |
| DB 접근 | Worker → Master PostgreSQL TCP 접속 필요 | 방화벽 정책으로 Worker IP만 5432 허용 |
| KT Cloud 콘솔 제약 | CLI/API 자동화 제한적 | 수동 프로비저닝 후 Ansible/SSH 기반 배포 스크립트 구성 |
| NAS 용량 한계 | 최대 10TB (AWS EFS 대비 제한) | 보관 기간 관리 + 자동 정리로 대응 |
