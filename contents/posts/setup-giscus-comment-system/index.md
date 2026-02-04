---
title: "블로그 댓글 시스템 Giscus 구축 및 연동"
description: "GitHub Discussions API를 활용한 Serverless 댓글 시스템 구현"
date: 2026-02-04
update: 2026-02-04
tags:
  - Gatsby
  - Giscus
series: "Gatsby 블로그 제작기"
---

정적 사이트 생성기(SSG) 기반의 블로그는 별도의 백엔드 데이터베이스가 존재하지 않아 동적인 댓글 기능을 구현하는 데 제약이 있다. 이를 해결하기 위해 기존에는 Utterances(Issues 기반)가 주로 사용되었으나, 권한 관리 및 스레드 구조의 한계가 존재하였다.

이에 대한 대안으로 **GitHub Discussions API**를 활용하는 **Giscus**를 도입하기로 결정하였다. 본 포스팅에서는 Giscus 시스템의 동작 원리를 이해하고, Gatsby 프로젝트의 `blog-config.js`에 이를 연동하는 과정을 정리한다.

---

## GitHub Repository 설정 및 Discussions 활성화

Giscus는 댓글 데이터를 GitHub 저장소의 Discussions 탭에 저장한다. 따라서 해당 기능이 비활성화되어 있다면 API가 정상적으로 동작하지 않는다.



1.  **Repository Settings 접근:** 블로그 소스 코드가 호스팅된 저장소의 `Settings` 탭으로 이동한다.
2.  **Features 설정:** `General` 메뉴 하단의 Features 섹션에서 **Discussions** 항목을 체크(Enable)한다.

이 과정이 선행되지 않으면 Giscus 봇이 댓글 스레드를 생성할 수 없다.

---

## Giscus 애플리케이션 구성

Giscus 앱과 저장소 간의 권한 위임 및 매핑 전략을 설정하기 위해 [Giscus.app](https://giscus.app/ko)에서 설정을 진행하였다.

### 1. 저장소 연결 및 접근 권한

저장소(Repository) 입력란에 `username/repository` 형식으로 대상 저장소를 지정한다. 이때 Giscus가 데이터를 읽고 쓸 수 있도록 해당 저장소는 반드시 **Public(공개)** 상태여야 한다.

### 2. Page - Discussion 매핑 전략

블로그 포스트와 댓글 스레드를 매핑하는 기준을 설정해야 한다. URL 경로가 변경되지 않는 한 가장 직관적인 매핑을 보장하는 **pathname** 방식을 선택하였다.

* **pathname:** 포스트의 URL 경로(slug)를 기준으로 Discussion 제목을 생성한다.
* **URL:** 전체 URL을 기준으로 한다. (프로토콜 변경 시 매핑이 깨질 위험 존재)
* **Title:** `og:title` 태그를 기준으로 한다.

### 3. Category 설정

Discussion이 생성될 카테고리를 지정한다. 일반적인 잡담과 구분하기 위해 `Announcements` 혹은 `General`을 선택한다. 댓글 기능의 안정성을 위해 '이 저장소에서 giscus가 discussion을 생성하고 수정하도록 허용합니다' 옵션을 활성화하여 봇의 권한을 위임하였다.

---

## 클라이언트 연동 및 Config 주입

설정이 완료되면 Giscus 사이트 하단에 `<script>` 태그 형태의 코드가 생성된다. 여기서 핵심 식별자인 `repoId`와 `categoryId`를 추출하여 프로젝트 설정 파일에 주입해야 한다.



### 환경 변수 및 메타 설정 적용

본 프로젝트는 `blog-config.js`에서 전역 설정을 관리한다. 생성된 스크립트에서 `data-` 속성으로 명시된 값들을 추출하여 아래와 같이 적용하였다.

**blog-config.js**

~~~javascript
module.exports = {
  // ... existing config
  giscus: {
    repo: "bh1848/hzeror",
    repoId: "R_kgDOL...", // data-repo-id 값 복사
    category: "Announcements", // data-category 값 복사
    categoryId: "DIC_kwDOL...", // data-category-id 값 복사
    mapping: "pathname",
    strict: "0",
    reactionsEnabled: "1",
    inputPosition: "bottom",
    lang: "ko",
  },
}
~~~

위 설정을 적용하고 배포를 수행하면, 포스트 하단에 Giscus 위젯이 렌더링되며 GitHub 계정을 통한 댓글 작성이 가능해진다.

---

Giscus 도입을 통해 별도의 백엔드 구축 없이 **Serverless** 형태의 댓글 시스템을 확보하였다. 데이터의 소유권이 GitHub 저장소에 귀속되므로 데이터 마이그레이션이 용이하며, 개발자 친화적인 UI/UX를 제공한다는 점에서 기술 블로그에 최적화된 선택이라 판단한다.