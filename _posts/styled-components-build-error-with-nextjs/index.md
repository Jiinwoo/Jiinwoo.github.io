---
title: StyledComponent v4 빌드 에러 및 nextjs에서 v5 이슈
date: 2020-05-23
tags:
  - react
  - styled-components
  - TIL
keywords:
  - createGlobalStyles
---

오늘 업무를 하던 도중 styled-components v4버전 대에 build할 때 에러가 있었고 그에러가 뭐였는지 기억이 안나는데
대충 깃헙이슈를 보니 v5 최신버전을 설치하거나 특정 type을 삭제하고 @type에 커스텀해서 추가하라는데 귀찮아서
그냥 v5를 새로설치했다. 

> 사실 앞 버전 올리는데 그냥 올리면 무슨일이 터질까 매우 고민을 했지만 v5 release 문서를 보면 ["no breaking changes!"](https://styled-components.com/releases) 
> 라는데 큰 변화없이 속도향상을 줄수있다고 적혀있다. 그래서 그냥 업데이트 했다.

그래서 설치하고 release branch에 배포했는데 html, body 등등 전역에 지정한 createGlobalStyle 메소드가 제대로 적용되지 않았다.

현재 v5 styled-components의 createGlobalStyle을 next js 의 _document.tsx 내에 next/head 컴포넌트 안에서 렌더링할 때
적용이 되지않는 문제가 있었다. 

여러가지 구글링을 해본 결과 _app.tsx에 createGlobalStyle을 적용해서 해결했다.

> 지금 styled-components github issue 가보면 이 관련 이슈가 꽤 있는것 같다 나는 귀찮아서 그냥 넘어갔지만 얼른 누가 해결해줬으면
>좋겠다. 
