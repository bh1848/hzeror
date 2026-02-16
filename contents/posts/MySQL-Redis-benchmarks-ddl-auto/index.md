---
title: "JPA ddl-auto를 이용한 벤치마크 환경 격리와 테스트 멱등성 확보"
description: "반복적인 MySQL 성능 테스트 시 발생하는 Primary Key 충돌 원인을 코드 레벨에서 분석하고, ddl-auto 옵션을 통해 테스트 멱등성을 확보한 과정을 기술한다."
date: 2026-02-09
tags: 
  - MySQL
  - SpringBoot
  - 테스트 환경
  - 해결 기록
series: "MySQL vs Redis 비교 벤치마크"
---

## 두 번째 실행에서 발생한 무결성 제약 조건 위반

MySQL과 Redis의 성능 비교 벤치마크를 수행하던 중, 연속적인 테스트 실행 시 `java.sql.SQLIntegrityConstraintViolationException`이 발생하며 애플리케이션이 비정상 종료되는 현상을 마주했다. 에러 메시지는 **Duplicate entry '1' for key 'PRIMARY'**였으며, 이는 명백히 이전 테스트의 잔존 데이터가 다음 테스트를 방해하고 있음을 의미했다. 벤치마크 테스트는 언제, 몇 번을 실행하더라도 동일한 결과가 보장되어야 한다는 **멱등성(Idempotency)** 원칙이 깨진 것이다.

> **멱등성(Idempotency)이란?**   
> 연산을 여러 번 반복하더라도 결과가 처음 한 번 수행했을 때와 동일하게 유지되는 성질을 의미한다. 벤치마크 환경에서 멱등성 확보는 이전 실험의 결과가 현재 실험에 노이즈를 주지 않도록 보장하는 핵심 요소다.

## 결정적 ID 생성 로직의 함정

문제의 원인은 벤치마크 데이터 생성 로직에 있었다. `MysqlBatchExperiment.java` 구현을 보면, Primary Key(ID)를 DB의 Auto Increment에 맡기지 않고 애플리케이션에서 직접 할당한다.

~~~java
// 10,000건의 데이터를 배치 단위로 생성하며 직접 ID를 부여한다.
int keyIndex = batch * BATCH_SIZE + i; // 1, 2, 3 ... 순차 증가
sql = "INSERT INTO test_data (id, name, key_index) VALUES (?, ?, ?)";
~~~

이 로직은 1회 차 실행 시 $1$번부터 $10,000$번까지의 ID를 생성한다. 문제는 DB가 초기화되지 않은 상태에서 2회 차를 실행할 때 발생한다. 애플리케이션은 다시 $1$번 ID부터 생성을 시도하지만, DB에는 이미 $1$번 레코드가 존재하므로 PK 중복 충돌이 발생한다. 즉, **ID 생성 로직이 결정적(Deterministic)이기 때문에, DB 상태 또한 매번 초기화되어야만 정합성이 유지되는 구조**다.

## 트랜잭션 관리의 한계와 ddl-auto의 채택

이 문제를 해결하기 위해 트랜잭션 롤백과 명시적 삭제 쿼리(TRUNCATE)를 고려했으나 다음과 같은 이유로 배제했다.

1.  **성능 측정 왜곡 (vs Rollback)**: `@Transactional`을 통한 롤백은 실제 디스크 I/O가 발생하는 커밋 단계를 생략하거나, DB 엔진의 버퍼 관리 방식에 영향을 주어 측정 결과에 노이즈를 섞는다.
2.  **인덱스 파편화 (vs DELETE)**: 단순 `DELETE` 수행 시 MySQL은 데이터를 즉시 지우지 않고 마킹(Marking)만 수행하며, 이후 B-Tree 인덱스 재정렬 비용이 발생한다. 잔존 데이터가 누적된 상태에서의 벤치마크는 초기 상태(Clean State)보다 불리한 조건을 가질 수밖에 없다.

결국 가장 담백한 해결책은 **애플리케이션 구동 시점에 스키마 자체를 재생성(Re-creation)**하는 것이다. Spring Data JPA는 `ddl-auto` 옵션을 통해 이를 지원한다.

## ddl-auto create 전략 적용

`application-mysql.yml` 설정 파일에서 Hibernate의 DDL 전략을 수정했다. 기존의 `update` 대신 `create` 옵션을 적용했다.

~~~yaml
spring:
  jpa:
    hibernate:
      # SessionFactory 시작 시 기존 테이블 DROP 후 신규 CREATE 실행
      ddl-auto: create
    open-in-view: false
~~~

`ddl-auto: create`는 단순히 테이블을 비우는 것이 아니라, 하이버네이트의 `SchemaExport` 도구가 현재 엔티티 메타데이터와 DB 방언(Dialect)을 대조하여 최적의 DDL을 새로 실행하는 과정이다. 이를 통해 매 실험마다 데이터베이스는 인덱스 파편화가 없는 완벽한 $0$건의 상태로 초기화된다. 실제 적용 후 재실행 테스트를 진행한 결과, 단 한 번의 PK 충돌 없이 안정적으로 성능을 측정할 수 있었다.

## 환경 격리를 통한 벤치마크 신뢰성 확보

테스트 멱등성 확보는 단순한 오류 해결을 넘어 실험 데이터의 신뢰도로 직결된다. 본 벤치마크는 `ddl-auto`를 통해 초기화된 환경 위에서 **HikariCP 커넥션 풀 초기화 및 JIT 컴파일러 예열을 위한 Warm-up 과정**을 선행 수행하고, `System.currentTimeMillis()`의 오차를 줄이기 위해 배치 단위 평균 역산 방식을 택하여 데이터의 정밀도를 높였다.

다만 `ddl-auto: create`는 운영 환경에서 배포 시 모든 데이터를 삭제하는 치명적인 리스크가 있다. 따라서 본 프로젝트에서는 `--spring.profiles.active=mysql` 옵션을 통해 테스트 환경을 엄격히 격리했다. 이러한 환경 분리는 향후 CI 파이프라인에서 자동화된 성능 회귀 테스트(Performance Regression Test)를 구축할 때도 안전한 기반이 된다.

---
**🔗 GitHub Repository:** [bh1848/mysql-redis-benchmark](https://github.com/bh1848/mysql-redis-benchmark)