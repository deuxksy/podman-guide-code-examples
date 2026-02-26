# tailscale-dev/docker-guide-code-examples/08-ts-heritage

이 예제는 Caddy 리버스 프록시와 Tailscale HTTPS 종료를 사용하는 방법을 보여줍니다.

## 현재 목표: HTTPS 연동 검증

Tailscale + Caddy 연동이 작동하고 `https://heritage.bun-bull.ts.net`로 접근 가능한지 확인합니다.

## 장기 목표: 다중 서비스 리버스 프록시

추후 단일 도메인에서 경로 기반 라우팅으로 여러 서비스에 접근할 수 있도록 확장합니다.

## 아키텍처

```
https://heritage.bun-bull.ts.net:443
     ↓
[heritage] Tailscale 사이드카 (HTTPS 종료)
     ↓
[Caddy] 리버스 프록시 :9091
     ↓
현재: 정적 파일 (/srv)
추후: /rutorrent → rutorrent 컨테이너
      /whisparr → whisparr 컨테이너
      /jellyfin → jellyfin 컨테이너
```

## 전제 조건

1. Tailscale Admin Console의 `DNS` 탭에서 HTTPS 인증서 활성화
2. Tailscale AuthKey 생성: https://login.tailscale.com/admin/settings/keys
3. `heritage.bun-bull.ts.net` 도메인이 자동으로 구성됨

## 빠른 시작

```bash
cd 08-ts-heritage
cp .env.example .env
# .env 파일을 편집하여 TS_AUTHKEY 추가
podman compose up -d
```

검증용 초기 정적 파일은 `site/` 디렉토리에 있습니다.

사이트 URL: `https://heritage.bun-bull.ts.net`

## 다중 서비스 확장 예시

`config/Caddyfile`에 다음을 추가:

```caddyfile
:9091 {
    handle_path /rutorrent* {
        reverse_proxy rutorrent:8080
    }
    handle_path /whisparr* {
        reverse_proxy whisparr:6969
    }
    handle_path /jellyfin* {
        reverse_proxy jellyfin:8096
    }
    # 기본: 정적 파일
    file_server
}
```

## 트러블슈팅

- Tailscale 로그: `podman compose logs heritage`
- Caddy 로그: `podman compose logs caddy`
- Tailscale Admin Console에서 HTTPS가 활성화되어 있는지 확인
