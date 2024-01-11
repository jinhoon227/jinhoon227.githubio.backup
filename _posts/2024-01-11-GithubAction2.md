---
layout: post
title:  "[CICD] Spring boot 환경 Github Action 에서 Docker, Docker-compose, EC2, RDS 적용기"
categories: SPRING CICD
tags : java spring jpa cicd
---

## 소개
이전글에서는 테스트환경을 자동화하는 과정을 소개했었다. 이번글에서는 자동으로 클라우드 서버에 배포하는 과정을 다뤄본다.

[SpringBoot[CICD] Spring boot 환경 Github Action 에서 Test 적용기](../GithubAction1)  

## CD Architecture

<img src="../../assets/img/posts/ci/ga2-1.png">

개발(DEVELOP)환경에서 구축할 CD 아키텍처는 위와 같다.
개발자가 Github 에 Develop 브랜치에서 Push 를 할경우에 Github Action 이 작동한다. Github Action 에서는 Docker 를 이용해 이미지를 만들고 Docker Hub 에 이미지를 푸시한다. EC2 에서는 Docker Hub 에서 이미지를 가져온다(Pull). EC2 에 있는 docker-compose 파일을 실행해 가져온 이미지를 컨테이너화 한다. 이렇게 Spring boot 를 EC2 에서 실행시키고, Spring boot 의 DB 는 AWS 의 RDS-MySQL 를 사용한다.

## Github Action

자동으로 배포되기 위해서 Github Action 을 사용했다. 아래 파일을 .github/workflows 에 넣어주면 된다. 전체적인 코드를 소개하고 디테일하게 하나씩 다뤄볼것이다. 이전에 작성한글과 중복되는 부분은 넘어가도록 하겠다.

### action-develop-cd.yml

```yaml
name: action-develop-cd

# 언제 이 파일의 내용이 실행될 것인지 정의
on:
  push:
    branches:
      - develop

# 코드의 내용을 이 파일을 실행하여 action을 수행하는 주체(Github Actions에서 사용하는 VM)가 읽을 수 있도록 권한을 설정
permissions:
  contents: read

# 실제 실행될 내용들을 정의합니다.
jobs:
  build:
    runs-on: ubuntu-latest # ubuntu 최신 버전에서 script를 실행
    steps:
      # 지정한 저장소(현재 REPO)에서 코드를 워크플로우 환경으로 가져오도록 하는 github action
      # submodule 을 사용하기 위한 설정을 추가
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          token: ${{secrets.ACTION_TOKEN}}
          submodules: true

      # open jdk 17 버전 환경을 세팅
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: "corretto"

      # 캐시를 사용하기위해 buildx 를 사용
      - name: Setup docker buildx
        uses: docker/setup-buildx-action@v2

      # Repository secrets 에 등록해둔 환경변수 파일 생성
      - name: Copy secrets to application
        env:
          OCCUPY_ENV: ${{ secrets.OCCUPY_ENV }}
          OCCUPY_SECRET_DIR: ./src/main/resources  # 레포지토리 내 빈 env.yml의 위치 (main)
          OCCUPY_SECRET_DIR_FILE_NAME: env.yml                 # 파일 이름

        # 환경변수 값 복사
        run: |
          echo $OCCUPY_ENV >> $OCCUPY_SECRET_DIR/$OCCUPY_SECRET_DIR_FILE_NAME

      # gradle을 통해 소스를 빌드.
      - name: Build with gradle
        run: |
          chmod +x ./gradlew
          ./gradlew clean build -x test

      # 도커 컴포즈 설정 파일 서버(EC2)로 전달
      - name: Send docker-compose.yml
        uses: appleboy/scp-action@master
        with:
          username: ubuntu
          host: ${{ secrets.KCS_HOST_DEV }}
          key: ${{ secrets.KCS_KEY_DEV }}
          source: "src/main/resources/backend-submodule/docker-compose-dev.yml"
          target: "/home/ubuntu/"

      # Docker hub 로그인
      - name: Login to dockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME}}
          password: ${{ secrets.DOCKER_TOKEN}}

      # Docker Hub 에 푸시
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile.dev
          push: true
          tags: ${{ secrets.DOCKER_REPOSITORY }}:latest
          cache-from: type=gha
          cache-to: type=gha, mode=max

      # appleboy/ssh-action@master 액션을 사용하여 지정한 서버에 ssh로 접속하고, script를 실행합니다.
      # 실행 시, docker-compose를 사용합니다.
      # useranme : ubuntu 우분투 기반 ec2 일 경우 기본이름
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          username: ubuntu
          host: ${{ secrets.KCS_HOST_DEV }}
          key: ${{ secrets.KCS_KEY_DEV }}
          script: |
            docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
            sudo docker pull {{ secrets.DOCKER_REPOSITORY }}:latest
            docker-compose -f docker-compose-dev.yml down
            docker rmi $(docker images -q)
            cp -f ./src/main/resources/backend-submodule/docker-compose-dev.yml .
            rm -r src
            docker-compose -f docker-compose-dev.yml up -d
```

