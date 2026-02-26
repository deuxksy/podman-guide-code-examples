# 08-ts-heritage 다중 서비스 지원 확장 설계

**날짜:** 2026-02-27
**목표:** 단일 Caddy 서비스 예제를 다중 서비스를 지원하는 구조로 확장

## 개요

현재 `08-ts-heritage` 예제는 단일 Caddy 서비스로 정적 파일만 제공합니다. 이를 다중 서비스가 경로 기반 라우팅으로 접근 가능한 구조로 확장합니다.

## 아키텍처

### 변경 전

```
https://heritage.bun-bull.ts.net:443
     ↓
[heritage] Tailscale 사이드카
     ↓
[Caddy] network_mode: service:heritage
     ↓
 정적 파일 (/srv)
```

### 변경 후

```
https://heritage.bun-bull.ts.net:443
     ↓
[heritage] Tailscale 사이드카 (독립)
     ↓
[Caddy] Tailscale 네트워크 + ts-net 공유
     ↓
     ├─→ /nginx → [nginx:80]  (ts-net)
     │
     └─→ /      → 정적 파일 (/srv)
```

## 변경사항

### 1. compose.yaml

**Caddy 서비스:**
- `network_mode: service:heritage` 제거
- `networks: [ts-net]` 추가
- `depends_on: - nginx` 추가

**nginx 서비스 (신규):**
```yaml
nginx:
  image: nginx:1.27-alpine
  container_name: nginx
  networks:
    - ts-net
  restart: unless-stopped
```

**Networks (신규):**
```yaml
networks:
  ts-net:
    driver: bridge
```

### 2. config/Caddyfile

```caddyfile
:9091 {
    handle_path /nginx* {
        reverse_proxy nginx:80
    }

    file_server
}
```

### 3. 파일 구조 변경

```
08-ts-heritage/
├── compose.yaml          # 변경
├── config/
│   ├── Caddyfile         # 변경
│   └── serve-config.json # 유지
├── site/
│   └── index.html        # 유지
├── .env                  # 유지
├── .env.example          # 유지
└── README.md             # 변경: nginx 접속 방법 추가
```

## 경로 라우팅

| URL 경로 | 대상 | 설명 |
|----------|------|------|
| `/` | 정적 파일 (`/srv`) | 기존 index.html |
| `/nginx` | nginx:80 | nginx welcome 페이지 |
| `/nginx/*` | nginx:80 | 경로 제거 후 전달 |

## 네트워크 구성

```
ts-net (bridge)
    ├── Caddy
    └── nginx

heritage (독립)
    └── Tailscale HTTPS 종료
```

## 테스트 계획

### 배포
```bash
podman compose down
# 파일 변경 후
podman compose up -d
podman compose ps
```

### 검증
1. `ts-net` 네트워크 생성 확인
2. Caddy, nginx가 `ts-net`에 연결된 것 확인
3. `curl https://heritage.bun-bull.ts.net/` → 정적 파일
4. `curl https://heritage.bun-bull.ts.net/nginx` → nginx welcome 페이지

## 롤백 계획

```bash
git checkout compose.yaml config/Caddyfile
podman compose up -d --force-recreate
```

## 향후 확장 가능성

이 구조를 사용하면 다음과 같이 추가 서비스를 쉽게 확장할 수 있습니다:

```caddyfile
:9091 {
    handle_path /nginx* { reverse_proxy nginx:80 }
    handle_path /rutorrent* { reverse_proxy rutorrent:8080 }
    handle_path /jellyfin* { reverse_proxy jellyfin:8096 }
    # ...
    file_server
}
```
