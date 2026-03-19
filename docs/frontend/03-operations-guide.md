# 프론트엔드 — 운영 및 관리 가이드

---

## 1. 인프라 정보

### 1.1 Docker Compose 서비스

| 컨테이너 | 이미지 | 포트 | 역할 |
|----------|--------|:----:|------|
| `vibe-crawler-frontend` | `vibe-frontend:latest` | 80, 443 | Vue SPA + Nginx |

### 1.2 서비스 접근 URL

| 환경 | URL |
|------|-----|
| 로컬 | http://localhost:5173 |
| 운영 | `http://<EC2_IP>:3001` (또는 `:80`) |

---

## 2. 빌드 및 배포

### 2.1 Docker 빌드

```bash
# ECR에 빌드 & Push (플랫폼 전체)
./infrastructure/scripts/build-and-push-to-ecr.sh
```

- Frontend 이미지는 **Nginx** 기반으로 빌드
- AMD64 플랫폼 강제 빌드 (Mac ARM64 ↔ EC2 AMD64 호환)

### 2.2 EC2 배포

```bash
# 기존 서버에 재배포
./infrastructure/scripts/deploy-to-ec2.sh <EC2_IP>
```

### 2.3 AWS 정보

| 항목 | 값 |
|------|-----|
| ECR 레포지토리 | `vibe-frontend` |
| Region | `ap-northeast-2` (서울) |

---

## 3. Nginx 설정

- Vue SPA의 `index.html`로 모든 경로 폴백 (SPA 라우팅)
- 백엔드 API 프록시: `/api/*` → `vibe-crawler-backend:6767`

---

## 4. 로그 확인

### Docker 환경

```bash
ssh -i heesun.pem ubuntu@<IP>
cd ~/vibe-deployment
docker-compose logs -f frontend
```

### 로컬 환경

```bash
yarn dev
# 브라우저 DevTools → Console / Network 탭
```

---

## 5. 에러 대응

1. **빌드 실패**: `yarn build` 로컬 실행으로 에러 확인
2. **API 연결 실패**: `VITE_API_BASE_URL` 환경변수 확인
3. **Nginx 502/504**: 백엔드 컨테이너 상태 확인 (`docker-compose ps`)
4. **캐시 문제**: 브라우저 캐시 클리어 또는 Nginx 재시작
