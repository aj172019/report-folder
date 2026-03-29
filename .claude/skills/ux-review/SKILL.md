# /ux-review - 특정 대상 UX 리뷰

## 설명
특정 Vue 컴포넌트나 페이지 파일을 대상으로 접근성·사용성·인터랙션을 심층 리뷰한다.
대상 파일의 import 관계를 추적하여 template/script/style 3단 검토를 수행하고, 라인별 이슈와 수정 코드를 제안한다.

## 사용법
```
/ux-review web-bo/src/views/member/MemberList.vue
/ux-review web-bo/src/views/targeting/
/ux-review web-bo/src/components/common/SearchFilter.vue
```

인자가 디렉토리면 해당 디렉토리 내 모든 `.vue` 파일을 대상으로 리뷰.

## 에이전트
uiux-expert (`.claude/agents/uiux-expert.md`)

## 실행 절차

### Step 1: 대상 파일 수집
- 파일 경로 → 해당 파일 1개
- 디렉토리 경로 → `**/*.vue` 글로브로 파일 목록 수집

### Step 2: import 관계 추적
각 대상 파일에서:
1. `<script setup>` 내 `import` 문 파싱
2. 사용된 컴포넌트, composable, service, store 파일 경로 수집
3. 1단계 import만 추적 (깊이 제한)

### Step 3: Template 검토
```
검수 항목:
- [ ] <img> alt 속성 존재
- [ ] 아이콘 버튼 aria-label 존재
- [ ] 폼 요소-라벨 연결
- [ ] v-html 사용 시 sanitize
- [ ] 커스텀 인터랙티브 요소(@click div/span)에 tabindex/role/키보드 이벤트
- [ ] 빈 상태 처리 (v-for, DataTable)
- [ ] 로딩 상태 표시
- [ ] 에러 메시지 텍스트 피드백
```

### Step 4: Script 검토
```
검수 항목:
- [ ] 폼 제출 시 중복 방지 (loading + disabled)
- [ ] API 에러 핸들링 (try/catch, toast/alert)
- [ ] 비동기 작업 로딩 상태 관리
- [ ] 컴포넌트 unmount 시 cleanup (onUnmounted)
```

### Step 5: Style 검토
```
검수 항목:
- [ ] 포커스 스타일 존재 (:focus, :focus-visible)
- [ ] hover/active 상태 스타일
- [ ] 반응형 미디어 쿼리 (필요한 경우)
- [ ] CSS 변수 사용 (하드코딩 없음)
```

### Step 6: 라인별 이슈 보고
```markdown
# UX 리뷰: {파일명}

## 요약
- 검토 파일: {대상 + import 파일 수}
- 발견 이슈: N (심각 N / 경고 N / 개선 N)

## 이슈 상세
### 1. [CRITICAL] {이슈 제목}
- **파일**: {파일:라인}
- **현재 코드**:
```html
{현재 코드}
```
- **수정 제안**:
```html
{수정 코드}
```
- **근거**: {WCAG 기준 또는 사용성 원칙}
```
