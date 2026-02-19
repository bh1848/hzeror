---
title: "Redis ZSet과 @Async를 활용한 비동기 분산 매칭 엔진 아키텍처 설계"
description: "다중 서버 환경의 매칭 대기열 병목과 동기 처리의 레이턴시 한계를 극복하기 위해, ZSet 분산 큐와 비동기 연산 격리 아키텍처를 구축한 과정을 기술한다."
date: 2026-02-19
tags: 
  - Redis
  - Spring Boot
  - 비동기
  - 아키텍처
  - 트러블 슈팅
series: "수챗 개발기"
---

## 다중 서버 환경과 인메모리 큐의 물리적 한계

수챗(Suchat) 프로젝트에서 1:1 무작위 채팅 기능을 구현하며, 초기에는 서버 메모리(JVM) 기반의 자료구조(Queue)를 매칭 대기열로 고려했다. 그러나 서버를 다중 인스턴스(Scale-out)로 확장하는 구조에서는 유저 간의 매칭 상태를 공유할 수 없는 치명적인 결함이 있다.

이 현상은 JVM 메모리에 종속된 로컬 자료구조가 분산 서버 인스턴스 간의 상태(State)를 물리적으로 공유하지 못하기 때문에 발생하는 것이다. 매칭 대기열을 서버 인스턴스와 완벽히 분리하기 위해 외부 인메모리 저장소인 Redis 도입이 필요하다.

## Redis List의 선형 탐색 병목과 ZSet 채택

Redis를 매칭 큐로 사용할 때 가장 먼저 고려되는 자료구조는 `List`다. 하지만 실무 환경에서는 매칭을 기다리던 유저가 도중에 이탈하는 '취소(Cancel)' 변수를 빈번하게 봤다.

Redis `List` 구조에서 특정 유저의 데이터를 찾아 삭제하는 연산(`LREM`)의 시간 복잡도는 $\mathcal{O}(N)$이다. 트래픽 밀집 시간에 대기열이 수만 명으로 늘어난다면, 단순한 취소 요청 하나가 Redis의 싱글 스레드를 블로킹(Blocking)하여 전체 시스템의 레이턴시를 치솟게 만들 수 있다.

