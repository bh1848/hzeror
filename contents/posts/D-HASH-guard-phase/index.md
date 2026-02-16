---
title: "Hot-key 승격 시점의 Latency Spike 억제를 위한 Guard Phase와 Pre-warming"
description: "D-HASH 알고리즘에서 Hot-key 승격 직후 발생하는 Alternate Node의 캐시 미스 문제를 분석하고, Guard Phase와 쓰기 병행 예열(Pre-warming)을 통해 트래픽 분산 시점의 성능 저하를 방지한 기법을 기술한다."
date: 2026-02-15
tags: 
  - D-HASH
  - 성능 개선
  - 분산 시스템
  - 해결 기록
series: "D-HASH 알고리즘 개발기"
---

## Hot-key 승격 직후 발생하는 일시적 성능 저하 현상

D-HASH 알고리즘은 특정 키(Key)에 대한 요청 횟수가 임계값($T$)을 초과하는 순간, 해당 키를 Hot-key로 승격시키고 트래픽의 일부를 대체 노드(Alternate Node, $A_k$)로 우회시킨다. 이는 메인 노드(Primary Node, $p(k)$)의 부하를 집중적으로 해소하기 위한 핵심 메커니즘이다.

그러나 초기 구현 단계에서 키가 Hot-key로 승격되어 $A_k$로 트래픽이 분산되는 시점에 **Latency Spike(지연 시간 급증)**가 관측되었다. 원인 분석 결과, 승격 직후 $A_k$에는 해당 데이터가 캐싱되어 있지 않은 'Cold State'였으며, 이로 인해 $A_k$로 향하는 첫 트래픽들이 연쇄적으로 **Cache Miss**를 유발했기 때문이다. 부하 분산을 위한 경로 변경이 오히려 백엔드 DB 접근 비용을 발생시켜 시스템 안정성을 저해하는 현상이 확인되었다.

## 데이터 지역성 부재에 따른 Cold Start 문제

Consistent Hashing 환경에서 모든 데이터는 해시 링상의 결정론적 위치인 $p(k)$에 매핑되어 저장된다. 반면 D-HASH가 동적으로 선택한 $A_k$는 논리적인 우회 경로일 뿐, 물리적인 데이터 복제본을 즉각적으로 보유하고 있지 않다.

> **[데이터 지역성(Data Locality)](https://www.google.com/search?q=Data+Locality)이란?**     
> 데이터가 실제로 필요한 계산 장치와 물리적으로 가까운 곳에 위치하려는 성질을 의미한다. 분산 캐시에서는 요청이 전달된 노드에 이미 데이터가 존재해야 네트워크 및 디스크 I/O 비용을 최소화할 수 있다.

트래픽이 급증하여 Hot-key로 판정되는 임계 시점은 시스템 리소스가 가장 타이트하게 사용되는 구간이다. 이 시점에 준비되지 않은 노드로 트래픽을 전송하는 것은 서비스 품질에 치명적이다. 따라서 트래픽 분산 이전에 $A_k$의 상태를 'Hot'하게 만드는 **유예 기간(Grace Period)** 설계가 필요하다.

## Guard Phase 도입을 통한 단계적 라우팅 전환

이 문제를 해결하기 위해 **Guard Phase(방어 구간)** 로직을 도입했다. 키가 Hot-key 임계값($T$)에 도달하더라도 즉시 라우팅을 변경하지 않고, 일정 횟수(Window Size, $W$)만큼은 기존 $p(k)$가 요청을 계속 처리하도록 강제하는 방식이다.

~~~python
def get_node(self, key: Any, op: str = "read") -> str:
    # ... (생략) ...
    
    # Hot-key 임계값 도달 시 승격 수행
    if cnt >= self.T:
        self._ensure_alternate(key) 
        delta = cnt - self.T # 승격 이후 누적 요청 수

        # [Guard Phase]
        # 승격 직후 W번의 요청 동안은 Alternate Node로 보내지 않고 Primary를 유지한다.
        # 이 기간을 통해 A_k가 데이터를 채울 시간을 확보한다.
        if delta < self.W:
            return self._primary_safe(key)

        # Guard Phase 종료 후 윈도우 기반 동적 라우팅 실시
        epoch = (delta - self.W) // self.W
        return self.alt[key] if (epoch % 2 == 0) else self._primary_safe(key)
~~~

Guard Phase가 지속되는 동안 시스템은 승격 상태를 유지하면서도 실제 트래픽 전환 시점을 뒤로 미룬다. 이 $W$번의 요청 기간은 $A_k$가 데이터를 적재할 수 있는 최소한의 시간적 여유를 제공한다.

## 쓰기 병행 예열(Pre-warming)을 통한 데이터 가용성 확보

Guard Phase의 실효성을 높이기 위해 **Pre-warming(예열)** 전략을 병행 설계했다. 클라이언트 사이드에서 쓰기(Write) 요청이 발생할 때, 해당 키가 Hot-key 후보이거나 이미 승격된 상태라면 $p(k)$뿐만 아니라 $A_k$에도 데이터를 비동기로 기록하는 방식이다.

~~~python
# 쓰기 시점에 Alternate Node가 선정되어 있다면 병행 기록 수행 (Pre-warming)
if hasattr(sharding, "_ensure_alternate"):
    sharding._ensure_alternate(k)
    a_node = sharding.alt.get(k)
    
    if a_node and a_node != p_node:
        # Primary와 Alternate 양쪽에 쓰기 요청을 전달하여 데이터 복제
        write_buckets[a_node].append(k)
~~~

이러한 **이중 쓰기(Dual Write)** 로직이 Guard Phase와 결합되면 실제 읽기 트래픽이 $A_k$로 넘어가는 시점($\delta \ge W$)에는 이미 $A_k$가 최신 데이터를 보유하게 된다. 이는 승격 시점의 초기 Cache Miss 발생률을 낮게 통제할 수 있는 근거가 된다.

## 승격 구간의 성능 하락 방지

Guard Phase와 Pre-warming 전략을 적용한 결과, Hot-key 승격 과정에서 관측되던 Latency Spike를 성공적으로 억제했다.

* **단계적 트래픽 전환**: 급격한 경로 변경으로 인한 'Cold Start' 충격을 방지하여, 부하 분산 로직이 시스템 전체의 응답 속도를 저해하지 않고 적용되도록 개선했다.
* **서비스 안정성 확보**: 실제 NASA 및 Synthetic Zipfian 워크로드 실험에서 승격 구간의 급격한 성능 하락 없이 부하 표준편차를 최대 $33.8\%$까지 개선하는 성과를 거두었다.

단순한 알고리즘적 설계를 넘어 분산 시스템의 물리적인 데이터 전파 속도와 캐시 적재 과정을 설계에 녹여낸 결과, 실무 환경에서도 신뢰할 수 있는 안정적인 로드 밸런싱 구조를 완성했다.

---
**🔗 GitHub Repository:** [bh1848/D-HASH](https://github.com/bh1848/D-HASH)