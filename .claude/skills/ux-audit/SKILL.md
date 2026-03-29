# /ux-audit - 종합 UX 감사

## 설명
web-bo의 접근성(WCAG 2.1 AA), 사용성, 인터랙션 패턴을 종합 감사하여 a11y 위반, 사용성 결함, 인터랙션 불일치를 탐지한다.

## 사용법
```
/ux-audit              # 전체 감사
/ux-audit a11y         # 접근성만
/ux-audit forms        # 폼 관련만 (라벨, 검증, 에러 메시지)
/ux-audit navigation   # 네비게이션/인터랙션만
```

## 에이전트
uiux-expert (`.claude/agents/uiux-expert.md`)

## 실행 절차

### Step 1: 접근성 (a11y) 감사

#### 1.1 이미지 alt 누락
```
대상: web-bo/src/**/*.vue
패턴: <img 태그에서 alt 속성이 없거나 빈 문자열이 아닌데 누락된 경우
Grep: "<img[^>]*(?<!alt=)[^>]*/?\s*>" (alt= 가 없는 img 태그)
```

#### 1.2 아이콘 버튼 aria-label 누락
```
대상: web-bo/src/**/*.vue
탐지:
1. <button> 또는 @click 이 있는 요소 내부에 Icon* 컴포넌트만 있고 텍스트 없는 경우
2. 해당 요소에 aria-label 이 없는 경우
```

#### 1.3 폼 라벨 미연결
```
대상: web-bo/src/**/*.vue
탐지:
1. <input>, <select>, <textarea>, InputText, Dropdown, Calendar 등 폼 요소
2. 인접한 <label> 의 for 속성이 폼 요소의 id와 매칭되지 않는 경우
3. 또는 <label> 내부에 폼 요소가 없는 경우
```

#### 1.4 모달 포커스 트래핑
```
대상: web-bo/src/**/*.vue
탐지:
1. Dialog, Modal 컴포넌트 사용 부분
2. PrimeVue Dialog는 기본 지원 — 커스텀 모달만 검사
3. focus-trap 라이브러리 또는 수동 구현 유무
```

#### 1.5 커스텀 인터랙티브 요소
```
대상: web-bo/src/**/*.vue
패턴: <div @click 또는 <span @click 에 tabindex, role, @keydown 중 하나라도 누락
Grep: "<(div|span)[^>]*@click" → tabindex 체크
```

#### 1.6 컬러 대비 검사
```
root.css 토큰에서 주요 전경/배경 조합의 대비비 계산:
- 본문 텍스트(--neutral-color-n-33 #333) on 배경(--neutral-white #fff) → 12.63:1 ✓
- 비활성 텍스트(--neutral-color-nsp #a8afbf) on 배경(#fff) → ?
- 서브컬러(--sub-color-sub-01 #f95729) on 배경(#fff) → ?
- 계산: 상대 휘도 비율 (L1 + 0.05) / (L2 + 0.05)
```

### Step 2: 사용성 감사

#### 2.1 로딩 상태
```
탐지:
1. service 호출(axios)이 있는 컴포넌트에서 loading ref/reactive 선언 확인
2. template에서 v-if="loading" 또는 LoadingComponent 사용 확인
3. 로딩 미표시 페이지 목록 보고
```

#### 2.2 빈 상태(empty state)
```
탐지:
1. DataTable 사용처에서 :emptyMessage 또는 #empty 슬롯 확인
2. v-for 사용처에서 배열.length === 0 분기 처리 확인
3. 미처리 목록 보고
```

#### 2.3 폼 에러 메시지
```
탐지:
1. VeeValidate Form/Field 사용처에서 ErrorMessage 컴포넌트 존재 확인
2. 또는 :class에 에러 스타일만 있고 텍스트 피드백 없는 경우
```

#### 2.4 중복 제출 방지
```
탐지:
1. @click 또는 @submit 핸들러가 있는 제출 버튼
2. :disabled 바인딩이 없거나 loading 상태와 연결 안 된 경우
```

#### 2.5 v-html XSS 위험
```
탐지:
1. v-html 디렉티브 전수 검색
2. 바인딩된 데이터의 소스 추적 (API 응답 → sanitize 여부)
```

### Step 3: 인터랙션 패턴 감사

#### 3.1 모달 ESC 닫기
- PrimeVue Dialog: 기본 지원 확인
- 커스텀 모달: `@keydown.esc` 또는 `@keyup.esc` 핸들러 확인

#### 3.2 DataTable 일관성
```
탐지:
1. 모든 DataTable 사용처에서 :sortable, :paginator, :rows, :rowsPerPageOptions 설정 수집
2. 설정값 편차 보고 (예: 어떤 테이블은 rows=10, 다른 건 rows=20)
```

#### 3.3 CRUD 흐름 일관성
- 목록→상세→수정→저장 패턴이 도메인별로 일관한지 비교

### Step 4: Chrome MCP 라이브 검수
1. `tabs_context_mcp`로 현재 브라우저 탭 확인
2. dev 서버 페이지 접속 (열려 있으면 사용, 없으면 새 탭 생성)
3. `javascript_tool`로 ARIA 속성, tabindex, role 전수 검사
   ```javascript
   // 예: aria-label 없는 버튼 찾기
   document.querySelectorAll('button:not([aria-label])').length
   ```
4. `computer`로 주요 페이지 스크린샷 캡처
5. `gif_creator`로 모달 열기/닫기, 탭 전환 등 인터랙션 GIF 기록
6. `get_page_text`로 텍스트 계층 구조·라벨 연결 검증

### Step 5: 보고서 출력
심각도별 이슈를 WCAG 기준·파일:라인과 함께 정리.
