# /query-audit - jOOQ 쿼리 감사

## 설명
Repository/Port 변경 파일을 대상으로 jOOQ 쿼리 품질을 검증한다. 안티패턴을 탐지하고 개선안을 제시한다.

## 사용법
```
/query-audit                           # git diff 기반 Repository 파일 자동 감지
/query-audit member/common/.../port/   # 특정 디렉토리 대상
```

## 대상 파일
- `**/domain/port/repository/**` 경로의 변경 파일
- `*Repository*.java`, `*Port*.java` 패턴
- jOOQ `DSLContext`를 사용하는 모든 클래스

## 체크리스트

### N+1 쿼리
- [ ] 루프 내에서 쿼리 호출이 없는지 확인
- [ ] `for`/`forEach`/`stream().map()` 안에서 Repository 메서드 호출 탐지
- **심각도**: BLOCKER
- **수정**: 배치 조회 또는 JOIN으로 변환

### 페이지네이션
- [ ] 대량 조회 가능성이 있는 쿼리에 `limit()`/`offset()` 또는 페이지네이션 파라미터 존재
- [ ] `fetchInto()` / `fetch()`가 전체 테이블을 읽을 가능성
- **심각도**: WARNING
- **수정**: 페이지네이션 파라미터 추가 또는 `limit()` 적용

### SELECT 컬럼 명시
- [ ] `select()` 없이 전체 레코드를 가져오는 경우 (`selectFrom()` 남용)
- [ ] 필요한 컬럼만 명시적으로 선택하는지 확인
- **심각도**: WARNING
- **수정**: 필요한 컬럼만 `select(TABLE.COL1, TABLE.COL2, ...)` 형태로 지정

### 트랜잭션 설정
- [ ] QueryService의 조회 메서드에 `@Transactional(readOnly = true)` 적용 여부
- [ ] CommandService의 변경 메서드에 `@Transactional` 적용 여부
- **심각도**: WARNING
- **수정**: 적절한 `@Transactional` 어노테이션 추가

### 안전한 UPDATE/DELETE
- [ ] WHERE 조건 없는 `update()` / `delete()` 문 탐지
- **심각도**: BLOCKER
- **수정**: 반드시 WHERE 조건 추가, 의도적 전체 삭제라면 명시적 주석

### JOIN 최적화
- [ ] 불필요한 JOIN이 없는지 확인 (사용하지 않는 테이블과의 JOIN)
- [ ] JOIN 조건이 인덱스를 활용하는지 확인
- **심각도**: WARNING
- **수정**: 불필요한 JOIN 제거, 필요 시 서브쿼리로 변환

### 트랜잭션 경계
- [ ] 트랜잭션이 Service 레이어에서만 시작되는지 확인
- [ ] Repository/Controller에서 `@Transactional`을 사용하지 않는지
- **심각도**: WARNING
- **수정**: 트랜잭션 어노테이션을 Service 레이어로 이동

## 출력 형식

```
## 쿼리 감사 결과

### 대상 파일
- `{파일 경로}` (N개 쿼리 분석)

### BLOCKER (N건)
1. `파일명:라인` — [N+1 쿼리] 루프 내에서 `findByKey()` 호출
   → 권장: `findByKeys(List<Long> keys)` 배치 메서드 추가

### WARNING (N건)
1. `파일명:라인` — [페이지네이션 누락] `fetchInto()` 전체 조회
   → 권장: `limit(pageSize).offset(offset)` 추가

### 이슈 없음
쿼리 품질 검증 통과.

### 요약
- 분석 쿼리: N개
- 총 이슈: N건 (BLOCKER: X, WARNING: Y)
```

## 원칙
- Repository/Port 파일이 변경되지 않았으면 실행하지 않는다
- 기존 쿼리의 문제는 지적하지 않음 — 변경/추가된 쿼리만 대상
- BLOCKER는 반드시 수정 필요, WARNING은 맥락에 따라 판단
