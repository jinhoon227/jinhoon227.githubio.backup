---
layout: post
title:  "[CICD] Spring boot 환경 Github Action 에서 Test 적용기 "
categories: SPRING CICD
tags : java spring jpa cicd
---

## CI - Github Action

Github 에 push, pull request 가 발생하면 자동으로 Test 를 수행하면 좋겠다고 생각했다. CI 를 할 수 있게 도와주는 프로그램으로 Jenkins, Travis CI, Github Action 등등 여러가지가 있다. Jenkins 는 많은 기업에서 사용하면서 인증받은 CICD 툴이다. 파이프라인을 한곳에서 관리할 수 있어 대규모 시스템에 적합하다. 하지만 설정하는 과정이 다소 복잡해서 처음 접하는 사람에게 어려울 수 있다. Travis CI 는 사용하기 편하지만 일정기간 이상 사용하면 유료로 전환된다. Github Action 은 사용하기 편하고, Github 관련해서 시각화가 잘되어있다. 아무래도 Github Action 이다보니 Github 와 연동되는게 많을것이다. 무료이고, 사용이 편한것을 찾다보니 Github Action 을 사용하게 되었다. (Github Action 적용 레포가 private 면 제한이 있다. public 이면 항상 무료다!)

## CI - Test

Github Action 에서 테스트를 적용할려면 `.github` 폴더 안에 `workflows` 폴더를 만들어줘야 한다. 해당 폴더를 만들고 안에 `yml` 파일을 넣어두면 조건에 따라서 알아서 실행해준다. 테스트 CI 를 해줄 `action-test.yml` 파일을 `.github/workflows/action-test.yml` 에 넣어주었다.

<img src="../../assets/img/posts/ci/ga1.png">


### 파일 코드

```yaml
name: action-test

# 하기 내용에 해당하는 이벤트 발생 시 github action 동작
on:
  push:
    branches:
      - main
      - develop

  pull_request:
    branches:
      - main
      - develop


# 참고사항
# push가 일어난 브랜치에 PR이 존재하면, push에 대한 이벤트와 PR에 대한 이벤트 모두 발생합니다.

# 단위 테스트 결과를 발행하기 위해 쓰기 권한을 주어야 합니다.
permissions: write-all

jobs:
  test: # 테스트를 수행합니다.
    runs-on: ubuntu-latest # 실행 환경 지정
    steps:
      - name: Checkout Repostiory
        uses: actions/checkout@v3 # github action 버전 지정(major version)
      
      - name: Set up JDK 17 # JAVA 버전 지정
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto' 
      
      # github action 에서 Gradle dependency 캐시 사용
      - name: Cache Gradle packages
        uses: actions/cache@v3
        with: # 캐시로 사용될 경로 설정
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }} # 캐시 키 설정
          restore-keys: |
            ${{ runner.os }}-gradle- # 복원 키 설정

      - name: Grant execute permission for gradlew # 실행할 수 있게 권한주기
        run: chmod +x gradlew

      - name: Test with Gradle
        run: ./gradlew test
      
      # 테스트 후 Result를 보기위해 Publish Unit Test Results step 추가
      - name: Publish Unit Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: ${{ always() }}  # 테스트가 실패하여도 Report를 보기 위해 `always`로 설정
        with:
          files: build/test-results/test/TEST-*.xml

      # 테스트 실패시 어디서 틀렸는지 알려줍니다.
      - name: Add comments to a pull request
        uses: mikepenz/action-junit-report@v3
        if: ${{ always() }}
        with:
          report_paths: build/test-results/test/TEST-*.xml
```

## on

```yaml
# 하기 내용에 해당하는 이벤트 발생 시 github action 동작
on:
  push:
    branches:
      - main
      - develop

  pull_request:
    branches:
      - main
      - develop
```

