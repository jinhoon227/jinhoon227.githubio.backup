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

