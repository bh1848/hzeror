---
title: "Gatsby v4 블로그 구축 및 배포 트러블슈팅"
description: "Node.js 런타임 호환성 이슈 해결과 GitHub Pages 배포 자동화"
date: 2026-02-04
update: 2026-02-04
tags:
  - Gatsby
  - Troubleshooting
series: "Gatsby 블로그 제작기"
---

Velog 사용 중 깔끔한 디자인에 이끌려 `gatsby-starter-hoodie` 템플릿으로 개인 블로그를 구축하기로 결정하였다.
하지만 해당 Starter는 **Gatsby v4**를 기반으로 구성되어 있어, 최신 Node.js 런타임 환경에서 **OpenSSL 호환성 문제 및 Dependency 충돌**이 발생하였다.

해당 Starter는 최근 활발한 유지 관리가 이루어지고 있지 않아, 최신 런타임 환경과의 호환성 문제가 발생할 수 있다.

본 포스팅에서는 Gatsby v4 기반 블로그를 구축하며 마주친 문제들과 이를 해결하기 위해 적용한 **런타임 격리, 의존성 정리, GitHub Pages 배포 전략**을 정리한다.

---

## Node.js 런타임 환경 구성 및 버전 격리

Gatsby v4 엔진은 **Node.js v14 환경에서 안정적으로 동작**하는 것으로 알려져 있다.
시스템 전역 Node.js 환경과의 충돌을 방지하고, 프로젝트 단위로 독립적인 실행 환경을 구성하기 위해 **nvm-windows**를 사용하였다.



---

## NVM 설치 중 발생한 Runtime Panic (Windows)

초기 환경 구성 시 `nvm install 14.21.3` 명령어를 통해 자동 설치를 시도하였다.
그러나 `npm-install.exe` 실행 단계에서 바이너리 로드에 실패하며, 아래와 같은 **Runtime Panic(메모리 참조 오류)** 이 발생하였다.

> 본 이슈는 **Windows 환경에서 nvm-windows 사용 시** 확인되었다.

~~~text
panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xc0000005 code=0x0 addr=0x0 pc=0x49800b]
~~~

자동 설치 스크립트 내부에서 사용하는 바이너리가 정상적으로 초기화되지 못한 것으로 판단하였다.

---

## Manual Setup을 통한 Node.js 환경 고정

자동화 스크립트의 불안정성을 우회하기 위해, Node.js 바이너리를 직접 배치하는 **수동 설치 방식**을 채택하였다.

1. **바이너리 확보**
   Node.js 공식 아카이브에서 `v14.21.3` Windows 바이너리를 직접 다운로드한다.

2. **디렉토리 매핑**
   `nvm` Root 경로 하위에 `v14.21.3` 디렉토리를 생성한 뒤, 다운로드한 바이너리를 수동으로 복사한다.

3. **런타임 활성화**
   터미널을 재실행한 후 다음 명령어로 Node.js 버전을 고정한다.

~~~bash
nvm use 14.21.3
~~~

이 방식으로 Runtime Panic 없이 Gatsby 개발 서버를 정상적으로 실행할 수 있었다.

---

## 의존성 충돌 및 Peer Dependency 해결

Node.js 런타임 문제를 해결한 이후에도, Gatsby v4와 일부 플러그인 간 **Peer Dependency 불일치**로 인해 `ERESOLVE` 에러가 발생하며 빌드가 중단되었다.



---

## 의존성 트리 재구성 전략

기존 설치 부산물이 충돌을 유발할 가능성이 높다고 판단하여, 의존성 트리를 **완전히 초기화한 뒤 강제로 재구성**하였다.

~~~bash
# Artifacts Cleanup
Remove-Item -Path .\node_modules -Recurse -Force
Remove-Item -Path .\package-lock.json -Force

# Atomic Installation
npm install gatsby@4 react@17 react-dom@17 --save-exact --legacy-peer-deps
~~~

- `--legacy-peer-deps`
  Peer Dependency 검사를 완화하여 Gatsby v4 기준 의존성 조합을 강제로 유지한다.
- `--save-exact`
  패치 버전 업데이트로 인한 예기치 않은 충돌을 방지한다.

이를 통해 개발 및 빌드 환경을 안정화하였다.

---

## GitHub Pages 배포 아키텍처 구성

GitHub Pages는 사용자 도메인이 아닌 경우 **서브 디렉토리 기반 호스팅 구조**를 사용한다.
본 프로젝트는 `/hzeror` 경로를 기준으로 서비스되므로, 정적 자산 경로 오류를 방지하기 위한 **Path Prefix 설정**이 필요했다.



---

## pathPrefix 및 배포 스크립트 설정

빌드 시 `--prefix-paths` 옵션을 적용하지 않으면 CSS 및 이미지 자산이 정상적으로 로드되지 않는다.
이를 방지하기 위해 다음과 같이 설정하였다.

**gatsby-config.js**

~~~javascript
module.exports = {
  pathPrefix: "/hzeror",
}
~~~

**package.json**

~~~json
{
  "scripts": {
    "deploy": "gatsby build --prefix-paths && gh-pages -d public"
  }
}
~~~

이후 `npm run deploy` 명령어만으로 GitHub Pages 배포를 자동화할 수 있었다.

---

## 블로그 운영 Workflow

안정적인 서비스 제공과 소스 코드 유실 방지를 위해 다음과 같은 운영 프로세스를 수립하였다.

1. **Local Test**
   `npm run start`로 로컬 환경에서 렌더링 결과 확인

2. **Production Deploy**
   `npm run deploy` 실행 후 `gh-pages` 브랜치에 정적 자산 배포

3. **Source Backup**
   원본 마크다운 및 설정 파일은 `main` 브랜치에 유지

~~~bash
git push origin main
~~~

---

Node.js v14는 현재 EOL 상태로 보안 업데이트가 지원되지 않는다.
그러나 본 템플릿은 Gatsby v4 의존성이 강하게 결합되어 있어, 상위 Node.js 버전으로의 마이그레이션은 **코어 엔진 수정이 수반되는 작업**이다.

따라서 당분간은 **런타임 격리 전략을 통해 안정성을 확보**하고, 향후에는 Gatsby v5 또는 Astro 기반으로의 마이그레이션을 검토할 예정이다.

본 포스팅에서 정리한 설정을 통해 Gatsby v4 기반 블로그를 안정적으로 운영할 수 있는 기반 환경을 확보하였다.