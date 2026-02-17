---
title: "데이터 정합성을 위한 Write-Primary 라우팅 설계"
description: "동적 라우팅 환경에서 발생할 수 있는 데이터 파편화와 정합성 불일치 문제를 분석하고, Write-Primary 정책을 통해 쓰기 경로를 단일화하여 무결성을 확보한 아키텍처를 기술한다."
date: 2026-02-16
tags: 
  - D-HASH
  - 분산 시스템
  - 설계
  - 데이터 정합성
  - 트러블 슈팅
series: "D-HASH 알고리즘 개발기"
---

## 동적 라우팅 환경의 데이터 파편화 리스크

D-HASH 알고리즘은 읽기(Read) 트래픽 분산을 위해 특정 키(Key)의 요청 경로를 메인 노드(Primary Node, $p(k)$)에서 대체 노드(Alternate Node, $a(k)$)로 실시간 변경한다. 로드 밸런싱 측면에서는 효과적이지만, 이 유연함은 **데이터 정합성(Consistency)** 측면에서 심각한 위협이 된다.    

만약 라우팅 로직이 읽기와 쓰기(Write) 구분 없이 동일하게 적용된다면, Hot-key 구간에서 쓰기 요청 또한 $a(k)$로 전달된다. 이후 트래픽이 감소하여 라우팅이 다시 $p(k)$로 복귀되면, $p(k)$에는 구버전 데이터가 남고 $a(k)$에만 신규 데이터가 존재하는 **데이터 파편화(Fragmentation)**가 발생한다.    

이는 클라이언트가 방금 갱신한 데이터를 읽지 못하는 'Stale Read' 현상을 유발하며, 데이터 무결성이 핵심인 캐시 시스템에서 치명적인 결함이다.    

## 읽기 최적화와 쓰기 정합성의 충돌

문제의 근본 원인은 **읽기 성능 최적화를 위한 동적 라우팅 규칙을 쓰기 경로에도 동일하게 적용**한 데 있다. 분산 시스템의 CAP 이론 관점에서, 파티션 감내(P) 환경의 가용성(A)을 높이기 위해 데이터를 동적으로 이동시키는 순간 일관성(C) 유지는 기술적으로 매우 복잡해진다.    



D-HASH 구조에서 $a(k)$는 트래픽 분산을 위한 '임시 저장소' 성격이 강하다. 이곳을 데이터의 원천(Source of Truth)으로 취급하여 쓰기 작업을 수행하면, 시스템 내에 서로 다른 데이터 버전이 공존하는 **Split-Brain** 유사 상황이 초래된다.    

## Write-Primary: 쓰기 경로의 물리적 고정

이 문제를 해결하기 위해 **Write-Primary** 정책을 도입했다. 핵심은 데이터의 **읽기 경로는 부하 상황에 따라 분기(Fork)하되, 쓰기 경로는 오직 불변의 해시 링(Consistent Hashing Ring)이 지정하는 $p(k)$로만 고정**하는 것이다.   

> **[Write-Primary Pattern](https://www.google.com/search?q=Write-Primary+Pattern)이란?**   
> 모든 쓰기 요청을 지정된 주(Primary) 노드에서만 처리하고, 읽기 요청은 여러 노드(Replica/Alternate)로 분산하여 성능을 확장하면서도 데이터 무결성을 유지하는 디자인 패턴이다.    

이 구조를 통해 어떤 시점에 쓰기가 발생하더라도 데이터는 항상 $p(k)$에 최신 상태로 기록된다. $a(k)$는 읽기 부하를 분담하는 역할만 수행하며, $a(k)$의 데이터 갱신은 별도의 동기화 메커니즘(Pre-warming 등)을 통해 $p(k)$를 복제해오는 방식으로 안전하게 처리된다.   

## 구현 로직과 정합성 보장

D-HASH의 라우팅 로직인 `algorithms.py`를 보면, `get_node` 메서드 진입 시 연산 타입(`op`)을 확인하여 쓰기 요청일 경우 동적 라우팅 로직을 완전히 배제하고 즉시 $p(k)$를 반환하도록 설계했다.    

~~~python
def get_node(self, key: Any, op: str = "read") -> str:
    """
    Routes a request to a node based on operation type.
    """
    # 1. Write Consistency: Always route to Primary Node
    # Hot-key 여부와 무관하게 쓰기는 무조건 Primary로 고정하여 정합성 보장
    if op == "write":
        return self._primary_safe(key)

    # 2. Read Optimization: Dynamic Routing
    # 읽기 요청만 부하 상태에 따라 Alternate Node로 분산
    return self._resolve_read_node(key)
~~~

## 아키텍처 단순화와 Split-Brain 방지

Write-Primary 정책 적용 결과, 동적 라우팅을 사용하는 복잡한 분산 환경에서도 데이터 흐름을 명확히 통제할 수 있게 되었다.   

* **정합성 결함 제거**: 쓰기 경로가 단일화되었으므로, 라우팅 변경 시점에 데이터가 유실되거나 노드 간에 파편화될 가능성을 **0%**로 차단했다.   
* **복잡도 감소**: 동적 라우팅이 쓰기 정합성에 영향을 주지 않으므로, 분산 락(Distributed Lock)이나 복잡한 쿼럼(Quorum) 쓰기 로직 없이도 완벽한 무결성을 유지한다.   

## Read-intensive 워크로드에 특화된 트레이드오프

Write-Primary 전략은 데이터 정합성을 확보하는 가장 확실한 방법이지만, 모든 쓰기 트래픽이 $p(k)$로 집중된다는 구조적 특성이 있다.    

> 이 아키텍처는 **읽기 비율이 압도적으로 높은(Read-intensive) 캐시 시스템**에 최적화된 설계다. 만약 쓰기 부하 자체가 병목인 'Write-heavy' 워크로드라면, 이 정책만으로는 부족하며 DB 샤딩(Sharding)이나 최종 일관성(Eventual Consistency) 모델로의 전환을 고려해야 한다.   

엔지니어링의 핵심은 모든 상황에 맞는 정답을 찾는 것이 아니라, 현재 시스템의 목적(Cache Read Performance)에 가장 적합한 트레이드오프를 선택하는 데 있다.   

---
**🔗 GitHub Repository:** [bh1848/D-HASH](https://github.com/bh1848/D-HASH)