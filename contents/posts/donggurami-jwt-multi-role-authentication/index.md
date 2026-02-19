---
title: "책임 연쇄 패턴으로 OCP를 준수하는 UUID 기반 다중 인증"
description: "JWT의 UUID만으로 다중 도메인(Admin, Leader, User)을 인증해야 하는 상황에서, 책임 연쇄 패턴과 List Injection을 활용하여 OCP를 준수한 아키텍처 설계 과정을 공유한다."
date: 2026-02-17
tags:
  - Java
  - Spring Security
  - JWT
  - 디자인 패턴
  - OCP
  - 트러블 슈팅
series: "동구라미 개발기"
---

## 다중 도메인 환경에서의 인증 한계와 로직 파편화

`Admin`, `Leader`, `User` 등 서로 다른 도메인 엔티티(테이블)가 공존하는 서비스에서 인증 파이프라인을 설계하며 한 가지 문제가 있었다. 클라이언트가 제출한 JWT 토큰 Payload에는 식별자인 `UUID`만 들어있을 뿐, 이 유저가 어떤 역할(테이블)에 속하는지 명시되어 있지 않았다.

초기에는 하나의 서비스 클래스 내에서 여러 Repository를 순차적으로 조회하는 `if-else` 분기문으로 구현하기 쉽다. 

~~~java
// Legacy Code: 도메인이 추가될수록 분기문이 무한정 길어진다.
public UserDetails loadUserByUuid(UUID uuid) {
    if (adminRepository.existsById(uuid)) return loadAdmin(uuid);
    else if (leaderRepository.existsById(uuid)) return loadLeader(uuid);
    else return loadUser(uuid);
}
~~~

이러한 절차지향적 구조는 새로운 역할이 추가될 때마다 핵심 인증 코드를 수정하게 만들어 **개방-폐쇄 원칙(OCP)**을 위반한다. 나는 이것을 막기 위해, 요청 처리를 순차적으로 위임하는 **책임 연쇄 패턴(Chain of Responsibility)**의 아이디어를 차용하여 역할별 인증 책임을 분리하기로 결정했다.

## 공통 인터페이스 추출을 통한 인증 책임 격리

가장 먼저 구체적인 구현체에 의존하지 않도록 상위 수준의 인터페이스를 정의했다. Spring Security의 의존성을 줄이고 프로젝트 상황에 맞게 `UUID`를 식별자로 받는 커스텀 인터페이스를 설계했다.

~~~java
public interface RoleBasedUserDetailsService {
    UserDetails loadUserByUuid(UUID uuid);
}
~~~

이후 각 역할별 서비스는 이 인터페이스를 구현(Implements)하여 격리된 클래스로 분리했다. 예를 들어 `CustomLeaderDetailsService`는 `Leader` 도메인의 비즈니스 로직에만 집중하며, 다른 도메인의 존재 자체를 알 필요가 없다.

~~~java
@Service
@RequiredArgsConstructor
@Slf4j
public class CustomLeaderDetailsService implements RoleBasedUserDetailsService {

    private final LeaderRepository leaderRepository;

    @Override
    public UserDetails loadUserByUuid(UUID uuid) {
        Leader leader = leaderRepository.findByLeaderUUID(uuid)
                .orElseThrow(() -> new UserException(ExceptionType.USER_NOT_EXISTS));

        UUID clubUUID = leaderRepository.findClubUUIDByLeaderUUID(leader.getLeaderUUID())
                .orElseThrow(() -> new UserException(ExceptionType.USER_NOT_EXISTS));

        return new CustomLeaderDetails(leader, clubUUID);
    }
}
~~~

## List Injection과 예외 기반의 동적 위임 매핑

이제 들어온 `UUID`를 처리할 수 있는 적절한 전략(Service)을 찾아 위임해 줄 매니저 클래스가 필요하다. 나는 Spring Framework의 강력한 의존성 주입(DI) 기능을 활용하여, 해당 인터페이스를 구현한 모든 빈(Bean)을 `List`로 자동 주입받도록 구성했다.

핵심 로직은 **예외(Exception)를 활용한 제어 흐름(Control Flow)**이다.

~~~java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserDetailsServiceManager {

    private final List<RoleBasedUserDetailsService> userDetailsServices;

    public UserDetails loadUserByUuid(UUID uuid) {
        // 주입된 서비스 리스트를 순회하며 조회를 시도(Try)한다.
        for (RoleBasedUserDetailsService service : userDetailsServices) {
            try {
                return service.loadUserByUuid(uuid);
            } catch (UserException ignored) {
                // 해당 도메인에 유저가 없어 예외가 터지면 무시하고 다음 서비스로 위임한다.
            }
        }
        // 모든 서비스에서 유저를 찾지 못했을 경우 최종 예외를 발생시킨다.
        throw new UserException(ExceptionType.USER_NOT_EXISTS);
    }
}
~~~

> **[OCP (개방-폐쇄 원칙)](https://gemini.google.com/app/b46838d855943e81?hl=ko)**  
> 소프트웨어 엔티티는 **확장에는 열려 있어야 하고, 변경에는 닫혀 있어야 한다**는 객체지향 설계 원칙이다. 본문의 매니저 클래스는 향후 새로운 도메인 서비스가 추가되더라도 `for`문 로직을 단 한 줄도 수정할 필요가 없으므로 이 원칙을 충실히 이행한다.

## 예외 제어 흐름의 트레이드오프와 실무 적용 임계점

내가 설계한 이 아키텍처는 비즈니스 로직 내의 `if-else` 분기를 완전히 제거하여 도메인 간의 결합도를 완벽히 끊어냈다. 하지만 설계자로서 이 코드가 가진 명확한 안티 패턴(Anti-pattern)과 임계점 역시 인지하고 있다.

일반적으로 자바에서 **예외를 제어 흐름으로 사용하는 것(Exception for Control Flow)**은 권장되지 않는다. 예외가 발생할 때마다 JVM은 스택 트레이스(Stack Trace)를 캡처하는 무거운 연산을 수행하기 때문이다. 유저가 `User` 테이블에 있다면, 앞서 실행된 `Admin`, `Leader` 서비스에서 터지는 `UserException`의 스택 캡처 비용이 고스란히 인증 레이턴시에 누적된다.

그럼에도 불구하고 나는 두 가지 실무적 근거로 이 방식을 현재 단계의 최종 구조로 채택했다. 
첫째, 현재 서비스의 사용자 도메인은 단 3개(`Admin`, `Leader`, `User`)로 고정되어 있어 루프 횟수와 예외 발생 비용이 시스템 성능에 미치는 영향이 극히 미미하다. 
둘째, JWT에 역할(Role) Claim을 명시하지 않기로 한 기존의 토큰 발급 스펙을 유지하면서도 OCP를 달성하기 위해, 코드의 간결성(유지보수성)을 성능보다 우선순위에 둔 의도적인 **트레이드오프** 결단이었다.

물론 트래픽이 고도화되거나 도메인이 확장된다면 이 지점은 반드시 CPU 병목을 유발할 것이다. 그 시점에는 JWT Payload에 `Role` 식별자를 추가 발급하도록 스펙을 변경하고, 리스트 순회 방식이 아닌 `Map<Role, Service>` 기반의 $O(1)$ 라우팅 아키텍처로 리팩토링하여 임계점을 돌파할 것이다.