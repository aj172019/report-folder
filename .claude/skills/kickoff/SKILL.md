# /kickoff - Jira 에픽/태스크 생성 및 팀 워크플로 시작

## 설명
PM으로서 Jira 에픽/태스크를 생성하고, 작업 규모에 맞는 에이전트 팀을 구성하여 분석→구현→검증 워크플로를 실행한다.

## 사용법
```
/kickoff 타겟팅 메타 조회 API 추가
/kickoff AFNBPB-318 필드 정의 CRUD
```
- 에픽 KEY를 지정하면 해당 에픽에 태스크를 추가한다.
- 에픽 KEY 없이 작업명만 입력하면 에픽을 조회/생성한다.

## 에이전트
pm (`.claude/agents/pm.md`)

## 실행 절차

### Step 1: 에픽 조회 또는 생성
- 인자에 에픽 KEY가 있으면 해당 에픽 사용 (`jira_search` 스킵)
- 없으면 `jira_search`로 기존 에픽 검색 → 적절한 에픽이 없으면 `jira_create_issue`로 신규 생성

### Step 2: 3태스크 생성
`jira_create_issue`로 분석/구현/검증 태스크를 각각 생성:
- `분석: {작업명}` (issuetype = `작업`)
- `구현: {작업명}` (issuetype = `작업`)
- `검증: {작업명}` (issuetype = `작업`)

에픽 연결: `jira_update_issue`로 `fields: {"parent": {"key": "{에픽KEY}"}}`

### Step 3: blocks 관계 설정
`jira_create_issue_link`로:
- 분석 → 구현 (blocks)
- 구현 → 검증 (blocks)

### Step 4: 작업 규모 판단

| 규모 | 기준 | 워크플로 |
|------|------|----------|
| **소** | 1~2파일 수정 | PM이 직접 처리 |
| **중** | 3~5파일 | PM + Developer (2인) |
| **대** | 6파일 이상 | PM + Analyzer + Developer + Reviewer (4인) |

### Step 5: 팀 구성 및 실행
작업 내용에 따라 BE/FE 에이전트를 선택:
- member 대상 → analyzer-be, developer-be, reviewer-be
- web-bo 대상 → analyzer-fe, developer-fe, reviewer-fe
- 양쪽 → BE/FE 에이전트 각각 spawn

순차 실행: Analyzer → Developer → Reviewer
피드백 루프 최대 3회, PM이 횟수 추적.

### Step 6: 최종 보고
사용자에게 Jira 이슈 링크와 변경 요약 전달.
에픽에 최종 요약 코멘트 1회 작성.
