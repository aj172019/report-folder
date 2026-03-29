# /accessibility - WCAG 2.1 AA 접근성 전수 감사

## 설명
웹 접근성(WCAG 2.1/2.2 Level A/AA) 준수 여부를 전수 감사한다. 컬러 대비, 키보드 접근, ARIA, 시맨틱 HTML을 정밀 검사하여 위반 항목을 보고한다.

## 사용법
```
/accessibility                          # web-bo 전체 감사
/accessibility web-bo/src/views/member/ # 특정 경로만
/accessibility contrast                 # 컬러 대비만
/accessibility keyboard                 # 키보드 접근성만
```

## 에이전트
uiux-expert (`.claude/agents/uiux-expert.md`)

## 실행 절차

### Step 1: 컬러 대비 검사 (Color Contrast)

#### WCAG AA 기준
- 일반 텍스트 (18px 미만): **4.5:1** 이상
- 큰 텍스트 (18px+ 또는 14px+ bold): **3:1** 이상
- UI 컴포넌트/그래픽: **3:1** 이상

#### 검사 방법
1. `web-bo/src/assets/css/uikit/root.css`에서 컬러 토큰 전체 추출
2. 주요 전경/배경 조합의 대비비 계산:
   ```
   대비비 = (L1 + 0.05) / (L2 + 0.05)
   L = 상대 휘도 (sRGB → 선형 → 가중합)
   ```
3. 컴포넌트별 실제 사용 조합 추적:
   - 버튼 텍스트 on 버튼 배경
   - placeholder 텍스트 on input 배경
   - 비활성(disabled) 텍스트 on 배경
   - 에러/경고 텍스트 on 배경
   - 링크 텍스트 on 본문 배경

