---
title: Flyway를 활용한 DB Migration하기 (1) - 정의 및 간단한 사용 방법
date: 2025-03-21 13:53:52 +/-TTTT
categories: [DB Migration]
tags: [Flyway]
---


## 들어가기 앞서
일반적인 소프트웨어 배포 환경은 로컬 → 통합 환경 → 테스트 환경 → 운영 환경으로 단계적으로 구성됩니다. <br/>
사내에서는 소스 코드의 경우 Git과 같은 형상 관리 도구를 통해 변경 이력을 관리해왔지만, DB의 경우 별도의 형상 관리 없이 각 환경마다 동일한 작업을 수동으로 반복 진행해왔습니다. <br/>
이럴 경우 누락이나 오류로 인한 장애 발생 위험, 변경 사항 파악이 어렵고, 그리고 문제 발생 시 이전 상태로 롤백하기 어렵습니다. <br/><br/>
이런 문제를 해결하기 위해 DB 형상 관리를 위한 툴로 `Flyway`나 `Liquibase`가 있습니다.<br/>
이 글에서는 `Flyway`의 동작 방식 및 사용 방법에 대해 설명합니다.<br.>

## Flyway란?
<img src="/assets/img/flyway-1/Flyway_logo.png" width="300" alt="Flyway Logo">

공식문서 바로가기 -> [Flyway 공식 문서](https://documentation.red-gate.com/flyway) <br/>
데이터베이스 마이그레이션 툴로, 데이터베이스의 변경 사항 추적, 업데이트 롤백을 보다 쉽게 도와줍니다.

- SQL / Java로 마이그레이션 스크립트 작성 가능
- 다양한 데이터베이스 지원 (PostgreSQL, MySQL, Oracle 등)
- Java API, Maven, Gradle 플러그인 제공

## 사용 시 이점
- **일관성 있는 환경 구성**
  - 모든 환경(개발, 테스트, 운영)에서 동일한 데이터베이스 스키마를 보장합니다.
- **변경 이력 추적**
  - 언제, 어떤 변경이 적용되었는지 명확하게 기록합니다.
- **자동화된 배포**
  - 수동 작업 없이 스크립트 기반으로 DB 변경 사항 적용합니다.
- **롤백 관리**
  - 버전 관리를 통해 이전 상태로의 복원 가능성 제공합니다.

## 동작 방식

1. DB가 비어있을 경우, `flyway_schema_history`을 생성합니다. 이 테이블은 DB 상태를 추적하는 역할을 합니다.
2. SQL 또는 Java로 작성된 마이그레이션 파일을 버전 번호 순서대로 실행합니다. <br/>
마이그레이션이 완료되면 스키마 히스토리 테이블에 버전 정보, 체크섬, 성공 여부 등을 기록합니다. <br/>
초기 상태 완료 후 버전 정보가 저장되며, 다음 마이그레이션 시에는 이미 적용된 이전 버전의 스크립트는 무시합니다.

## 버전 관리
Flyway는 마이그레이션 스크립트의 버전을 파일명을 통해 관리합니다. 기본 경로인 `/resources/db/migration`에 스키마를 정의하는 네이밍 규칙에 따라 SQL 파일을 생성해야 합니다.

<img src="/assets/img/flyway-1/naming.png" alt="Naming">

### V(Versioned) Migration
`V{버전 번호}__{설명}.sql`

- 순차적인 스키마 변경 (테이블 생성/변경, 컬럼 추가 등)
- 버전 번호 순서대로 실행
- 예시
  - `V1__create_user_table.sql`
  - `V2__add_email_column.sql`
  - `V2.1__add_indexes.sql`

### R (Repeatable) 마이그레이션
`R__{설명}.sql`

- 자주 변경되는 뷰, 함수, 프로시저, 트리거 등 관리
- 모든 V 마이그레이션 이후에 실행됨
- 파일 내용의 체크섬이 변경되면 재실행
- 항상 최신 버전만 유지됨
- 예시
- `R__Create_views.sql`
- `R__Create_stored_procedures.sql`
  
### U (Undo) 마이그레이션 (Teams 버전 전용)
`U{버전 번호}__{설명}.sql`

- 특정 V 마이그레이션을 취소
- flyway undo 명령으로 실행 (Teams 버전에서만 지원)
- 무료 버전에서 롤백이 필요하면 새 V 마이그레이션으로 변경 취소를 구현해야 함
- 예시
  - `U2__undo_add_email_column.sql` (V2 마이그레이션 취소)

## 실제 사용해보기

### 플러그인 & 의존성 추가

Gradle에서 수동 마이그레이션을 위한 플러그인과 의존성을 다음과 같이 추가합니다.

참고) PostgreSQL & Flyway 10.x 버전 이상과 함께 사용할 때 의존성 문제가 발생합니다. 깃헙에 관련 이슈도 많이 올라와있습니다.

