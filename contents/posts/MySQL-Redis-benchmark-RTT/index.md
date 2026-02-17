---
title: "네트워크 RTT에 따른 Redis 처리량 병목 분석"
description: "Redis 벤치마크 시 이론적 성능 대비 낮은 OPS가 측정되는 현상을 네트워크 RTT 관점에서 분석하고, 'Client Side Latency'를 지표로 채택한 엔지니어링 의도를 기술한다."
date: 2026-02-11
tags: 
  - Redis
  - 네트워크
  - 측정
  - 해결 기록
series: "MySQL vs Redis 벤치마크"
---

## 낮은 Latency에도 불구하고 제한된 처리량(OPS) 발생

Redis는 단일 인스턴스에서도 초당 10만 회 이상의 연산(OPS)을 처리할 수 있는 고성능 In-Memory DB로 알려져 있다. 하지만 본 프로젝트의 벤치마크 결과, Redis의 응답 속도는 MySQL 대비 압도적으로 빠름에도 불구하고($0.17ms$ vs $1.05ms$), 절대적인 처리량(Throughput) 수치는 기대했던 '수백만 OPS'에 미치지 못했다.

서버의 CPU나 메모리 리소스는 충분한 여유(Idle)가 있었음에도 클라이언트가 더 이상의 부하를 생성해내지 못하는 현상이 관측되었다. 이는 **병목(Bottleneck)이 DB 서버가 아닌 요청자(Client) 구간에 존재하는 상황**이다.

## 동기식(Synchronous) 구조와 Stop-and-Wait 프로토콜

문제의 원인은 벤치마크 클라이언트의 구현 방식인 **Blocking I/O**와 물리적 한계인 **RTT(Round Trip Time)**의 결합에 있다. 코드를 살펴보면, 각 요청은 이전 요청이 완료되어 응답을 받을 때까지 대기(Block)하는 구조다.

~~~java
// 요청 -> 대기 -> 응답수신 -> 다음 요청 (Stop-and-Wait)
for (int i = 0; i < BATCH_SIZE; i++) {
    redisTemplate.opsForValue().get(key); // 동기식 호출, Thread는 응답 시까지 Block된다.
}
~~~

이 구조는 전형적인 **Stop-and-Wait** 프로토콜로 동작하며, 네트워크 선로를 점유하는 시간보다 대기하는 시간이 더 길어지는 비효율을 초래한다.



## 전체 레이턴시 구성 요소와 물리적 한계

이 구조에서 클라이언트가 체감하는 전체 레이턴시($T_{total}$)는 다음과 같이 정의된다.

$$T_{total} = T_{proc} + T_{rtt} + T_{serialize}$$

* $T_{proc}$: Redis 내부에서 데이터를 처리하는 시간 (수 $\mu s$ 수준)
* $T_{rtt}$: 네트워크 패킷이 왕복하는 시간
* $T_{serialize}$: 데이터 직렬화/역직렬화 비용

Redis 내부 처리 시간($T_{proc}$)이 $1\mu s$ 수준으로 극한으로 짧더라도, 데이터가 네트워크를 타고 왕복하는 시간($T_{rtt}$)이 $0.1ms$라면, 클라이언트의 최소 응답 시간은 물리적으로 $0.1ms$ 이하가 될 수 없다.

> **[RTT](https://www.google.com/search?q=Round+Trip+Time)란?**   
> 데이터 패킷이 출발지에서 목적지로 이동하고, 다시 출발지로 돌아오는 데 걸리는 시간을 의미한다. 로컬호스트(Loopback) 환경이라도 OS의 네트워크 스택(TCP/IP Stack)을 거치는 비용이 발생하며, 이는 성능의 물리적 하한선(Lower Bound)이 된다.

## 네트워크 바운드(Network Bound)와 최대 OPS 산출

결국 동기식 클라이언트에서의 최대 OPS는 레이턴시의 역수($\frac{1}{Latency}$)로 수렴하게 된다. 만약 평균 레이턴시가 $0.2ms$라면, 이론상 낼 수 있는 최대 OPS는 초당 $5,000$회($\frac{1}{0.0002}$)에 불과하다.

이 현상은 Redis 서버의 성능 부족이 아니라, 클라이언트가 네트워크 응답을 기다리느라 다음 요청을 보내지 못하는 **'Network Bound'** 상태에 해당한다.

## Pipelining 대신 Client Side Latency를 지표로 선택한 이유

이러한 제약을 극복하고 처리량을 높이려면 **Pipelining(파이프라이닝)** 기술이나 **비동기(Async) I/O**를 도입해야 한다. 응답을 기다리지 않고 명령어를 연속으로 전송하면 RTT의 영향을 상쇄할 수 있기 때문이다.

하지만 본 벤치마크에서는 의도적으로 동기식 방식을 유지하고, 측정 지표를 **'Client Side Latency'**로 명확히 정의했다.

> **동기식을 유지한 이유**  
> 대부분의 웹 애플리케이션 비즈니스 로직은 데이터 조회 후 그 결과를 바탕으로 다음 로직을 수행하는 순차적(Sequential) 흐름을 가진다. 따라서 파이프라이닝을 통한 최대 처리량보다는, **"애플리케이션이 `get()`을 호출했을 때 언제 제어권을 돌려받는가?"**가 더 현실적인 지표다.

또한 DB 내부의 순수 연산 시간뿐만 아니라, 직렬화 비용과 네트워크 I/O 비용을 모두 포함해야 MySQL(Disk I/O 위주)과 Redis(Memory Access 위주)의 실제 체감 성능 차이를 공정하게 비교할 수 있다.

## 서버 성능이 아닌 아키텍처가 결정하는 처리량

이번 실험을 통해 **"Redis가 빠르다는 사실과 시스템 전체가 빠르다는 것은 별개의 문제"**임을 확인했다.

Redis 도입 시 단순히 엔진의 속도만 믿을 것이 아니라, 클라이언트와 서버 간의 거리(Network Topology)와 호출 패턴(Sync vs Async)이 전체 시스템의 성능에 더 큰 영향을 미친다. 향후 대량의 데이터 수집이나 로깅과 같이 '즉각적인 응답 대기가 불필요한' 시나리오에서는 Redis Pipeline을 적용하여 RTT 병목을 해소하는 별도의 최적화가 필요하다.

---
**🔗 GitHub Repository:** [bh1848/mysql-redis-benchmark](https://github.com/bh1848/mysql-redis-benchmark)