## Checkout Repository

```yaml
- name: Checkout repository
        uses: actions/checkout@v3
        with:
          token: ${{secrets.ACTION_TOKEN}}
          submodules: true
```

Github Action 에서 Repository 를 가져오는 과정이다. 여기서 눈여겨볼점은 `token: ${{secrets.ACTION_TOKEN}}` 과 ` submodules: true` 이다. 해당 과정은 submodule 을 사용하면 적어주어야 한다. submodule 을 사용하지 않는다면 필요없다.

### 서브모듈 이란?

기존의 Repository 가 있고, 거기안에 있는 새로운 Repository 를 만들면 이를 서브모듈이라고 한다.
보통 public Repository 에서 외부로 노출되면 안되는 설정들을 따로 private Repository 인 서브모듈로 만들어서 관리한다. 나같은 경우에는 application.yml 을 local, dev, prod 인 3가지로 나눴다. dev, prod 설정은 외부로 노출되면 안되기 때문에 서브모듈을 사용했다. 

### 주의점1

처음 서브모듈을 사용했을때 설정을 잘못해서 적용이 제대로 안된적이 있다. 적용이 안되면 Github Action 에서 checkout 할 때 못가져온다. 아래 사진처럼 화살표 폴더가 떠야 적용이 된거다.

<img src="../../assets/img/posts/ci/ga2-2.png">

### 주의점2

`token: ${{secrets.ACTION_TOKEN}}` 에서 토큰을 사용하는데 이 토큰은 Settings > Developer Settings > Personal access tokens > Tokens (classic) 에서 만든 토큰이다. 이 토큰은 만료날짜를 무한으로 하면 적용이 안된다는 버그가 있다. 토큰 설정은 아래와 같다.

<img src="../../assets/img/posts/ci/ga2-3.png">

## Docker buildx

```yaml
# 캐시를 사용하기위해 buildx 를 사용
      - name: Setup docker buildx
        uses: docker/setup-buildx-action@v2
```

해당 과정은 옵션이다. buildx 를 사용해 캐시를 적용했다. 캐시가 필요없으면 사용하지 않아도 된다.

## 환경변수 파일 생성

```yaml
# Repository secrets 에 등록해둔 환경변수 파일 생성
      - name: Copy secrets to application
        env:
          OCCUPY_ENV: ${{ secrets.OCCUPY_ENV }}
          OCCUPY_SECRET_DIR: ./src/main/resources  # 레포지토리 내 빈 env.yml의 위치 (main)
          OCCUPY_SECRET_DIR_FILE_NAME: env.yml                 # 파일 이름

        # 환경변수 값 복사
        run: |
          echo $OCCUPY_ENV >> $OCCUPY_SECRET_DIR/$OCCUPY_SECRET_DIR_FILE_NAME
```

서브모듈을 사용해 비밀정보를 관리할 수 도 있지만, ENV 파일 내용을 Repositry Secrets 에 등록해두어서 관리할 수 도 있다. 이전글에서는 이 방식으로 환경변수를 관리했어서 학습차원에서 남겨두었다. 관리해야할 파일이 많아지면 서브모듈로 넘어가는게 좋다.