> **[Redis ZSet (Sorted Set)](https://gemini.google.com/app/b46838d855943e81?hl=ko)**  
> 요소(Value)마다 가중치(Score)를 부여하여 자동으로 정렬을 유지하는 자료구조다. 탐색과 삭제 연산에 $\mathcal{O}(\log N)$의 성능을 보장한다.

이 병목을 해소하기 위해 **ZSet** 자료구조를 도입했다. 유저 식별자(`account`)를 Value로 두고, 큐 진입 시간(`System.currentTimeMillis()`)을 Score로 설정하여 FIFO 정렬을 강제하고 취소 연산 비용을 낮추기 위해서다.

~~~java
@Service
@Transactional
@RequiredArgsConstructor
public class RoomSecureService {
    private static final String MATCH_QUEUE = "MatchQueue"; // 단일 매칭 큐 Key
    private final RedisTemplate<String, String> matchRedisTemplate;
    private ZSetOperations<String, String> matchQueue;

    @PostConstruct
    public void init() {
        matchQueue = matchRedisTemplate.opsForZSet();
    }

    // 매칭 참가: 시간(ms)을 Score로 사용하여 자동 오름차순(FIFO) 정렬 보장
    // 시간 복잡도: O(log N)
    public void addToMatchingQueue(String account) {
        matchQueue.add(MATCH_QUEUE, account, System.currentTimeMillis());
        // ... 웹소켓 알림 전송 로직
        performMatchingAsync(account);
    }

    // 매칭 취소: List(O(N)) 대비 압도적으로 빠른 O(log N) 연산
    public void removeCancelParticipants(String account) {
        matchQueue.remove(MATCH_QUEUE, account);
        // ... 웹소켓 알림 전송 로직
    }
}
~~~

## 동기식 매칭 로직의 스레드 점유 지연

ZSet을 통해 큐 자체의 자료구조적 병목은 해소했다. 하지만 큐에 유저를 넣은 직후 실제 매칭을 성사시키는 `performMatching` 로직에서 또 다른 문제가 있었다.

클라이언트 요청을 받은 톰캣(Tomcat) 워커 스레드가 큐 삽입 후 매칭 연산(조회, UUID 생성, DB 업데이트)까지 모두 동기적으로 수행한 뒤에야 제어권을 반환하는 구조였다.

이 현상은 요청 접수(I/O Bound)와 매칭 알고리즘 처리(CPU/DB Bound)가 단일 스레드 내에 강하게 결합되어 유휴 스레드 풀의 고갈을 유발하기 때문에 발생하는 것이다. 결국 트래픽이 몰리면 매칭 성공 여부와 상관없이 시스템 전체의 응답 속도가 떨어지는 병목이 생길 수밖에 없다.

## @Async와 CompletableFuture를 통한 비동기 연산 격리(Isolation)

이 문제를 해결하기 위해 비동기 기반의 **연산 격리(Isolation)**가 필요하다. 클라이언트의 큐 진입 요청 즉시 워커 스레드를 반환하여 레이턴시를 최소화하고, 무거운 매칭 로직은 별도의 스레드 풀로 오프로딩(Off-loading)하기 위해서다.

나는 `@Async`와 `CompletableFuture`를 결합하여 메인 스레드의 블로킹 없는 매칭 엔진을 구현했다.

~~~java
    // @Async를 통한 별도 스레드 풀 오프로딩 (Non-blocking)
    @Async
    public CompletableFuture<String> performMatchingAsync(String account) {
        CompletableFuture<String> future = new CompletableFuture<>();

        // 큐에 2명 이상 존재할 경우 매칭 성사
        if (matchQueue.size(MATCH_QUEUE) > 1) {
            // ZSet은 Score 오름차순이므로 0번째, 1번째가 가장 오래 대기한 유저 (FIFO)
            String participant1 = Objects.requireNonNull(matchQueue.range(MATCH_QUEUE, 0, 0)).iterator().next();
            String participant2 = Objects.requireNonNull(matchQueue.range(MATCH_QUEUE, 1, 1)).iterator().next();
            String chatRoomId = UUID.randomUUID().toString();

            // DB 업데이트 및 알림 비동기 처리
            updateMemberRoomId(participant1, chatRoomId);
            updateMemberRoomId(participant2, chatRoomId);
            sendMatchingNotification(participant1, chatRoomId, participant2);
            sendMatchingNotification(participant2, chatRoomId, participant1);

            // 매칭된 유저를 큐에서 제거 (O(log N))
            matchQueue.remove(MATCH_QUEUE, participant1);
            matchQueue.remove(MATCH_QUEUE, participant2);

            future.complete(chatRoomId);
        } else {
            // 매칭 실패 시 비동기 예외 처리
            future.completeExceptionally(new IllegalStateException("매칭 큐에 충분한 참가자가 없습니다."));
        }

        return future;
    }
~~~

## 동시성 제어와 트레이드오프

이것으로 매칭 큐의 취소 탐색 복잡도를 $\mathcal{O}(N)$에서 $\mathcal{O}(\log N)$으로 압축하고, 워커 스레드의 레이턴시 지연을 원천 차단하는 논블로킹 엔진을 완성했다.

하지만 이 시스템은 극단적인 분산 환경에서 문제점이 있다. 현재 로직은 `matchQueue.range`로 참가자를 조회하고 `matchQueue.remove`로 삭제하기까지 미세한 시간차가 있다. 다수의 서버 인스턴스가 동시에 `performMatchingAsync`를 호출할 경우 경쟁 상태(Race Condition)가 발생할 위험이 있다.

현재 트래픽 수준에서는 분산 락(Distributed Lock)의 도입이 오히려 시스템 성능을 저해한다고 판단하여 조회 후 삭제 구조를 채택했다. 만약, 동접자가 폭발적으로 증가하면 Redis Lua Script 기반의 원자적(Atomic) 구조로 매칭 엔진을 업그레이드할 필요가 있다.