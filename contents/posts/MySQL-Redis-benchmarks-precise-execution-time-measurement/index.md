---
title: "currentTimeMillis의 해상도 한계와 산술 평균 측정"
description: "Redis 벤치마크 수행 중 단건 조회 시간이 0ms로 측정되는 현상을 분석하고, System.currentTimeMillis()의 정밀도 한계를 배치 단위의 산술 평균 도출로 극복한 과정을 기술한다."
date: 2026-02-10
tags: 
  - Java
  - Redis
  - 측정
  - 트러블 슈팅
series: "MySQL vs Redis 벤치마크"
---

## Redis 조회 0ms 측정 현상과 도구의 신뢰성 문제

MySQL과 Redis의 성능 비교를 위해 `SELECT` 및 `GET` 연산 속도를 측정하던 중 데이터 신뢰성 문제를 발견했다. 로컬 환경에서 MySQL은 평균 $1.05ms$ 대의 응답 속도를 보인 반면, Redis는 로그상 지속적으로 **0ms**가 기록되는 현상을 확인했다.

물리적인 네트워크 왕복 시간(RTT)과 직렬화(Serialization) 비용을 고려할 때 소요 시간이 물리적 '0'이 될 수는 없다. 이 현상은 Redis의 성능이 압도적이기 때문이 아니라, **측정 도구의 해상도가 대상의 속도를 따라잡지 못하는 한계** 때문에 발생하는 것이다.

## OS 타이머 인터럽트와 System.currentTimeMillis()의 해상도

문제의 핵심은 Java의 시간 측정 API인 `System.currentTimeMillis()`가 가진 정밀도(Precision)와 해상도(Resolution)의 한계에 기인한다. 이 메서드는 호출 시점의 운영체제(OS) 시스템 시각을 밀리초($ms$) 단위의 정수로 반환한다.

> **[Time Resolution](https://www.google.com/search?q=Time+Resolution+in+OS)**  
> 측정 도구가 구분할 수 있는 최소 시간 단위를 의미한다. `currentTimeMillis()`는 최소 $1ms$ 단위의 해상도를 가지며, OS 인터럽트 주기에 따라 정밀도가 변동하므로 마이크로초($\mu s$) 단위 연산을 포착하지 못한다.

Redis의 단건 조회는 보통 수십~수백 마이크로초($\mu s$) 내외로 처리된다. 실제 소요 시간이 $0.17ms$라 하더라도, `currentTimeMillis()`가 반환하는 값은 정수형(`long`)이므로 소수점 이하는 **0으로 버림(Truncation) 처리**되어 지표의 왜곡을 초래한다.

## List 수집과 DoubleStream 산술 평균을 통한 오차 상쇄

나는 `System.nanoTime()` 도입을 고려했으나, 시스템 체감 레이턴시의 전반적인 경향성 파악이라는 목적에 맞춰 구현 복잡도를 낮추는 **'단건 누적 후 산술 평균'** 방식을 채택했다.

~~~java
// RedisBatchExperiment.java 핵심 로직
private List<Long> performBatchOperations(int batch, String operationType) {
    List<Long> operationTimes = new ArrayList<>();
    for (int i = 1; i <= BATCH_SIZE; i++) {
        // 단건 명령어마다 소요 시간(long)을 측정하여 List에 수집한다.
        // 대부분 0ms가 기록되나, 간헐적인 지연(1ms 이상)도 함께 수집된다.
        operationTimes.add(executeRedisCommand(
                () -> redisTemplate.opsForValue().get(key)
        ));
    }
    return operationTimes;
}

// AbstractBatchExperiment.java의 평균 도출 로직
protected double average(List<Long> times) {
    // 0과 1 이상이 섞인 정수 리스트를 DoubleStream으로 묶어 산술 평균을 산출한다.
    return times.stream()
            .mapToLong(Long::longValue)
            .average()
            .orElse(0.0);
}
~~~

이 아키텍처는 개별 응답 시간이 $0$으로 절사되더라도, $1,000$번의 요청 중 OS 타이머 해상도에 걸려 $1ms$ 이상으로 기록되는 샘플들을 묶어 소수점 단위의 평균 지표($double$)를 도출하는 통계적 보정 방식이다.

## 통계적 보정의 성과와 실무적 한계 인지

이 방식을 통해 벤치마크를 수행한 결과, Redis의 조회 성능은 0ms가 아닌 통계적 평균 **$0.17ms$** 수준으로 측정되었다. MySQL($1.05ms$) 대비 약 **6.6배** 빠른 읽기 성능을 정량적으로 증명하는 성과를 얻었다.

하지만 나는 벤치마크 설계자로서 이 코드가 가진 명확한 측정 임계점(Threshold)을 인지하고 설계했다. 정수 버림 처리된 데이터들을 기반으로 산출된 산술 평균은 전체적인 성능의 경향성을 파악하는 데는 유효하나, 간헐적으로 튀는 **Coordinated Omission(지연 시간 누락)** 현상을 왜곡 없이 담아내지 못한다.

현재의 비교 실험 수준에서는 이 로직이 가장 합리적인 트레이드오프지만, 향후 시스템 한계 상황에서의 P99, P99.9 같은 정밀한 꼬리 지연(Tail Latency) 분석이 요구되는 환경으로 넘어간다면 본 로직을 폐기하고 **JMH(Java Microbenchmark Harness)**와 같은 전용 프레임워크로 측정 인프라를 이전해야 한다.