## Send docker-compose

```yaml
# 도커 컴포즈 설정 파일 서버(EC2)로 전달
      - name: Send docker-compose.yml
        uses: appleboy/scp-action@master
        with:
          username: ubuntu
          host: ${{ secrets.KCS_HOST_DEV }}
          key: ${{ secrets.KCS_KEY_DEV }}
          source: "src/main/resources/backend-submodule/docker-compose-dev.yml"
          target: "/home/ubuntu/"

```

도커컴포즈 파일을 EC2 서버로 보내는 작업이다. 위와 같이 하면 EC2 서버에 `/home/ubuntu/src/main/resources/backend-submodule/docker-compose-dev.yml` 경로로 파일이 생긴다. EC2 에서는 docker-compose 파일을 실행시켜주어야 하기때문에 EC2 에 보내준다.

위 방법을 사용하지 않고 docker-compose 를 EC2 에 직접올리는 방법도 있다. docker-compose 는 변경이 잦지 않으므로 EC2에 직접올려도 괜찮다고 생각한다. 하지만 docker-compose 를 수정하기위해 EC2 를 접속하는과정은 귀찮다. GitHub 에서 관리할 수 있게하기위해 EC2 로 복사하는 방법을 선택했다.

## Docker login & push

```yaml
# Docker hub 로그인
      - name: Login to dockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME}}
          password: ${{ secrets.DOCKER_TOKEN}}

      # Docker Hub 에 푸시
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile.dev
          push: true
          tags: ${{ secrets.DOCKER_REPOSITORY }}:latest
          cache-from: type=gha
          cache-to: type=gha, mode=max
```

DockerHub 에 도커이미지를 푸시해주는 과정이다. 기존에는 아래 코드를 사용했다.

```yaml
# dockerfile을 통해 이미지를 빌드하고, 이를 docker repo로 push
      # DOCKER_REPOSITORY : [아이디]/[레포명]
      - name: Docker build & push to docker repo
        run: |
          docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
          docker build -t ${{ secrets.DOCKER_REPOSITORY }}:latest -f Dockerfile.dev .
          docker push ${{ secrets.DOCKER_REPOSITORY }}:latest
```

하지만 docker buildx 를 캐시를 이용하기위해 사용하면서 `docker/login-action@v2` 과 `docker/build-push-action@4` 를 이용했다. 여기서는 PASSWORD 대신에 TOKEN 이용했는데 TOKEN 은 DockerHub 에서 발급받을 수 있다. DOCKER_REPOSITORY 의 경우 [사용자명]/[레포명] 이다. 예를 들면 jinhoon227/backend 이렇게이다. 

### 주의점

처음에 이미지 이름을 jinhoon227/backend/dev:latest 이런식으로 작성했었는데 에러가 났다. 이름을 제대로 지으라고 말이다. [사용자명]/[레포명]:버전 이런식으로 네이밍을 해줘야한다. 그리고 반드시 사용자명은 일치해야한다. 도커허브 계정이 jinhoon227 인데 jin/backend:latest 로 하면 jinhoon227 과 jin 은 다르기때문에 푸시할 수 없다.

## Deploy to server
```yaml
# appleboy/ssh-action@master 액션을 사용하여 지정한 서버에 ssh로 접속하고, script를 실행합니다.
      # 실행 시, docker-compose를 사용합니다.
      # useranme : ubuntu 우분투 기반 ec2 일 경우 기본이름
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          username: ubuntu
          host: ${{ secrets.KCS_HOST_DEV }}
          key: ${{ secrets.KCS_KEY_DEV }}
          script: |
            docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
            sudo docker pull {{ secrets.DOCKER_REPOSITORY }}:latest
            docker-compose -f docker-compose-dev.yml down
            docker rmi $(docker images -q)
            cp -f ./src/main/resources/backend-submodule/docker-compose-dev.yml .
            rm -r src
            docker-compose -f docker-compose-dev.yml up -d
```

