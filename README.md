# 6주차 — Dockerfile과 이미지 빌드·공유

> 강의: **06. 도커파일** (48쪽) · 실습 과정 및 핵심 명령어 정리

## 1. 학습 목표

- Dockerfile의 주요 명령어 `FROM`, `WORKDIR`, `COPY`, `RUN`, `EXPOSE`, `CMD`, `ENV`의 기능을 이해한다.
- Dockerfile을 이용해 나만의 이미지를 만들고 실행한다.
- 이미지 레이어와 빌드 캐시가 Dockerfile 작성 순서에 따라 달라지는 이유를 설명한다.
- 이미지 태그·배포·멀티 아키텍처 빌드(`buildx`) 개념을 이해한다.

## 2. Dockerfile이 필요한 이유

5주차에는 이미 만들어진 이미지를 `docker run`으로 실행했다. 하지만 컨테이너 내부에서 바꾼 내용은 컨테이너 삭제 시 잃을 수 있다. 6주차에서는 **애플리케이션 코드 + Dockerfile → `docker build` → 사용자 이미지** 흐름으로 자신의 프로그램을 재현 가능하게 포장한다.

- **Dockerfile:** 이미지를 만드는 레시피
- **`docker build`:** 레시피에 따라 이미지를 만드는 작업
- **이미지:** 실행에 필요한 프로그램·라이브러리·설정이 포함된 산출물
- **`docker run`:** 생성된 이미지로 컨테이너 실행

## 3. Dockerfile 주요 명령어 7개

| 명령어 | 역할 | 예시 |
|---|---|---|
| `FROM` | 기반 이미지 선택 | `FROM python:3.12-slim` |
| `WORKDIR` | 이후 명령의 작업 디렉터리 지정 | `WORKDIR /app` |
| `COPY` | 빌드 컨텍스트의 파일을 이미지에 복사 | `COPY requirements.txt .` |
| `RUN` | **이미지 빌드 시** 명령 실행 | `RUN pip install -r requirements.txt` |
| `EXPOSE` | 컨테이너가 사용하는 포트 문서화 | `EXPOSE 5000` |
| `CMD` | 컨테이너 실행 시 기본 명령 지정 | `CMD ["python", "app.py"]` |
| `ENV` | 이미지의 기본 환경 변수 설정 | `ENV APP_TITLE="DevOps 방명록"` |

`EXPOSE`만으로 PC 포트가 외부에 공개되지는 않는다. 실제 호스트 포트 연결은 `docker run -p 8081:5000`과 같이 한다.

## 4. 방명록 앱 실습 (강의 19~27쪽)

### 4.1 실습 폴더 구성

강의에서 제공되는 `guestbook.zip`을 `~/devops/guestbook`에 풀어 사용한다. **이 파일이 없으면 아래 빌드 실습을 그대로 실행할 수 없다.**

```text
guestbook/
├── app.py
├── requirements.txt
├── templates/index.html
├── Dockerfile
├── Dockerfile.multistage
├── .dockerignore
└── README.md
```

강의에서는 주차별로 `guestbook` 폴더를 복제하지 않고 **하나의 폴더를 계속 수정하고 Git으로 변경 이력을 기록**하라고 설명한다.

### 4.2 기존 배포 이미지 실행

```bash
docker run -d -p 8080:5000 --name demo ghcr.io/makyraen/inhatc-devops-guestbook:v1
# 브라우저 http://localhost:8080 확인
# 실습 후
docker rm -f demo
```

태그를 생략하면 `latest`를 시도하기 때문에 저장소에 `latest`가 없으면 이미지 다운로드가 실패할 수 있다. 따라서 강의처럼 **`:v1`을 명시**한다.

### 4.3 Dockerfile 읽고 이미지 만들기

```bash
cd ~/devops/guestbook
cat Dockerfile
docker build -t guestbook:v1 .
docker images
docker run -d -p 8081:5000 --name gb guestbook:v1
# 브라우저 http://localhost:8081 확인
```

- `-t guestbook:v1`: 이미지 이름과 태그 지정
- 마지막 `.`: **현재 디렉터리를 빌드 컨텍스트로 사용**
- `8081:5000`: 호스트 8081 포트와 컨테이너 5000 포트 연결

**예제 Dockerfile의 기본 흐름(학습용 구성 예시, 제공 ZIP의 실제 Dockerfile과 동일하다고 주장하지 않음):**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

이 예시는 `requirements.txt`, `app.py`가 있는 Flask 앱을 가정한다. 실제 강의 제공 파일에는 추가 사용자 설정과 환경변수가 있을 수 있으므로 **받은 Dockerfile을 기준으로 비교**해야 한다.

## 5. 환경 변수 우선순위 (강의 22쪽)

