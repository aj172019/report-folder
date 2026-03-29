# /helm-deploy - 헬름 차트 이미지 빌드 & 배포

헬름 차트의 프론트엔드 이미지를 빌드하고 NCloud Container Registry에 푸시한다.

## 사용법

```
/helm-deploy <chart-name>
```

- `<chart-name>`: 대상 차트 (예: `if-scalar`, `scalar`, `swagger`)

## 절차

1. NCloud Container Registry 로그인
```bash
docker login incheondfs-dev.kr.ncr.ntruss.com -u <NCLOUD_ACCESS_KEY> -p <NCLOUD_SECRET_KEY>
```

2. 차트 디렉토리로 이동하여 build.sh 실행
```bash
cd helm-charts/helm-charts/addon-charts/<chart-name>/<ui-dir>
bash build.sh
```

- `if-scalar` → `scalar-ui/build.sh`
- `scalar` → `scalar-ui/build.sh`
- `swagger` → `swagger-ui/build.sh`

3. 빌드 & 푸시 완료 후 사용자에게 ArgoCD에서 해당 차트 restart 안내

## 주의사항

- Docker Desktop이 실행 중이어야 한다
- `build.sh`는 `linux/amd64` 플랫폼으로 빌드한다
- 이미지 태그는 `custom`으로 고정 (dev 환경)
- ArgoCD restart는 수동으로 진행해야 한다
