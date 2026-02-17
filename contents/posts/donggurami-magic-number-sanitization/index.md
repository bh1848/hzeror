---
title: "Magic Number 검증 및 전역 XSS 방어 체계 구축"
description: "확장자 변조를 통한 Web Shell 공격과 XSS 취약점을 방지하기 위해 파일 헤더 검증 및 Spring InitBinder를 활용한 전역 Sanitization 시스템을 구축한다."
date: 2026-02-17
tags:
  - Java
  - Spring Boot
  - Security
  - XSS
  - 파일 업로드
  - 트러블 슈팅
---

## 외부 입력값 신뢰의 위험성과 보안 홀

웹 애플리케이션 보안의 제1원칙은 외부에서 유입되는 모든 데이터는 오염되었다고 가정하는 것이다. 하지만 실무에서는 개발 편의성이나 구현상의 실수로 인해 이 원칙이 무너지며 치명적인 보안 사고로 이어진다.     

가장 흔한 사례는 파일 업로드 취약점이다. 단순히 파일명 확장자(`.jpg`, `.png`)만 검사할 경우, 공격자가 악성 스크립트(`shell.php`)의 확장자만 위장하여 업로드하면 서버 제어권을 탈취당하는 **Web Shell** 공격에 노출된다.     

또한, 게시글이나 댓글 입력란에 `<script>` 태그를 삽입하는 **XSS** 공격은 서비스 신뢰도를 파괴한다. 모든 컨트롤러 메서드마다 수동으로 필터링 로직을 넣는 것은 휴먼 에러를 유발하므로, 시스템 입구에서 동작하는 **전역 방어 체계**가 필요하다.        

## 파일 무결성 검증: 확장자가 아닌 바이트 헤더 분석

파일의 확장자는 OS나 사용자가 임의로 붙인 라벨일 뿐, 실제 데이터의 형식을 보장하지 않는다. 따라서 파일의 **바이너리 헤더(Binary Header)**를 직접 열어 실제 내용을 대조해야 한다.        

### FileSignatureValidator를 통한 매직 넘버 검증

모든 파일 포맷은 시작 부분에 고유한 16진수 값인 **Magic Number**를 가진다. JPEG는 `FF D8 FF`, PNG는 `89 50 4E 47`로 시작하는 특성을 활용해 아래와 같이 검증기를 구현했다.       

~~~java
@Component
public class FileSignatureValidator {

    // 허용할 파일 타입별 매직 넘버 매핑
    private static final Map<String, List<String>> MAGIC_NUMBERS = Map.of(
        "JPG", List.of("FFD8FF"),
        "PNG", List.of("89504E47")
    );

    public void validate(MultipartFile file) {
        try (InputStream inputStream = file.getInputStream()) {
            // 성능을 위해 파일 전체를 읽지 않고, 헤더 판별에 필요한 앞부분 8바이트만 부분 로드한다.
            byte[] headerBytes = new byte[8];
            if (inputStream.read(headerBytes) == -1) {
                throw new FileException(FileErrorResult.EMPTY_FILE);
            }

            String fileHex = bytesToHex(headerBytes);
            
            // 추출한 16진수 시그니처와 허용 리스트(Allow List) 비교
            if (!isValidSignature(fileHex)) {
                throw new FileException(FileErrorResult.INVALID_FILE_TYPE);
            }
        } catch (IOException e) {
            throw new FileException(FileErrorResult.FILE_READ_ERROR);
        }
    }
}
~~~

이 구조는 확장자를 조작하더라도 바이너리 데이터가 실제 이미지 형식이 아니면 즉시 차단한다. 특히 `InputStream`을 통해 필요한 바이트만 읽어내므로 대용량 파일 검사 시에도 **레이턴시(Latency)** 증가를 최소화했다.        

> **[Web Shell](https://gemini.google.com/app/b46838d855943e81?hl=ko)이란?**        
> 공격자가 원격에서 웹 서버를 제어할 수 있도록 업로드하는 악성 스크립트 파일이다. 서버가 위장된 파일을 실행 가능한 코드로 인식하게 되면 시스템 권한을 탈취당할 수 있다. 본문에서는 이를 바이트 단위 검사로 원천 차단했다.       

## Global Sanitization: 라이프사이클 개입을 통한 자동 방어

XSS 방어를 위해 모든 DTO 필드마다 수동으로 태그를 제거하는 방식은 유지보수가 불가능하다. Spring MVC의 데이터 바인딩 라이프사이클에 개입하여, 데이터가 Java 객체에 매핑되기 직전에 일괄 처리하는 전략을 채택했다.        

### @InitBinder와 Jsoup 기반 전역 처리

Spring의 `WebDataBinder`에 커스텀 에디터를 등록하면 모든 `String` 타입 입력값에 대해 전처리가 가능하다.     

~~~java
@RestControllerAdvice
public class SanitizationBinder {

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // 모든 String 타입 입력에 대해 HTML 태그 제거 처리
        binder.registerCustomEditor(String.class, new StringSanitizer());
    }

    public static class StringSanitizer extends PropertyEditorSupport {
        @Override
        public void setAsText(String text) {
            if (text == null) {
                setValue(null);
            } else {
                // Jsoup 라이브러리를 사용하여 화이트리스트 기반 태그 제거
                String sanitized = Jsoup.clean(text, Safelist.none());
                setValue(sanitized);
            }
        }
    }
}
~~~

이 설정을 통해 개발자가 보안 로직 작성을 누락하더라도, 컨트롤러에 도달하는 데이터는 이미 정제된 상태가 된다. 이는 **공통 관심사(Cross-cutting Concern)**를 인프라 계층으로 격리하여 비즈니스 로직의 순수성을 지키는 설계다.     

> **논리적 100% 차단의 근거와 트레이드오프**        
> 본 아키텍처가 확장자 변조 및 누락 사고를 **논리적으로 100% 차단**한다고 단언할 수 있는 근거는 두 가지다.          
> 첫째, 허용된 포맷 외에는 모두 거부하는 **White-list** 방식을 채택했기 때문이다.       
> 둘째, 개별 로직이 아닌 Spring의 데이터 바인딩 직전 단계인 **`@InitBinder`**에 전역 적용함으로써 개발자의 코드 누락 가능성을 원천 봉쇄했기 때문이다.   
>   
> 물론 이미지 파일 내부에 텍스트 형태로 악성 코드를 삽입하는 정교한 스테가노그래피 공격까지 막으려면 추가적인 스캔 엔진이 필요하겠으나, 애플리케이션 레벨에서 발생할 수 있는 휴먼 에러와 일반적인 변조 공격에 대해서는 완전한 방어 체계를 구축했다고 평가할 수 있다.        

## 보안 아키텍처 강화에 따른 시스템 무결성 확보

바이트 레벨 검증과 바인딩 시점의 전역 처리를 통해 서비스 보안성을 극대화했다. 확장자 변조 공격은 시스템 입구에서 차단되며, 신규 API 개발 시 발생할 수 있는 보안 처리 누락 가능성을 아키텍처 레벨에서 제거했다. 결과적으로 보안 로직과 비즈니스 로직을 분리함으로써 코드 품질과 안전성을 동시에 확보했다.        

---
**🔗 GitHub Repository:** [bh1848/USW-Circle-Link-Server](https://github.com/bh1848/USW-Circle-Link-Server)