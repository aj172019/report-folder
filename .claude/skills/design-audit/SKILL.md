# /design-audit - 디자인 시스템 전체 감사

## 설명
web-bo의 디자인 시스템(CSS 토큰, 컴포넌트, 스타일 일관성)을 전체 감사하여 토큰 중복정의, 클래스-토큰 값 불일치, 하드코딩, Welfare 래퍼 미사용 등의 결함을 탐지한다.

## 사용법
```
/design-audit
/design-audit web-bo/src/views/member/
```

## 에이전트
designer (`.claude/agents/designer.md`)

## 실행 절차

### Step 1: CSS 토큰 카탈로그 수집
`web-bo/src/assets/css/uikit/root.css`를 파싱하여 전체 토큰 목록 추출.
- 컬러: `--sub-color-*`, `--neutral-*`, `--black-*`, `--color-*`
- 타이포: `--h1`~`--h6`, `--body_*`, `--fz_*`, `--caption`
- 거리: `--d_*` (약 130개)
- 보더: `--br_*`
- 굵기: `--w-*`

### Step 2: 토큰 중복정의 탐지
root.css 내에서 동일 CSS 변수명이 2회 이상 선언된 경우를 찾는다.
```
탐지 방법: root.css에서 "--" 로 시작하는 변수 선언을 모두 추출 → 변수명 기준 그룹핑 → count > 1인 항목 보고
```
알려진 버그: `--d_100` (line 144, 160), `--d_153` (line 174, 176)

### Step 3: 클래스-토큰 값 불일치 탐지
spacing.css에서 `.wf-{방향}-{N}` 클래스가 `var(--d_{N})`이 아닌 다른 토큰을 참조하는 경우.
```
탐지 방법:
1. spacing.css에서 ".wf-(mt|mb|ml|mr|mx|my|pt|pb|pl|pr|px|py|p|m|gap|top|left|right|bottom)-{숫자}" 패턴 추출
2. 각 클래스의 CSS 속성에서 var(--d_{숫자}) 참조값 추출
3. 클래스명의 숫자 != 참조 토큰의 숫자 → 불일치 보고
```
알려진 버그: `.wf-mt-28 { margin-top: var(--d_27) }`, `.wf-px-15 { ... var(--d_14) }`

### Step 4: 하드코딩 탐지
`var()` 없이 CSS에 직접 값을 쓴 경우를 탐지한다.

대상: `web-bo/src/**/*.css`, `web-bo/src/**/*.vue` (root.css 제외)

탐지 패턴:
- 색상 하드코딩: `#[0-9a-fA-F]{3,8}` (단, CSS 변수 정의 라인 제외)
- 크기 하드코딩: `[0-9]+\.?[0-9]*px` (단, `var()` 내부, CSS 변수 정의 라인, `0px` 제외)
- rem 하드코딩: `[0-9]+\.?[0-9]*rem` (동일 조건)

알려진 사례: `right: 14px` (welfare-input.css:262)

### Step 5: Welfare 래퍼 사용률 분석
```
탐지 방법:
1. web-bo/src/components/uikit/ 하위 파일에서 래퍼 컴포넌트명 목록 수집
2. src/views/, src/components/ 내 .vue 파일에서 PrimeVue 직접 import 검색
3. Welfare 래퍼가 존재하는 PrimeVue 컴포넌트의 직접 사용 → WARNING
4. 비율 계산: Welfare 사용 / (Welfare + PrimeVue 직접) × 100
```

### Step 6: Chrome MCP 라이브 검수
1. `tabs_context_mcp`로 현재 브라우저 탭 확인
2. dev 서버가 열려 있으면 해당 탭 사용, 없으면 `tabs_create_mcp`로 새 탭 생성하여 dev 서버 접속
3. `computer`로 주요 페이지 스크린샷 캡처 → 시각적 일관성 비교
4. `javascript_tool`로 computed style 검사 (실제 적용된 CSS 값 vs 토큰 값)
5. `read_page`로 렌더된 DOM 구조에서 스타일 클래스 사용 현황 확인

### Step 7: 보고서 출력
심각도별(CRITICAL/WARNING/INFO) 이슈를 파일:라인과 함께 정리.
- CRITICAL: 토큰 중복정의, 클래스-토큰 값 불일치
- WARNING: 하드코딩, PrimeVue 직접 사용
- INFO: 미사용 토큰, CSS 변수 사용률 통계, 라이브 검수 결과
