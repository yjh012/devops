# 5주차 — Docker와 GitHub Issue

> 강의: **05. 도커 w/ GitHub Issue** (67쪽) · 수업 핵심 개념과 실습 절차를 정리한 학습 노트

## 1. 학습 목표

- 가상 머신(VM)과 컨테이너의 차이를 이해한다.
- **이미지(Image)**와 **컨테이너(Container)**의 관계를 이해한다.
- Docker 주요 명령어(`run`, `ps`, `logs`, `exec`, `stop`, `rm`, `images`)와 옵션을 설명한다.
- nginx 컨테이너를 실행하고 포트 매핑과 접속을 점검하는 방법을 익힌다.
- GitHub의 **Issue → Branch → 실습 기록 → Commit/Push → PR → Merge → Issue 종료** 과정을 이해한다.

## 2. 핵심 개념

### 2.1 Docker가 필요한 이유

동일한 프로그램이라도 컴퓨터마다 Python 버전, 설치 라이브러리, 운영 환경이 다르면 실행이 실패할 수 있다. Docker는 **애플리케이션과 실행에 필요한 파일·라이브러리·설정을 묶어 일관된 실행 환경**을 만들도록 돕는다. 강의에서는 VM과 컨테이너의 차이를 다루며, 리눅스 컨테이너가 호스트의 리눅스 커널을 공유하는 점을 강조한다.

| 구분 | 의미 | 비유 |
|---|---|---|
| 이미지 | 컨테이너를 만들기 위한 읽기 중심의 템플릿 | 붕어빵 틀 |
| 컨테이너 | 이미지를 실행해 만든 격리된 실행 단위 | 붕어빵 |
| `docker run` | 이미지로 컨테이너를 생성·시작 | 틀로 붕어빵 만들기 |

컨테이너 내부의 쓰기 계층에만 저장한 변경 사항은 **컨테이너를 삭제하면 사라진다**. 이후 6주차의 이미지 빌드, 7주차의 볼륨이 이 문제와 연결된다.

### 2.2 명령어 정리

| 명령어 | 설명 |
|---|---|
| `docker --version` | Docker CLI 버전 확인 |
| `docker run hello-world` | 간단한 테스트 이미지 실행 |
| `docker run -d -p 8080:80 --name web nginx` | 이름 `web`으로 nginx를 백그라운드 실행, PC의 8080포트를 컨테이너의 80포트에 연결 |
| `docker ps` / `docker ps -a` | 실행 중 / 종료된 컨테이너까지 조회 |
| `docker images` | 로컬 이미지 확인 |
| `docker logs web` | 컨테이너 로그 확인 |
| `docker exec -it web bash` | 실행 중인 컨테이너 안에서 셸 실행 |
| `docker stop web` | 컨테이너 중지 |
| `docker rm web` | 중지된 컨테이너 삭제 |
| `docker rm -f web` | 실행 중이라도 강제 삭제(주의) |

옵션 `-d`는 백그라운드 실행, `-p 호스트:컨테이너`는 포트 연결, `--name`은 이름 지정, `-e`는 환경 변수 지정, `-it`는 대화형 터미널, `--rm`은 종료 시 컨테이너 자동 삭제를 뜻한다. `-v`는 호스트 경로를 컨테이너에 연결하고 `-w`는 작업 디렉터리를 설정한다.

## 3. 수업 실습 절차와 명령어

> 아래는 **PDF에 제시된 실습을 재현하기 위한 명령어**이다. 실제 실행 결과나 스크린샷은 실행한 후 기록한다.

### 3.1 Docker 실행 환경 점검

```bash
docker --version
docker info
docker run --rm hello-world
```

`hello-world`가 정상 메시지를 출력하면 기본 Docker 실행을 확인할 수 있다.

### 3.2 nginx 웹 서버 실행 및 접속 확인

```bash
docker run -d -p 8080:80 --name web nginx
docker ps
curl -I http://localhost:8080
docker logs web
```

- 브라우저에서 `http://localhost:8080` 접속 시 nginx 기본 페이지를 확인한다.
- `curl -I`는 HTTP 응답 헤더를 확인한다.
- `docker ps`에서 컨테이너 이름, 실행 상태, `8080->80` 포트 매핑을 확인한다.
- **주의:** 다른 프로그램이 8080 포트를 사용하거나 `web`이라는 이름이 이미 있으면 충돌할 수 있다.

### 3.3 여러 컨테이너와 포트 충돌

```bash
docker run -d -p 8081:80 --name web2 nginx
docker run -d -p 8082:80 --name web3 nginx
docker ps
```

서로 다른 컨테이너가 **동일한 호스트 포트**를 동시에 사용할 수 없으므로 각기 다른 포트로 연결한다. 이미지가 같더라도 컨테이너는 별도로 관리된다.

### 3.4 컨테이너 내부 접근·환경 변수·버전 분리

```bash
docker exec -it web bash
# 컨테이너 안에서 exit로 나올 수 있다.

docker run -d -p 8083:80 -e MY_NAME=Kam --name env_test nginx
docker exec env_test env | grep MY_NAME

docker run --rm python:3.9-slim python --version
docker run --rm python:3.12-slim python --version
```

Python 이미지를 바꿔 실행하면 호스트 컴퓨터에 해당 버전의 Python을 별도로 설치하지 않고도 버전을 비교할 수 있다. 강의에는 현재 디렉터리를 마운트하여 동일한 `test.py`를 두 Python 버전에서 실행하는 예시도 있다.

