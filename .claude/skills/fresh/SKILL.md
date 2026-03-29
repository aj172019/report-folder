# /fresh - 클린 마이그레이션

볼륨 초기화 → PostgreSQL 기동 → Flyway 마이그레이션을 순차 실행한다.

## 절차

1. `docker-compose down -v`
2. `docker-compose up -d postgres`
3. PostgreSQL healthy 대기 (최대 30초)
4. `docker-compose run --rm flyway migrate`

모든 명령은 `developer/` 디렉토리에서 실행한다. 각 단계 실패 시 즉시 중단하고 보고한다.

5. 마이그레이션 성공 후 `docker-compose up -d` 로 ERD 뷰어 등 전체 서비스를 기동한다.
