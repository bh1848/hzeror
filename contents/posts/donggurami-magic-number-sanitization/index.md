---
title: "Magic Number 검증 및 전역 XSS 방어 체계 구축"
description: "확장자 변조를 통한 Web Shell 공격과 XSS 취약점을 방어하기 위해 파일 헤더 직접 검증 및 Spring InitBinder 전역 Sanitization 시스템을 설계하고 구현했다."
date: 2026-02-17
tags:
  - Java
  - Spring Boot
  - Security
  - XSS
  - 파일 업로드
  - 트러블 슈팅
series: "동구라미 개발기"
---

## 외부 입력값 신뢰의 위험성과 보안 홀

웹 애플리케이션 보안을 설계하며 내가 세운 제1원칙은 **"외부에서 유입되는 모든 데이터는 오염되었다"**고 가정하는 것이었다. 실무 환경에서는 개발 편의성이나 구현상의 작은 실수가 치명적인 보안 사고로 이어지는 경우를 많이 확인했기 때문이다.

특히 파일 업로드 취약점에 주목했다. 단순히 확장자(`.jpg`, `.png`)만 검사할 경우, 공격자가 악성 스크립트(`shell.php`)의 이름만 바꿔 업로드하면 서버 제어권을 탈취당하는 **Web Shell** 공격에 무방비로 노출될 수 있음을 파악했다.

또한, 게시글 입력란 등을 통한 **XSS** 공격 역시 서비스의 신뢰도를 파괴하는 핵심 위협이었다. 모든 컨트롤러마다 수동으로 필터링 로직을 넣는 방식은 반드시 휴먼 에러가 발생할 수밖에 없다고 판단했고, 시스템 입구에서 이를 원천 차단할 수 있는 **전역 방어 체계**를 직접 구축하기로 결정했다.

## 파일 무결성 검증: 확장자가 아닌 바이트 헤더 분석

사용자가 제출한 파일의 확장자는 언제든 조작이 가능하므로 이를 신뢰하지 않기로 했다. 대신 파일의 실제 내용을 대조하기 위해 **바이너리 헤더(Binary Header)**를 직접 읽어 분석하는 로직을 설계했다.

### FileSignatureValidator를 통한 매직 넘버 검증

모든 파일 포맷은 시작 부분에 고유한 16진수 값인 **Magic Number**를 가진다. 나는 JPEG(`FF D8 FF`)와 PNG(`89 50 4E 47`) 등 허용할 포맷의 시그니처를 정의하고, 아래와 같이 검증기를 구현했다.

~~~java
@Component
public class FileSignatureValidator {

    // 허용할 파일 타입별 매직 넘버 매핑 (White-list 방식 채택)
    private static final Map<String, List<String>> MAGIC_NUMBERS = Map.of(
        "JPG", List.of("FFD8FF"),
        "PNG", List.of("89504E47")
    );

    public void validate(MultipartFile file) {
        try (InputStream inputStream = file.getInputStream()) {
            // 파일 전체를 로드하지 않고 헤더 판별에 필요한 앞부분 8바이트만 읽어 성능을 챙겼다.
            byte[] headerBytes = new byte[8];
            if (inputStream.read(headerBytes) == -1) {
                throw new FileException(FileErrorResult.EMPTY_FILE);
            }

            String fileHex = bytesToHex(headerBytes);
            
            // 추출한 시그니처와 허용 리스트(Allow List)를 직접 비교 검증한다.
            if (!isValidSignature(fileHex)) {
                throw new FileException(FileErrorResult.INVALID_FILE_TYPE);
            }
        } catch (IOException e) {
            throw new FileException(FileErrorResult.FILE_READ_ERROR);
        }
    }
}
~~~

이 방식을 통해 확장자를 속인 악성 파일을 바이너리 레벨에서 즉시 차단했다. 특히 메모리에 전체 파일을 올리지 않고 `InputStream`으로 8바이트만 읽어내는 방식을 사용하여, 대용량 파일 검사 시에도 **레이턴시(Latency)** 증가를 최소화했다.

