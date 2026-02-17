---
title: "JPA ddl-auto를 이용한 스키마 초기화"
description: "반복적인 MySQL 성능 테스트 시 발생하는 Primary Key 충돌 원인을 분석하고, ddl-auto 옵션을 통해 인덱스 파편화 없는 순수(Clean) 테스트 환경을 구축한 과정을 기술한다."
date: 2026-02-09
tags: 
  - MySQL
  - Spring Boot
  - 테스트 환경
  - 트러블 슈팅
series: "MySQL vs Redis 벤치마크"
---

## 반복 실행 시 발생하는 무결성 제약 조건 위반

MySQL과 Redis의 성능 비교 벤치마크를 수행하던 중, 2회 차 실행 시 `java.sql.SQLIntegrityConstraintViolationException`이 발생하며 애플리케이션이 비정상 종료되는 현상이 관측되었다.   

로그에는 **Duplicate entry '1' for key 'PRIMARY'**라는 명확한 에러 메시지가 남았다. 이는 이전 테스트에서 생성된 데이터가 정리되지 않아 다음 테스트의 쓰기 작업을 방해하는 상황이다. 즉, 벤치마크 테스트는 언제, 몇 번을 실행하더라도 동일한 초기 상태와 결과를 보장해야 한다는 **멱등성** 원칙이 깨진 것이다.   

> **[멱등성](https://www.google.com/search?q=Idempotency+in+Computer+Science)이란?**    
> 연산을 여러 번 반복하더라도 결과가 처음 한 번 수행했을 때와 동일하게 유지되는 성질을 의미한다. 벤치마크 환경에서 멱등성 확보는 노이즈 없는 정확한 측정을 위한 필수 전제 조건이다.   

## 결정적(Deterministic) ID 생성 로직의 함정

문제의 원인은 벤치마크 데이터 생성 로직의 특성에 있다. 통제된 실험을 위해 Primary Key(ID)를 DB의 Auto Increment에 맡기지 않고 애플리케이션 레벨에서 직접 할당한다.    

~~~java
// 10,000건의 데이터를 배치 단위로 생성하며 결정적(Deterministic) ID를 부여한다.
// 1회 차: 1 ~ 10,000 생성 / 2회 차: 1 ~ 10,000 생성 시도 -> 충돌
int keyIndex = batch * BATCH_SIZE + i; 
sql = "INSERT INTO test_data (id, name, key_index) VALUES (?, ?, ?)";
~~~

이 로직은 1회 차 실행 시 $1$번부터 $10,000$번까지의 ID를 생성한다. 문제는 DB가 초기화되지 않은 상태에서 2회 차를 실행할 때 발생한다. 애플리케이션은 다시 $1$번 ID부터 생성을 시도하지만, DB에는 이미 $1$번 레코드가 존재하므로 PK 중복 충돌이 발생한다.   

이 현상은 **ID 생성 로직이 결정적이기 때문에, DB 상태 또한 매번 초기화되어야만 정합성이 유지되는 구조**에서 기인한다.   

## DELETE와 Rollback이 벤치마크에 부적합한 이유

이 문제를 해결하기 위해 트랜잭션 롤백이나 명시적 삭제(`DELETE`, `TRUNCATE`)를 고려했으나, 정밀한 벤치마크를 위해 다음과 같은 이유로 배제했다.   

1.  **성능 측정 왜곡 (vs Rollback)**: `@Transactional`을 통한 롤백은 실제 디스크 I/O가 발생하는 커밋(Commit) 단계를 생략하거나, DB 엔진의 버퍼 풀(Buffer Pool) 상태에 영향을 주어 측정 결과에 노이즈를 섞는다.    
2.  **인덱스 파편화 (vs DELETE)**: 단순 `DELETE` 수행 시 MySQL(InnoDB)은 데이터를 즉시 물리적으로 지우지 않고 마킹(Marking)만 수행한다. 이로 인해 B-Tree 인덱스 재정렬 비용이 발생하며, 잔존 데이터가 누적된 상태에서의 벤치마크는 초기 상태(Clean State)보다 불리한 조건을 갖게 된다.    

> **Clean State 유지의 중요성**     
> 벤치마크는 항상 '동일한 출발선'에서 시작해야 한다. 인덱스 파편화(Fragmentation)나 더티 페이지(Dirty Page)가 남아있는 환경은 실험의 독립 변수를 통제하지 못하는 것과 같다.   

## ddl-auto: create 전략을 통한 스키마 재생성

결국 가장 확실한 해결책은 **애플리케이션 구동 시점에 스키마 자체를 재생성(Re-creation)**하는 것이다. Spring Data JPA의 `ddl-auto` 옵션을 사용하여 이를 구현했다.    

`application-mysql.yml` 설정 파일에서 Hibernate의 DDL 전략을 `update`에서 `create`로 수정했다.    

~~~yaml
spring:
  jpa:
    hibernate:
      # SessionFactory 시작 시 기존 테이블 DROP 후 신규 CREATE 실행
      # SchemaExport 도구가 메타데이터를 기반으로 최적의 DDL을 수행한다.
      ddl-auto: create
    open-in-view: false
~~~

`ddl-auto: create` 전략은 하이버네이트의 `SchemaExport` 도구를 통해 기존 테이블을 `DROP`하고, 현재 엔티티 메타데이터에 맞춰 `CREATE` 문을 새로 실행한다. 이를 통해 매 실험마다 데이터베이스는 인덱스 파편화가 전혀 없는 완벽한 $0$건의 상태로 초기화된다.   

## 환경 격리를 통한 운영 리스크 통제

적용 결과, 재실행 시 단 한 번의 PK 충돌 없이 안정적으로 성능을 측정할 수 있었으며, HikariCP Warm-up 과정과 결합하여 신뢰도 높은 데이터를 확보했다.    

다만 `ddl-auto: create`는 운영 환경에서 배포 시 모든 데이터를 삭제하는 치명적인 리스크가 있다. 따라서 본 프로젝트에서는 `--spring.profiles.active=mysql` 옵션을 통해 벤치마크용 프로파일을 분리하여 운영 환경과 엄격히 격리했다. 이러한 환경 분리는 향후 CI 파이프라인에서 자동화된 성능 회귀 테스트(Performance Regression Test)를 구축할 때도 안전한 기반이 된다.   

---
**🔗 GitHub Repository:** [bh1848/mysql-redis-benchmark](https://github.com/bh1848/mysql-redis-benchmark)