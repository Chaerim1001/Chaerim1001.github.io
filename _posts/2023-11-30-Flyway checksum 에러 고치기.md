---
title: Flyway checksum 에러 고치기
date: 2023-11-30
categories: [DB]

tags: [Flyway]
---

> 본 포스트는 아래의 환경을 기준으로 작성되었습니다. <br>
> Springboot 3.0.1 <br>
> Java 17 <br>
> MariaDB 11
{: .prompt-info}

## Flyway란?
flyway란 **데이터베이스 형상 관리**를 도와주는 도구입니다.

### springboot 설정
저희 팀은 springboot로 api 서버를 개발하고 있으며 **flyway**를 사용해서 데이터베이스 형상관리를 하고 있습니다.
springboot에서는 아래와 같이 dependency를 추가하고 sql문을 작성해주면 쉽게 적용할 수 있습니다.

![springboot_flyway](https://velog.velcdn.com/images/chaerim1001/post/0cf933bd-a9a6-4a35-b23a-34dead29f552/image.png)

``` 
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-mysql'
```

![](https://velog.velcdn.com/images/chaerim1001/post/961f36af-65b9-4689-8196-d01376087a50/image.png)

라이브러리 추가 후에는 `resources/db/migration` 위치에 sql 파일을 추가해주시면 됩니다.
파일 이름은 `V ${version} __ ${name}.sql`로 지정하면 되고 version 순서대로 관리가 됩니다!

## checksum 에러 발생
이렇게 잘 적용해서 사용하던 중에..
member 테이블을 만들어 사용하던 중 요구사항의 변경으로 인해 테이블의 컬럼을 추가해야하는 상황이 생겼는데요.

그때 데이터베이스와 연동이 안되는 이슈가 발생했습니다. 이런 에러가 발생했네요..

![](https://velog.velcdn.com/images/chaerim1001/post/a1e8ddf0-b4bc-472c-b123-6cf9e65c21db/image.png)

에러의 내용을 보면 `Migration checksum mismatch for migration version` 으로 checksum의 버전이 일치하지 않아서 생기는 문제입니다!

### (1) flyway_schema_history
데이터베이스에 접속하면 자동으로 생성된 `flyway_schema_history` 테이블을 확인할 수 있는데요. flyway는 해당 테이블의 `checksum` 컬럼을 통해 파일 내용을 해싱해서 저장하고 제일 최신의 checksum과 현재 상황을 비교합니다.


![](https://velog.velcdn.com/images/chaerim1001/post/8f889e10-4c19-4cc5-b20d-df0264fbd9b2/image.png)

현재 적용된 DB의 내용과 마지막 checksum으로 저장된 내용이 달라 발생하는 에러였던 것입니다!

### (2) repair()을 통한 해결
`repair()` 을 통해서 checksum을 재설정해주면 문제를 해결할 수 있습니다.

저는 FlywayConfig 클래스를 만들어 마이그레이션 전에 repair()을 통해 어긋난 checksum을 맞춰주는 작업을 수행하였습니다. 

```java
@Configuration
public class FlywayConfig {

    @Bean
    public FlywayMigrationStrategy cleanMigrateStrategy() {
        return flyway -> {
            flyway.repair();
            flyway.migrate();
        };
    }
}
```

<br>
> 참고 <br>
> https://www.baeldung.com/spring-boot-flyway-repair <br>
