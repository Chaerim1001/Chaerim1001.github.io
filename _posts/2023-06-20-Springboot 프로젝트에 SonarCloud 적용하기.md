---
title: Springboot 프로젝트에 SonarCloud 적용하기
date: 2023-06-20
categories: [Spring]

tags: [Springboot, SonarCloud]
---

> 본 포스트는 아래의 환경을 기준으로 작성되었습니다. <br>
> Springboot 3.0.1 <br>
> Java 17 <br>
> Gradle 7.6
{: .prompt-info}

## 정적 코드 분석
**정적 코드 분석**이란 단어 그대로 소스 코드를 실행하지 않고 정적으로 코드를 분석하는 것입니다.
**소스 코드의 품질을 높이기 위해 잠재적인 버그나 코딩 컨벤션에 어긋난 부분을 찾는 것**을 의미하는데요.

## 정적 코드 분석을 사용하는 이유
정적 코드 분석을 사용하면 흔히 **코드 스멜(code smell)** 이라고 불리는 문제들과 보안 취약점 등의 문제를 사전에 찾아낼 수 있습니다.

* 잠재적으로 버그가 발생할 수도 있는 코드를 찾을 수 있다.
* 코드 스타일(코딩 컨벤션) 위반 여부를 판단할 수 있다.
* 오타를 찾아낼 수 있다.
* 사용되지 않는 코드를 찾아낼 수 있다.
* 잠재적인 보안 취약점을 발견할 수 있다.

## SonarQube
<a href="https://docs.sonarqube.org/latest/">SonarQube</a>는 현재 30개 이상의 프로그래밍 언어에서 버그, 코드 스멜, 보안 취약점을 발견할 목적으로 사용하는 오픈 소스 코드 품질 관리 플랫폼입니다.
다양한 빌드 시스템, CI 엔진과 통합되어 Devops 실천을 지원하기에 많은 프로젝트에서 정적 분석 도구로 선택되어 사용되는 도구입니다.

## SonarCloud

<a href="https://docs.sonarcloud.io/?_gl=1*12cgd2m*_gcl_aw*R0NMLjE2ODcxODY4MDcuQ2p3S0NBanctYi1rQmhCLUVpd0E0ZnZLckdZb2Fta2hnc2pOdmxhZFFlaS1PYkM2T2NKbWNrbWJSQWU0NTZuMlNYN2UtT0x0TGJYa3p4b0NmaDhRQXZEX0J3RQ..*_gcl_au*MTYwNzA3MjkzOS4xNjg3MDk0NDk1*_ga*MTI5OTg1ODczOC4xNjg3MDk0NDk1*_ga_9JZ0GZ5TC6*MTY4NzE4NTkwNC4yLjEuMTY4NzE4NjgwNi42MC4wLjA.">SonarCloud</a> 또한 SonarQube와 같이 코드 품질을 관리하기 위해 사용하는 도구입니다. Github과 Bitbucket 같은 코드 저장소와 원활하게 통합되어 코드 품질 분석의 자동화를 편리하게 지원합니다.

## SonarQube VS SonarCloud
SonarQube와 SonarCloud 모두 코드 품질 관리 도구입니다. 그렇다면 이 둘은 어떤 차이가 있을까요?

* SonarQube는 **온프레미스 환경**에서 사용할 수 있는 오픈소스 코드 품질 관리 플랫폼으로, SonarQube를 사용하기 위해서는 자체 서버에 설치하고 직접 관리해야 합니다.
* 반면에 SonarCloud는 **클라우드 기반의 호스팅 서비스**로 SonarQube와 유사한 기능을 제공하지만 클라우드 상에서 서비스를 제공하기 때문에 자체 서버를 설치하거나 유지 관리할 필요가 없습니다.

보안 상의 이슈로 클라우드 기반 서비스를 사용할 수 없거나, SonarQube에서 제공하는 다양한 플러그인을 사용하고 싶은 경우에는 SonarQube를 선택하고 그 외에는 직접 서버를 관리할 수고가 없는 SonarCloud를 활용하는 것이 좋다고 생각합니다. 

## 프로젝트에 SonarCloud 적용하기
### (1) SonarCloud 계정 만들기
SonarCloud를 사용하기 위해 <a href="https://www.sonarsource.com/products/sonarcloud/">홈페이지 </a>에 접속해 계정을 생성해줍니다!

<img src="https://velog.velcdn.com/images/chaerim1001/post/e2d80935-a0c7-4b71-97b5-5259782c6442/image.png" alt="" width=500px height=500px>

### (2) SonarCloud Organization 생성 및 설치

생성한 계정으로 로그인 후 Github과의 연결을 위해 SonarCloud 사용을 원하는 레포지토리가 있는 Organization을 import해줍니다.

