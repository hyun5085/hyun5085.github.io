---
layout: post
title: Docker & CI/CD 핵심 개념 및 활용 가이드
description: >
   GitHub Actions를 이용한 CI/CD부터 Docker 기초, 이미지·컨테이너 관리, Dockerfile 작성, Compose, 볼륨, 네트워크, AWS 배포까지 실전 예제와 함께 단계별로 상세 정리합니다.
image: /assets/img/blog/Docker_CICD.png
sitemap: false
---


# Docker & CI/CD 핵심 개념 및 활용 가이드

현대 소프트웨어 개발에서 **CI/CD**와 **컨테이너 기반 배포**는 생산성과 안정성을 동시에 높여 줍니다. 이 가이드에서는 GitHub Actions를 활용한 CI/CD 파이프라인 구성과 Docker의 기본 개념부터 고급 활용까지 단계별로 자세히 다룹니다.

---

## 1. CI/CD with GitHub Actions

### 1.1 CI/CD 개요

* **Continuous Integration (CI)**: 코드 통합 시 자동으로 빌드·테스트를 실행해 문제 조기 발견
* **Continuous Deployment (CD)**: 통합된 코드를 자동으로 운영 환경에 배포하여 릴리즈 주기 단축
* **Benefits**: 빠른 피드백, 일관된 배포, 인적 오류 최소화

### 1.2 Workflow 구성 요소

* **Workflow**: GitHub 레포지토리의 `.github/workflows/*.yaml` 파일로 정의
* **Event**: 워크플로우를 트리거하는 GitHub 이벤트 (`push`, `pull_request`, `workflow_dispatch`, `schedule` 등)
* **Job**: 병렬 또는 순차로 실행되는 단위 작업, 독립적 실행 환경 정의(`runs-on`)
* **Step**: Job 내부에서 순서대로 실행, `uses`(Action 재사용) 또는 `run`(직접 명령 실행)