```bash
# test.py가 현재 폴더에 준비되어 있을 때
docker run --rm -v "$PWD:/app" -w /app python:3.9-slim python test.py
docker run --rm -v "$PWD:/app" -w /app python:3.12-slim python test.py
```

## 4. 강의의 '혼자서 해보기' — nginx 3개

강의 63쪽 실습 조건: 이름 `nginx1`, `nginx2`, `nginx3`, 포트 `8091`, `8092`, `8093`로 각각 실행하고 페이지를 서로 다르게 수정한 후 캡처와 `curl` 결과를 README에 기록한다.

```bash
docker run -d -p 8091:80 --name nginx1 nginx
docker run -d -p 8092:80 --name nginx2 nginx
docker run -d -p 8093:80 --name nginx3 nginx

# 강의 목표에 맞춘 페이지 변경 예시(설명용)
docker exec nginx1 sh -c 'echo "<h1>Welcome to nginx1</h1>" > /usr/share/nginx/html/index.html'
docker exec nginx2 sh -c 'echo "<h1>Welcome to nginx2</h1>" > /usr/share/nginx/html/index.html'
docker exec nginx3 sh -c 'echo "<h1>Welcome to nginx3</h1>" > /usr/share/nginx/html/index.html'

curl http://localhost:8091
curl http://localhost:8092
curl http://localhost:8093
docker ps
```

**실행 후 첨부할 증빙:** `week05/images/`에 세 웹페이지 캡처와 `docker ps` 캡처를 저장한다. 캡처가 없는 상태에서 완료라고 주장하지 않는다.

실습 완료 후 이름을 확인하고 불필요한 컨테이너를 정리한다.

```bash
docker stop nginx1 nginx2 nginx3
docker rm nginx1 nginx2 nginx3
# web, web2, web3 등도 필요 없을 때 별도 정리
```

## 5. GitHub Issue 기반 협업 흐름 (강의 55~66쪽)

1. **Issue 생성:** 해야 할 일과 완료 조건을 체크리스트로 작성한다. 예: `5주차 Docker 실행 환경 확인 및 nginx 실행`.
2. **Branch 생성:** 기본 브랜치에 직접 수정하지 않고 해당 작업용 브랜치를 만든다.
3. **실습 및 기록:** 명령 출력과 학습 내용을 `week05/README.md`에 기록한다.
4. **Commit/Push:** 작업 내역을 Git에 저장하고 GitHub로 올린다.
5. **Pull Request:** 작업 브랜치의 변경 사항을 기본 브랜치에 합쳐 달라고 요청한다.
6. **Merge:** 검토 후 반영한다. PR 본문에 `Close #이슈번호`를 넣고 기본 브랜치로 병합하면 연결된 Issue를 자동 종료할 수 있다.

```bash
# 실제 기본 브랜치 이름(main 등)과 이슈 번호 확인 후 사용
git switch main
git pull --ff-only
ISSUE_NO=6  # 강의 예시 번호; 자신의 실제 이슈 번호로 교체
git switch -c "docs/issue-${ISSUE_NO}-docker-start"
# README 등을 수정한 다음
git add week05/
git commit -m "5주차: Docker 실행 환경 및 nginx 확인 (#${ISSUE_NO})"
git push -u origin "docs/issue-${ISSUE_NO}-docker-start"
```

GitHub 웹에서 PR 생성 시 본문에 `Close #6`처럼 **실제 이슈 번호**를 넣는다. Merge 이후 이슈가 닫혔는지 확인한다. 강의 65~66쪽에는 `gh pr create`, `gh pr merge --squash --delete-branch` 방식도 안내되어 있다.

## 6. 오류·주의 사항

- **포트 충돌:** `port is already allocated`가 나오면 사용 중인 컨테이너를 확인하고 다른 포트를 사용한다.
- **이름 충돌:** `web`이 이미 있다면 기존 컨테이너 확인 후 재사용·삭제 또는 다른 이름을 선택한다.
- **데이터 보존:** 컨테이너 내부에서 변경한 HTML은 컨테이너를 삭제하면 사라질 수 있다.
- **명령어 구분:** `docker run`은 새 컨테이너 생성·시작, `docker start`는 기존 중지된 컨테이너 시작이다.

## 7. 학습 정리

Docker 이미지 하나로 독립된 여러 컨테이너를 만들 수 있다. 외부 접속은 포트 매핑으로 연결하며, 컨테이너의 상태·로그·내부 환경을 명령어로 점검한다. DevOps 작업은 실행만 하고 끝내지 않고 **Issue에 목표를 정의하고 README에 근거를 남기며 PR로 합치는 과정**까지 포함한다.

## 8. 실제 실행 증빙 기록란

- Docker 버전: `실행 후 입력`
- `hello-world` 확인: `실행 후 입력`
- nginx 웹페이지 캡처: `week05/images/...` (촬영 후 추가)
- `curl` 응답: `실행 후 기록`
- 관련 Issue / PR 주소: `생성 후 입력`

**출처:** 수업자료 「05. 도커 w/ GitHub Issue」, 특히 실습·옵션(27~54쪽), GitHub Issue(55~66쪽). 별도 표시한 페이지 수정 명령어는 강의 과제 조건에 따른 **재현용 예시**이다.
