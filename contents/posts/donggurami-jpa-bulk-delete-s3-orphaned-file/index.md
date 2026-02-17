---
title: "JPA Bulk Delete 최적화 및 S3 고아 파일 방지 전략"
description: "JPA Cascade 삭제의 N+1 문제를 Bulk Operation으로 해결하고, S3 고아 파일(Orphaned File) 발생을 방지하는 동기화 로직을 구현한다."
date: 2026-02-17
tags:
  - Java
  - Spring Boot
  - JPA
  - AWS S3
  - 트러블 슈팅
---

## Cascade 삭제의 함정과 스토리지 비용 누수

'동아리(Club) 삭제' 기능은 단순히 `clubs` 테이블의 row 하나를 지우는 작업이 아니다. 동아리에 종속된 회원, 게시글, 사진, 신청서 등 10여 개의 연관 테이블 데이터가 함께 삭제되어야 하는 **강한 결합(Tight Coupling)** 구조를 가진다.      

초기 구현에서는 JPA의 `CascadeType.REMOVE` 편의 기능에 의존했다. 하지만 이는 심각한 성능 저하와 운영 비용 문제를 야기했다.      

1.  **N+1 쿼리 폭탄**: JPA는 영속성 컨텍스트 관리를 위해 삭제 대상 엔티티를 모두 `SELECT` 한 후 단건으로 `DELETE` 쿼리를 날린다. 동아리원이 100명이라면 동아리 삭제 한 번에 최소 101번의 쿼리가 발생한다.   
2.  **S3 고아 파일(Orphaned File)**: DB의 메타데이터(이미지 경로)는 트랜잭션 롤백으로 복구가 되지만, 외부 저장소인 AWS S3에 요청한 파일 삭제는 롤백되지 않는다. 로직 순서가 잘못되면 DB 데이터는 지워졌는데 S3에는 파일이 남는 데이터 불일치가 발생하여 스토리지 비용이 낭비된다.   

이러한 비효율을 제거하기 위해 **JPQL 기반의 Bulk Delete**와 **선제적 S3 키 조회 전략**을 도입했다.      

## JPQL Bulk Operation을 통한 쿼리 압축

JPA의 Dirty Checking이나 Cascade 삭제는 대량의 데이터를 지울 때 적합하지 않다. 영속성 컨텍스트를 거치지 않고 DB에 직접 `DELETE` 문을 날리는 벌크 연산(Bulk Operation)이 필요하다.

### ClubRepositoryCustomImpl 구현

테이블별로 단 한 번의 쿼리만 실행되도록 최적화하여 DB 서버의 부하를 최소화했다.

~~~java
@Repository
@RequiredArgsConstructor
public class ClubRepositoryCustomImpl implements ClubRepositoryCustom {

    private final EntityManager em;

    @Override
    public void deleteClubAndDependencies(Long clubId) {
        // 1. 자식 테이블 데이터 일괄 삭제 (회원, 지원서 등)
        em.createQuery("DELETE FROM ClubMember cm WHERE cm.club.id = :clubId")
          .setParameter("clubId", clubId)
          .executeUpdate();

        em.createQuery("DELETE FROM Aplict a WHERE a.club.id = :clubId")
          .setParameter("clubId", clubId)
          .executeUpdate();

        // 2. 부모 테이블(Club) 삭제
        em.createQuery("DELETE FROM Club c WHERE c.id = :clubId")
          .setParameter("clubId", clubId)
          .executeUpdate();
        
        // 3. 영속성 컨텍스트 초기화
        em.clear();
    }
}
~~~

> **[Phantom Read](https://gemini.google.com/app/b46838d855943e81?hl=ko)란?**   
> 한 트랜잭션 내에서 같은 쿼리를 두 번 실행했을 때, 첫 번째 쿼리에서 없던 레코드가 두 번째 쿼리에서 나타나는 현상이다. 본문에서는 벌크 연산 후 영속성 컨텍스트를 비우지 않으면 이미 삭제된 데이터가 1차 캐시에 남아 조회되는 논리적 불일치를 경고하기 위해 사용했다.

## 분산 저장소 간 데이터 정합성 확보 전략

DB 트랜잭션과 S3 API 호출은 하나의 원자적(Atomic) 트랜잭션으로 묶을 수 없다. 따라서 운영 비용 절감을 위해 **고아 파일을 남기지 않는 선제적 삭제 전략**을 채택했다.

### 선제적 Key 조회 및 삭제 프로세스

무작정 삭제하는 것이 아니라, 삭제 대상 파일의 S3 Key(경로)를 먼저 확보하는 것이 핵심이다.

~~~java
@Transactional
public void deleteClub(Long clubId) {
    // 1. 삭제될 이미지들의 S3 Key를 먼저 조회 (SELECT)
    List<String> s3Keys = clubRepository.findImageKeysByClubId(clubId);

    // 2. 외부 스토리지(S3) 파일 삭제 요청
    if (!s3Keys.isEmpty()) {
        s3FileUploadService.deleteFiles(s3Keys);
    }

    // 3. DB 데이터 벌크 삭제 (Bulk Delete)
    clubRepository.deleteClubAndDependencies(clubId);
}
~~~

> **분산 트랜잭션의 한계와 트레이드오프**   
> S3 파일 삭제 후 DB 삭제를 진행할 때, S3 삭제는 성공했지만 DB 삭제 중 예외가 발생하면 클라이언트는 '존재하지 않는 이미지'를 참조하게 된다. 하지만 이는 스토리지 누수로 인한 지속적인 비용 발생보다 관리적 측면에서 비용이 저렴하다. 실무에서는 이러한 데이터 불일치를 해결하기 위해 별도의 배치(Batch) 프로그램을 돌려 S3와 DB를 정기적으로 동기화하는 방식을 병행하기도 한다.

## 쿼리 압축을 통한 Latency 개선 및 공학적 근거

최적화 적용 결과, 동아리 삭제 시 발생하는 쿼리 수가 데이터 양에 관계없이 **최소 90% 이상 감소**하는 구조적 이득을 확인했다.

* **구조적 성능 비교 (연관 테이블 10개, 자식 데이터 $N$개 기준)**:      
    * **개선 전 (Cascade)**: $2N + 2$ Queries (데이터가 늘어날수록 쿼리 수 폭발)    
    * **개선 후 (Bulk Delete)**: 고정 10 Queries (데이터 양에 독립적인 성능 유지)   

**이러한 수치가 가능한 근거는 다음과 같다.**     
첫째, JPA의 기본 삭제 방식은 영속성 컨텍스트의 상태 관리를 위해 삭제 대상 전체를 먼저 조회(`SELECT`)한 후 단건 삭제를 수행하지만, **벌크 연산은 DB 엔진에서 직접 조건절 삭제를 수행**하여 중간 단계를 생략하기 때문이다. 

둘째, 네트워크 레이턴시 측면에서 수백 번의 쿼리 왕복(Round-trip)이 단 10회로 압축되면서 통신 오버헤드가 제거되었다. 이를 통해 대용량 데이터 환경에서도 일정한 API 성능을 보장하는 **예측 가능한 시스템**을 구축했다.

불필요한 네트워크 왕복(Round-trip)이 제거되면서 **레이턴시(Latency)**가 획기적으로 개선되었다. 대용량 데이터 환경에서 ORM의 추상화된 편의성보다 데이터베이스 메커니즘을 고려한 명확한 SQL 작성이 시스템 안정성에 더 큰 기여를 한다는 사실을 확인했다.

---
**🔗 GitHub Repository:** [bh1848/USW-Circle-Link-Server](https://github.com/bh1848/USW-Circle-Link-Server)