강의 방명록 앱은 제목(`APP_TITLE`)과 색상(`THEME_COLOR`)을 환경 변수로 제어한다.

```bash
# 앱 내부 기본값보다 Dockerfile의 ENV가 우선하며,
# 실제 컨테이너 실행 시 docker run -e로 덮어쓸 수 있다.
docker run -d -p 8081:5000 --name gbv2 \
  -e APP_TITLE="나의 DevOps 방명록" \
  -e THEME_COLOR="#16a34a" guestbook:v1
```

**설정 우선순위:** `app.py 기본값 < Dockerfile ENV < docker run -e`.

※ 위 명령은 `gbv2` 이름 및 포트 8081을 사용하므로 기존 `gb`가 같은 포트를 사용 중이라면 먼저 중지하거나 포트를 변경한다.

## 6. 레이어와 캐시 (강의 7~10쪽, 24~27쪽)

Dockerfile의 명령들은 이미지를 만드는 단계가 된다. 변경되지 않은 단계는 빌드 캐시를 재활용할 수 있다.

```dockerfile
# 권장되는 배치의 원리
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

`app.py`만 수정했을 때 `requirements.txt`가 그대로라면 의존성 설치 단계를 재사용할 가능성이 높다. 반대로 **자주 바뀌는 소스 전체를 먼저 COPY**하면 이후 캐시 재사용이 줄어들 수 있다.

```bash
docker build -t guestbook:v1 .
docker build -t guestbook:v1.1 .
# 두 번째 빌드의 출력에서 CACHED 여부 확인
```

강의에서는 나쁜 순서의 `Dockerfile.bad`와 비교하여 빌드 효율 차이를 관찰한다. 캐시는 환경과 변경된 파일에 따라 결과가 달라질 수 있다.

## 7. 이미지 배포 및 멀티 아키텍처 (강의 28~39쪽)

이미지에 태그를 붙이고 레지스트리(GitHub Container Registry: `ghcr.io`, Docker Hub)에 올리면 다른 사람이 내려받아 실행할 수 있다.

```bash
# 강의의 GHCR 이미지 태그 형식: 아래 '내아이디'는 실제 계정으로 교체
docker build -t ghcr.io/내아이디/guestbook:v1 .
docker push ghcr.io/내아이디/guestbook:v1

# 강의의 멀티 아키텍처 빌드 형식
docker buildx build --platform linux/amd64,linux/arm64 \
  -t ghcr.io/내아이디/guestbook:v1 --push .
```

- `linux/amd64`와 `linux/arm64`: 서로 다른 CPU 아키텍처
- `buildx`: 여러 플랫폼용 이미지 빌드 지원
- 실제 `push`는 레지스트리 인증과 접근 권한이 필요하다. 배포하지 않았다면 배포 성공으로 기록하지 않는다.

## 8. 보안과 실행 시 주의사항

- `USER appuser`처럼 **root가 아닌 사용자**로 실행하는 구성이 더 안전하다(강의 21쪽).
- `.dockerignore`로 불필요한 파일이나 민감한 파일이 이미지에 포함되지 않도록 관리한다.
- 로컬 환경에 필요한 Python 라이브러리가 없어 `ModuleNotFoundError`가 나더라도 Dockerfile을 통한 환경 재현으로 해결할 수 있다(강의 20쪽).
- `guestbook:v1`이라는 같은 이름의 컨테이너를 여러 개 띄울 때는 호스트 포트와 컨테이너 이름이 중복되지 않아야 한다.

## 9. 학습 정리

Dockerfile은 **이미지를 코드로 재현하는 방법**이다. 명령어의 실행 시점을 구분하고(`RUN`: 빌드, `CMD`: 시작), 의존성 파일부터 복사해 캐시를 활용하며, 환경 변수로 이미지 재빌드 없이 실행 설정을 바꿀 수 있다. 태그와 레지스트리를 활용하면 동일한 이미지를 다른 환경에서도 실행할 수 있다.

## 10. 실제 실행 증빙 기록란

- 받은 `guestbook.zip`의 폴더 확인: `실행 후 기록`
- `docker build -t guestbook:v1 .` 결과: `실행 후 기록`
- `http://localhost:8081` 확인 캡처: `실행 후 첨부`
- 캐시 빌드 출력의 `CACHED` 확인: `실행 후 기록`
- GHCR/도커허브 이미지 주소(실제로 배포한 경우): `배포 후 입력`

**출처:** 수업자료 「06. 도커파일」(48쪽), Dockerfile 명령어·캐시·이미지 실습·이미지 공유·혼자 해보기. Dockerfile 전체 예시는 **학습용 재구성**이며 강의 ZIP 파일의 원문과 동일하다고 표시하지 않았다.
