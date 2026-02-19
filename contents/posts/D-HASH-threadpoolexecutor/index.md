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

D-HASH 알고리즘의 성능 임계치를 검증하기 위해 부하 테스트를 수행하던 중, Redis 서버의 자원은 널널함에도 불구하고 클라이언트의 **처리량(Throughput)**이 특정 구간에서 정체되는 현상을 확인했다.

프로파일링 결과, 이 현상은 서버 성능이 부족해서가 아니라 부하를 생성하는 벤치마크 툴 자체가 느려서 발생하는 **클라이언트 사이드 병목(Client-side Bottleneck)** 때문에 발생하는 것이다.

## Stop-and-Wait 구조의 물리적 한계

문제의 근본 원인은 내가 작성한 초기 테스트 클라이언트가 **Stop-and-Wait** 방식의 단일 스레드 동기식(Blocking I/O)으로 구현되어 있다는 점이다.

> **[Stop-and-Wait Protocol](https://www.google.com/search?q=Stop-and-Wait+Protocol)**  
> 송신자가 요청을 보낸 후 응답(ACK)을 받을 때까지 대기하는 구조다. 이 구조에서 최대 처리량(OPS)은 물리적으로 $\frac{1}{RTT + ProcessingTime}$을 넘을 수 없으므로, 네트워크 왕복 시간(RTT)이 실제 서버의 처리 성능 측정값을 왜곡하게 된다.

## GIL과 I/O Bound 작업의 특성 활용

나는 이 병목을 해소하기 위해 Python의 `concurrent.futures.ThreadPoolExecutor`를 도입하여 아키텍처를 **비동기 병렬 처리** 방식으로 전면 개편했다.

Python의 GIL(Global Interpreter Lock)은 CPU 연산 병렬화를 제한하지만, Socket 통신과 같은 Blocking I/O 대기 중에는 GIL을 해제한다. 따라서 노드 통신과 같은 I/O Bound 작업에서는 멀티 스레딩을 통해 완벽한 병렬 부하 주입이 가능하다고 판단했다.

~~~python
def benchmark_cluster(keys, sharding, ...):
    # 1. 노드별 데이터 파티셔닝 (Bucketing)
    # 락 경합(Lock Contention)을 막기 위해 타겟 노드별로 키를 사전 그룹화한다.
    write_buckets = defaultdict(list)
    # ... (중략)

    # 2. ThreadPoolExecutor를 이용한 병렬 실행
    # 각 노드 버킷에 전담 스레드를 할당하여 독립적으로 I/O를 수행한다.
    with ThreadPoolExecutor(max_workers=max(1, len(write_buckets))) as ex:
        for total, samples in ex.map(_io_write, write_buckets.items()):
            write_node_totals.append(total)
            write_all_samples.extend(samples)

    # 3. 전체 수행 시간(Wall Clock) 도출
    # 병렬 처리이므로 '가장 늦게 종료된 스레드'의 시간이 전체 소요 시간이 된다.
    cluster_wall = max(write_node_totals) if write_node_totals else 0.0
    throughput = (total_ops / cluster_wall)
~~~

## 18만 OPS 달성 및 테스트 환경의 한계 인지

테스트 환경 전환 후 벤치마크를 수행한 결과, 기존 방식에서는 도달하지 못했던 **$160,000 \sim 180,000$ OPS** 수준의 처리량을 안정적으로 기록했다. 이는 클라이언트 내부의 대기 시간(Queueing Delay)을 제거하여 순수한 서버의 능력을 이끌어낸 성과다.

나는 벤치마크를 설계하는 엔지니어로서, 이 방식이 현재 Python 생태계 내에서는 최적의 모델이지만 **스레드 컨텍스트 스위칭 오버헤드**라는 명확한 임계점을 가짐을 인지하고 있다. 현재 수준의 수십만 단위 트래픽 시뮬레이션에서는 이 구조가 완벽히 동작하지만, 만약 수백만 OPS 이상의 초거대 트래픽 극한 테스트가 요구된다면 Python의 스레드 한계를 인정하고 `memtier_benchmark`와 같은 C/C++ 기반 전용 벤치마크 도구로 인프라를 이전해야 할 것이다.