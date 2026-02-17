---
title: "ThreadPoolExecutor를 이용한 비동기 벤치마크 환경 구축"
description: "동기식 부하 테스트 시 발생하는 클라이언트 사이드 병목 현상을 분석하고, Python ThreadPoolExecutor를 도입하여 실제 운영 환경 수준의 OPS를 검증할 수 있는 고정밀 테스트 환경을 구축한다."
date: 2026-02-17
tags: 
  - D-HASH
  - Python
  - 비동기
  - 측정
  - 트러블 슈팅
series: "D-HASH 알고리즘 개발기"
---

## Blocking I/O로 인한 클라이언트 사이드 병목 현상

D-HASH 알고리즘의 성능 임계치를 검증하기 위해 부하 테스트를 수행하던 중, Redis 서버의 CPU 자원은 충분함에도 불구하고 클라이언트의 **처리량(Throughput)**이 특정 구간에서 정체되는 현상을 확인했다.      

프로파일링 결과, 이는 서버 성능이 부족한 것이 아니라 부하를 생성하는 벤치마크 툴 자체가 느려서 발생하는 **클라이언트 사이드 병목(Client-side Bottleneck)**이었다.       

## RTT가 측정 지표를 왜곡하는 문제

또한 동기식 요청 구조로 인해 네트워크 왕복 시간(RTT)이 측정 결과에 포함되면서, 실제 서버의 처리 성능($Processing Time$)보다 네트워크 지연($Network Latency$)이 지표를 왜곡하는 문제가 발생했다. 이는 $ms$ 단위의 미세한 성능 차이를 비교해야 하는 분산 알고리즘 검증에 부적합하다.      



## Stop-and-Wait 구조의 물리적 한계

문제의 근본 원인은 테스트 클라이언트가 **Stop-and-Wait** 방식의 Blocking I/O로 구현되어 있다는 점이다. 단일 스레드에서 요청을 보내면 응답(ACK)을 받을 때까지 애플리케이션은 블로킹(대기) 상태가 된다.       

> **[Stop-and-Wait Protocol](https://www.google.com/search?q=Stop-and-Wait+Protocol)이란?**     
> 송신자가 패킷을 보낸 후 수신자의 응답을 받을 때까지 다음 패킷을 보내지 않고 대기하는 통신 방식이다. 이 구조에서 최대 처리량(OPS)은 물리적으로 $\frac{1}{RTT + ProcessingTime}$을 넘을 수 없다.        

로컬 환경(Loopback)이라도 OS 네트워크 스택을 경유하는 비용($RTT$)은 반드시 존재한다. 서버의 연산 속도가 아무리 빨라도 클라이언트가 $RTT$만큼 대기해야 하므로, $OPS$(Operations Per Second)는 낮게 측정되고 평균 레이턴시는 실제보다 높게 기록된다.      

## GIL과 I/O Bound 작업의 특성

이 병목을 해소하기 위해 Python의 `concurrent.futures.ThreadPoolExecutor`를 도입하여 **비동기 병렬 처리** 방식으로 아키텍처를 전환했다.      

> **I/O Bound 작업과 GIL 해제**     
> Python의 GIL(Global Interpreter Lock)은 CPU 연산 병렬화를 제한하지만, socket 입출력과 같은 Blocking I/O 대기 중에는 GIL을 해제한다. 따라서 D-HASH의 노드 통신과 같은 I/O Bound 작업에서는 멀티 스레딩을 통해 실질적인 성능 향상을 얻을 수 있다.       

## 노드별 버킷팅과 병렬 실행 구현

구체적인 구현 전략은 다음과 같다:       

1.  **노드별 버킷팅(Bucketing)**: 전체 키(Key) 리스트를 타겟 노드별로 사전 그룹화하여 락 경합(Lock Contention)을 최소화한다.        
2.  **스레드 풀 격리**: 각 노드 버킷에 전담 스레드를 할당하여 다중 연결을 통해 동시에 부하를 주입한다.               
3.  **Max Wall Clock 기준 측정**: 개별 요청 시간의 단순 합산이 아닌, 병렬 작업 중 '가장 늦게 종료된 스레드의 시간'을 기준으로 전체 처리량을 계산하여 동시성 효과를 반영한다.        

~~~python
def benchmark_cluster(keys, sharding, ...):
    # 1. 노드별 데이터 파티셔닝 (Bucketing)
    write_buckets = defaultdict(list)
    # ... (생략)

    # 2. ThreadPoolExecutor를 이용한 병렬 실행
    # I/O 중심 작업이므로 노드 수에 맞춰 워커 스레드 생성
    with ThreadPoolExecutor(max_workers=max(1, len(write_buckets))) as ex:
        # 각 스레드가 독립적으로 I/O 수행
        for total, samples in ex.map(_io_write, write_buckets.items()):
            write_node_totals.append(total)
            write_all_samples.extend(samples)

    # 3. 전체 수행 시간(Wall Clock) 도출
    # 병렬 처리이므로 가장 오래 걸린 스레드의 시간이 전체 소요 시간이 된다.
    cluster_wall = max(write_node_totals) if write_node_totals else 0.0
    throughput = (total_ops / cluster_wall)
~~~

## 비동기 전환을 통한 18만 OPS 달성

테스트 환경 전환 후 벤치마크를 수행한 결과, 기존 동기식 방식에서는 도달하지 못했던 **$160,000 \sim 180,000$ OPS** 수준의 처리량을 안정적으로 기록했다. 이는 실제 운영 환경의 Redis 클러스터가 처리할 수 있는 한계 성능에 근접한 수치다.     

## 순수 서버 성능 측정과 Tail Latency

측정 데이터의 신뢰도 또한 개선되었다. 병렬 처리를 통해 클라이언트 내부의 대기 시간(Queueing Delay)을 제거함으로써, 측정된 레이턴시가 실제 서버의 처리 능력을 반영하도록 했다.       

또한 `_weighted_percentile` 로직을 통해 수집된 샘플의 가중치를 반영하여 P99, P99.9 지표를 산출함으로써, 평균값에 가려진 간헐적 지연(Jitter) 현상을 정확히 포착했다.     

## Python 스레드의 오버헤드와 memtier_benchmark 검토

벤치마크 수행 시 대상 시스템(Target System)뿐만 아니라 측정 도구(Test Harness)의 성능 한계를 파악하는 것이 중요하다. 측정 도구의 성능이 대상 시스템보다 낮다면, 그 결과는 무의미하다.       

> 이번 `ThreadPoolExecutor` 도입은 Python 환경에서 구현할 수 있는 최적의 비동기 모델이지만, 스레드 컨텍스트 스위칭 오버헤드라는 한계가 존재한다. 향후 수백만 OPS 이상의 극한 성능 테스트가 요구될 경우, `memtier_benchmark`와 같은 C/C++ 기반 전용 도구 도입을 검토해야 한다.       

---
**🔗 GitHub Repository:** [bh1848/D-HASH](https://github.com/bh1848/D-HASH)