- [Flyway 10 Gradle Plugin: No database found to handle jdbc:postgresql://... #3774](https://github.com/flyway/flyway/issues/3774) <br/>
- [No database found to handle ... [db connection url] (even after specialized dependency inclusion) #3998](https://github.com/flyway/flyway/issues/3998)

comment처럼 일단은 buildScript 추가 및 관련 버전을 다 동일하게 맞춰서 해결해줍니다.

```gradle
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.flywaydb:flyway-database-postgresql:11.4.0")
    }
}

plugins {
    // 기존 플러그인들
    id 'org.flywaydb.flyway' version '11.4.0'
}

dependencies {
    // 기존 의존성들
    
    // Flyway 의존성
    implementation("org.flywaydb:flyway-core:10.5.0") 
    implementation("org.flywaydb:flyway-database-postgresql:11.4.0")
}

// Flyway 설정 추가
flyway {
    url = 'jdbc:postgresql://localhost:5432/mytestdb'
    user = 'postgres'
    password = 'admin'
    schemas = ['test']
    locations = ['filesystem:src/main/resources/db/migration']
    baselineOnMigrate = true
}
```
{: file="build.gradle"}

### application.yml 설정

Spring Boot를 사용한 자동 마이그레이션을 위해 yml 파일도 다음과 같이 설정합니다:
```yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mytestdb?currentSchema=test
    username: postgres
    password: admin
    driver-class-name: org.postgresql.Driver

  # Flyway 설정 추가
  flyway:
    schemas: test
    baseline-on-migrate: true # 기존 DB에 적용 시 사용
    baseline-version: 1
    locations: classpath:db/migration
    enabled: false # 애플리케이션 시작 시 자동 마이그레이션 비활성화
```
{: file="Application.yml"}

- `baseline-on-migrate`: 이미 존재하는 데이터베이스에 처음으로 Flyway를 적용할 때, true로 설정하면 Flyway는 현재 DB 상태를 baseline-version으로 표시하고 이후 버전의 마이그레이션만 적용합니다.

### 스키마 등록 및 적용
간단하게  `V1__init.sql` 파일을 작성합니다. 해당 파일은 `resources/db/migration`에 네이밍 규칙에 맞게 작성되어야합니다.

```sql
CREATE TABLE person (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    age INT
);
```
{: file="/src/resources/db/migration/V1__init.sql"}

마이그레이션을 적용하는 방법은 두 가지입니다
- Gradle 명령어를 통한 수동 마이그레이션: `./gradlew flywayMigrate`
- Spring Boot 애플리케이션 시작 시 자동 마이그레이션 (application.yml에서 enabled: true로 설정한 경우)

마이그레이션이 성공적으로 실행되면, Flyway는 `flyway_schema_history`에 실행 이력을 기록합니다. 각 마이그레이션 버전의 실행 시간, 상태, 체크섬 등을 확인할 수 있습니다.

<img src="/assets/img/flyway-1/flyway_schema_history.png" alt="Flyway Schema History">


### 관련 명령어
- 마이그레이션 실행

```shell
./gradlew flywayMigrate
```

- 마이그레이션 정보 확인

```shell
./gradlew flywayInfo
```
<img src="/assets/img/flyway-1/flyway_info.png" alt="Flyway Info">

- 마이그레이션 체크섬 검증

적용된 마이그레이션 스크립트의 체크섬을 검증하여 스크립트가 변경되지 않았는지 확인합니다. 변경된 경우 오류를 발생시킵니다.
```shell
./gradlew flywayValidate
```

- 마이그레이션 수동 수리

오류가 발생한 마이그레이션을 수리합니다. 실패한 마이그레이션 항목을 flyway_schema_history 테이블에서 제거하고, 불완전한 마이그레이션으로 인한 잠금 상태를 해제합니다.
```shell
./gradlew flywayRepair 
```


---
참고 자료

- [Flyway 정의, DB 마이그레이션 도구](https://velog.io/@hyun-jii/Flyway-%EC%A0%95%EC%9D%98-DB-%EB%A7%88%EC%9D%B4%EA%B7%B8%EB%A0%88%EC%9D%B4%EC%85%98-%EB%8F%84%EA%B5%AC)
- [[우테코] Flyway를 사용한 데이터베이스 스키마 형상 관리](https://engineerinsight.tistory.com/206)
- [Flyway를 통한 Postgres 형상관리 적용](https://migo-dev.tistory.com/entry/Flyway%EB%A5%BC-%ED%86%B5%ED%95%9C-Postgres-%ED%98%95%EC%83%81%EA%B4%80%EB%A6%AC-%EC%A0%81%EC%9A%A9)
