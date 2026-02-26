# 다중 서비스 지원 확장 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 08-ts-heritage 예제를 단일 Caddy 서비스에서 다중 서비스를 지원하는 구조로 확장

**Architecture:** Caddy의 `network_mode: service:heritage`를 제거하고 공유 `ts-net` 브리지 네트워크를 도입합니다. nginx 서비스를 추가하여 `/nginx` 경로로 접근 가능하게 하고, 기존 정적 파일 제공은 유지합니다.

**Tech Stack:** Podman, Docker Compose, Caddy, nginx, Tailscale

---

## Task 1: ts-net 네트워크 정의 추가

**Files:**
- Modify: `compose.yaml` (파일 끝에 networks 섹션 추가)

**Step 1: compose.yaml에 networks 섹션 추가**

```yaml
networks:
  ts-net:
    driver: bridge
```

**Step 2: 문법 검증**

```bash
podman compose config
```

Expected: 에러 없이 설정 출력

**Step 3: Commit**

```bash
git add compose.yaml
git commit -m "feat: ts-net 공유 네트워크 정의 추가"
```

---

## Task 2: Caddy 서비스 네트워크 설정 변경

**Files:**
- Modify: `compose.yaml` (caddy 서비스)

**Step 1: network_mode 제거, networks 추가**

기존:
```yaml
caddy:
  image: caddy:2.11.1
  container_name: caddy
  network_mode: service:heritage
  depends_on:
    - heritage
```

변경:
```yaml
caddy:
  image: caddy:2.11.1
  container_name: caddy
  networks:
    - ts-net
  depends_on:
    - heritage
```

**Step 2: 문법 검증**

```bash
podman compose config
```

Expected: 에러 없이 설정 출력, caddy.networks에 ts-net 포함

**Step 3: Commit**

```bash
git add compose.yaml
git commit -m "refactor: caddy 네트워크 설정을 network_mode에서 공유 네트워크로 변경"
```

---

## Task 3: nginx 서비스 추가

**Files:**
- Modify: `compose.yaml` (caddy 서비스 다음에 nginx 서비스 추가)

**Step 1: nginx 서비스 정의 추가**

```yaml
  nginx:
    image: nginx:1.27-alpine
    container_name: nginx
    networks:
      - ts-net
    restart: unless-stopped
```

**Step 2: 문법 검증**

```bash
podman compose config
```

Expected: 에러 없이 설정 출력, nginx 서비스 포함

**Step 3: Commit**

```bash
git add compose.yaml
git commit -m "feat: nginx 서비스 추가"
```

---

## Task 4: Caddy의 depends_on에 nginx 추가

**Files:**
- Modify: `compose.yaml` (caddy 서비스의 depends_on)

**Step 1: depends_on에 nginx 추가**

변경 전:
```yaml
  depends_on:
    - heritage
```

변경 후:
```yaml
  depends_on:
    - heritage
    - nginx
```

**Step 2: 문법 검증**

```bash
podman compose config
```

Expected: 에러 없이 설정 출력

**Step 3: Commit**

```bash
git add compose.yaml
git commit -m "deps: caddy가 nginx에 의존하도록 설정"
```

---

## Task 5: Caddyfile에 nginx reverse_proxy 설정 추가

**Files:**
- Modify: `config/Caddyfile`

**Step 1: handle_path 블록 추가**

변경 전:
```caddyfile
:9091 {
    file_server
}
```

변경 후:
```caddyfile
:9091 {
    handle_path /nginx* {
        reverse_proxy nginx:80
    }

    file_server
}
```

**Step 2: Caddyfile 문법 검증**

```bash
podman run --rm -v $(pwd)/config/Caddyfile:/etc/caddy/Caddyfile caddy:2.11.1 caddy validate --config /etc/caddy/Caddyfile
```

Expected: `Valid configuration`

**Step 3: Commit**

```bash
git add config/Caddyfile
git commit -m "feat: /nginx 경로에 대한 reverse_proxy 설정 추가"
```

---

## Task 6: README에 nginx 접속 방법 추가

**Files:**
- Modify: `README.md`

**Step 1: README에 nginx 접속 정보 추가**

"다중 서비스 확장 예시" 섹션 다음에 추가:

```
## 테스트된 서비스

### nginx
- URL: `https://heritage.bun-bull.ts.net/nginx`
- 설명: nginx welcome 페이지 (경로 기반 라우팅 테스트용)
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: nginx 접속 방법 추가"
```

---

## Task 7: 통합 테스트 및 검증

**Files:**
- Test: 배포 및 접속 테스트

**Step 1: 기존 컨테이너 정지**

```bash
podman compose down
```

Expected: 컨테이너 정지 및 제거

**Step 2: 새 설정으로 배포**

```bash
podman compose up -d
```

Expected: heritage, caddy, nginx 컨테이너 시작

**Step 3: 컨테이너 상태 확인**

```bash
podman compose ps
```

Expected: 3개 컨테이너 모두 Up 상태

**Step 4: 네트워크 확인**

```bash
podman network ls | grep ts-net
```

Expected: `ts-net` 브리지 네트워크 존재

**Step 5: Caddy 네트워크 확인**

```bash
podman inspect caddy | grep -A5 "Networks"
```

Expected: `ts-net` 네트워크에 연결됨

**Step 6: 기존 정적 파일 접근 테스트**

```bash
curl -s https://heritage.bun-bull.ts.net/ | head -20
```

Expected: 기존 index.html 내용 (HTML 태그 포함)

**Step 7: nginx 접근 테스트**

```bash
curl -s https://heritage.bun-bull.ts.net/nginx | head -20
```

Expected: nginx welcome 페이지 (HTML에 "nginx" 포함)

**Step 8: 로그 확인**

```bash
podman compose logs caddy | tail -20
podman compose logs nginx | tail -10
```

Expected: 에러 없음

**Step 9: Commit (성공 시)**

```bash
git add .
git commit -m "test: 다중 서비스 통합 테스트 완료"
```

---

## Task 8: 롤백 테스트 (선택 사항)

**Files:**
- Test: 롤백 절차 검증

**Step 1: 롤백**

```bash
podman compose down
git checkout HEAD~1 compose.yaml config/Caddyfile
podman compose up -d --force-recreate
```

Expected: 이전 단일 서비스 구조로 복원

**Step 2: 정상 동작 확인**

```bash
curl -s https://heritage.bun-bull.ts.net/ | head -20
```

Expected: 정적 파일 정상 제공

**Step 3: 재배포**

```bash
podman compose down
git checkout main compose.yaml config/Caddyfile
podman compose up -d
```

Expected: 다중 서비스 구조로 복원

---

## 완료 체크리스트

- [ ] ts-net 네트워크 생성됨
- [ ] Caddy가 ts-net에 연결됨
- [ ] nginx가 ts-net에 연결됨
- [ ] `/` 경로로 정적 파일 접근 가능
- [ ] `/nginx` 경로로 nginx 접근 가능
- [ ] Caddy 로그에 에러 없음
- [ ] nginx 로그에 에러 없음
- [ ] README 업데이트됨

## 롤백 절차

문제 발생 시:

```bash
podman compose down
git checkout HEAD~1 compose.yaml config/Caddyfile README.md
podman compose up -d --force-recreate
```
