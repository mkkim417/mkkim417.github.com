---
layout: post
title: 윈도우7에서 동작하지 않는 Electron 기반 앱 동작하게 만들기
excerpt_separator:  <!--more-->
categories:
  - Electron
tags:
  - Electron
  - Chromium
  - d3dcompiler_47.dll
  - .net framework
---

열심히 만든 Electron 기반의 키오스크가 완성되었습니다

프로그램을 USB에 담아 설치하러 가는데 첫번째 설치할 대상이 Windows7 기반의 키오스크네요

Windows7 기반의 키오스크가 있는 건 알고 있었고, Electron은 Windows7을 지원합니다 그래서 자신만만하게 설치했습니다

프로그램이 실행은 되나 화면이 뜨지 않습니다..........

<span style="color:red">런칭이 미뤄졌습니다ㅠㅠ</span>

## 무엇이 문제였는가

~~[윈도우7의 지원은 2020년 1월 14일 종료](https://support.microsoft.com/ko-kr/help/4057281/windows-7-support-ended-on-january-14-2020)되었..~~

Electron 도입 전에 [지원하는 플랫폼](https://www.electronjs.org/docs/tutorial/support)을 먼저 확인 했을 때에는 분명 Windows 7 이상을 지원한다고 쓰여 있었습니다.

그리고 아무리 보안 지원이 종료되었다 하더라도 어쩔수 없이 현실적으로 키오스크 등의 시스템에서는 윈도우7은 물론 무려 XP가 사용되기도 합니다...

<img src="https://media.techeblog.com/images/windowsxp.jpg" />

윈도우 XP 지원은 못하더라도 키오스크에서 사용하고 있는 윈도우7은 어떻게든 지원해야 합니다.

이런 문제가 발생한 이유가 뭘까요?

제가 해당 앱을 개발했을 당시 Electron 9.0.1을 사용했는데, 해당 이슈는 무려 Electron 6부터 발생하는 이슈였습니다.

- [Electron Issue #19569 : Electron 6.0.0 not support Windows 7](https://github.com/electron/electron/issues/19569)
- [Update for the d3dcompiler_47.dll component on Windows Server 2012, Windows 7, and Windows Server 2008 R2](https://support.microsoft.com/en-us/help/4019990/update-for-the-d3dcompiler-47-dll-component-on-windows)
- [Chromium Issue 920704: New d3dcompiler_47.dll needed](https://bugs.chromium.org/p/chromium/issues/detail?id=920704)

Electron 6이 내부적으로 `d3dcompiler_47.dll` 라는 파일을 통해(내장된 Chromium에서) 렌더링을 하는데 해당 파일의 `10.0.17763.132` 버전이 Black screen 이슈가 있다고 하네요. 그래서 프로그램은 실행되나 화면이 뜨지 않았던 것이죠.

Chromium에서는 해당 이슈를 인지하고 해당 파일의 버전을 일단 `10.0.17134.12` 으로 다운그레이드합니다.

지금은 Electron 10이 릴리즈되었고 내장된 크로미움 버전이 v83→v85로 업데이트되었기 때문에 윈도우7에서도 해당 문제는 발생하지 않을 것으로 보입니다 ~~(모든 키오스크가 업데이트되어서 확인은 못해봤습니다)~~

## 해결하기

하지만 Electron 프레임워크가 해결책을 내놓기 전에 자체적으로 해결해야 했고, 직접 해결하는 방법은 간단했습니다.

**키오스크에 [닷넷 프레임워크 4.7.1+](https://www.microsoft.com/en-us/download/details.aspx?id=56116) 을 설치하는 것!**

하지만 역시나 단순하게 풀리지 않습니다. 키오스크에서 닷넷 프레임워크 설치에 실패합니다 ㅠㅠ

인증서 체인은 처리되었지만, 신뢰공금자에 의해 신뢰되지 않는 루트 인증서에서 중지되었습니다.

- [x64 기반 시스템용 Windows 7 보안 업데이트(KB2813430)](https://www.microsoft.com/ko-kr/download/details.aspx?id=39115)

해당 오류가 발생한다면, 위 윈도우 업데이트를 설치 후, 재부팅하고 다시 닷넷 프레임워크 설치를 진행하시면 됩니다.

자 이제 멀쩡하게 윈도우 7에서도 Electron 기반의 키오스크가 실행되는 것을 보실 수 있습니다.

<span style="color:green;font-weight:bold;">런칭!</span>