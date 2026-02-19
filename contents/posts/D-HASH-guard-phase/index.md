---
title: "Hot-key 승격 지연을 위한 Guard Phase 설계"
description: "D-HASH 알고리즘에서 Hot-key 승격 직후 발생하는 Alternate Node의 캐시 미스 문제를 분석하고, Guard Phase와 쓰기 병행 예열(Pre-warming)을 통해 트래픽 분산 시점의 성능 저하를 방지한 기법을 기술한다."
date: 2026-02-15
tags: 
  - D-HASH
  - 성능 개선
  - 분산 시스템
  - 트러블 슈팅
series: "D-HASH 알고리즘 개발기"
---

## 트래픽 분산 직후 발생하는 Latency Spike

D-HASH 알고리즘은 특정 키(Key)에 대한 요청 횟수가 임계값($T$)을 초과하면, 해당 키를 Hot-key로 승격시키고 트래픽의 일부를 대체 노드(Alternate Node, $A_k$)로 우회시킨다. 이는 메인 노드($p(k)$)의 부하를 해소하기 위해 내가 고안한 핵심 메커니즘이다.

그러나 초기 구현 단계에서 키가 승격되어 $A_k$로 트래픽이 분산되는 순간, 급격한 **Latency Spike(지연 시간 급증)**가 관측되었다.

이 현상은 승격 직후 $A_k$가 해당 데이터를 보유하지 않은 **'Cold State'**이기 때문에 발생하는 것이다. 준비되지 않은 노드로 유입된 트래픽은 연쇄적인 Cache Miss를 유발하고, 이는 백엔드 DB의 부하로 전이되어 시스템 전체의 안정성을 저해한다.

## 데이터 지역성 부재와 Cold Start

Consistent Hashing 환경에서 모든 데이터는 해시 링상의 결정론적 위치인 $p(k)$에 매핑된다. 반면 내가 동적으로 선택한 $A_k$는 논리적인 우회 경로일 뿐, 물리적인 데이터 복제본을 즉각적으로 보유하고 있지 않다.

> **[Cache Cold Start](https://www.google.com/search?q=Cache+Cold+Start)**  
> 시스템이나 노드가 처음 가동될 때 캐시가 비어 있어 요청을 처리하지 못하는 상태를 의미한다. 내가 설계한 D-HASH의 동적 라우팅은 실행 중인 시스템 내에서 인위적으로 Cold Start 상황을 빈번하게 유발하는 특성이 있다.

트래픽이 급증하여 Hot-key로 판정되는 시점은 시스템 리소스가 가장 타이트한 구간이다. 이때 $A_k$에서 Cache Miss가 발생하면, 데이터 로딩을 위한 네트워크 I/O 비용이 추가되어 분산 처리가 오히려 독이 되는 결과를 초래한다.

## Guard Phase와 Active Pre-warming의 결합

나는 이 딜레마를 해결하기 위해 **Guard Phase(방어 구간)**와 **Active Pre-warming(능동적 예열)** 로직을 결합하여 설계했다. 

키가 임계값($T$)에 도달하더라도 즉시 라우팅을 변경하지 않고, 일정 횟수(Window Size, $W$)만큼은 기존 $p(k)$가 요청을 계속 처리하도록 강제하는 것이다. 그리고 이 유예 기간 동안 $A_k$에 데이터를 비동기로 복제하여 미리 심어둔다.

~~~python
def get_node(self, key: Any, op: str = "read") -> str:
    # ... (중략)
    if cnt >= self.T:
        self._ensure_alternate(key) 
        delta = max(0, cnt - self.T) # 승격 이후 누적 요청 수

        # [Guard Phase]
        # 승격 직후 W번의 요청 동안은 Alternate Node로 보내지 않고 Primary를 유지한다.
        # 이 기간(Latency Hiding)을 통해 A_k가 데이터를 적재할 시간을 확보한다.
        if delta < self.W:
            return self._primary_safe(key)
~~~

~~~python
# 클라이언트 쓰기 시점에 Alternate Node가 선정되어 있다면 병행 기록 수행 (Pre-warming)
if hasattr(sharding, "_ensure_alternate"):
    sharding._ensure_alternate(k)
    a_node = sharding.alt.get(k)
    
    # Primary와 Alternate 양쪽에 쓰기 요청을 전달하여 데이터 복제
    # 이는 정합성을 위한 Write가 아니라, Cache Warming을 위한 이중 쓰기(Dual Write)다.
    if a_node and a_node != p_node:
        write_buckets[a_node].append(k)
~~~

이러한 논리적 지연 설계를 통해, 실제 읽기 트래픽이 $A_k$로 넘어가는 시점($\delta \ge W$)에는 이미 $A_k$가 최신 데이터를 보유하게 만들어 Cache Miss를 원천 차단했다.

## 네트워크 대역폭 한계 인지 및 개선 지표

이 아키텍처를 적용한 결과, 실제 NASA 및 Synthetic 워크로드 실험에서 승격 구간의 Latency Spike가 제거되었으며 노드 간 부하 표준편차는 최대 **$33.8\%$**까지 개선되었다.

하지만 나는 이 해결책이 가진 네트워크 임계점을 인지하고 있다. Active Pre-warming은 $A_k$로의 추가적인 쓰기 트래픽을 유발하므로 대규모 클러스터에서는 네트워크 대역폭(Bandwidth)을 낭비하는 **트레이드오프**가 존재한다. 그럼에도 불구하고 Hot-key 상황의 병목은 대역폭보다는 CPU와 Latency에서 발생하므로, 약간의 I/O 비용을 지불하고 시스템의 응답 속도 안정성을 챙기는 것이 현재 워크로드에서 더 합리적인 리스크 통제라고 판단했다.