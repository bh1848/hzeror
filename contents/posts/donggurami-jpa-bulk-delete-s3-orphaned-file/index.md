---
title: "JPA Bulk Delete 최적화 및 분산 환경의 고아 파일 방지 전략"
description: "JPA Cascade 삭제의 N+1 문제를 Bulk Operation으로 해결하고, S3 고아 파일(Orphaned File) 방지를 위해 의도적인 계층 파괴를 감수한 트레이드오프를 공유한다."
date: 2026-02-17
tags:
  - Java
  - Spring Boot
  - JPA
  - AWS S3
  - 트러블 슈팅
---

## Cascade 삭제 방식의 성능 병목과 대안 모색

'동아리(Club) 삭제' 기능을 설계하며 데이터 정합성과 성능 사이의 간극을 확인했다. 동아리 엔티티 하나를 삭제하려면 그에 종속된 회원, 게시글, 사진, 신청서 등 10여 개의 연관 테이블 데이터가 함께 지워져야 하는 **강한 결합(Tight Coupling)** 구조였기 때문이다.

초기에는 구현의 편의성을 위해 JPA의 `CascadeType.REMOVE`에 의존했으나, 이는 심각한 성능 저하를 야기했다. 이 현상은 JPA가 영속성 컨텍스트 관리를 위해 삭제 대상 전체를 메모리로 먼저 조회(`SELECT`)한 후 단건으로 삭제(`DELETE`)를 수행하는 **N+1 쿼리 폭탄** 메커니즘 때문에 발생하는 것이다. 나는 이 방식이 데이터가 늘어날수록 OOM(Out Of Memory) 위험과 API 레이턴시를 무한정 늘릴 것이라 판단했고, 애플리케이션 메모리를 거치지 않고 DB 엔진에 직접 쿼리를 인가하는 **벌크 연산(Bulk Operation)**으로 최적화하기로 결정했다.

## JPQL Bulk Operation 도입과 쿼리 압축

나는 영속성 컨텍스트의 더티 체킹(Dirty Checking)을 포기하는 대신, 성능을 극대화할 수 있는 **Bulk Delete** 방식을 채택했다. 이를 통해 수백 번 발생하던 삭제 쿼리를 테이블당 단 1회로 압축하여 DB 서버의 부하를 최소화했다.

~~~java
@Repository
@RequiredArgsConstructor
public class ClubRepositoryCustomImpl implements ClubRepositoryCustom {

    @PersistenceContext
    private EntityManager em;
    
    // S3 삭제를 위한 서비스 주입 (의도적 계층 파괴)
    private final S3FileUploadService s3FileUploadService;

    @Override
    public void deleteClubAndDependencies(Long clubId) {
        // 하위 테이블 데이터를 조건절로 일괄 삭제하여 쿼리 발생을 억제한다.
        em.createQuery("DELETE FROM ClubMembers cm WHERE cm.club.clubId = :clubId")
          .setParameter("clubId", clubId)
          .executeUpdate();

        // ... (중략: 해시태그, 신청서 등 타 연관 테이블 삭제)
~~~

## 분산 저장소 간 데이터 무결성과 의도된 계층 위반

관계형 DB와 AWS S3는 서로 다른 시스템이므로 하나의 원자적(Atomic) 트랜잭션으로 묶을 수 없다. 따라서 물리 파일이 스토리지에 영구히 남는 **고아 파일(Orphaned File)** 문제를 차단하기 위해, 벌크 삭제 로직 내부에 S3 삭제 프로세스를 통합했다.

~~~java
        // (ClubRepositoryCustomImpl 내부 연속)
        // 1. 부모 엔티티 삭제 전, 삭제될 이미지들의 S3 Key를 먼저 확보한다.
        List<String> clubIntroPhotoKeys = em.createQuery(
                        "SELECT cip.clubIntroPhotoS3Key FROM ClubIntroPhoto cip WHERE cip.clubIntro.club.clubId = :clubId", String.class)
                .setParameter("clubId", clubId)
                .getResultList();

        // 2. 외부 저장소(S3) 파일 삭제를 선제적으로 요청한다.
        List<String> s3Keys = new ArrayList<>(clubIntroPhotoKeys);
        if (!s3Keys.isEmpty()) {
            s3FileUploadService.deleteFiles(s3Keys);
        }

        // 3. 물리 파일 삭제 요청이 끝난 후 최상위 부모 테이블을 삭제한다.
        em.createQuery("DELETE FROM Club c WHERE c.clubId = :clubId")
                .setParameter("clubId", clubId)
                .executeUpdate();
    }
}
~~~

> **영속성 컨텍스트 불일치(Persistence Context Inconsistency)**  
> 벌크 연산은 1차 캐시를 무시하고 DB에 직접 쿼리를 날리기 때문에, 메모리(캐시)에는 남아있지만 DB에서는 삭제된 상태가 발생할 수 있다. 현재 코드에는 벌크 연산 후 `em.clear()`를 명시하지 않았으나, 동아리 삭제 액션이 트랜잭션의 마지막 사이클이므로 별도의 컨텍스트 초기화 없이 트랜잭션을 종료하도록 설계했다.

## 쿼리 압축 지표 및 아키텍처의 실무적 임계점

최적화 적용 결과, 동아리 삭제 시 발생하는 쿼리 수가 데이터 양에 관계없이 **최소 90% 이상 감소**하는 성과를 확인했다. (기존 $2N+2$ 쿼리 $\rightarrow$ 고정 10 쿼리) 대용량 환경에서 수백 번의 쿼리 왕복(Round-trip)이 단 10회로 압축되면서 레이턴시가 획기적으로 개선되었다.

하지만 나는 이 코드가 가진 명확한 아키텍처적 **안티 패턴(Anti-pattern)과 임계점**을 인지하고 있다. 

데이터 접근을 담당하는 Repository 계층 내부에 외부 API를 호출하는 `S3FileUploadService`가 주입된 것은 **계층형 아키텍처(Layered Architecture)의 관심사 분리 원칙을 위반**한 것이다. 그럼에도 이 방식을 택한 이유는 단일 트랜잭션 범위(Repository Method) 안에서 DB 데이터 삭제와 S3 파일 삭제의 실행 컨텍스트를 최대한 밀결합하여, 개발자가 서비스 단에서 삭제 순서를 실수로 누락하는 휴먼 에러를 막기 위한 의도적인 **트레이드오프**였다.

만약 이 서비스가 MSA(Microservices Architecture)로 전환되거나 트래픽이 고도화된다면 이 방식은 유효하지 않다. 단일 쿼리가 락(Lock)을 오래 점유하여 **Long Transaction**을 유발하고, S3 삭제 지연이 DB 커넥션 풀을 고갈시킬 위험이 있기 때문이다. 향후에는 계층 분리를 원상 복구하고, 메시지 큐(Kafka)와 이벤트 리스너를 도입하여 DB 상태만 변경(Soft Delete)한 뒤 S3 삭제는 Worker 노드에서 비동기로 처리하도록 아키텍처를 수정해야 한다.