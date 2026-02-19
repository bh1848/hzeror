---
title: "Redis RTR로 JWT 보안 한계 극복하기"
description: "JWT 탈취의 구조적 한계를 극복하기 위해 Access Token 수명을 단축하고, Redis TTL 기반의 Refresh Token 로테이션(RTR)을 적용하여 서버 통제권을 확보한 과정을 기술한다."
date: 2026-02-19
tags: 
  - Security
  - JWT
  - Redis
  - Spring Boot
  - 트러블 슈팅
series: "수챗 개발기"
---

## Stateless 인증의 역설과 통제권 상실

수챗(Suchat) 프로젝트의 인증 아키텍처를 설계하며 무상태(Stateless) 기반의 JWT를 도입했다. 하지만 JWT는 한 번 발급되면 탈취되더라도 만료 전까지 서버에서 무효화할 수 없다.

이 현상은 암호화된 토큰 자체가 검증 수단이 되어, 서버가 클라이언트의 세션 상태에 개입할 수 있는 물리적 연결 고리가 단절되었기 때문에 발생하는 것이다. 탈취된 Access Token(AT)이 악용되는 것을 막으면서도 서버의 제어 통제권을 회복할 하이브리드 토큰 아키텍처가 필요하다.

## Access Token 수명 단축과 Refresh Token 로테이션(RTR)

AT의 탈취 리스크를 방어하는 가장 확실한 방법은 수명을 극단적으로 줄이는 것이다. `JwtProvider` 클래스에서 `ACCESS_TOKEN_EXPIRATION_TIME`을 30분(1800000ms)으로 단축했다. 사용자가 30분마다 다시 로그인해야 하는 불편함을 방지하기 위해 2주(1209600000ms) 수명의 Refresh Token(RT)을 발급하여 Redis에 매핑하여 저장했다.

> **[Refresh Token Rotation (RTR)](https://gemini.google.com/app/b46838d855943e81?hl=ko)**  
> 엑세스 토큰 갱신 시 기존 리프레시 토큰을 일회용으로 간주하여 폐기하고 새로운 리프레시 토큰을 발급하는 기법이다. 탈취된 리프레시 토큰의 반복적인 재사용 공격(Replay Attack)을 차단할 수 있다.

나는 토큰 갱신 요청(`renewToken`)이 들어올 때마다 기존 RT를 삭제하고 새로운 RT를 덮어씌우는 RTR 로직을 `JwtService`에 구현했다.


## Redis TTL 기반의 화이트리스트 파이프라인 설계

서버가 개입하는 로직은 Redis TTL 체계를 통해 완성했다. 사용자가 로그아웃(`signOut`)하거나 회원 탈퇴(`withdraw`)를 요청하면 즉시 Redis에서 해당 RT 데이터를 찾아 하드 딜리트(`delete`)했다.

토큰 재발급을 요청할 때, 전달된 RT의 암호학적 서명이 유효하더라도 Redis에 해당 키(`hasKey`)가 존재하지 않으면 강제로 `RefreshTokenException`을 발생시켜 비정상 세션을 끊어내는 방어 로직을 작성했다.

~~~java
@Component
public class JwtProvider {
    // ... 생략 ...

    // 리프레시 토큰 검증: Redis 내부 존재 여부(hasKey)를 통한 서버 통제권 확보
    public boolean validateRefreshToken(String refreshToken) throws RefreshTokenException {
        try {
            Jws<Claims> claims = Jwts.parserBuilder().setSigningKey(secretKey).build().parseClaimsJws(refreshToken);
            
            // 1차 검증: 토큰 자체의 만료 기한 확인
            if (claims.getBody().getExpiration().before(new Date())) {
                throw new RefreshTokenException(ExceptionType.REFRESH_TOKEN_EXPIRED);
            }
            
            // 2차 검증: Redis 화이트리스트 대조를 통한 강제 로그아웃/만료 세션 식별
            String redisKey = REFRESH_TOKEN_PREFIX + refreshToken;
            Boolean hasKey = redisTemplate.hasKey(redisKey);
            if (!Boolean.TRUE.equals(hasKey)) {
                throw new RefreshTokenException(ExceptionType.INVALID_REFRESH_TOKEN);
            }
            return true;
        } catch (JwtException e) {
            throw new RefreshTokenException(ExceptionType.INVALID_REFRESH_TOKEN);
        }
    }
}
~~~

~~~java
@Service
public class JwtService {
    // ... 생략 ...

    // RTR 적용: 기존 토큰 폐기 및 신규 토큰 발급
    private String replaceRefreshToken(HttpServletResponse response, String oldRefreshToken, String account) {
        // 기존 리프레시 토큰 삭제로 재사용(Replay) 공격 차단
        redisTemplate.delete(JwtProvider.REFRESH_TOKEN_PREFIX + oldRefreshToken);

        // 새로운 리프레시 토큰 생성 및 Redis 저장
        String newRefreshToken = jwtProvider.createRefreshToken();
        jwtProvider.addRefreshTokenToHeaderAndSaveInRedis(response, newRefreshToken, account);
        return newRefreshToken;
    }
}
~~~

## 아키텍처 트레이드오프와 In-memory 캐싱 레이어 분리

이 방법으로 인증망의 무상태성을 최대한 유지하면서도, 유효하지 않은 세션을 즉각 차단하는 서버의 제어권을 성공적으로 확보했다. 

하지만 이 방식은 완전한 Stateless의 장점을 일부 포기하고 Stateful 아키텍처로 회귀한다는 구조적 트레이드오프를 가지고 있다. 토큰 갱신 시점(30분 주기)마다 Redis I/O가 강제 발생하며, 시스템 전반에 네트워크 레이턴시 병목이 생길 수 있다. 이를 해소하기 위해 In-memory 캐싱 레이어 분리가 필요하다.

그럼에도 API 요청마다 DB 검증 쿼리를 날려야 하는 전통적인 세션 방식과 비교하면, 30분에 한 번 발생하는 Redis `hasKey` 조회 연산은 시간 복잡도 $\mathcal{O}(1)$로 처리되므로 성능 저하를 막을 수 있다고 판단했다. 트래픽이 증가할 경우, 현재의 단일 Redis 노드 인프라가 단일 장애점(SPOF)으로 작용할 수 있으므로 Redis Cluster 구축을 통한 가용성 개선이 필요하다.