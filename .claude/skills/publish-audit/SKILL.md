# /publish-audit - 퍼블리싱 품질 감사

## 설명
HTML/CSS 퍼블리싱 품질을 감사하여 `!important` 남용, CSS 클래스 중복정의, 클래스명 오타, 시맨틱 HTML 위반, 인라인 스타일 사용 등을 탐지한다.

## 사용법
```
/publish-audit                          # 전체 src/ 감사
/publish-audit web-bo/src/views/member/ # 특정 경로만
```

## 에이전트
publisher (`.claude/agents/publisher.md`)

## 실행 절차

### Step 1: `!important` 감사
```
대상: web-bo/src/**/*.{css,vue}
Grep 패턴: "!important"
분석:
1. 각 !important 사용처의 파일:라인 수집
2. 유틸리티 클래스(spacing.css 등) 내 !important → 허용 (INFO)
3. 컴포넌트/페이지 내 !important → WARNING
4. 셀렉터 특이성 조정으로 제거 가능한지 판단
```

### Step 2: CSS 클래스 중복 정의
```
대상: web-bo/src/assets/css/**/*.css
탐지 방법:
1. 각 CSS 파일에서 셀렉터(클래스명) 추출
2. 동일 클래스명이 동일 파일 내에서 2회 이상 정의된 경우 보고
3. 서로 다른 파일에서 동일 클래스 정의 → 별도 보고 (의도적일 수 있음)
알려진 사례: .wf-mb-10 (spacing.css 내 중복)
```

### Step 3: 클래스명 오타 탐지
```
대상: web-bo/src/assets/css/**/*.css
탐지 방법:
1. wf-* 또는 wf_* 패턴의 클래스명 전체 추출
2. 영단어 사전 매칭으로 의미 분석
3. 의심스러운 클래스명 보고
알려진 사례: .wf-feetback (올바른 철자: feedback)
```

### Step 4: 시맨틱 HTML 감사
```
대상: web-bo/src/**/*.vue의 <template> 영역
검수 항목:
1. <div> 남용: 의미에 맞는 시맨틱 태그 대체 가능 여부
   - 내비게이션 역할 → <nav>
   - 메인 콘텐츠 → <main>
   - 구획 → <section>/<article>
   - 헤더/푸터 → <header>/<footer>
2. <button> vs <a>: @click.prevent만 있고 href 없는 <a> → <button>
3. 테이블: <thead>/<tbody> 구분, <th scope>
4. 리스트: 관련 항목 나열에 <ul>/<ol>/<li> 사용
```

### Step 5: 크로스 브라우저 호환성
```
대상: web-bo/src/**/*.{css,vue}
탐지:
1. ::-webkit-scrollbar → Firefox 미지원 경고
2. vendor prefix 필요 속성 미적용
3. CSS Grid/Flexbox gap → IE 미지원 (IE 지원 대상인 경우만)
```

### Step 6: 인라인 스타일 감사
```
대상: web-bo/src/**/*.vue
Grep 패턴: ":style=" 또는 "style="
분석:
1. :style 바인딩 사용처 전수 검색
2. 정적 인라인 style (동적이 아닌 경우) → CSS 클래스로 대체 권장
3. 동적 :style은 허용하되 하드코딩 값 확인
```

### Step 7: Chrome MCP 라이브 검수
1. `navigate`로 대상 페이지 접속
2. `computer`로 렌더링 스크린샷 → 레이아웃 깨짐, 오버플로 감지
3. `javascript_tool`로 DOM 구조 검사:
   ```javascript
   // div 중첩 깊이 측정
   function maxDepth(el, depth) {
     return Math.max(depth, ...Array.from(el.children).map(c => maxDepth(c, depth + 1)));
   }
   maxDepth(document.body, 0);
   ```
4. `javascript_tool`로 computed style 검사 (!important 실제 영향 확인)

### Step 8: 보고서 출력
```markdown
# 퍼블리싱 감사 보고서

## 요약
- 검사 파일 수: N
- !important 사용: N회 (제거 가능 N)
- CSS 중복 정의: N건
- 클래스명 오타: N건
- 시맨틱 위반: N건
- 인라인 스타일: N건

## !important 사용처
| # | 파일:라인 | 셀렉터 | 제거 가능 | 비고 |
|---|----------|--------|----------|------|

## CSS 클래스 중복
| # | 클래스명 | 파일:라인(1) | 파일:라인(2) | 동일/상이 |
|---|---------|------------|------------|----------|

## 클래스명 오타
| # | 현재 클래스 | 파일:라인 | 추정 올바른 이름 |
|---|-----------|----------|----------------|

## 시맨틱 HTML 위반
| # | 현재 태그 | 파일:라인 | 권장 태그 | 근거 |
|---|----------|----------|----------|------|

## 인라인 스타일
| # | 파일:라인 | 스타일 내용 | CSS 클래스 대체 가능 |
|---|----------|-----------|-------------------|
```