EC2 에서 Springboot 를 띄우는 과정이다. username 의 경우 EC2 ubuntu 라면 기본값으로 ubuntu 로 되어있다. EC2 여도 다른 이미지 기반이면 (Amazon Linux, MacOS 등) 이면 기본값이 다 다르다. KCS_HOST_DEV 의 경우 EC2 생성시 퍼블릭 호스트 이름이다. 예시로 `ec2-6-35-23-114.ap-northeast-2.compute.amazonaws.com` 이런형식이다. KCS_KEY_DEV 의 경우 EC2 생성시에 .pem 키 값이다. 맥이라면 cat kcs.pem 하면 키 값이 나오는데 

```
-----BEGIN RSA PRIVATE KEY-----
secretkeysecretkeysecretkeysecretkeysecretkeysecretkey
secretkeysecretkeysecretkeysecretkeysecretkeysecretkey
secretkeysecretkeysecretkeysecretkeysecretkeysecretkey
...
-----END RSA PRIVATE KEY-----
```

저 부분을 다 넣어줘야 한다. 처음에 `-----BEGIN RSA PRIVATE KEY-----` 와 `-----END RSA PRIVATE KEY-----` 이걸 안넣어주었다가 실행이 안되었다.

만약 DockerHub 에서 private repository 로 팠으면 반드시 DockerHub 에 로그인해주어야 한다. 그 다음 DockerHub 에서 푸시해놨던 이미지를 가져온다. 그리고 이전에 실행했던 컨테이너를 중지하고, 필요없는 이미지들을 삭제한다.

그리고 docker-compose 파일을 EC2 에 복사할때 경로가 `/src/main/resources/backend-submodule/docker-compose-dev.yml` 이거라고 했다. 해당 경로에 있는 파일을 home 으로 복사하고 src 폴더를 제거해주었다. 나는 home 에서 docker-compose 를 실행하기 위해서다.

## Dockerfile.dev V1

```yaml
FROM openjdk:17-alpine
ARG JAR_FILE=build/libs/*.jar
ARG SPRING_PROFILE=dev

COPY ${JAR_FILE} app.jar

ENV spring_profile=${SPRING_PROFILE}

ENTRYPOINT ["java", "-Dspring.profiles.active=${spring_profile}", "-Duser.timezone=Asia/Seoul", "-jar", "/app.jar"]
```

첫번째 도커파일로는 위에꺼를 사용했다. `openjdk:17-alpine` 의 경우 -alpine 이 붙어있는데 이는 좀 더 용량이 가벼운 파일이다. 100Mb 정도 작은데 조금이라도 용량을 줄이고자 사용했다. 이 외에도 여러방식으로 도커이미지 용량을 줄일 수 있는데 최대한 줄일 수 있으면 줄이는게 좋다. 

`ARG SPRING_PROFILE=dev` `ENV spring_profile=${SPRING_PROFILE} ` `"-Dspring.profiles.active=${spring_profile}"` 코드는 개발환경 yml 을 실행시키라고 지정해주기위해 사용했다. 나는 로컬, 개발, 운영환경이 분리되어있기 때문에 이를 통해 환경을 분리해주었다.

## Dockerfile.dev V2

```yaml
FROM openjdk:17-alpine as builder
WORKDIR app
ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM openjdk:17-alpine
WORKDIR app
COPY --from=builder app/dependencies/ ./
COPY --from=builder app/spring-boot-loader/ ./
COPY --from=builder app/snapshot-dependencies/ ./
COPY --from=builder app/application/ ./

ENTRYPOINT ["java", "-Dspring.profiles.active=${ACTIVE_SPRING_PROFILE}", "-Duser.timezone=Asia/Seoul", "org.springframework.boot.loader.launch.JarLauncher"]
```

도커 파일을 좀 더 효율적으로 실행시키기위해 변경했다. jar 파일을 4개의 레이어로 추출해 레이어별로 복사하는 과정을 거쳤다. 도커는 캐싱을 해두는데 변경이 되지 않았다면 캐싱을 활용한다. 그러니 변경이 잦은 부분을 최대한 늦게 복사하도록 한다. 

