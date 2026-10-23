# 7주차 — Docker Compose, Redis, Volume, Scale

> 강의: **07. 도커 컴포즈** (62쪽) · 여러 컨테이너의 연결과 운영 실습 정리

## 1. 학습 목표

- 여러 컨테이너를 하나의 `compose.yaml`로 정의하고 실행하는 방법을 이해한다.
- 웹 앱과 Redis가 **서비스 이름**으로 통신하는 원리를 설명한다.
- 볼륨이 데이터 유지에 필요한 이유와 `down -v`의 위험성을 구분한다.
- `.env`, healthcheck, nginx 프록시, `--scale web=3`의 역할을 이해한다.

## 2. Docker Compose 개념

5주차의 `docker run`은 개별 컨테이너 실행에 사용했다. 7주차에는 웹 앱과 Redis처럼 역할이 다른 여러 컨테이너를 **`compose.yaml` 파일로 함께 정의**한다.

```bash
# 강의 실습의 guestbook 폴더에서 실행
cd ~/devops/guestbook
docker compose config
docker compose up -d
docker compose ps
docker compose logs web
```

| 명령어 | 의미 |
|---|---|
| `docker compose config` | compose 파일을 해석해 설정 확인 |
| `docker compose up -d` | 필요한 네트워크·볼륨·컨테이너 생성 및 백그라운드 시작 |
| `docker compose ps` | 서비스 컨테이너 상태 확인 |
| `docker compose logs web` | web 서비스 로그 확인 |
| `docker compose exec web sh` | 실행 중인 web 서비스 컨테이너 내부 접근 |
| `docker compose down` | Compose 컨테이너와 기본 네트워크 제거(볼륨은 기본적으로 유지) |
| `docker compose down -v` | 관련 볼륨까지 제거하므로 데이터 삭제 위험 |

## 3. 방명록 + Redis: 컨테이너 협력 (강의 5~16쪽)

방명록 앱이 글을 프로세스 메모리에만 저장하면 앱 재시작 시 글이 사라질 수 있다. 강의는 웹 앱과 Redis를 분리하여 **웹 앱이 재시작돼도 데이터가 유지되도록 하는 구조**를 소개한다.

```text
브라우저 → web(방명록 앱 :5000) → redis(데이터 저장소 :6379)
```

같은 Compose 네트워크에서는 서비스 이름 `redis`로 접속할 수 있다. 웹 앱의 연결 설정에서 `localhost`는 **자기 컨테이너**이므로 다른 컨테이너의 Redis 주소가 아니다. Redis의 포트를 호스트에 게시하지 않아도 같은 내부 네트워크 서비스에서 접근할 수 있다.

## 4. 볼륨과 데이터 유지 (강의 17~25쪽)

- 컨테이너가 교체되더라도 Redis 데이터가 필요하면 `redis-data` 같은 **named volume**에 저장한다.
- 볼륨을 사용하고 Redis 영속화 설정도 올바르면 앱/컨테이너 재생성 후 데이터를 복구할 수 있다.
- `docker compose down`과 `docker compose down -v`는 다르다. **`-v`는 볼륨을 제거하므로 중요한 데이터를 잃을 수 있다.**

```bash
docker compose down
docker compose up -d
# 기존 글이 남는지 확인: 수업에서 제시한 검증 포인트
```

볼륨만 있다고 무조건 모든 데이터가 즉시 안전하게 보존되는 것은 아니다. 저장소의 실제 영속화 설정도 중요하다.

## 5. 환경 변수와 비밀번호 관리 (강의 34~37쪽)

강의에서는 Redis 비밀번호를 `compose.yaml`에 직접 적는 문제를 지적하고, `.env`로 값을 분리한다.

```dotenv
# .env.example — 값이 아닌 양식만 공개
REDIS_PASSWORD=비밀번호
```

```yaml
# compose.yaml 내 설정 예시(강의에서 설명한 해당 부분만 발췌)
environment:
  REDIS_PASSWORD: "${REDIS_PASSWORD:?비밀번호 설정 필요}"
```

- `compose.yaml`, `.env.example`은 저장소에 올릴 수 있다.
- 실제 비밀번호가 담긴 **`.env`는 `.gitignore`에 추가**한다.
- `.dockerignore`에도 `.env`를 제외하여 빌드 파일에 복사되지 않도록 한다.
- `docker compose config` 출력에는 대체된 비밀번호가 보일 수 있으므로 공유할 때 주의한다.
- `.env`는 비밀번호를 암호화하는 방법이 아니라 **Git 추적에서 분리하는 방법**이다.

## 6. healthcheck와 depends_on (강의 38~41쪽)

Redis 컨테이너가 단순히 실행 중이라는 사실과 **요청을 처리할 준비가 끝났다는 사실**은 다르다. 따라서 강의는 다음과 같은 조건을 설명한다.

```yaml
# 강의의 핵심 설정을 간추린 예시
web:
  depends_on:
    redis:
      condition: service_healthy
redis:
  healthcheck:
    test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
    interval: 5s
    timeout: 3s
    retries: 5
```

