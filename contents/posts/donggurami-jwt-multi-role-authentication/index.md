---
title: "전략 패턴으로 OCP를 준수하는 다중 역할 인증 아키텍처"
description: "if-else 분기로 점철된 인증 로직을 전략 패턴(Strategy Pattern)으로 리팩토링하여 개방-폐쇄 원칙(OCP)을 달성한 과정을 공유한다."
date: 2026-02-17
tags:
  - Java
  - Spring Security
  - JWT
  - 디자인 패턴
  - OCP
  - 트러블 슈팅
---

## 다중 도메인 환경에서의 인증 로직 파편화 문제

`Admin`, `Leader`, `User` 등 서로 다른 도메인 엔티티가 공존하는 서비스에서, 초기 인증 로직은 단일 서비스 클래스 내에 **if-else** 분기문으로 구현되기 쉽다. 이러한 절차지향적 코드는 역할이 추가될 때마다 메인 로직을 수정하게 만들어 시스템의 안정성을 해친다.      

~~~java
// Legacy Code: 역할이 추가될수록 비즈니스 로직이 오염된다.
public UserDetails loadUserByUsername(String username, Role role) {
    if (role.equals(Role.ADMIN)) {
        return adminRepository.findByUsername(username);
    } else if (role.equals(Role.LEADER)) {
        // Leader 도메인 특화 로직...
    } else {
        // User 도메인 특화 로직...
    }
}
~~~

이러한 구조는 **단일 책임 원칙(SRP)**과 **개방-폐쇄 원칙(OCP)**을 모두 위반한다. 새로운 역할(예: `Professor`)이 추가될 때마다 핵심 인증 코드를 수정해야 하며, 이는 곧 **회귀 테스트 비용 증가**와 **유지보수성 저하**로 직결된다. 이를 해결하기 위해 **전략 패턴(Strategy Pattern)**을 도입하여 역할별 인증 책임을 물리적으로 분리하기로 했다.      

## RoleBasedUserDetailsService 인터페이스 설계를 통한 책임 분리

가장 먼저 구체적인 구현체에 의존하지 않도록 상위 수준의 인터페이스를 정의한다. Spring Security의표준 인터페이스인 `UserDetailsService`를 상속받되, 각 구현체가 자신이 담당하는 **역할(Role)을 식별**할 수 있는 메서드를 추가한다.       

~~~java
public interface RoleBasedUserDetailsService extends UserDetailsService {
    /**
     * 해당 서비스 구현체가 처리할 수 있는 역할(Role)을 반환한다.
     * 전략 선택의 식별자(Key)로 사용된다.
     */
    Role getRole();
}
~~~

각 역할별 서비스는 이 인터페이스를 구현(Implements)하여 격리된 클래스로 분리된다. `AdminService`, `LeaderService` 등은 이제 서로의 존재를 알 필요가 없다.       

~~~java
@Service
@RequiredArgsConstructor
public class CustomLeaderDetailsService implements RoleBasedUserDetailsService {

    private final LeaderRepository leaderRepository;

    @Override
    public Role getRole() {
        return Role.LEADER;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Leader leader = leaderRepository.findByMemberId(username)
                .orElseThrow(() -> new UsernameNotFoundException("Leader not found"));
        return new CustomLeaderDetails(leader);
    }
}
~~~

## Spring List Injection을 활용한 동적 전략 바인딩

이제 요청된 역할에 맞는 적절한 전략(Service)을 선택해 줄 **Context(Manager)**가 필요하다. Spring Framework의 강력한 의존성 주입(DI) 기능을 활용하면, 별도의 등록 로직 없이도 유연한 구조를 만들 수 있다.        

핵심은 `List<RoleBasedUserDetailsService>` 주입이다. Spring은 해당 인터페이스를 구현한 모든 빈(Bean)을 자동으로 리스트에 담아 주입해 준다.      

~~~java
@Service
@RequiredArgsConstructor
public class UserDetailsServiceManager {

    private final List<RoleBasedUserDetailsService> userDetailsServices;

    public UserDetails loadUserByUsername(String username, Role role) {
        UserDetailsService service = getService(role);
        return service.loadUserByUsername(username);
    }

    private UserDetailsService getService(Role role) {
        // 주입된 서비스 리스트를 순회하며 역할(Key)이 일치하는 구현체를 찾는다.
        return userDetailsServices.stream()
                .filter(service -> service.getRole() == role)
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Unsupported role: " + role));
    }
}
~~~

> **[OCP (개방-폐쇄 원칙)](https://gemini.google.com/app/b46838d855943e81?hl=ko)란?**   
> 소프트웨어 엔티티(클래스, 모듈 등)는 **확장에는 열려 있어야 하고, 변경에는 닫혀 있어야 한다**는 설계 원칙이다. 위 코드에서 새로운 `Professor` 역할을 추가하려면 기존 `Manager` 코드는 단 한 줄도 수정할 필요 없이, `CustomProfessorService` 클래스만 새로 만들면 된다.        

## 리스트 순회 방식의 성능 효율성 및 아키텍처 검증

리스트를 순회하며 적합한 서비스를 찾는 방식에 대해 성능 우려가 있을 수 있다. `List`를 Stream으로 순회하는 로직은 $O(N)$의 시간 복잡도를 가지며, `Map<Role, Service>` 구조를 사용하면 $O(1)$ 조회가 가능하다.        

하지만 인증 서비스의 역할(Role) 개수는 일반적으로 10개 미만으로 유지되므로, $N$ 값이 매우 작다. 따라서 Map 초기화의 복잡성을 감수하는 것보다, **Spring의 자동 리스트 주입을 활용하여 코드 간결성과 OCP 준수를 우선시하는 것**이 실무적으로 더 합리적인 선택이다.        

이 아키텍처를 `JwtFilter`와 인증 파이프라인에 적용함으로써 비즈니스 로직 내의 `if-else` 분기를 완전히 제거하고, 각 도메인 개발자가 독립적으로 인증 로직을 확장할 수 있는 환경을 구축했다.       

---
**🔗 GitHub Repository:** [bh1848/USW-Circle-Link-Server](https://github.com/bh1848/USW-Circle-Link-Server)