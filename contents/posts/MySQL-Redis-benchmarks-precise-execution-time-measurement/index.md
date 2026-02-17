---
title: "currentTimeMillis의 해상도 한계와 배치 단위 개선"
description: "Redis 벤치마크 수행 중 단건 조회 시간이 0ms로 측정되는 현상을 분석하고, System.currentTimeMillis()의 정밀도 한계를 배치 단위 측정으로 극복한 과정을 기술한다."
date: 2026-02-10
tags: 
  - Java
  - Redis
  - 측정
  - 오차 개선
series: "MySQL vs Redis 벤치마크"
---

## Redis 조회 0ms 측정 현상과 도구의 신뢰성 문제

MySQL과 Redis의 성능 비교를 위해 `SELECT` 및 `GET` 연산 속도를 측정하던 중 데이터의 신뢰도를 낮추는 문제가 발견되었다. 로컬 환경에서 MySQL은 평균 $1.05ms$ 대의 응답 속도를 보인 반면, Redis는 로그상 지속적으로 **0ms**가 기록되는 현상이 확인되었다.

아무리 In-Memory DB가 빠르다고 해도 물리적인 네트워크 왕복 시간(RTT)과 직렬화(Serialization) 비용이 존재하므로 소요 시간이 '0'이 될 수는 없다. 이는 Redis의 성능이 측정 범위를 넘어설 만큼 빠르기 때문이 아니라, **측정 도구의 해상도가 대상의 속도를 따라잡지 못한 상황**이다.

## OS 타이머 인터럽트와 System.currentTimeMillis()의 해상도

문제의 원인은 Java의 시간 측정 API인 `System.currentTimeMillis()`가 가진 정밀도(Precision)와 해상도(Resolution)의 한계에 있다. 이 메서드는 호출 시점의 운영체제(OS) 시스템 시각을 밀리초($ms$) 단위의 `long` 타입 정수로 반환한다.

> **[Time Resolution](https://www.google.com/search?q=Time+Resolution+in+OS)이란?**  
> 측정 도구가 구분할 수 있는 최소 시간 단위를 의미한다. `currentTimeMillis()`는 $1ms$ 단위의 해상도를 가지므로, 원칙적으로 $1ms$보다 짧은 시간은 측정 및 표현이 불가능하다.

특히 Windows OS 환경에서는 시스템 타이머 인터럽트 주기에 따라 정밀도가 약 $10\sim15ms$ 단위로 떨어지기도 한다. 따라서 마이크로초($\mu s$) 단위로 수행되는 연산을 측정하기에는 부적합하다.

## 정수형 변환 과정에서의 데이터 소실

Redis의 단순 Key-Value 조회는 보통 수십~수백 마이크로초($\mu s$) 내외로 처리된다. 만약 실제 소요 시간이 $0.17ms$라 하더라도, `currentTimeMillis()`가 반환하는 값은 정수형이기 때문에 **0으로 버림(Truncation) 처리**된다.

이 현상은 벤치마크 결과의 왜곡을 초래하며, 성능 차이를 정량적으로 증명해야 하는 실험의 목적을 저해한다.

## 단건 측정의 한계와 System.nanoTime()의 고려

이 문제를 해결하기 위해 `System.nanoTime()`을 사용하는 방안을 고려했다. `nanoTime()`은 나노초($ns$) 단위의 고해상도 측정을 지원하므로 $0.17ms$와 같은 미세한 시간도 포착할 수 있다.

하지만 이번 벤치마크의 목적은 '단건 실행의 정밀한 프로파일링'이 아니라 '애플리케이션 레벨에서 체감하는 평균 처리량'을 비교하는 것이다. 따라서 나노초 단위의 측정보다는 **전체 흐름의 평균을 구하는 방식**이 더 적합하다고 판단했다.

## 배치 단위 역산(Inverse Calculation) 방식을 통한 오차 상쇄

최종적으로 단건 실행 시간을 측정하는 방식에서, **배치(Batch) 단위의 총 소요 시간을 측정한 뒤 실행 횟수만큼 나누는 '평균 역산 방식'**으로 설계를 변경했다.

~~~java
// AS-IS: 단건 반복 측정 (0ms 발생 및 오차 누적)
for (int i = 0; i < 1000; i++) {
    long start = System.currentTimeMillis();
    redisTemplate.opsForValue().get(key);
    long end = System.currentTimeMillis(); 
    // 실제 수행 시간이 1ms 미만일 경우 0으로 기록됨
}

// TO-BE: 배치 단위 총량 측정 후 평균 역산
long batchStart = System.currentTimeMillis();
for (int i = 0; i < 1000; i++) {
    redisTemplate.opsForValue().get(key);
}
long batchEnd = System.currentTimeMillis();

// 전체 소요 시간(ms) / 실행 횟수 = 평균 레이턴시(ms)
double average = (batchEnd - batchStart) / 1000.0;
~~~

이 방식은 개별 연산의 미세한 변동성을 상쇄하고, OS 타이머의 정밀도 한계보다 훨씬 긴 시간(예: $1,000$회 실행 시 약 $170ms$)을 측정하므로 오차를 최소화할 수 있다.

## 0ms에서 0.17ms로: 유효한 평균 레이턴시 확보

측정 방식을 변경한 후 재실험을 수행한 결과, Redis의 조회 성능은 0ms가 아닌 평균 **$0.17ms$** 수준임이 확인되었다.

> **실험 결과 요약**    
> MySQL의 평균 응답 속도가 $1.05ms$인 것에 비해, Redis는 $0.17ms$로 측정되었다. 이는 Redis가 MySQL 대비 약 **6.6배** 빠른 읽기 성능을 보임을 정량적으로 입증한다.

## 평균의 함정과 Coordinated Omission

배치 단위 평균 역산 방식은 실무적으로 유효한 근사치를 제공하지만, **Coordinated Omission(지연 시간 누락)** 문제를 완전히 해결하지는 못한다. 특정 시점에 GC(Garbage Collection) 등으로 튀는 지연 시간(Jitter)이 발생할 경우 평균값에 희석되어 보이지 않을 수 있다.

## 정밀 벤치마킹을 위한 JMH 도입 검토

따라서 더욱 엄밀한 측정이 요구되는 환경이라면 Java 진영의 표준 벤치마크 프레임워크인 **JMH(Java Microbenchmark Harness)** 도입을 고려해야 한다.

> **[JMH](https://www.google.com/search?q=Java+Microbenchmark+Harness)란?**  
> OpenJDK에서 개발한 벤치마크 프레임워크다. JVM의 웜업(Warm-up), JIT 컴파일 최적화 등을 자동으로 제어하며, 평균뿐만 아니라 P99, P99.9와 같은 백분위 지표를 제공하여 '꼬리 지연(Tail Latency)'까지 정밀하게 분석할 수 있다.

이번 프로젝트에서는 전체적인 성능 경향성 파악이 목적이었기에 구현 비용이 낮은 평균 역산 방식을 택했으나, 시스템의 한계 상황(Tail Latency)을 테스트할 때는 측정 도구의 선택 또한 중요한 트레이드오프임을 인지해야 한다.

---
**🔗 GitHub Repository:** [bh1848/mysql-redis-benchmark](https://github.com/bh1848/mysql-redis-benchmark)