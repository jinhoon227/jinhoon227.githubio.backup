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

해당 코드도 옵션이다. 위의 옵션을 사용하면 테스트 실패시 어떤 라인이 잘못됐는지 알려준다.

## 마무리
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