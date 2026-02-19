---
title: "SameSite 정책 이원화로 크로스 도메인 쿠키 유실 해결하기"
description: "CORS 환경에서 발생하는 리프레시 토큰(Cookie) 차단 문제를 해결하기 위해, HttpOnly 보안을 유지하며 Set-Cookie 헤더를 직접 제어하고 환경별 정책을 동적으로 이원화한 아키텍처를 공유한다."
date: 2026-02-19
tags:
  - Java
  - Spring Security
  - JWT
  - CORS
  - Cookie
  - 트러블 슈팅
series: "동구라미 개발기"
---

## 크로스 도메인 분리에 따른 쿠키 전송 차단 현상

프론트엔드와 백엔드의 도메인이 완전히 분리된 크로스 도메인(Cross-Domain) 환경에서 인증 파이프라인을 구축하며 세션 유실 문제를 마주했다. 사용자가 로그인을 완료하고 서버가 리프레시 토큰을 쿠키에 담아 응답했음에도, 클라이언트의 후속 API 요청에서는 해당 쿠키가 서버로 전달되지 않는 현상을 확인했다.

이 현상은 구글 크롬을 비롯한 모던 브라우저의 강화된 보안 정책 때문이다. 브라우저는 타 도메인 간의 요청(Cross-site Request)에서 CSRF 공격을 방지하기 위해 쿠키의 기본 정책을 `SameSite=Lax`로 강제한다. 즉, 명시적으로 제어하지 않는 한 서드파티 컨텍스트에서는 쿠키가 자동으로 버려지게 되어 인증 갱신 메커니즘이 붕괴되는 구조적 한계가 있다.

## 원시 문자열 포맷팅을 통한 Low-Level 헤더 제어

이 문제를 해결하려면 쿠키 속성을 `SameSite=None; Secure`로 덮어씌워야 한다. 나는 Spring 프레임워크가 제공하는 추상화된 `ResponseCookie` 객체를 사용하는 대신, `HttpServletResponse.setHeader`를 통해 `Set-Cookie` 속성을 원시 문자열(Raw String)로 직접 포맷팅하는 **Low-Level 제어 방식**을 채택했다.

~~~java
@Component
public class JwtProvider {

    // 로컬과 운영 환경의 프로퍼티를 분리하여 동적으로 주입받는다.
    @Value("${security.cookie.secure:true}")
    private boolean cookieSecure;

    @Value("${security.cookie.same-site:Lax}")
    private String cookieSameSite;

    // ... (중략) ...

    /**
     * 리프레시 토큰을 HttpOnly 쿠키로 설정 (SameSite/ Secure 프로퍼티 반영)
     */
    private void setRefreshTokenCookie(HttpServletResponse response, String refreshToken) {
        int maxAge = (int) (REFRESH_TOKEN_EXPIRATION_TIME / 1000); // 604,800초 (7일)
        
        // 환경 변수 기반의 동적 쿠키 속성 조립
        String sameSite = (cookieSameSite == null || cookieSameSite.isBlank()) ? "Lax" : cookieSameSite;
        String securePart = cookieSecure ? "; Secure" : "";
        
        // 프레임워크의 캡슐화를 우회하고 문자열 포맷팅으로 직접 헤더에 인가한다.
        String cookieValue = String.format("refreshToken=%s; Path=/; HttpOnly; Max-Age=%d; SameSite=%s%s",
                refreshToken, maxAge, sameSite, securePart);
        response.setHeader("Set-Cookie", cookieValue);
    }
}
~~~

프레임워크의 추상화된 객체를 우회한 이유는 시스템의 명시성과 통제권을 완전히 확보하기 위해서다. 톰캣(Tomcat) 버전에 따라 `SameSite` 속성 매핑이 무시되거나 오작동하는 엣지 케이스(Edge Case)를 원천 차단하고, 문자열 수준에서 브라우저가 요구하는 스펙과 정확히 일치하는 HTTP 표준 헤더를 응답에 꽂아 넣기 위해 이 방식을 설계했다.

> **[SameSite](https://gemini.google.com/app/b46838d855943e81?hl=ko)**  
> 웹 애플리케이션에서 CSRF 공격을 방지하기 위해 쿠키가 같은 웹 사이트 요청에서만 전송되도록 제어하는 속성이다. 나는 크로스 도메인 환경에서의 정상적인 통신을 위해 이를 `None`으로 해제하고, 데이터 탈취를 막기 위해 `HttpOnly` 속성을 조합했다.

## 보안 설정 이원화의 의사결정과 트레이드오프

브라우저 규격상 `SameSite=None`을 사용하려면 통신 암호화를 보장하는 `Secure` 속성이 반드시 함께 선언되어야 한다. 하지만 이는 로컬 개발 환경(HTTP)에서는 쿠키가 생성조차 되지 않는 개발 사이클의 병목을 유발한다.

나는 이 딜레마를 해결하기 위해 `@Value` 어노테이션을 활용하여 **실행 환경(Profile)에 따른 쿠키 정책 이원화 아키텍처**를 구축했다. 로컬(Dev) 환경에서는 `--security.cookie.same-site=Lax`, `--security.cookie.secure=false`를 주입하여 HTTP 환경에서의 디버깅을 허용하고, 운영(Prod) 환경에서는 `None`과 `true`를 주입하여 보안 규격을 강제했다.

물론, `SameSite=None` 설정은 본질적으로 **CSRF(크로스 사이트 요청 위조)** 공격 표면을 외부로 노출하는 명확한 **트레이드오프(Trade-off)**를 가진다. 그럼에도 내가 이 방식을 고수한 실무적 근거는 다음과 같다. 해당 쿠키는 상태를 변경하는 비즈니스 API 로직에 직접 사용되는 것이 아니라, 오직 `AccessToken` 재발급 엔드포인트에서만 제한적으로 사용되기 때문이다. 나는 클라이언트의 로컬 스토리지에 토큰을 저장하여 XSS(크로스 사이트 스크립팅) 공격에 노출되는 치명적인 결함보다, 재발급 API에 한정된 CSRF 리스크를 수용하고 `HttpOnly` 방어막을 챙기는 것이 시스템 보안 측면에서 더 우위에 있다고 판단했다.

## 3PC Phase-Out 정책에 따른 실무 임계점 인지

이 아키텍처는 현재의 브라우저 스펙 내에서 프론트-백엔드 간 원활한 인증 인프라를 $100\%$ 보장한다. 하지만 나는 설계자로서 이 방식이 영구적이지 않은 시한부 해결책이라는 구조적 한계점(Threshold)을 명확히 인지하고 있다.

애플(Safari ITP)을 시작으로 구글 크롬까지 **서드파티 쿠키 전면 차단(3rd Party Cookie Phase-Out)** 정책을 추진하고 있다. 만약 브라우저 레벨에서 서드파티 컨텍스트의 `Set-Cookie` 자체를 하드 드롭(Hard Drop)하는 시대가 도래하면, `SameSite=None` 속성을 통한 우회는 더 이상 유효하지 않게 된다. 

이러한 보안 임계점을 마주할 경우, 현재의 헤더 조작 로직에 의존하는 대신 인프라 레벨의 리팩토링이 필요하다. Nginx나 AWS API Gateway 등 리버스 프록시(Reverse Proxy)를 앞단에 배치하여 클라이언트와 서버를 동일한 루트 도메인(First-Party Context)으로 묶어내는 방식(BFF 패턴)으로 아키텍처를 진화시켜야만 지속 가능한 인증 체계를 유지할 수 있다.