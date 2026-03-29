# /mig - Flyway 마이그레이션

Flyway 마이그레이션을 실행한다.

```bash
cd developer && docker-compose run --rm flyway migrate
```

실행 후 결과를 간단히 보고한다.

마이그레이션 성공 시 `cd developer && docker-compose up -d` 로 ERD 뷰어 등 전체 서비스를 기동한다.
