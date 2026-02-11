---
title: "동기 방식 요청의 대기 시간 병목 해결과 비동기 테스트 환경 구축"
description: "D-HASH 알고리즘 검증 과정에서 발생한 테스트 클라이언트의 병목 현상을 분석하고, ThreadPoolExecutor와 Redis 파이프라인 도입을 통해 고부하 환경을 구현한 과정을 정리한다."
date: 2026-02-11
update: 2026-02-11
tags:
  - D-HASH
  - ThreadPoolExecutor
  - 트러블슈팅
series: "D-HASH 개발기"
---

분산 캐시 시스템인 **D-HASH**를 개발하면서 가장 큰 난관은 역설적이게도 '알고리즘'이 아닌 '테스트 도구'에 있었다. 분산 시스템의 성능을 제대로 측정하려면 초당 수십만 건의 요청을 단시간에 쏟아부어야 하는데, 정작 테스트 클라이언트가 이 부하를 견디지 못해 알고리즘의 한계를 보기 전에 병목이 먼저 발생했기 때문이다.

테스트 도구의 성능이 테스트 대상의 성능을 가로막는 아이러니한 상황을 어떻게 해결했는지, 그 고민의 과정을 기록해본다.


## CPU는 놀고 있는데 처리량은 제자리다

초기 테스트 도구는 단순한 루프 기반의 동기 방식으로 구현했다. 하나의 요청이 완전히 끝나야 다음 요청을 보내는 구조였는데, 여기서 치명적인 병목이 발생했다.

### 왜 동기 방식은 부하 테스트에 부적합할까?
* **네트워크 I/O의 늪**: Redis 서버에 요청을 보내고 응답을 받는 시간 동안 CPU는 유의미한 연산 없이 유휴 상태(Idle)로 머물게 된다. 이 '대기 시간의 누적'은 초당 수십만 건을 처리해야 하는 환경에서 치명적이다.
* **현실과의 괴리**: 실제 서비스 환경은 수많은 클라이언트가 동시에 접속하는 동시성 환경이지만, 직렬적인 동기 테스트는 이를 재현하지 못해 알고리즘의 실제 성능을 왜곡한다.
* **리소스 낭비**: 모니터링 결과, Redis 서버는 충분히 자원이 남아도는 반면 테스트 클라이언트는 물리적인 네트워크 대기 시간 때문에 목표 OPS(Operations Per Second)를 달성하지 못하고 있었다.


## ThreadPoolExecutor와 파이프라인의 결합

부하 테스트는 전형적인 **I/O Bound** 작업이다. Python에는 GIL(Global Interpreter Lock) 제약이 있지만, 네트워크 응답을 기다리는 I/O 작업 시에는 스레드가 제어권을 넘겨줄 수 있다. 나는 표준 라이브러리인 `concurrent.futures.ThreadPoolExecutor`를 활용해 병렬성을 확보하기로 했다.

하지만 단순히 스레드만 늘리는 것은 컨텍스트 스위칭 오버헤드를 유발할 수 있다. 이를 방지하기 위해 각 스레드가 **Redis Pipeline**을 사용하여 요청을 배치(Batch) 단위로 묶어 보내도록 설계하여, 네트워크 왕복 횟수(RTT) 자체를 획기적으로 줄였다.



### 실제 구현 코드

~~~python
from concurrent.futures import ThreadPoolExecutor

def benchmark_cluster(keys, sharding, pipeline_size=500):
    # [비동기 부하 테스트 핵심 로직]
    # 1. ThreadPoolExecutor로 노드별 병렬 I/O 수행
    # 2. Redis Pipeline으로 네트워크 RTT 최소화 (기본 B=500)
    
    # 쓰기(Write) 연산 병렬 실행
    with ThreadPoolExecutor(max_workers=max(1, len(write_buckets))) as ex:
        # 각 노드에 할당된 요청 버킷을 병렬로 처리하여 Wall-clock Time 단축
        for total, samples in ex.map(_io_write, write_buckets.items()):
            write_node_totals.append(total)
            write_all_samples.extend(samples)

    # 읽기(Read) 연산 병렬 실행
    with ThreadPoolExecutor(max_workers=max(1, len(read_buckets))) as ex:
        for total, samples in ex.map(_io_read, read_buckets.items()):
            read_node_totals.append(total)
            read_all_samples.extend(samples)
~~~



## 주요 최적화 포인트: 무엇에 집중했는가?

단순히 도구를 쓰는 것을 넘어, 최적의 효율을 위해 다음 두 지점에 집중했다.

### Worker 수와 배치 사이즈의 조율
`max_workers`를 실제 Redis 노드 수에 맞추고, 실험을 통해 검증된 최적의 파이프라인 크기($B=500$)를 설정하여 스레드 오버헤드와 네트워크 효율 사이의 균형을 맞췄다.

### 통계적 정확성 확보
배치(Pipeline) 단위로 요청을 처리하면 레이턴시 통계가 왜곡될 수 있다. 이를 방지하기 위해 각 샘플에 가중치를 부여하는 `_weighted_percentile` 로직을 구현하여, 병렬 환경에서도 정확한 **P95, P99 Tail Latency**를 산출할 수 있게 했다.


## 비동기 전환 후 성능 측정의 신뢰성 확보

전환 결과는 놀라웠다. 클라이언트 병목으로 초당 수만 건에 머물렀던 처리량은 **초당 15만~18만 건(ops/s)** 이상으로 급증했다.

1.  **극한의 부하 재현**: 테스트 클라이언트의 성능 한계를 제거함으로써, 비로소 Redis 서버의 네트워크 대역폭(Bandwidth)을 모두 소모할 만큼의 고부하 시나리오를 재현할 수 있게 되었다.
2.  **가설의 증명**: 동시성 환경이 갖춰지자, 핫키 분산이 레이턴시 안정화에 얼마나 기여하는지를 수치로 명확히 증명하는 근거가 되었다.


## 테스트 도구는 정직해야 한다

"테스트 도구가 테스트 대상의 성능을 가로막아서는 안 된다"는 원칙을 다시 한번 절감했다. 단순히 비동기 라이브러리를 가져다 쓰는 것을 넘어, 언어의 특성(GIL)과 대상 시스템의 특성(Redis Pipeline)을 깊게 이해하고 결합했을 때 비로소 신뢰할 수 있는 벤치마크 환경이 만들어졌다.