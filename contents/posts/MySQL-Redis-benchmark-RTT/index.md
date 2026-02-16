---
title: "동기식 I/O 환경에서 네트워크 RTT가 Redis 처리량에 미치는 영향"
description: "Redis 벤치마크 시 이론적 성능 대비 낮은 OPS가 측정되는 현상을 네트워크 RTT 관점에서 분석하고, 'Client Side Latency'를 지표로 채택한 엔지니어링 의도를 기술한다."
date: 2026-02-11
tags: 
  - Redis
  - 네트워크
  - 측정
  - 해결 기록
series: "MySQL vs Redis 비교 벤치마크"
---

## 이론과 괴리가 있는 낮은 처리량(OPS)

Redis는 단일 인스턴스에서도 초당 10만 회 이상의 연산(OPS: Operations Per Second)을 처리할 수 있는 고성능 In-Memory DB로 알려져 있다. 하지만 본 프로젝트의 벤치마크 결과, Redis의 응답 속도는 MySQL 대비 압도적으로 빠름에도 불구하고($0.17ms$ vs $1.05ms$), 절대적인 처리량(Throughput) 수치는 Redis 공식 문서나 벤치마크 자료에서 보던 '수백만 OPS'에는 미치지 못했다.

서버의 CPU나 메모리 리소스는 충분히 여유가 있었음에도 불구하고 클라이언트가 더 이상의 부하를 생성해내지 못하는 현상, 즉 **병목(Bottleneck)이 DB가 아닌 요청자(Client) 측에 존재하는 상황**이었다.

## 동기식(Synchronous) 구조와 RTT의 지배

문제의 원인은 벤치마크 클라이언트의 구현 방식인 **Blocking I/O**와 물리적 한계인 **RTT(Round Trip Time)**의 결합에 있다. `RedisBatchExperiment.java`의 코드를 살펴보면, 각 요청은 이전 요청이 완료되어 응답을 받을 때까지 대기(Block)하는 구조다.

~~~java
// 요청 -> 대기 -> 응답수신 -> 다음 요청 (Stop-and-Wait)
for (int i = 0; i < BATCH_SIZE; i++) {
    redisTemplate.opsForValue().get(key); // 동기식 호출
}
~~~

이 구조에서 전체 레이턴시($T_{total}$)는 **'DB 처리 시간($T_{proc}$) + 네트워크 왕복 시간($T_{rtt}$)'**으로 구성된다. Redis 내부에서 데이터를 찾는 시간($T_{proc}$)이 $1\mu s$(마이크로초) 수준으로 극한으로 짧더라도, 데이터가 네트워크를 타고 왕복하는 시간($T_{rtt}$)이 $0.1ms$라면, 클라이언트가 체감하는 최소 응답 시간은 $0.1ms$ 이상이 된다.

> **RTT(Round Trip Time)란?**    
> 데이터 패킷이 출발지에서 목적지로 이동하고, 다시 출발지로 돌아오는 데 걸리는 시간을 의미한다. 로컬호스트(Loopback) 환경이라도 OS의 네트워크 스택을 거치는 비용이 발생하며, 실제 클라우드 환경에서는 이 시간이 성능의 물리적 상한선이 된다.

결국 동기식 클라이언트에서의 최대 OPS는 $\frac{1}{Latency}$로 수렴하게 된다. $Latency$가 $0.2ms$라면, 이론상 낼 수 있는 최대 OPS는 초당 5,000회($\frac{1}{0.0002}$)에 불과하다. Redis 서버가 놀고 있어도 클라이언트는 네트워크 응답을 기다리느라 다음 요청을 보내지 못하는 **'Network Bound'** 상태인 것이다.

## Client Side Latency

이러한 현상을 해결하고 처리량(Throughput)을 극한으로 높이려면 **Pipelining(파이프라이닝)** 기술이나 **비동기(Async) I/O**를 사용해야 한다. 응답을 기다리지 않고 명령어를 쏟아부으면 RTT의 영향을 상쇄할 수 있기 때문이다.

하지만 본 벤치마크에서는 의도적으로 동기식 방식을 유지하고, 측정 지표를 **'Client Side Latency'**로 명확히 정의했다. 그 이유는 다음과 같다.

1.  **실무적 정합성**: 대부분의 웹 애플리케이션 비즈니스 로직은 데이터 조회 후 그 결과를 바탕으로 다음 로직을 수행하는 순차적(Sequential) 흐름을 가진다. 따라서 파이프라이닝을 통한 최대 처리량보다는, **"애플리케이션이 `get()`을 호출했을 때 언제 응답을 받는가?"**가 더 현실적인 지표다.
2.  **전체 비용의 가시화**: DB 내부의 순수 연산 시간뿐만 아니라, 직렬화/역직렬화(Serialization) 비용과 네트워크 I/O 비용을 모두 포함한 시간을 측정해야 MySQL(Disk I/O 위주)과 Redis(Memory Access 위주)의 실제 체감 성능 차이를 공정하게 비교할 수 있다.

## 서버 성능이 아닌, 아키텍처의 성능

이번 실험을 통해 **"Redis가 느린 것이 아니라, 네트워크를 타는 방식이 성능을 결정한다"**는 사실을 재확인했다. Redis 도입 시 단순히 "빠르다"는 사실만 믿을 것이 아니라, 클라이언트와 서버 간의 거리(Network Topology)와 호출 패턴(Sync vs Async)이 전체 시스템의 성능에 더 큰 영향을 미칠 수 있음을 인지해야 한다.

향후 대량의 데이터 수집이나 로깅과 같이 '응답 대기가 불필요한' 시나리오에서는 Redis Pipeline을 적용하여 RTT 병목을 해소하고 처리량을 극대화하는 별도의 벤치마크를 수행할 가치가 있다.

---
**🔗 GitHub Repository:** [bh1848/mysql-redis-benchmark](https://github.com/bh1848/mysql-redis-benchmark)