웹 앱의 healthcheck는 수업에서 `/readyz` 경로를 Python으로 요청하도록 구성했다. `healthcheck`는 상태 점검과 초기 실행 순서에 도움을 주지만 **장애 난 컨테이너를 자동으로 재생성한다는 뜻은 아니다**.

## 7. web 컨테이너 3개와 nginx (강의 42~48쪽)

```bash
docker compose up -d --scale web=3
```

- `--scale web=3`: web 서비스 컨테이너 3개 실행 목표
- web 3개가 모두 호스트 `8080`에 직접 연결되려고 하면 **포트 충돌**이 발생한다.
- 따라서 web에서는 호스트 포트 공개를 빼고, 앞단의 **nginx**가 호스트 포트 8080으로 요청을 받아 웹 컨테이너들에 분산하도록 한다.

```text
브라우저 (localhost:8080)
        ↓
nginx (Reverse Proxy / Load Balancer)
     ↙    ↓    ↘
  web-1  web-2  web-3 (각각 내부 :5000)
        ↓
      Redis
```

강의에서 사용하는 nginx 설정(45쪽):

```nginx
server {
    listen 80;
    resolver 127.0.0.11 valid=5s;
    location / {
        set $backend http://web:5000;
        proxy_pass $backend;
    }
}
```

여기서 `web`은 Compose의 서비스 이름이고 `127.0.0.11`은 Docker 내부 DNS 주소이다. 강의의 설명처럼 동적으로 이름을 다시 조회하도록 변수 기반 `proxy_pass`를 사용한다.

```bash
docker compose config
docker compose up -d --scale web=3
docker compose ps
curl http://localhost:8080
```

## 8. '3개 실행'과 '3개 유지'의 차이 (강의 48쪽)

```bash
# 수업에서 제시된 상황의 예시
docker stop guestbook-web-2
docker compose ps
```

강의에서는 `docker stop`으로 수동 종료한 컨테이너는 `restart: unless-stopped`로도 바로 재시작되지 않는다는 점을 보여준다. 즉, **`web=3`으로 한번 실행하는 것과 항상 3개를 보장하는 것은 다른 문제**다.

강의의 즉각적인 수동 복구 방안:

```bash
docker compose up -d --scale web=3
```

**추가 학습 아이디어(강의 내용과 구분):** 정기 점검 스크립트로 실행 수를 확인하고 부족하면 위 명령을 실행하는 방법, 혹은 원하는 복제 수를 지속적으로 조정하는 오케스트레이터를 사용하는 방법을 고려할 수 있다. 자동 복구 방법의 적합성과 한계는 별도 4번 과제에서 논의한다.

## 9. 트러블슈팅

| 현상 | 확인 사항 |
|---|---|
| `port is already allocated` | web 복제본에서 같은 호스트 포트 8080을 공개하지 않았는지 확인 |
| Redis가 `unhealthy` | `.env` 비밀번호와 Redis healthcheck 비밀번호가 같은지 확인 |
| web이 `unhealthy` | `docker compose logs web`에서 Redis 연결 오류 확인 |
| `docker compose config` 실패 | `.env` 파일 유무, `${REDIS_PASSWORD:?…}` 확인 |
| 앱 재시작 후 글이 사라짐 | Redis 연결 여부, 웹 앱의 Memory 모드 여부, 영속화 설정 확인 |
| 서비스가 3개였는데 2개만 동작 | 수동 종료 여부 확인, `docker compose up -d --scale web=3`로 재조정 |

## 10. 학습 정리

Compose는 여러 컨테이너의 구성을 코드로 관리한다. 서비스 이름을 사용하면 컨테이너 간 통신을 정의할 수 있고, Redis와 볼륨을 이용해 앱과 데이터를 분리할 수 있다. healthcheck는 서비스 준비 상태를 확인하며, nginx와 scale은 요청 분산을 구현한다. **컨테이너 3개를 기동하는 설정만으로 자동 복구가 보장되지는 않는다**는 한계를 이해하는 것이 중요하다.

## 11. 실습 증빙 기록란 (실제로 확인 후 작성)

- `docker compose ps` 결과(healthy 포함): `실행 후 기록`
- 브라우저 방명록 화면: `실행 후 캡처`
- Redis 재시작 후 글 보존 확인: `실행 후 기록`
- nginx 프록시 접속 결과: `실행 후 기록`
- `--scale web=3` 실행 및 `docker compose ps` 캡처: `실행 후 첨부`
- `.env`가 Git 추적 대상에서 제외되었는지: `실행 후 확인`

**출처:** 수업자료 「07. 도커 컴포즈」(62쪽), 특히 볼륨, 환경 변수, healthcheck, nginx, scale 및 실습 검증 내용. 일부 구성 예시는 핵심 구조만 발췌했으며 **완성된 실행용 `compose.yaml`이 아니다**.