on 은 어떤 조건에서 해당 파일이 작동할지 적어주는 것이다. 나는 push, pull request 가 발생했을시에 작동하도록 했다. 그리고 main, develop 브랜치 대상으로만 설정했다. 만약 feature/first-feature-branch, feature/second-feature-branch 같이 feature 하위 폴더 브랜치에도 동작하게 하고싶다면 feature/** 를 적어주면 된다.

## JDK 17

```yaml
      - name: Set up JDK 17 # JAVA 버전 지정
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto' # OpenJDK 배포사 corretto, temurin
```

자바의 어떤 버전으로 진행하는지 적어주어야 한다. 본인이 사용하는 JDK 버전을 잘 확인하고 사용하자. 보통 Spring Boot 3.0 버전 이상을 쓰면 JDK 17 을 사용한다.

## 캐시 적용

```yaml
      - name: Cache Gradle packages
        uses: actions/cache@v3
        with: # 캐시로 사용될 경로 설정
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }} # 캐시 키 설정
          restore-keys: |
            ${{ runner.os }}-gradle- # 복원 키 설정
```

해당 과정은 필수 과정은 아니다. path 에 적힌 `~/.gradle/caches`, `~/.gradle/wrapper` 를 (의존성파일) 을 캐싱해두어 다음에 실행할때 캐싱된 파일을 이용해 빌드 시간을 좀 더 줄일 수 있다. 30% 가량 빌드시간을 줄 일 수 있는것으로 보인다. 

## 권한 설정

```yaml
      - name: Grant execute permission for gradlew # 실행할 수 있게 권한주기
        run: chmod +x gradlew
```

해당 파일은 우분투 리눅스 최신버전으로 실행된다, 리눅스 환경에서 Java 빌드명령어(gradlew)를 실행시켜줄 권한을 주어야한다. 

## 테스트 실행

```yaml
      - name: Test with Gradle
        run: ./gradlew test
```

테스트를 수행한다! 나는 여기서 에러가 발생했는데 아래와 같다.

<img src="../../assets/img/posts/ci/ga2.png">

`contextLoads() FAILED` 에러가 발생했는데, 구글링해보니 워낙 원인이 많은에러라 찾기 어려웠다. 좀 더 Spring Boot 관련해서 찾아보니 DB 설정이 안되어있어서 발생한 에러라는 글이 있었다.

<img src="../../assets/img/posts/ci/ga3.png">

`@SpringBootTest` 어노테이션이 달려있으면 DB 에 연결해서 테스트를 진행하는데 파일에서는 따로 DB 설정을 하지 않았기에 발생한 에러다. 만약 MySql 을 사용한다면 MySql DB 설정을 따로 파일에 명시해 주어야 한다. 하지만 단위테스트를 할때 MySql 보다는 h2 인메모리 DB 를 사용하는게 가볍기 때문에 빌드도 빨리 할 수 있다. 그래서 h2 DB 를 사용하기위해 추가로 설정을 해주었다.

build.gradle 파일을 아래와 같이 설정한다.
```gradle
...

dependencies {
	...

	// 테스트 설정
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testRuntimeOnly 'com.h2database:h2'
}

...
```

test 폴더 안에 resource 폴더를 만들고 안에 application.yml 를 넣어서 test 용 설정을 추가한다.

```yaml
# 테스트용 설정 - 인메모리 db 사용
spring:
  datasource:
    url: jdbc:h2:mem:test
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        format_sql: true
        
logging:
  level:
    org.hibernate.SQL: debug
```

테스트 할때는 h2 DB 인 인메모리 DB 를 사용하기 때문에 파일에 추가적으로 DB 설정을 할 필요가 없다. 이렇게함으로써 `contextLoads() FAILED` 에러를 해결할 수 있었다.

### MySql 로 테스트 하고 싶다면

H2 는 매우 가벼운 DB 라 속도가 빠르다는 장점이 있지만, 그만큼 제공하는 기능이 적기때문에 MySQL 에서 사용하던것을 적용하지 못할 수 있다. 그래서 실제환경과 동일하게 MySQL 을 구축하고 싶다면 아래코드를 추가하자.

```yaml

      - name: Set up MySQL
        uses: mirromutth/mysql-action@v1.1
        with:
          host port: 3306
          container port: 3306
          mysql database: 'nainga_test'
          mysql user: 'test'
          mysql password: ${{ secrets.DB_PASSWORD }}
```

해당 코드를 추가하면 Github Action 에서 MySql 이미지를 가져와 띄운다. 그래서 h2 를 사용할때보다 시간이 좀 걸린다(10초 정도 더?) 

### gradlew test --full-stacktrace

```yaml
      - name: Test with Gradle
        run: ./gradlew test --full-stacktrace
```

테스트를 돌리다보니 틀렸다고는 나오는데 자세하게 안알려줘서 정확히 오류를 찾기어려웠다. 그래서 추적하기위해서 `--full-stacktrace` 를 붙여주면 어디서 잘못됬는지 알려준다. 더 딮하게 알고 싶으면 `./gradlew test -i` 를 사용해보자. 정말 모든 로그를 출력한다. 그래서 더 찾기 어려울워서 적당히 알고싶으면 위에껄 사용하자.


## 단위테스트 시각화

```yaml
  # 테스트 후 Result를 보기위해 Publish Unit Test Results step 추가
      - name: Publish Unit Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: ${{ always() }}  # 테스트가 실패하여도 Report를 보기 위해 `always`로 설정
        with:
          files: build/test-results/test/TEST-*.xml # 로컬에 저장할 위치
```

해당 코드는 옵션이다. 위의 옵션을 사용하면 아래와 같이 테스트 통과가 어떻게 됐는지 알려준다.

<img src="../../assets/img/posts/ci/ga4.png">

위의 시각화 기능 사용중에 아래 에러가 발생했다.

<img src="../../assets/img/posts/ci/ga5.png">

`github.GithubException.GithubException: 403 {"message": "Resource not accessible by integration", "documentation_url": "https://docs.github.com/rest/checks/runs#create-a-check-run"}` 리소스에 접근할 수 없다고 하는걸 보고 권한문제임을 알 수 있다. 아무래도 보고서를 작성하다보니 쓰기권한이 필요했다.
그래서 위에 아래와 같은 코드를 추가해주었다.

> `permissions: write-all`

## 단위테스트 시각화2

```yaml
- name: Add comments to a pull request
        uses: mikepenz/action-junit-report@v3
        if: ${{ always() }}
        with:
          report_paths: build/test-results/test/TEST-*.xml
```

해당 코드도 옵션이다. 위의 옵션을 사용하면 테스트 실패시 아래 사진처럼 어떤 라인이 잘못됐는지 알려준다.

<img src="../../assets/img/posts/ci/ga8.png">

## 환경 변수 추가 설정

다시 한번 `contextLoads() FAILED` 에러가 발생했다!

<img src="../../assets/img/posts/ci/ga9.png">

발생 이유는 다음과 같았다. 해당 어플리케이션에서는 외부 API 를 요청한다. 그리고 외부 API 를 요청할때 KEY 를 발급해서 해당 키를 주면 요청에 응답해준다. 문제는 이 KEY 를 아무생각 없이 github 에 올리면 다른 사람도 사용할 수 있다. 유료 API 라면 다른 사람이 무분별한 사용으로 요금폭탄을 맞을 수 도 있는것이다. 그래서 이런 중요한 키 정보들은 파일을 따로 관리하여 github 에 올리지 않는다. 여기서 문제가 생긴다.. github action 에서 해당 키 값이 없으니 이를 테스트하면 외부 API 를 요청할 수 없어 테스트 통과가 안된다는점이다.

열심히 구글링 해본결과 아래 방식을 사용한다.

### env.yml 파일 설정

env.yml 파일로 환경변수를 저장한다. 그리고 application.yml 에서 해당 env.yml 파일을 포함하도록 한다. 로컬에서는 env.yml 을 사용해서 빌드와 테스트를 문제없이 할 수 있다.(env.yml 은 .gitignore 에 등록하여 꼭 github 에 올라가지 않도록하게 하자.)
application.yml 이 env.yml 파일을 포함하는 방법은 아래와 같다. main 말고도 test 에 있는 applciation.yml 에도 설정해주자.

```yaml
# 이 파일은 develop level properties
spring:
  config:
    import: env.yml
...
```

env.yml 예시로는 아래와 같다. google 비밀키를 예시로 사용한다.

```yaml
google-key: googlesecretkey
```

### repository secrets 등록

github action 에서는 참고할 env.yml 이 없다. 그래서 workflow 진행과정에서 env.yml 을 만들것이다. 그렇기에 github action 에서 사용할 변수는 repository secrets 에 등록해야 된다.
repository secrets 등록방법은 github > settings > 좌측메뉴에서 Secrets and variables > Actions 들어가면 repository secrets 가 있고 환경변수명을 정해주고 비밀키를 넣어주면된다. 아래사진처럼 GOOGLE_API_KEY 로 환경변수를 설정해주었다.

<img src="../../assets/img/posts/ci/ga10.png">

repository secrets 키는 다른사람이 볼 수 없다. 그래서 여기에 키 값을 등록하고 github action 에서 사용한다. github action 에서는 위의 repository secrets 에 등록해둔 키 값으로 env.yml 을 만드는 것이다. 그러면 github action 에서도 문제없이 테스트와 빌드를 할 수 있다.

### workflow 파일 수정

action-test 파일을 아래와 같이 수정한다.

```yaml
name: action-test

# 하기 내용에 해당하는 이벤트 발생 시 github action 동작
on:
  push:
    branches:
      - main
      - develop

  pull_request:
    branches:
      - main
      - develop


# 참고사항
# push가 일어난 브랜치에 PR이 존재하면, push에 대한 이벤트와 PR에 대한 이벤트 모두 발생합니다.

# 단위 테스트 결과를 발행하기 위해 쓰기 권한을 주어야 합니다.
permissions: write-all

jobs:
  test: # 테스트를 수행합니다.
    runs-on: ubuntu-latest # 실행 환경 지정
    steps:
      - name: Checkout Repostiory
        uses: actions/checkout@v3 # github action 버전 지정(major version)
      
      - name: Set up JDK 17 # JAVA 버전 지정
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto' 

      - name: Copy secrets to application
        env:
          GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }} # 구글키를 repository secrets 에서 가져옴
          OCCUPY_SECRET_DIR: ./src/main/resources  # 레포지토리 내 빈 env.yml의 위치 (main)
          OCCUPY_SECRET_DIR_FILE_NAME: env.yml                 # 파일 이름

        # secrets 값 복사
        run: |
          echo "google-key: $GOOGLE_API_KEY" >> $OCCUPY_SECRET_DIR/$OCCUPY_SECRET_DIR_FILE_NAME
      
      # github action 에서 Gradle dependency 캐시 사용
      - name: Cache Gradle packages
        uses: actions/cache@v3
        with: # 캐시로 사용될 경로 설정
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }} # 캐시 키 설정
          restore-keys: |
            ${{ runner.os }}-gradle- # 복원 키 설정

      - name: Grant execute permission for gradlew # 실행할 수 있게 권한주기
        run: chmod +x gradlew

      - name: Test with Gradle
        run: ./gradlew test
      
      # 테스트 후 Result를 보기위해 Publish Unit Test Results step 추가
      - name: Publish Unit Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: ${{ always() }}  # 테스트가 실패하여도 Report를 보기 위해 `always`로 설정
        with:
          files: build/test-results/test/TEST-*.xml

      # 테스트 실패시 어디서 틀렸는지 알려줍니다.
      - name: Add comments to a pull request
        uses: mikepenz/action-junit-report@v3
        if: ${{ always() }}
        with:
          report_paths: build/test-results/test/TEST-*.xml
```

위는 전체코드이고 추가된 코드는 아래와 같다.

```yaml
- name: Copy secrets to application
  env:
    GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }} # 구글키를 repository secrets 에서 가져옴
    OCCUPY_SECRET_DIR: ./src/main/resources  # 레포지토리 내 빈 env.yml의 위치 (main)
    OCCUPY_SECRET_DIR_FILE_NAME: env.yml                 # 파일 이름

  # secrets 값 복사
  run: |
    echo "google-key: $GOOGLE_API_KEY" >> $OCCUPY_SECRET_DIR/$OCCUPY_SECRET_DIR_FILE_NAME
```

Repository secrets 에 등록한 키는 ${{ secrets.XXX }} 를 통해 접근할 수 있다. 등록해둔 GOOGLE_API_KEY 를 쓸려면 ${{ secrets.GOOGLE_API_KEY }} 이렇게 적으면 되는것이다.

이 구글키를 이용해 env.yml 을 만들어서 넣어준다. 해당과정에 삽질을 엄청 했는데... 처음에 `echo $GOOGLE_API_KEY >> $OCCUPY_SECRET_DIR/$OCCUPY_SECRET_DIR_FILE_NAME` 이렇게 했었다.
이렇게했더니 키값을 못찾는 문제가 발생했다. 10시간 동안 삽질한 결과, 키이름을 안적어준게 문제였다. 저렇게 하면 env.yml 파일이 아래처럼 만들어진다.

```yaml
googlesecretkey
```

눈이 좋은분이라면 눈치챘을것이다. googlesecretkey 가 누구의 비밀키인지 정의가 안되어있다. 원래라면
아래와같이 만들어져야한다.

```yaml
google-key: googlesecretkey
```

이 처럼 앞에 어떤 키인지 알려줘야하는데 키값만 떡하니 있으니 못찾은것이다. 그래서 `echo "google-key: $GOOGLE_API_KEY" >> $OCCUPY_SECRET_DIR/$OCCUPY_SECRET_DIR_FILE_NAME`
를 사용해서 만들어주었다. 전자의 방법으로 할려면 Repository secrets 에 GOOGLE_API_KEY 를 `google-key: googlesecretkey` 처럼 키와 밸류를 같이 적어주어야한다.

### 참고로

여기서 살짝 벗어난 주제지만 action-test.yml 파일은 test 만 수행하는 workflow 다. 그런데 환경변수가 불러오는 코드에서 `OCCUPY_SECRET_DIR: ./src/main/resources  # 레포지토리 내 빈 env.yml의 위치 (main)` 로 main 에다가 env.yml 을 만든다. test 면 ./src/test/resources 에서 만들어야 되는게 아니냐라는 의문이 들 수 있다.

springboot 를 좀 사용했던분들이라면 답을 알겠지만, 모르는분들이나 까먹었던분들도 있으실까봐 적어둔다.
springboot 에서 test 를 할때, resources 에 있는 파일의 경우 test 폴더에 없으면 main 에 있는파일을 먼저 참고한다. 이게 무슨 뜻이냐면 테스트할때 test/resources/application.yml 이 없다면 main/resoures/application.yml 을 참고한다는 뜻이다. (물론 test 에 있다면 이를 우선으로 적용한다.)

즉, 여기서 main 에 env.yml 을 만드는 이유는, "내" 프로젝트 에서는 로컬에서도 test 에 env.yml 을 생성하지 않았기때문이다. 왜? 냐면 test 에도 env.yml 만들면 관리할 파일이 늘어나기에 귀찮아지기 때문이다. 물론 필요성이 생긴다면 분리하겠지만 지금까지는 아니다.(application.yml 은 main, test 둘 다 만든 이유는 test 에서는 h2 database 를 사용하기때문이다. 이렇듯 분리할 필요성이 있으면 분리하면 된다.)

### 내 최종파일

계속 진행하면서 코드가 바뀌다보니 코드가 변경이 많았다. 그래서 현재 최종 코드 파일을 적어본다.
세부사항으로 달라진건 다음과같다.

1. H2 가 아닌 MySql 사용
2. `./gradlew test --full-stacktrace` 로 오류발생시 추적가능
3. 서브모듈을 사용한 설정값 관리

3번의 서브모듈을 사용하는것은 처음보는 글일것이다. 해당글은 [서브모듈이란?](../GithubAction2) 글을 참고하면 좋다. 서브모듈을 사용하면 좀 더 설정값을 편하게 관리할 수 있다.  

```yaml
name: action-test

# 하기 내용에 해당하는 이벤트 발생 시 github action 동작
on:
  push:
    branches:
      - main
      - develop

  pull_request:
    branches:
      - main
      - develop


# 참고사항
# push가 일어난 브랜치에 PR이 존재하면, push에 대한 이벤트와 PR에 대한 이벤트 모두 발생합니다.

# 단위 테스트 결과를 발행하기 위해 쓰기 권한을 주어야 합니다.
permissions: write-all

jobs:
  test: # 테스트를 수행합니다.
    runs-on: ubuntu-latest # 실행 환경 지정
    steps:
      - name: Checkout Repostiory
        uses: actions/checkout@v3 # github action 버전 지정(major version)
        # 아래는 서브모듈 사용으로 추가
        with: 
          token: ${{secrets.ACTION_TOKEN}} 
          submodules: true

      - name: Set up JDK 17 # JAVA 버전 지정
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto' # OpenJDK 배포사 corretto, temurin

      - name: Set up MySQL
        uses: mirromutth/mysql-action@v1.1
        with:
          host port: 3306
          container port: 3306
          mysql database: 'nainga_test'
          mysql user: 'test'
          mysql password: ${{ secrets.DB_PASSWORD }}

      # github action 에서 Gradle dependency 캐시 사용
      - name: Cache Gradle packages
        uses: actions/cache@v3
        with: # 캐시로 사용될 경로 설정
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }} # 캐시 키 설정
          restore-keys: |
            ${{ runner.os }}-gradle- # 복원 키 설정

      - name: Grant execute permission for gradlew # 실행할 수 있게 권한주기
        run: chmod +x gradlew

      - name: Test with Gradle
        run: ./gradlew test --full-stacktrace

      # 또한, Github Actions 상에서는 머신의 IP 주소를 특정할 수가 없어서 도로명 주소 API 사용이 불가능하기 때문이 이와 관련된 테스트도 Skip!
      # 테스트 후 Result를 보기위해 Publish Unit Test Results step 추가
      - name: Publish Unit Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: ${{ always() }}  # 테스트가 실패하여도 Report를 보기 위해 `always`로 설정
        with:
          files: build/test-results/test/TEST-*.xml

      # 테스트 실패시 어디서 틀렸는지 알려줍니다.
      - name: Add comments to a pull request
        uses: mikepenz/action-junit-report@v3
        if: ${{ always() }}
        with:
          report_paths: build/test-results/test/TEST-*.xml

```

## BLOCKING 설정

해당 파일을 workflows 폴더 안에 넣었다면 설정해둔 트리거가 작동할때마다 test 검사를 진행할것이다. 근데 test 검사를 수행하고 잘못된게 있다면 강제로 Merge 를 막아야되지 않겠는가? 착한 팀원이라면 당연히 merge 를 안하겠지만, 어쨋든 실수로라도 막아두기위해 github 에서는 강제로 merge 를 막아주는 기능을 제공한다.

Github 사이트에서 설정한다.

> Repository -> settings -> branches

위의 경로로 들어가 설정해둔 룰이 있다면 편집해주고 없다면 Add rule 을 해준다.

<img src="../../assets/img/posts/ci/ga6.png">

그 다음 통과해주어야할 설정을 검색해서 넣어주면 된다. 그러면 테스트 실패시 merge 가 자동으로 막힌다!

<img src="../../assets/img/posts/ci/ga7.png">


## Reference

[https://velog.io/@kimseungki94/Jenkins-vs-Github-Action-%EC%96%B4%EB%96%A4%EA%B1%B8-%EC%93%B0%EB%8A%94%EA%B2%8C-%EC%A2%8B%EC%9D%84%EA%B9%8C](https://velog.io/@kimseungki94/Jenkins-vs-Github-Action-%EC%96%B4%EB%96%A4%EA%B1%B8-%EC%93%B0%EB%8A%94%EA%B2%8C-%EC%A2%8B%EC%9D%84%EA%B9%8C)  
[https://kotlinworld.com/399](https://kotlinworld.com/399)  
[https://velog.io/@ohzzi/%EC%9A%B0%EC%95%84%ED%95%9C%ED%85%8C%ED%81%AC%EC%BD%94%EC%8A%A4-4%EA%B8%B0-220802-F12-%EA%B0%9C%EB%B0%9C%EC%9D%BC%EC%A7%80](https://velog.io/@ohzzi/%EC%9A%B0%EC%95%84%ED%95%9C%ED%85%8C%ED%81%AC%EC%BD%94%EC%8A%A4-4%EA%B8%B0-220802-F12-%EA%B0%9C%EB%B0%9C%EC%9D%BC%EC%A7%80)  
[https://devs0n.tistory.com/25](https://devs0n.tistory.com/25)  