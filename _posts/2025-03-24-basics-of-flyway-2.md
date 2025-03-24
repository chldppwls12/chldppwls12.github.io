---
title: Flyway를 활용한 DB Migration하기(2) - 협업 환경에서의 문제점 및 해결 방안
date: 2025-03-24 13:02:52 +/-TTTT
categories: [DB Migration]
tags: [Flyway]
---


## 들어가기 앞서
<img src="/assets/img/flyway-1/Flyway_logo.png" width="300" alt="Flyway Logo">

지난 글 [Flyway를 활용한 DB Migration하기(1) - 기본 및 간단한 사용 방법](https://chldppwls12.github.io/posts/basics-of-flyway/)에서 Flyway가 무엇인지, 기본 사용법에 대해 알아봤습니다.<br/>
하지만 실제 프로젝트는 혼자 진행하는 것이 아니기 때문에 단순히 기본 방식으로 적용하기는 어렵습니다.<br/>
해당 글에서는 실제 사내에 Flyway를 도입 시, 발생할 수 있는 문제와 이에 따라 어떻게 사용하는 것이 좋을지에 대해 설명하겠습니다.

## 발생할 수 있는 문제

### 1. 배포 순서에 따른 충돌

여러 개발자가 동시에 서로 다른 기능을 개발하는 실제 프로젝트 상황을 고려해보겠습니다.

* 기능 A: V10로 개발 중
* 기능 B: V11로 개발 중

만약 기능 A의 개발이 지연되고, 기능 B가 먼저 출시된다면 문제가 발생합니다. <br/>
V10가 먼저 프로덕션에 배포된 상태에서 나중에 V10를 배포하려고 하면 Flyway는 이미 더 높은 버전이 적용되었다는 오류를 발생시킵니다.
```
Migration V10 has a version older than the current version
```

이런 경우 개발자는 기능 A의 마이그레이션 스크립트를 V10에서 V11으로 변경해야 하는 추가 작업이 필요합니다.

### 2. 버전 충돌

여러 개발자가 동시에 같은 기능을 개발하는 상황을 고려해보겠습니다. <br/>
기능 A를 개발 중인 개발자들은 이전 배포 버전이 V9임을 확인하고, 각각 V10으로 마이그레이션 스크립트를 작성합니다.<br/>
배포 과정에서 동일한 버전 V10의 마이그레이션 파일이 여러 개 존재하게 되어 오류를 발생시킵니다.

```
Found more than one migration with version 10
```

### 3. 환경 별 DB 상태 불일치

개발, 테스트, 스테이징, 프로덕션 환경에서 각각 다른 마이그레이션 상태가 발생할 수 있습니다.<br/>
각 환경마다 배포 시점과 적용된 마이그레이션이 다르면, 환경 간 스키마 불일치가 발생합니다. <br/>
이로 인해 특정 환경에서만 동작하는 코드가 생기거나, 테스트 환경에서 검증된 기능이 프로덕션에서 오작동하는 문제가 발생할 수 있습니다.

## 효과적인 적용 방안
결국 어떻게 마이그레이션 파일을 관리하는 것이 좋을 지 팀 내에서 의논하는 것이 좋습니다.

### 1. 타임스탬프 기반 버전 관리
순차적 버전 번호(V1, V2, V3) 대신 타임스탬프를 사용하여 버전 충돌 문제를 해결할 수 있습니다.
`V{YYYYMMHHmmss}_{Jira 티켓번호}__{설명).sql`

예시

* `V202503241045_JIRA-130_create_user_tables`
* `V202503241046_JIRA-130__add_order_idxs.sql`

각 개발자는 마이그레이션 스크립트 작성 시작 시점의 타임스탬프를 사용하면, Flyway는 이 타임스탬프를 순으로 정렬하여 실행합니다.<br/>
(PR 전에 논리적으로 관련된 변경사항들을 적절히 정리하도록 합니다.)

### 2. 개발 중 반복 기능 마이그레이션 활용
개발 과정에서는 `R__ `접두사를 사용하여 반복 가능한 마이그레이션 스크립트를 활용할 수 있습니다.
```sql
-- 개발 중에는 이 파일을 자유롭게 수정 가능
-- R__dev_user_tables.sql
CREATE TABLE IF NOT EXISTS users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(100),
    email VARCHAR(255)
);
```
PR 전에 최종 버전을 V 접두사를 가진 마이그레이션 파일로 변환합니다.
```sql
-- V202503241045_JIRA-130__create_user_tables.sql
-- 최종 버전으로 고정
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    username VARCHAR(100),
    email VARCHAR(255)
);
```
이렇게 하면 개발 과정에서는 자유롭게 스크립트를 수정하고, 최종 버전만 버전 관리 시스템에 커밋할 수 있습니다.

### 3. 폴더 구조 및 환경 설정
[지마켓 기술 블로그](https://dev.gmarket.com/76)처럼 환경 별 마이그레이션 파일을 분리해서 사용합니다. <br/>
스키마와 데이터 관리를 분리해 사용합니다.
* `migration`: 테이블 구조, 인덱스, 제약조건 등 스키마 관련 변경
* `seed`: 기준 데이터, 코드 테이블, 초기 설정 값 등

```
db/
├── migration/                      # 스키마 변경 스크립트
│   ├── common/                     # 모든 환경 공통 마이그레이션
│   │   └── V*.sql
│   ├── local/                      # 로컬 개발 환경 마이그레이션
│   │   └── V*.sql
│   ├── dev/                        # 개발 서버 환경 마이그레이션
│   │   └── V*.sql
│   ├── test/                       # 테스트 환경 마이그레이션
│   │   └── V*.sql
│   └── prod/                       # 프로덕션 환경 마이그레이션
│       └── V*.sql
│
├── seed/                           # 초기 데이터 스크립트
    ├── local/                      # 로컬 개발 환경 시드 데이터
    │   └── V*.sql
    ├── dev/                        # 개발 서버 시드 데이터
    │   └── V*.sql
    ├── test/                       # 테스트 환경 시드 데이터
    │   └── V*.sql
    └── prod/                       # 프로덕션 시드 데이터
        └── V*.sql
```

각 환경에 맞게 application.yml을 수정해주면 됩니다.

```yml
# 로컬 개발 환경
flyway.locations=db/migration/local/,db/seed/local/

# 개발 서버 환경
flyway.locations=db/migration/dev/,db/seed/dev/

# 테스트 환경
flyway.locations=db/migration/test/,db/seed/test/

# 프로덕션 환경
flyway.locations=db/migration/prod/,db/seed/prod/
```

### CI/CD 설정
- 빌드 전 검증 단계 추가

Jenkins 파이프라인에 Flyway 검증 단계를 추가하여 배포 전 스크립트 오류를 미리 확인할 수 있습니다

```groovy
pipeline {
    agent any
    stages {
        stage('Flyway Validate') {
            steps {
                sh "flyway -configFiles=config/flyway-${env.ENVIRONMENT}.conf validate"
            }
        }
        ...
    }
}
```

- 배포 단계에 통합

데이터베이스 마이그레이션을 애플리케이션 배포 전에 실행하도록 파이프라인을 구성합니다

```groovy
pipeline {
    agent any
    stages {
        stage('Database Migration') {
            steps {
                sh "flyway -configFiles=config/flyway-${env.ENVIRONMENT}.conf migrate"
            }
        }
        stage('Application Deploy') {
            steps {
                sh "./deploy-app.sh"
            }
        }
    }
}
```
이 구성으로 데이터베이스 스키마가 먼저 업데이트된 후 애플리케이션이 배포되어, 스키마 변경과 애플리케이션 코드의 동기화를 보장합니다.

## 결론
Flyway는 강력한 데이터베이스 마이그레이션 도구이지만, 협업 환경에서 효과적으로 사용하기 위해서는 적절한 전략이 필요합니다. <br/>
팀 상황에 맞는 가이드라인을 수립하고 모든 팀원이 이를 일관되게 따르는 것이 중요할 것 같습니다.

---
참고 자료
- [Flyway로 함께 일하기, 버전 충돌 피하기 with Spring](https://medium.com/@132262b/flyway%EB%A1%9C-%ED%95%A8%EA%BB%98-%EC%9D%BC%ED%95%98%EA%B8%B0-%EB%B2%84%EC%A0%84-%EC%B6%A9%EB%8F%8C-%ED%94%BC%ED%95%98%EA%B8%B0-with-spring-b77ea57b06ba)
- [[SpringBoot] 데이터베이스 마이그레이션 툴 Flyway 도입기](https://ywoosang.tistory.com/18)
- [Testcontainers로 통합테스트 만들기](https://dev.gmarket.com/76)
- [[우테코] Flyway를 사용한 데이터베이스 스키마 형상 관리](https://engineerinsight.tistory.com/206)
