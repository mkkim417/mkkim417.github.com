---
layout: post
title: Electron과 React로 키오스크 데스크톱 앱 만들기
excerpt_separator:  <!--more-->
categories:
  - Electron
tags:
  - Electron
  - React
---

프론트엔드와 백엔드를 모두 개발할 수 있는 Javascript의 장점을 가장 크게 가져간 부분이 크로스 플랫폼 개발 분야가 아닐까 합니다.

모바일 앱은 Flutter, React native, Xamarin, Nativescript, Cordova, Ionic 등 여러가지 선택지가 있겠지만 데스크톱 앱 분야에서는 단연 Electron이 대장 자리를 차지하고 있습니다.

Electron은 Chromium과 Node.js, HTML, CSS, Javascript 등의 웹 기술을 이용하여 크로스 플랫폼(Windows, MacOS, Linux) 대상 프로그램을 개발할 수 있는 오픈소스 플랫폼입니다.

VS Code, Slack, WhatsApp, Twitch, 잔디, Mattermost, Discord 등 이미 많은 대형 업체에서도 Electron을 도입하여 안정적으로 프로그램을 배포하고 운영하고 있습니다.

## Electron Architecture

Electron을 아주 가볍게 살펴보겠습니다. Electron은 메인 프로세스와 렌더러 프로세스로 구성되어 있습니다.

웹 개념으로 볼 때 메인 프로세스는 Back-end, 렌더러는 Front-end로 볼 수 있는데요.

### 메인 프로세스

메인 프로세스에서는 Electron에서 제공하는 저수준의 API(운영체제의 파일 선택 Dialog 등)와 node.js 패키지 등을 사용할 수 있고, 렌더러 프로세스를 생성할 수 있습니다.

### 렌더러 프로세스

렌더러 프로세스는 Front-end라고 말씀드렸는데요. 유저와 상호작용하는 GUI 역할을 합니다. 우리가 익숙한 HTML, CSS등으로 유저와 상호작용하는 GUI를 만들어낼 수 있습니다.

특히 Chromium 기반이기 때문에 크로스 브라우징을 개발자가 고려하지 않아도 웹과 다르게 모든 플랫폼에서 같은 모습을 보여줄 수 있습니다.

### 프로세스 간 통신

렌더러 프로세스에서 직접 파일 선택 Dialog 등을 여는 일은 직접 할 수 없고, 메인 프로세스에 요청을 전달해야 합니다. 이러한 프로세스 간 통신은 이벤트를 발생시키고 처리하는 `ipcRenderer`와 `ipcMain`을 통해 동작합니다. 

## 일단 시작해보기

저는 create-react-app을 통해 먼저 react 앱을 생성하겠습니다.

PC 앱을 사용하는데 페이지가 이동하면서 깜박거리거나 하면 유저의 앱 사용성을 떨어트릴 수 있기 때문에 SPA로 개발하고자 React를 선택했습니다.

Electron에서는 React 외에 Vue를 사용해도 되고, 프레임워크 없는 HTML, CSS를 사용해도 되고, 이미 Electron+React, Electron+Vue 환경을 구축해놓은 boilerplate도 많이 있으니 참고하세요.

`npx create-react-app my-kiosk`

위 명령어를 통해 새로운 React app을 생성합니다.

자 이제 생성된 my-kiosk 폴더로 들어가셔서 필요한 electron 패키지를 설치합니다.

`yarn add electron`

그리고 root 폴더에 다음과 같은 `electron.js` 를 생성해주세요.

```javascript
const { app, BrowserWindow, ipcMain }  = require('electron');
const path = require('path');

let mainWindow;

function createWindow() {
  mainWindow = new BrowserWindow({
    center: true,
    width: 800,
    height: 800,
    webPreferences: {
      nodeIntegration: true,
    }
  });

  mainWindow.loadURL('http://localhost:3000');

  mainWindow.on('closed', () => {
    mainWindow = undefined;
  });
}

app.on('ready', createWindow);

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

app.on('activate', () => {
  if (mainWindow === null) {
    createWindow();
  }
});
```

파일을 살펴보시면

Electron app이 ready가 되는 시점에 createWindow가 호출되며 BrowserWindow를 생성합니다.

우리가 만든 앱의 메인 GUI가 될 화면입니다.

개발환경인 경우 react가 기본적으로 `http://localhost:3000`에 로컬 서버로 동작하므로 앱이 실행되고 첫 페이지로 해당 URL을 load 하도록 설정했습니다.

아직 Electron 앱을 실행할 수 없는 상태인데요. 관련된 script를 package.json에 설정해주어야 합니다.

먼저 다음 명령어로 의존성 패키지를 추가 설치해줍니다.

`yarn add concurrently cross-env wait-on —dev`

그리고 `package.json` 을 부분 변경해줍니다.

```json
...
"main": "electron.js",
"scripts": {
    "react-start": "react-scripts start",
    "react-build": "react-scripts build",
    "react-test": "react-scripts test",
    "react-eject": "react-scripts eject",
    "start": "concurrently \"cross-env BROWSER=none yarn react-start\" \"wait-on http://localhost:3000 && electron .\"",
  },
...
```

create-react-app이 만들어준 스크립트는 모두 `react-`를 붙여 옮겨주었구요.

프로그램의 진입점을 electron.js로 수정해주시고,

start 스크립트 실행 시에 React 앱을 실행하고, Electron 앱을 실행할 수 있도록 실행 스크립트를 수정해줍니다.

자 이제 `yarn start` 하시면!

<img src="/assets/images/20201006/electron-first.png" />

내가 만든 ~~사이트~~프로그램이 실행되었습니다! (감격)

## 키오스크 만들기

분명히 키오스크 만들기가 제목이었는데, 키오스크는 어디갔죠?

다시 electron.js로 돌아가서 다음과 같이 수정해주세요.

```javascript
  mainWindow = new BrowserWindow({
    center: true,
    width: 800,
    height: 800,
    kiosk: true,
    alwaysOnTop: true,
    resizable: false,
    frame: false,
    webPreferences: {
      nodeIntegration: true,
    }
  });

```

mainWindow를 선언할 때 kiosk: true로 설정하면 키오스크처럼 전체화면으로 동작하며, frame: false 설정을 통해 프레임 없는 윈도우를 만들 수 있습니다.

다시 종료 후 실행해보면,

<img src="/assets/images/20201006/electron-kiosk.png" />

자 이제 이렇게 내가 만든 키오스크가 완성되었습니다 (??)

## CSS Tip

제작하는 키오스크에 따라 다르겠지만, 일반 웹 앱처럼 텍스트가 드래그된다던가, 이미지가 드래그 된다면 사용성을 해칠 수 있습니다.

아래와 같은 CSS 요소를 설정해두면 이와 같은 사용성 해침을 방지할 수 있습니다.

```css
*, *::after, *::before {
	-webkit-user-select: none;
	-webkit-user-drag: none;
	-webkit-app-region: no-drag;
	cursor: default;
}
```