# 프론트엔드 — 이슈 리스트 및 히스토리

---

## 1. 미해결 이슈

| # | 분류 | 내용 | 비고 |
|---|------|------|------|
| 1 | Orval | `watch` 콜백 실행 타이밍 불안정 | Vue 3 Composition API에서 간헐적 발생 |
| 2 | 구조 | 일부 pages 컨포넌트가 너무 큼 (ProjectDetailPage 117KB) | features로 로직 분리 필요 |

---

## 2. 해결된 버그

| # | 증상 | 원인 | 해결 |
|---|------|------|------|
| 1 | AI Improvement 버튼 클릭 시 저장되지 않은 데이터로 실행 | 사용자가 업로드 후 저장하지 않고 바로 클릭 | UI 플로우에 저장 강제 단계 추가 |

---

## 3. 기술 주의사항

### 3.1 Orval + Vue 3 watch 타이밍

- **문제**: Orval 자동 생성 API 훅에서 `watch` 콜백 실행 타이밍이 불안정
- **대응**: `watchEffect` 대신 명시적 `watch`와 `nextTick` 조합 사용
- **예시**:
```typescript
// ❌ 불안정
watchEffect(() => {
  // Orval 훅의 데이터 참조
})

// ✅ 안정
watch(
  () => queryResult.data.value,
  async (newData) => {
    await nextTick()
    // 데이터 처리
  }
)
```

### 3.2 다중 테넌트 (워크스페이스)

- 워크스페이스 전환 시 이전 워크스페이스 데이터가 잠시 표시될 수 있음
- 전환 시 Vue Query 캐시 무효화 필요
- 백엔드에서 `workspace_id` 필터로 데이터 격리 보장됨

### 3.3 API 코드 재생성

- 백엔드 API 스펙 변경 후 반드시 `yarn generate:api` 실행
- 생성된 `src/api/` 코드는 직접 수정 금지
- Orval 설정: `orval.config.ts`

### 3.4 아토믹 디자인 규칙

- atoms에 API 호출/store 접근 금지
- molecules에서 atoms만 import (다른 molecules 참조 피할 것)
- 새 컨포넌트 작성 시 반드시 계층 확인 (비즈니스 로직 → organisms/features, 레이아웃 → templates)
