---
title: "System.currentTimeMillis()의 정밀도 한계와 배치 단위 측정을 통한 오차 개선"
description: "Redis 벤치마크 수행 중 단건 조회 시간이 0ms로 측정되는 현상을 분석하고, System.currentTimeMillis()의 정밀도 한계를 배치 단위 측정으로 극복한 과정을 기술한다."
date: 2026-02-10
tags: 
  - Java
  - Redis
  - 측정
  - 오차 개선
series: "MySQL vs Redis 비교 벤치마크"
---

## 너무 빨라서 측정되지 않는 Redis 응답 속도

MySQL과 Redis의 성능 비교를 위해 `SELECT` 및 `GET` 연산 속도를 측정하던 문제를 발견했다. 로컬 환경에서 MySQL은 평균 $1.05ms$ 대의 응답 속도를 보였으나, Redis는 로그상 지속적으로 **0ms**가 찍히는 것이었다. 아무리 In-Memory DB가 빠르다고 해도 물리적인 네트워크 왕복 시간(RTT)과 직렬화 비용이 존재하는데, 소요 시간이 '0'이라는 것은 측정 방식에 문제가 있다는 것이다. 이는 빠른 성능을 증명한 것이 아니라, **측정 도구가 대상의 속도를 따라잡지 못한 상황**이다.

## OS 종속적인 시간 정밀도의 한계

문제의 원인은 Java의 시간 측정 API인 `System.currentTimeMillis()`의 정밀도(Precision)와 해상도(Resolution) 차이에 있다. 우리가 흔히 사용하는 이 메서드는 호출 시점의 운영체제(OS) 시스템 시각을 밀리초(ms) 단위의 `long` 타입 정수로 반환한다.

> **정밀도(Precision)와 해상도(Resolution)란?**  
> 정밀도는 측정값이 실제 값에 얼마나 근접한지를, 해상도는 측정 도구가 구분할 수 있는 최소 단위를 의미한다. `currentTimeMillis()`는 1ms 단위의 해상도를 가지므로, 1ms보다 짧은 시간은 측정 불가능하다.

특히 Windows OS 환경에서는 시스템 타이머 인터럽트 주기에 따라 정밀도가 약 $10\sim15ms$ 단위로 떨어지기도 한다. Redis의 단순 Key-Value 조회는 보통 수십~수백 마이크로초($\mu s$) 단위로 처리된다. 즉, 실제 소요 시간이 $0.15ms$라 하더라도, `currentTimeMillis()`가 인지할 수 있는 최소 단위인 $1ms$에 미치지 못하므로 **정수형 변환 과정에서 0으로 버림 처리**되는 것이다.

## 대수의 법칙, 배치 단위 측정

이 문제를 해결하기 위해 `System.nanoTime()`을 사용하는 방법도 고려했다. `nanoTime()`은 나노초($ns$) 단위의 고해상도 측정을 지원하지만, 벤치마크의 목적이 '애플리케이션 레벨에서 체감하는 평균 응답 속도'였기에, 극소 시간 측정보다는 **전체 흐름의 평균을 구하는 방식**이 더 적합하다고 판단했다.

따라서 단건 실행 시간을 측정하는 방식에서, **배치(Batch) 단위의 총 소요 시간을 측정한 뒤 횟수만큼 나누는 '평균 역산 방식'**으로 설계를 변경했다.

~~~java
// AS-IS: 단건 측정 (0ms 발생 가능성 높음)
for (int i = 0; i < 1000; i++) {
    long start = System.currentTimeMillis();
    redisTemplate.opsForValue().get(key);
    long end = System.currentTimeMillis(); // 차이가 1ms 미만이면 0으로 측정됨
}

// TO-BE: 배치 단위 총량 측정
long batchStart = System.currentTimeMillis();
for (int i = 0; i < 1000; i++) {
    redisTemplate.opsForValue().get(key);
}
long batchEnd = System.currentTimeMillis();
double average = (batchEnd - batchStart) / 1000.0;
~~~

이 방식은 개별 연산의 미세한 변동성을 상쇄하고, OS 타이머의 정밀도 한계보다 훨씬 긴 시간(예: 1,000회 실행 시 약 $150ms$)을 측정하므로 오차를 최소화할 수 있다. 실제 구현 코드에도 이와 같은 로직을 적용하여 10,000회의 연산을 1,000회씩 배치로 나누어 측정했다.

## 숨겨져 있던 0.17ms를 찾아내다

측정 방식을 변경한 후 재실험을 수행한 결과, Redis의 조회 성능은 0ms가 아닌 평균 **$0.17ms$** 수준임이 확인되었다. 이는 MySQL($1.05ms$) 대비 약 6.6배 빠른 수치다. 단순히 "Redis가 빠르다"는 정성적 결론에 그치지 않고, 신뢰할 수 있는 정량적 데이터를 확보하게 된 것이다.

## 마이크로 벤치마킹의 함정과 JMH

배치 단위 평균 역산 방식은 실무적으로 유효한 근사치를 제공하지만, **Coordinated Omission(지연 시간 누락)** 문제를 완전히 해결하지는 못한다. 특정 시점에 GC(Garbage Collection) 등으로 튀는 지연 시간(Jitter)이 발생할 경우 평균값에 희석될 수 있기 때문이다.

따라서 더욱 엄밀한 측정이 요구되는 환경이라면 Java 진영의 표준 벤치마크 프레임워크인 **JMH(Java Microbenchmark Harness)** 도입을 고려해야 한다. JMH는 JVM의 웜업(Warm-up)과 최적화 단계를 자동으로 제어하고, 평균뿐만 아니라 P99, P99.9와 같은 백분위 지표를 제공하여 '꼬리 지연(Tail Latency)'까지 분석할 수 있게 해준다. 이번 프로젝트에서는 전체적인 경향성 파악이 목적이었기에 평균 역산 방식을 택했으나, 시스템의 한계 상황을 테스트할 때는 측정 도구의 선택 또한 트레이드오프의 영역임을 인지해야 한다.

---
**🔗 GitHub Repository:** [bh1848/mysql-redis-benchmark](https://github.com/bh1848/mysql-redis-benchmark)