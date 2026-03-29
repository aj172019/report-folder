# /write-plan - 구현 계획 수립

## 설명
코드 작성 전 구현 계획을 수립한다. 작업을 2-7개 단계로 분해하고, 각 단계마다 대상 파일·참조 파일·검증 기준을 명시한다.

## 사용법
```
/write-plan 타겟팅 메타 CRUD API 구현
/write-plan 회원 등급 변경 이벤트 처리
```

## 절차

### Step 1: 요구사항 정리
- 작업 목표를 1-2문장으로 요약
- 영향 범위 (어떤 레이어, 어떤 모듈)
- 선행 조건 (Analyzer 분석 결과, 기존 패턴 등)

### Step 2: 단계 분해
작업을 **2-7개 단계**로 분해한다. 각 단계는 다음을 포함:

```
### 단계 N: {단계 제목}
- **대상 파일**: 생성/수정할 파일 경로
- **참조 파일**: 패턴을 따를 기존 파일 경로
- **참조 스킬**: 적용할 스킬 (해당 시)
- **작업 내용**: 구체적으로 무엇을 구현하는지
- **검증 기준**: 이 단계가 완료되었다고 판단하는 조건
```

### Step 2b: DDD 신규 도메인 작업 시 단계 분해 템플릿

신규 도메인 모듈을 처음부터 구현하는 경우, **domain-outward 순서**를 따른다:

```
### 단계 0: 도메인 모델 설계
- 참조 스킬: /domain-model
- 작업: Exception 계층 → Value Object → Entity + Collection VO
- 검증: 컴파일 성공, create()에 유효하지 않은 값 → 예외 발생

### 단계 1: Port 인터페이스 정의
- 참조 스킬: /port-adapter
- 작업: Repository (Command/Query) → Assembler → Resolver 인터페이스
- 검증: 인터페이스에 프레임워크 의존 없음

### 단계 2: DTO 설계
- 참조 스킬: /dto-design
- 작업: Request (record) → Query (class + assign*) → Response (class + of(Query))
- 검증: 컴파일 성공, Request에 validation 어노테이션 존재

### 단계 3: Service 구현
- 작업: CommandService → QueryService
- 검증: 컴파일 성공

### 단계 4: Controller 구현
- 작업: REST endpoints, CommonResponse 반환
- 검증: 컴파일 성공

### 단계 5: Infrastructure Adapter 구현
- 참조 스킬: /port-adapter
- 작업: Repository Impl (jOOQ) → Assembler Impl (batch load)
- 검증: 컴파일 성공, Assembler에 루프 내 쿼리 없음

### 단계 6: 빌드 검증
- 작업: ./gradlew build
- 검증: 빌드 성공
```

### Step 2c: 기존 기능 수정/확장 시

기존 기능을 수정하는 경우, 영향받는 레이어만 선별:
- 변경이 필요한 가장 안쪽 레이어부터 바깥으로 진행
- 기존 패턴 100% 준수 (유사 코드 참조)
- 불필요한 레이어 생성 금지

### Step 3: 의존성 확인
- 단계 간 의존성을 명시 (어떤 단계가 선행되어야 하는지)
- 병렬 실행 가능한 단계가 있으면 표시

### Step 4: 계획 출력
아래 형식으로 계획을 출력한다:

```
## 구현 계획: {작업 제목}

### 목표
{1-2문장 요약}

### 영향 범위
- 모듈: {모듈명}
- 레이어: {Controller, Service, Repository, Entity, DTO 등}

### 단계별 계획

#### 단계 1: {제목}
- 대상: `{파일 경로}`
- 참조: `{기존 파일 경로}`
- 스킬: {/domain-model, /port-adapter, /dto-design 등}
- 작업: {구체적 내용}
- 검증: {완료 조건}

#### 단계 2: ...
(반복)

### 의존성
- 단계 0 → 단계 1 (Entity가 있어야 Port 인터페이스 정의 가능)
- 단계 1 → 단계 5 (Port가 있어야 Adapter 구현 가능)
- 단계 2, 단계 3 (병렬 가능 — DTO와 Service 독립)

### 최종 검증
- `./gradlew build` 성공
- {추가 검증 항목}
```

## 원칙
- **Karpathy Rule #4**: 성공 기준을 정의하고, 검증될 때까지 반복
- 각 단계의 검증 기준은 구체적이고 실행 가능해야 한다 ("동작하게 하기" ✗ → "빌드 성공 + GET 요청 시 200 응답" ✓)
- 불확실한 부분이 있으면 계획에 명시하고 사용자에게 질문한다
- **DDD 신규 도메인**: 반드시 domain-outward 순서 (Domain → Port → DTO → Service → Controller → Infrastructure)
- **기존 기능 수정**: 최소 변경 원칙, 영향받는 레이어만
