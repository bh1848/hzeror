---
title: "@Scheduled 기반 자동 클리닝으로 Ghost Data 제거"
description: "미인증 유령 회원 누적으로 인한 스토리지 낭비와 고유키(PK) 충돌을 방지하기 위해, 트랜잭션 기반의 백그라운드 스케줄러를 도입하여 더티 데이터를 클리닝한 과정을 기술한다."
date: 2026-02-19
tags: 
  - Spring Boot
  - Scheduler
  - 데이터베이스
  - 트러블 슈팅
series: "수챗 개발기"
---

## 미인증 유령 회원(Ghost Data) 누적의 위험성

수챗(Suchat) 프로젝트의 회원가입은 이메일 인증을 완료해야만 정식 회원으로 인정되는 2단계 구조를 갖는다. 인증 전의 데이터는 임시 테이블(`MemberTemp`)에 보관된다. 하지만 가입 도중 브라우저를 닫거나 이탈한 유저의 데이터가 DB에 무한정 적재되는 문제가 있었다.

이 문제는 유저가 이메일 인증을 완료하지 않고 이탈했을 때, 해당 데이터가 시스템의 생명주기 통제에서 벗어나 영구적인 고아 데이터(Orphan Data)로 남기 때문에 발생하는 것이다. 이는 단순한 스토리지 낭비를 넘어, 이탈했던 유저가 다시 동일한 이메일이나 아이디로 가입을 시도할 때 식별자(PK/UK) 충돌을 일으켜 가입이 차단되는 치명적인 UX 결함을 초래한다. 이를 해결하기 위해 생명주기가 끝난 데이터를 서버가 능동적으로 정리하는 백그라운드 클리닝 파이프라인이 필요하다.

> **[Ghost Data (더티 데이터)](https://gemini.google.com/app/b46838d855943e81?hl=ko)**  
> 비즈니스 로직의 결함이나 유저의 비정상적인 이탈로 인해 데이터베이스에 무의미하게 적재되어 시스템 리소스를 갉아먹는 유령 데이터를 뜻한다.

## @Scheduled 기반 자동화 클리닝 파이프라인 구축

인증 유효 시간이 만료된 데이터를 주기적으로 식별하여 하드 딜리트하는 로직을 구현했다. Spring의 `@Scheduled` 어노테이션을 활용하여 매분(`0 * * * * *`) 백그라운드에서 실행되는 배치(Batch) 성격의 스케줄러 서비스를 설계했다.

~~~java
@Service
@RequiredArgsConstructor
@Slf4j
public class EmailAuthSchedulerService {

    // 이메일 토큰 만료 30분 후 임시회원 삭제 유예 기간
    private static final long EMAIL_TOKEN_AFTER_EXPIRATION_TIME_VALUE = 30L;

    private final EmailTokenRepository emailTokenRepository;
    private final MemberTempService memberTempService;

    // 만료 시간(30분 전)이 지났으나 처리되지 않은 토큰 조회
    public List<EmailToken> findExpiredFalse(LocalDateTime localDateTime) {
        return emailTokenRepository.findByExpirationDateBeforeAndExpiredIsFalse(
            localDateTime.minusMinutes(EMAIL_TOKEN_AFTER_EXPIRATION_TIME_VALUE)
        );
    }

    // 유령회원(Ghost Data) 클리닝 파이프라인
    @Transactional
    @Scheduled(cron = "0 * * * * * ")
    public void removeMember() {
        log.info("{} - 이메일 인증을 수행하지 않은 유저 검증 시작", LocalDateTime.now());
        
        List<EmailToken> emailTokens = findExpiredFalse(LocalDateTime.now());

        // 대상 식별 후 반복 삭제 처리
        for (EmailToken emailToken : emailTokens) {
            Long account = emailToken.getMemberTemp().getId();
            memberTempService.deleteFromId(account);
        }
        
        log.info("{} - 이메일 인증을 수행하지 않은 유저 검증 종료", LocalDateTime.now());
    }
}
~~~

토큰 자체의 만료 시간(5분)이 경과한 직후 바로 삭제하지 않고, 추가적인 유예 시간(30분)을 부여(`minusMinutes(30L)`)하여 네트워크 지연이나 재시도 가능성을 열어두는 안전장치를 마련한 것이다.

## 트랜잭션 관리와 N+1 삭제 최적화의 한계

`removeMember` 메서드에 `@Transactional`을 선언하여, 조회된 여러 건의 더티 데이터를 삭제하는 일련의 과정이 하나의 원자적(Atomic) 작업으로 수행되도록 데이터 정합성을 확보했다. 단건이라도 삭제 중 예외가 발생하면 전체 롤백을 수행하여 데이터베이스의 무결성을 유지하기 위해서다.

하지만 현재 로직은 `emailTokens` 리스트를 순회하며 `deleteFromId`를 개별 호출하는 구조를 띠고 있다. 이는 한 번의 `SELECT` 이후 $N$번의 `DELETE` 쿼리가 발생하는 $\mathcal{O}(N)$ 선형 연산이다. 현재의 트래픽 규모에서는 1분 주기로 삭제 대상이 극소수이므로 시스템 병목이 없다고 판단하여 루프 기반의 삭제 로직을 채택했다. 그러나 분당 수천 명의 가입 이탈자가 발생하면 트랜잭션 점유 시간이 길어져 DB 데드락이나 커넥션 풀 고갈을 유발할 수 있다. 규모가 커지면 `DELETE FROM member_temp WHERE id IN (...)` 형태의 벌크(Bulk) 연산으로 쿼리를 최적화할 필요가 있다.

## 분산 서버 환경에서의 스케줄링 충돌 위험성

이 방법으로 DB의 스토리지를 최적화하고 사용자의 재가입 불가 문제를 해결할 수 있었다.

그러나 이 시스템은 Scale-out 된 다중 서버 환경(Multi-Instance)에서 문제가 발생할 수 있다. Spring의 `@Scheduled`는 기본적으로 애플리케이션이 구동되는 로컬 JVM 메모리 내에서 독립적으로 실행된다. 서버가 3대로 증설되면, 매분 0초마다 3대의 서버가 동시에 DB에 접근하여 동일한 유령 회원 데이터를 삭제하려 시도하는 경쟁 상태(Race Condition)가 발생한다.

이는 스케줄러의 제어권이 중앙 집중화되지 않아 다중 노드 간의 실행 동기화가 불가능하기 때문에 발생하는 것이다. 현재는 단일 서버 환경이므로 로컬 스케줄러만으로도 충분한 성능과 안정성을 보장한다. 하지만 서버 인스턴스가 확장되는 시점에는 ShedLock이나 Redis 기반의 분산 락(Distributed Lock)을 도입하여 오직 하나의 노드만 배치 워커(Worker)로 동작하도록 통제하거나, AWS EventBridge와 Lambda를 활용해 애플리케이션 외부로 배치 파이프라인을 완전히 분리할 필요가 있다.