#### 알려진 검사 대상
| 전경 | 배경 | 대비비 | 판정 |
|------|------|--------|------|
| `--neutral-color-n-33` (#333) | `--neutral-white` (#fff) | 12.63:1 | PASS |
| `--neutral-color-nsp` (#a8afbf) | #fff | ? | 검사 필요 |
| `--sub-color-sub-01` (#f95729) | #fff | ? | 검사 필요 |

### Step 2: 색상 단독 정보 전달 (Use of Color)

색상만으로 정보를 전달하는 패턴 탐지:
```
대상: web-bo/src/**/*.vue
탐지:
1. 상태 표시에 색상만 사용 (텍스트/아이콘 없이 배경색으로만 구분)
2. 필수 필드를 빨간 테두리로만 표시 (텍스트 라벨 없음)
3. 에러 상태를 색상으로만 표시 (아이콘/텍스트 보조 없음)
4. 차트/그래프에서 색상만으로 데이터 구분
```
- **수정**: 색상 + 아이콘/텍스트/패턴 조합으로 정보 전달

### Step 3: 키보드 접근성 (Keyboard Accessibility)

#### 3.1 Tab 순서
```
탐지:
1. tabindex > 0 사용 → WARNING (자연스러운 DOM 순서 권장)
2. tabindex="-1" 이 인터랙티브 요소에 적용 → CRITICAL
3. @click 있는 <div>/<span>에 tabindex="0" 누락 → CRITICAL
```

#### 3.2 포커스 트래핑 (모달)
```
탐지:
1. PrimeVue Dialog → 기본 지원 확인
2. 커스텀 모달 → focus-trap 구현 확인
3. 모달 열릴 때 포커스 이동 → 닫힐 때 원래 요소로 복귀
```

#### 3.3 키보드 패턴
```
검수 항목:
- [ ] ESC → 모달/드롭다운 닫기
- [ ] Enter/Space → 버튼/링크 활성화
- [ ] Arrow keys → 드롭다운/탭/라디오 탐색
- [ ] 커스텀 위젯에 적절한 키보드 핸들러
```

#### 3.4 포커스 스타일
```
대상: web-bo/src/**/*.{vue,css}
탐지:
1. :focus { outline: none } 또는 :focus { outline: 0 } → CRITICAL
2. :focus-visible 대체 스타일 없이 outline 제거 → CRITICAL
3. 포커스 링이 배경과 구분되지 않는 경우 → WARNING
```

### Step 4: ARIA 및 시맨틱 HTML

#### 4.1 시맨틱 HTML 우선
```
검수:
- [ ] <nav> 대신 <div class="nav"> 사용 → WARNING
- [ ] <main> 대신 <div class="main-content"> → WARNING
- [ ] <section>/<article> 대신 <div> 나열 → INFO
- [ ] <button> 대신 <div @click> → CRITICAL
- [ ] 제목 계층 (<h1>~<h6>) 건너뛰기 → WARNING
```

#### 4.2 ARIA role 적합성
```
검수:
- [ ] role="button" 에 tabindex="0" + @keydown.enter/@keydown.space 존재
- [ ] role="dialog" 에 aria-modal="true" + aria-labelledby 존재
- [ ] role="tab" + role="tabpanel" 에 aria-selected + aria-controls 연결
- [ ] aria-label/aria-labelledby 중복 사용 없음
```

#### 4.3 라이브 리전
```
검수:
- [ ] 동적 알림/토스트에 aria-live="polite" 또는 role="alert"
- [ ] 검색 결과 갱신 시 aria-live 영역으로 결과 수 안내
- [ ] 로딩 상태 변경 시 aria-busy 적용
```

#### 4.4 폼 접근성
```
검수:
- [ ] 모든 입력 필드에 연결된 <label> (for-id 또는 wrapping)
- [ ] 필수 필드에 aria-required="true"
- [ ] 에러 메시지가 aria-describedby로 필드와 연결
- [ ] 에러 발생 시 aria-invalid="true" 설정
- [ ] fieldset/legend으로 관련 필드 그룹핑
```

### Step 5: Chrome MCP 라이브 검수
1. `tabs_context_mcp`로 브라우저 탭 확인
2. dev 서버 접속
3. `javascript_tool`로 자동 검사:
   ```javascript
   // aria-label 없는 아이콘 버튼
   document.querySelectorAll('button:not([aria-label]):not(:has(span.p-button-label))').length;
   // label 없는 input
   document.querySelectorAll('input:not([aria-label]):not([id])').length;
   // outline:none 적용된 포커스 가능 요소
   // tabindex 양수값 사용
   document.querySelectorAll('[tabindex]').forEach(el => {
     if (parseInt(el.getAttribute('tabindex')) > 0) console.warn('tabindex > 0:', el);
   });
   ```
4. `computer`로 Tab 키 탐색 테스트 스크린샷
5. `gif_creator`로 키보드 탐색 흐름 기록

### Step 6: 보고서 출력

```markdown
# 접근성 감사 보고서

## 요약
- 검사 파일: N개
- CRITICAL: N건 (즉시 수정 필요)
- MAJOR: N건 (출시 전 수정)
- MINOR: N건 (점진적 개선)

## CRITICAL 이슈
| # | 파일:라인 | WCAG 기준 | 이슈 | 수정 방법 |
|---|----------|----------|------|----------|

## MAJOR 이슈
| # | 파일:라인 | WCAG 기준 | 이슈 | 수정 방법 |
|---|----------|----------|------|----------|

## MINOR 이슈
| # | 파일:라인 | WCAG 기준 | 이슈 | 수정 방법 |
|---|----------|----------|------|----------|

## 컬러 대비 검사 결과
| 전경 토큰 | 배경 토큰 | 대비비 | AA 일반 | AA 대형 | 사용처 |
|-----------|----------|--------|---------|---------|--------|

## 키보드 탐색 테스트 결과
| 페이지 | Tab 순서 정상 | 포커스 표시 | 트랩 없음 | 비고 |
|--------|-------------|-----------|----------|------|
```

## 원칙
- WCAG 2.1 AA 기준을 엄격히 적용 (AAA는 권장 사항으로만 언급)
- PrimeVue 기본 접근성 지원을 신뢰하되, 커스텀 영역은 직접 검사
- 기존 코드의 문제도 보고 (접근성은 전체 페이지 품질이므로)
- 자동 검사 가능한 항목은 자동화, 수동 판단 필요한 항목은 명시
