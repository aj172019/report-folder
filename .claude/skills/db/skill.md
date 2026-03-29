# /db - 데이터베이스 접속

PostgreSQL 데이터베이스에 접속하여 SQL을 실행한다.

## 사용법

```
/db <환경> <SQL>
```

- 환경: `local`, `dev`, `prod` (생략 시 `local`)
- SQL: 실행할 쿼리 (생략 시 접속 테스트)

## 환경별 접속 정보

### local (로컬)
```bash
docker exec fnbplus_postgresql psql -U root -d icndb -c "<SQL>"
```
로컬 Docker 컨테이너 직접 접속.

### dev (개발)
```bash
psql "postgresql://dbadmin:icnadmin12%21@private-postgresql-read-write-e7eaef56.ktcloud.io:5432/icndb?sslmode=prefer&channel_binding=disable" -c "<SQL>"
```

### prod (운영)
**운영 DB는 읽기 전용으로만 접속한다. DML(INSERT/UPDATE/DELETE)은 절대 실행하지 않는다.**
```bash
psql "postgresql://dbadmin:mEsUw9nYnEU2KjmGyf8eiZbwDm2s474U@private-postgresql-read-write-c5d703aa.ktcloud.io:5432/icndb?sslmode=prefer&channel_binding=disable" -c "<SQL>"
```

## 규칙

1. 환경을 인자로 받거나, 사용자에게 질문하여 확정한다
2. `prod` 환경은 SELECT만 허용한다. DML 요청 시 거부하고 사유를 안내한다
3. SQL 실행 후 결과를 간결하게 보고한다
4. 대량 결과(100행 초과)는 `LIMIT`을 붙여 실행하고, 전체 건수를 별도 안내한다