### 1.3 예시 파이프라인

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [ main, develop, 'feature/*' ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'adopt'
          java-version: '17'

      - name: Build and test
        run: ./gradlew clean test --info

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Copy artifact to server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          source: './build/libs/*.jar'
          target: '~/deploy/'

      - name: Restart service
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            docker-compose -f ~/deploy/docker-compose.yml down
            docker-compose -f ~/deploy/docker-compose.yml up -d --build
```

---

## 2. Docker 기본 이해

Docker는 리눅스 컨테이너 기술을 바탕으로 애플리케이션 배포 환경을 표준화합니다.

### 2.1 컨테이너 vs 가상 머신

| 구분       | 컨테이너(Container) | 가상 머신(VM) |
| -------- | --------------- | --------- |
| 격리 수준    | OS 레벨 네임스페이스    | 하드웨어 가상화  |
| 부팅 시간    | 빠름 (초 단위)       | 느림 (분 단위) |
| 리소스 오버헤드 | 낮음              | 높음        |

### 2.2 이미지와 레이어

* **이미지(Image)**: 컨테이너 실행에 필요한 불변 파일 시스템 스냅샷
* **레이어(Layer)**: Dockerfile 각 명령어(`RUN`, `COPY` 등)가 추가하는 증분 데이터
* **캐시 활용**: 레이어 캐시 덕분에 반복 빌드가 빨라짐

```bash
docker pull nginx:alpine  # 도커 허브에서 이미지 가져오기
docker images            # 로컬에 저장된 이미지 목록
```

---

## 3. Dockerfile: 커스텀 이미지 작성

Dockerfile은 **명령어의 순차적 나열**로 이미지 빌드 과정을 정의합니다.

### 3.1 주요 명령어

* `FROM <image>`: 베이스 이미지 지정
* `LABEL key=value`: 메타데이터 추가
* `WORKDIR <path>`: 작업 디렉터리 설정
* `COPY/ADD <src> <dest>`: 파일 복사 (ADD는 URL/압축 해제 지원)
* `RUN <command>`: 이미지 빌드 시 실행 명령
* `ENV <key>=<value>`: 환경 변수
* `EXPOSE <port>`: 문서화 목적 포트 노출
* `ENTRYPOINT` / `CMD`: 컨테이너 실행 시 기본 명령

### 3.2 멀티 스테이지 빌드

* **목적**: 빌드 도구를 최종 이미지에서 제거해 용량 최적화

```dockerfile
# Build stage
FROM gradle:7.5-jdk17 AS builder
WORKDIR /home/gradle/project
COPY . .
RUN gradle build --no-daemon

# Runtime stage
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY --from=builder /home/gradle/project/build/libs/app.jar ./app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build -t myapp:latest .
```

---

## 4. Docker Compose: 다중 컨테이너 정의 및 실행

Compose는 **YAML 파일** 하나로 멀티 컨테이너 애플리케이션을 관리합니다.

### 4.1 기본 구성

* `version`: Compose 파일 버전
* `services`: 컨테이너 정의 블록
* `volumes`, `networks`: 공통 자원 정의

```yaml
version: '3.9'
services:
  web:
    build: .
    ports:
      - "80:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
    depends_on:
      - db

  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - dbdata:/var/lib/mysql

volumes:
  dbdata:
networks:
  default:
    driver: bridge
```

```bash
docker-compose up -d    # 백그라운드 실행
docker-compose ps      # 서비스 상태 확인
docker-compose logs -f # 실시간 로그
```

---

## 5. 데이터 관리 및 볼륨

### 5.1 볼륨(Volume)

* Docker가 관리하는 데이터 저장소
* 컨테이너 제거 후에도 데이터 유지

```bash
docker volume create appdata
docker run -d -v appdata:/data myapp
```

### 5.2 바인드 마운트(Bind Mount)

* 호스트 디렉토리를 컨테이너에 직접 연결
* 코드 개발 시 실시간 변경 반영에 유리

```bash
docker run -d -v "$(pwd)/config:/app/config" myapp
```

---

## 6. 네트워킹: 서비스 간 통신

* **bridge**: 기본 Docker 네트워크 드라이버
* **host**: 호스트 네트워크 사용
* **overlay**: Swarm 모드에서 클러스터 네트워크 구현

```bash
docker network create front-tier
docker run --network front-tier --name api myapp-api
docker run --network front-tier --name web myapp-web
```

---

## 7. 고급 활용: 이미지 최적화 & 레지스트리

* **이미지 경량화**:

   * 멀티 스테이지 빌드
   * 불필요한 패키지 제거
   * `docker image prune`
* **레지스트리 사용**:

   * Docker Hub, AWS ECR, GitHub Container Registry
   * `docker login` → `docker push` → `docker pull`

```bash
docker tag myapp:latest myrepo/myapp:v1.0
docker push myrepo/myapp:v1.0
```

---

## 8. 배포: AWS EC2 + Docker Compose

1. **EC2 준비**: Ubuntu 인스턴스 생성, Docker & Docker Compose 설치
2. **CI/CD 연동**: GitHub Actions에서 SSH로 EC2 접속
3. **자동 배포 스크립트**:

   ```bash
   ssh ec2-user@host << 'EOF'
     cd ~/deploy
     git pull origin main
     docker-compose down
     docker-compose up -d --build
   EOF
   ```
4. **모니터링**: `docker stats`, CloudWatch, ELK 스택 등

---

## 마무리 및 추가 팁

* **테스트 환경**: GitHub Actions에서 `docker-compose`로 테스트 컨테이너 실행
* **보안**: 이미지 스캔(Trivy), 최소 권한 원칙 준수
* **로깅 & 모니터링**: Prometheus, Grafana, ELK 스택 연동

> 이 가이드를 통해 Docker와 CI/CD 환경을 체계적으로 이해하고, 실제 프로젝트에 적용해 보세요!

