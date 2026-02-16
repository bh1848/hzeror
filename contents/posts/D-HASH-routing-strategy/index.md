---
title: "분산 캐시 정합성 유지를 위한 Write-Primary 라우팅 설계"
description: "동적 라우팅 환경에서 발생할 수 있는 데이터 파편화와 정합성 불일치 문제를 분석하고, Write-Primary 정책을 통해 쓰기 경로를 단일화하여 무결성을 확보한 아키텍처를 기술한다."
date: 2026-02-16
tags: 
  - D-HASH
  - 분산 시스템
  - 설계
  - 데이터 정합성
series: "D-HASH 알고리즘 개발기"
---

## 동적 라우팅 도입에 따른 데이터 파편화 리스크

D-HASH 알고리즘은 읽기(Read) 트래픽 분산을 위해 특정 키(Key)의 요청 경로를 메인 노드(Primary Node, $p(k)$)에서 대체 노드(Alternate Node, $a(k)$)로 실시간 변경한다. 로드 밸런싱 측면에서는 효과적이지만, 이 유연함은 **데이터 정합성(Consistency)** 측면에서 심각한 위협이 될 수 있다.

만약 라우팅 로직이 읽기와 쓰기(Write) 구분 없이 동일하게 적용된다면, Hot-key 구간에서 쓰기 요청 또한 $a(k)$로 전달된다. 이후 트래픽이 감소하여 라우팅이 다시 $p(k)$로 복귀되면, $p(k)$에는 최신 데이터가 없고 $a(k)$에만 데이터가 남는 **데이터 파편화(Fragmentation)**가 발생한다. 사용자는 방금 갱신한 데이터를 읽지 못하거나 과거의 데이터를 읽게 되는 'Stale Read' 현상을 겪게 되며, 이는 데이터 무결성이 핵심인 분산 시스템에서 치명적인 결함이다.

## 읽기/쓰기 경로의 물리적 결합(Coupling)

문제의 근본 원인은 **읽기 성능 최적화를 위한 동적 라우팅 규칙을 쓰기 경로에도 동일하게 적용**한 데 있다. 분산 시스템의 CAP 정리 관점에서 파티션 감내(P) 환경의 가용성(A)을 높이기 위해 데이터를 복제하거나 이동시키는 순간, 일관성(C) 유지는 기술적으로 매우 복잡해진다.

D-HASH 구조에서 $a(k)$는 트래픽 분산을 위한 '임시 캐시' 성격이 강하다. 이곳을 데이터의 최종 저장소(Source of Truth)로 취급하여 쓰기 작업을 수행하면, 시스템 내에 서로 다른 데이터가 공존하는 **Split-Brain** 유사 상황이 초래된다. 따라서 동적 라우팅 환경에서도 데이터의 유일한 정합성 기준점은 변하지 않아야 한다.

## Write-Primary 정책을 통한 쓰기 경로 단일화

이 문제를 해결하기 위해 **Write-Primary** 정책을 도입했다. 이는 데이터의 **읽기 경로는 부하 상황에 따라 분기(Fork)하되, 쓰기 경로는 오직 불변의 해시 링(Consistent Hashing Ring)이 지정하는 $p(k)$로만 고정**하는 전략이다.

> **[Consistent Hashing](https://www.google.com/search?q=Consistent+Hashing)이란?**  
> 노드의 추가나 삭제 시 데이터 재배치를 최소화하기 위해 해시 링(Hash Ring) 구조를 사용하는 분산 기법이다. D-HASH는 이 링 위에 동적 경로 변경 로직을 얹어 트래픽 유연성을 확보했다.

> **[Write-Primary](https://www.google.com/search?q=Write-Primary+Pattern)란?**  
> 모든 쓰기 요청을 지정된 주(Primary) 노드에서만 처리하고, 읽기 요청은 여러 노드로 분산하여 성능을 확장하면서도 데이터 무결성을 유지하는 디자인 패턴이다.

D-HASH의 핵심 로직인 `algorithms.py`를 보면, `get_node` 메서드 진입 시 연산 타입(`op`)을 확인하여 쓰기 요청일 경우 동적 라우팅 로직을 완전히 배제하고 즉시 $p(k)$를 반환하도록 설계했다.

~~~python
def get_node(self, key: Any, op: str = "read") -> str:
    """
    Routes a request to a node.
    - WRITE: Always routed to Primary Node to maintain consistency.
    - READ: Routed to Primary or Alternate based on hot-status.
    """
    # 1. Write Consistency: Always P_k
    # Hot-key 여부와 무관하게 쓰기는 무조건 Primary로 고정하여 정합성 보장
    if op == "write":
        return self._primary_safe(key)
~~~

이 구조를 통해 어떤 시점에 쓰기가 발생하더라도 데이터는 항상 $p(k)$에 최신 상태로 기록된다. $a(k)$는 읽기 부하를 분담하는 역할만 수행하며, $a(k)$의 데이터 갱신은 Guard Phase 내의 Pre-warming(이중 쓰기) 등을 통해 $p(k)$의 데이터를 복제해오는 방식으로 안전하게 처리된다.

## 분산 환경 내 데이터 무결성 보장

Write-Primary 정책 적용 결과, 동적 라우팅을 사용하는 복잡한 분산 환경에서도 데이터 흐름을 명확히 통제할 수 있게 되었다.

* **정합성 결함 제거**: 쓰기 경로가 단일화되었으므로, 라우팅 변경 시점에 데이터가 유실되거나 노드 간에 파편화될 가능성을 **0%**로 차단했다.
* **시스템 아키텍처 단순화**: 동적 라우팅이 쓰기 정합성에 영향을 주지 않으므로, 분산 락(Distributed Lock)이나 복잡한 데이터 버전 관리 로직 없이도 완벽한 무결성을 유지할 수 있다.

## 워크로드 특성에 따른 전략적 선택

Write-Primary 전략은 데이터 정합성을 확보하는 가장 확실한 방법이지만, 모든 쓰기 트래픽이 $p(k)$로 집중된다는 특성이 있다. 따라서 이 아키텍처는 **읽기 비율이 압도적으로 높은(Read-intensive) 캐시 시스템**에 최적화된 설계임을 명확히 인지해야 한다.

만약 쓰기 부하 자체가 병목인 'Write-heavy' 워크로드라면, 이 정책만으로는 부족할 수 있다. 이 경우 데이터베이스 레벨의 샤딩(Sharding)을 강화하거나, 비즈니스 허용 범위 내에서 최종 일관성(Eventual Consistency) 모델로의 전환을 고려해야 한다. 엔지니어링의 핵심은 모든 상황에 맞는 정답을 찾는 것이 아니라, 현재 시스템의 목적에 가장 적합한 트레이드오프(Trade-off)를 선택하는 데 있다.

---
**🔗 GitHub Repository:** [bh1848/D-HASH](https://github.com/bh1848/D-HASH)