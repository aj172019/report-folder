# /design-tokens - CSS 토큰 카탈로그·건강도 분석

## 설명
root.css의 CSS 변수(디자인 토큰)를 카테고리별로 정리하고, 사용처 분석, 미사용 토큰 식별, 동일 값 토큰(병합 후보) 탐지, 신규 토큰 제안을 수행한다.

## 사용법
```
/design-tokens            # 전체 카테고리
/design-tokens color      # 컬러 토큰만
/design-tokens spacing    # 스페이싱(거리) 토큰만
/design-tokens typography # 타이포그래피 토큰만
```

## 에이전트
designer (`.claude/agents/designer.md`)

## 실행 절차

### Step 1: root.css 전체 파싱
`web-bo/src/assets/css/uikit/root.css`에서 모든 CSS 변수를 추출하여 카테고리별 분류.

카테고리 분류 기준:
- **color**: `--sub-color-*`, `--neutral-*`, `--black-*`, `--color-*`, `--line_*`, `--bg_*`, `--bg-color-*`
- **spacing**: `--d_*`
- **typography**: `--h1`~`--h6`, `--sub_tit_*`, `--body_*`, `--fz_*`, `--caption`, `--font-*`, `--w-*`, `--ls_*`
- **border**: `--br_*`
- **shadow**: `--inset_*`
- **z-index**: `--z-*`
- **gap**: `--gap-*`
- **layout**: `--width_*`, `--height_*`, `--size_*`

인자가 주어진 경우 해당 카테고리만 분석.

### Step 2: 토큰 카탈로그 테이블
```markdown
| 토큰명 | 값 | 카테고리 | 사용처 수 |
|--------|-----|---------|----------|
```

### Step 3: 사용처 검색
각 토큰에 대해 `var(--토큰명)` 패턴으로 전체 프로젝트 검색.
- 검색 대상: `web-bo/src/**/*.{css,vue,ts}`
- root.css 내 정의 라인은 제외

### Step 4: 미사용 토큰 식별
사용처 수가 0인 토큰 목록 보고.
```markdown
## 미사용 토큰 (삭제 후보)
| 토큰명 | 값 | 정의 위치 |
|--------|-----|----------|
```

### Step 5: 동일 값 토큰 (병합 후보)
같은 값을 가진 서로 다른 토큰 식별.
```markdown
## 동일 값 토큰 (병합 후보)
| 값 | 토큰 1 | 토큰 2 | 비고 |
|----|--------|--------|------|
```
예: `--neutral-b-6`(#f0f0f0)과 `--neutral-color-nf-0`(#f0f0f0)

### Step 6: 하드코딩 값 → 신규 토큰 제안
자주 하드코딩되는 값(3회 이상)을 수집하여 신규 토큰 생성 제안.
```markdown
## 신규 토큰 제안
| 하드코딩 값 | 사용 횟수 | 제안 토큰명 | 비고 |
|------------|----------|------------|------|
```

### Step 7: 건강도 요약
```markdown
## 토큰 건강도 요약
- 총 토큰 수: N
- 사용 중: N (N%)
- 미사용: N (N%)
- 중복 정의: N
- 동일 값 쌍: N
- 하드코딩 대체 가능: N값
```