### 주의점1

Spring Boot 3.2 버전을 이상을 사용한다면 `org.springframework.boot.loader.launch.JarLauncher` 를 사용한다. 그전 버전에서는 `org.springframework.boot.loader.JarLauncher`  이다.

### 주의점2

여기서도 `-Dspring.profiles.active=${ACTIVE_SPRING_PROFILE}` 사용해 개발모드로 설정해준다. Dockerfile.dev 첫번째 버전에서는 Dockerfile 에서 이를 명시해주었고 두번째 버전에서도 이처럼 사용할려했는데, 아래와같이 dev 설정을 찾지못하는 문제가 발생했다. 그래서 이를 다음에나오는 docker-compose 에 명시해주는 해결되었다.

<img src="../../assets/img/posts/ci/ga2-4.png">

## docker-compose-dev

```yaml
version: '3'

services:
  server:
    image: jinhoon227/backend:latest
    container_name: server
    ports:
      - 8080:8080
    environment:
      - "ACTIVE_SPRING_PROFILE=dev"
```

docker-compose 는 **여러개**의 docker 컨테이너를 생성 또는 삭제할 수 있게해준다. 나는 하나밖에 사용하지 않아서 굳이 사용할 필요는 없었지만 확장성과 가독성측면에서 사용했다.

environment 는 앞서 도커파일 두번째 버전에서 설명한 환경변수 설정이다. 도커파일에 있는 `ACTIVE_SPRING_PROFILE` 에 `dev` 를 넣어준다.

## EC2 & RDS

EC2
- 월별 750시간까지 무료 (EC2 인스턴스 하나를 풀로 돌려도 남는 시간)

RDS
- RDS 인스턴스 1개 무료 사용 가능
- 월별 750시간까지 무료
- 범용(SSD) 데이터베이스 스토리지 20GB 제한.
- 데이터베이스 백업 및 DB 스냅샷용 스토리지 20GB

AWS 프리티어는 위의조건과 같다. 그렇다고 무작정 사용하지말고 조건들을 잘 체크해야한다. 특히 RDS 는 과금되는 요소가 기본으로 체크되어있는 경우많으니 잘 확인하고 생성하자. RDS 는 아래글을 참고했다.

[RDS 인스터스 생성하기](https://developer111.tistory.com/52)

그리고 생성하기전에 꼭 region 을 잘 확인하자. 기본으로 버지니아 북부로 되어있는데 서울로 변경해야한다. 아무래도 물리적인 거리가 멀면 실제 통신하는데도 오래걸리는것은 당연하다. 리전을 잘못설정해서 변경할려면 거의 새로 다시 만드는 수준과 동일하니 잘 보고 생성하자. 

또 보안그룹을 잘 설정해주어야 한다. EC2 의 경우 8080(SpringBoot), 443(Http), 80(Https), 3306(MySQL) 를 열어주어야 한다. 그리고 RDS 는 위의 EC2 에서 접속을 요청하니 EC2 에서 설정한 보안그룹을 추가해 주어야 한다.

그리고 EC2 를 설치했다면 거기서 docker 와 docker-compose 를 설치해주어야 한다. 아래글 참고할 수 있다.

[EC2 에서 docker, docker-compose 설치](https://narup.tistory.com/223)

## 마무리

해당 과정을 수행하면서 가장 많이 고민한것은 노출되면 안되는 정보는 어떻게 관리하는가? 이다. 처음에는 Github 에서 제공해주는 Repository secret 에서 관리를 했지만 수정과 관리에 어려움이 있다. 그래서 찾다보니 서브모듈이라는게 좋은게 있어 이용할 수 있었다.

## Reference

[https://velog.io/@bjo6300/Springboot-docker-compose%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%B4-springboot-nginx-%EC%97%B0%EB%8F%99%ED%95%98%EA%B8%B0](https://velog.io/@bjo6300/Springboot-docker-compose%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%B4-springboot-nginx-%EC%97%B0%EB%8F%99%ED%95%98%EA%B8%B0)