# 5주차 — Docker와 GitHub Issue

## 학습 목표
- 이미지와 컨테이너의 차이를 이해한다.
- Docker 명령어로 컨테이너를 생성·확인·정리한다.
- Issue → Branch → PR → Merge로 실습을 기록한다.

## 핵심 개념
- **이미지**: 실행 환경과 파일을 묶은 템플릿.
- **컨테이너**: 이미지로부터 만들어 실행되는 인스턴스.
- **포트 매핑 `-p 8080:80`**: PC의 8080 포트 요청을 컨테이너의 80 포트에 연결.

## 강의 기반 실습 명령어
```bash
docker --version
docker run --rm hello-world
docker run -d -p 8080:80 --name web nginx
curl -I http://localhost:8080
docker ps
docker logs web
docker stop web
docker rm web
```

## 수업 실습에서 알게 된 점
동일한 이미지를 사용해도 포트와 컨테이너 이름을 다르게 지정하면 여러 인스턴스를 동시에 실행할 수 있다. 컨테이너 내부 변경은 삭제 시 사라질 수 있으므로, 영속 데이터는 별도 관리가 필요하다.

## GitHub 협업 기록 방식
1. Issue에 목표와 완료 조건 작성
2. 작업 브랜치 생성
3. 실습 내용과 실행 결과를 README에 기록
4. Commit 및 Push 후 PR 생성
5. 검토 후 Merge하고 Issue 종료 확인

## 실제 수행 증빙 (본인 실습 후 첨부)
- [ ] `docker ps` 실행 화면
- [ ] `curl` 응답 화면
- [ ] GitHub Issue 및 Merge 화면

> 참고: DevOps 05. 도커 w/ GitHub Issue, 강의자료 3쪽, 56~66쪽. 위 명령어는 강의 기반 재현 절차이며 이 파일은 실행 성공을 증명하지 않는다.
