# /review - 변경 코드 빌드/테스트/코드 리뷰

## 설명
변경된 코드를 빌드/테스트/컨벤션 관점에서 검증하고 이슈를 보고한다. 사소한 컨벤션 이슈는 직접 수정한다.

## 사용법
```
/review                # git diff 기반 자동 감지
/review member         # 백엔드만
/review web-bo         # 프론트엔드만
```

## 에이전트
- member 대상: reviewer-be (`.claude/agents/reviewer-be.md`)
- web-bo 대상: reviewer-fe (`.claude/agents/reviewer-fe.md`)
- 자동 감지: git diff 결과에서 대상 레포 판별

## 실행 절차

### Step 1: 변경 파일 확인
`git diff --name-only` (또는 `git diff --staged --name-only`)로 변경 파일 목록 확인.
- `member/` 파일 변경 → 백엔드 검증
- `web-bo/` 파일 변경 → 프론트엔드 검증
- 양쪽 변경 → 둘 다 검증

### Step 2: 빌드/테스트 실행
- **BE**: `cd member && ./gradlew build && ./gradlew test`
- **FE**: `cd web-bo && yarn lint && yarn type-check && yarn build`

### Step 3: 코드 리뷰
변경된 파일을 대상으로 코드 리뷰 수행:
- 기존 컨벤션 준수 여부
- 누락된 레이어나 파일
- 프로젝트 규정 위반 여부
- 불필요한 코드나 중복
- (FE) 접근성 속성, CSS 토큰 사용

### Step 4: 이슈 보고서 출력

**통과:**
```
## 검증 통과
- 빌드: 성공
- 테스트: 성공
- 코드 리뷰: 이슈 없음 (또는 사소한 이슈 N건 직접 수정)
```

**피드백:**
```
## 검증 피드백
### 빌드/테스트
- 결과: 실패
- 에러: `파일명:라인` — 에러 내용

### 코드 리뷰
1. `파일명:라인` — 이슈 설명 및 수정 권장
```