![](https://velog.velcdn.com/images/chaerim1001/post/9b15d0f1-3825-4c04-83ab-598d80da5bf8/image.png)

step을 따라가며 원하는 organization의 원하는 repository를 선택해주시면 됩니다!

![](https://velog.velcdn.com/images/chaerim1001/post/a5e605bf-ffe2-44bd-b229-5a526d196ee6/image.png)

원하는 repository를 가져온 뒤에는 name과 key를 설정해주고 (이후 프로젝트 연결에 사용됩니다.) plan을 선택해주시면 됩니다.

![](https://velog.velcdn.com/images/chaerim1001/post/eccde557-1b0b-440b-a5a6-73e9884c3288/image.png)

### (3) Project 생성하기
Organization 생성이 끝났으면 이제 Project를 생성하면 됩니다. Project는 전 단계에서 불러온 repository를 선택하여 생성할 수 있습니다.

![](https://velog.velcdn.com/images/chaerim1001/post/70f0db48-0795-4576-8e52-473b5159974a/image.png)

프로젝트를 생성하면 아래와 같이 Code 분석에 대한 대시보드를 확인할 수 있습니다!

![](https://velog.velcdn.com/images/chaerim1001/post/7d3a1b02-aa3b-4a45-84bf-9d23bd8f0883/image.png)

### (4) Automatic Analysis 끄기
SonarCloud는 기본적으로 **Automatic Analysis**를 지원합니다. 그러나 우리는 CI 작업 과정에서 분석을 수행할 것이기 때문에 해당 옵션은 off 후 진행하겠습니다. (CI-based Analysis는 Automatic Analysis와 함께 실행하면 충돌이 나므로 반드시 비활성화 해야합니다.)

`Administration > Analysis Method` 에서 Off로 바꿀 수 있습니다. 

![](https://velog.velcdn.com/images/chaerim1001/post/50b60691-7d19-441d-95fe-09e70d5a3cc4/image.png)

### (5) Springboot 프로젝트 세팅

`build.gradle`에 plugin과 설정을 추가해 줍니다.
아래에서 사용한 `organizationKey`와 `projectKey`는 SonarCloud 프로젝트 `information`에서 확인 가능합니다.

```yaml
{
    plugin {
        id 'jacoco'
        id 'org.sonarqube' version '4.2.1.3168'
    }

    sonar {
        properties {
            property 'sonar.host.url', 'https://sonarcloud.io'
            property 'sonar.organization', 'organizationKey'
            property 'sonar.projectKey', 'projectKey'
            property 'sonar.coverage.jacoco.xmlReportPaths', 'build/reports/jacoco/test/jacocoTestReport.xml'
        }
    }

    jacocoTestReport {
        reports {
            xml.enabled true
        }
    }
}
```

SonarCloud 기본 설정만으로는 프로젝트의 **Test Coverage**를 확인할 수 없기 때문에 추가적인 설정이 필요합니다.
 <a href="https://docs.sonarcloud.io/enriching/test-coverage/overview/?_gl=1*1bw594d*_gcl_aw*R0NMLjE2ODcxODk2NDkuQ2p3S0NBanctYi1rQmhCLUVpd0E0ZnZLckNCaEM2dVJDamdNWk5TajI5MUFVWGpscXlEYkppQUVsUkp2VzdqcUYxRy0xaE9SbWJuSEF4b0N4VllRQXZEX0J3RQ..*_gcl_au*MTYwNzA3MjkzOS4xNjg3MDk0NDk1*_ga*MTI5OTg1ODczOC4xNjg3MDk0NDk1*_ga_3BQP5VHE6N*MTY4NzE4NjgxOS41LjEuMTY4NzE4OTc3My4wLjAuMA..*_ga_9JZ0GZ5TC6*MTY4NzE4NTkwNC4yLjEuMTY4NzE4OTc3My42MC4wLjA.">안내 링크</a>로 들어가면 언어에 맞게 추가 설정에 대한 내용이 잘 나와있습니다.

![](https://velog.velcdn.com/images/chaerim1001/post/4b0bee49-20b7-4bc6-8739-8002beca9916/image.png)

현재는 자바를 사용하고 있기 때문에 <a href="https://docs.sonarcloud.io/enriching/test-coverage/java-test-coverage/"> Jacoco에 대한 설정</a> 도 함께 추가해주었습니다. 다른 언어를 사용하시는 분들은 그에 맞는 설정을 찾아 추가해주시면 됩니다.

### (6) Github Actions Workflow 작성하기

이제 `Github Actions workflow`를 작성해 ci 작업과 SonarCloud를 연동해보겠습니다.

```yml
name: CI
on:
  push:
    branches:
      - release
      - develop
  pull_request:
jobs:
  sonarcloud:
    name: SonarCloud
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Set up JDK 17
        uses: actions/setup-java@v2
        with:
          java-version: 17
          distribution: 'temurin'
          cache: 'gradle'
      - name: Cache SonarCloud packages
        uses: actions/cache@v3
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: ${{ runner.os }}-sonar
      - name: Build and analyze
        run: ./gradlew build jacocoTestReport sonar --info
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

ci workflow 파일에서는 두가지 secret key가 사용됩니다.
* `GITHUB_TOKEN`: Github에서 제공하는 토큰이므로 따로 설정해줄 필요는 없습니다.
* `SONAR_TOKEN`: SonarCloud에 대한 액세스를 인증하는데 사용되는 토큰으로, SonarCloud의 `Security` 페이지에서 생성할 수 있습니다.

SonarCloud의 Security 페이지에서 SONAR_TOKEN을 생성하고 이를 Repository의 환경변수로 설정해주시면 됩니다!

![](https://velog.velcdn.com/images/chaerim1001/post/015a080d-24e4-4266-8dc3-2bf538afde5a/image.png)


### (7) 결과 확인
이렇게 설정을 마치고 PR을 올리면 SonarCloud와 잘 연동되어 나타나는 것을 확인할 수 있습니다.

![](https://velog.velcdn.com/images/chaerim1001/post/2c20dd73-0720-4863-b041-dc1adb92d19a/image.png)


SonarCloud에서도 PASS라고 뜨는 것을 확인할 수 있습니다!

![](https://velog.velcdn.com/images/chaerim1001/post/6cf29f1b-1661-426b-a02c-e66c365c4c88/image.png)

<br>

> 참고 <br>
> https://gong-check.github.io/dev-blog/BE/%EC%98%A4%EB%A6%AC/sonarcloud/sonarcloud/ <br>
> https://vanslog.io/posts/devops/interate-sonarcloud-with-checkstyle/ <br>
> https://hudi.blog/static-analysis/ <br>
> https://github.com/sonarsource/sonarcloud-github-action <br>