> **[Web Shell](https://gemini.google.com/app/b46838d855943e81?hl=ko)**  
> 공격자가 원격에서 웹 서버를 제어할 수 있도록 업로드하는 악성 스크립트 파일이다. 나는 위장된 파일이 실행 가능한 코드로 인식되는 상황을 원천 차단하기 위해 바이트 단위 검사 로직을 도입했다.

## Global Sanitization: 라이프사이클 개입을 통한 자동 방어

XSS 방어를 위해 DTO 필드마다 수동으로 태그를 제거하는 방식은 유지보수가 불가능하다고 판단했다. 대신 Spring MVC의 데이터 바인딩 라이프사이클에 개입하여, 데이터가 객체에 매핑되기 직전에 일괄 처리하는 전략을 선택했다.

### @InitBinder와 Jsoup 기반 전역 처리

나는 Spring의 `WebDataBinder`에 커스텀 에디터를 등록하여 모든 `String` 타입 입력값에 대해 전처리를 수행하도록 설정했다. 

~~~java
@ControllerAdvice
public class SanitizationBinder {

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(String.class, new PropertyEditorSupport() {
            @Override
            public void setAsText(String text) {
                super.setValue(sanitizeContent(text));
            }
        });
    }

    private String sanitizeContent(String content) {
        if (content == null) return null;

        // XSS 방어를 수행하되, 텍스트 에디터 서식 유지를 위한 필수 태그만 White-list로 허용
        Safelist safelist = Safelist.none()
                .addTags("a", "b", "strong", "i", "em", "u", "ul", "ol", "li", "p", "br")
                .addAttributes("a", "href");

        return Jsoup.clean(content, safelist);
    }
}
~~~

이 설계를 통해 개발자가 보안 로직 작성을 누락하는 실수(휴먼 에러)를 하더라도, 컨트롤러에는 이미 정제된 데이터만 도달하게 만들었다. 이는 **공통 관심사(Cross-cutting Concern)**를 인프라 계층으로 격리하여 비즈니스 로직의 순수성을 지켜낸 결과다.

## 전역 필터링의 실무 적용 임계점과 보안-사용성 트레이드오프

내가 설계한 이 아키텍처는 **확장자 변조 및 보안 로직 누락 사고에 대한 발생 가능성을 설계 단계에서 원천 제거**했다. 특히 XSS 방어에 있어 `Safelist.none()`으로 모든 HTML 태그를 날려버리는 쉬운 선택을 하지 않고, 커스텀 White-list를 구성하여 `<a>`, `<strong>` 등의 서식 태그를 허용했다. 이는 보안을 이유로 사용자의 Rich Text 입력(사용성)을 과도하게 제한하지 않기 위한 의도적인 **트레이드오프** 결단이었다.

하지만 모든 기술이 그렇듯 이 방식 역시 명확한 성능적 임계점을 가진다.

첫째, 모든 `String` 타입 데이터에 Jsoup 파싱을 강제하므로 대용량 텍스트 요청 시 **CPU 연산 오버헤드**가 발생할 수 있다. 악의적인 사용자가 10MB 이상의 더미 문자열을 전송할 경우, 서버는 DOM 파싱 과정에서 과도한 스레드 자원을 소모하게 된다. 

둘째, 이미지 내부에 텍스트 형태로 악성 코드를 삽입하는 **스테가노그래피(Steganography)** 공격처럼 바이너리 데이터 자체가 오염된 경우는 이 검증만으로 완벽히 막을 수 없다.

나는 이러한 한계를 인지하고 있으며, 이를 보완하기 위해 웹 서버 단에서 **Request Payload 크기를 엄격히 제한**하고 있다. 100% 무결한 방어는 존재하지 않지만, 애플리케이션 레벨에서 통제 가능한 **휴먼 에러와 일반적인 Web Shell, XSS 공격만큼은 확고히 방어**할 수 있는 체계를 성공적으로 구축했다.