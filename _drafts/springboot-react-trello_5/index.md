---
title: Spring Boot, React 사용해 Trello 클론코딩하기 ( 5 )
date: 2020-06-04
tags:
  - springboot
  - react
keywords:
  - trello
  - springboot
  - react
---

1. ~~Provider 구현체 DaoAuthenticationProvider~~
2. ~~UserDetailsService 구현체 CustomUserDetailsService~~
3. ~~User Entity 및 UserPrincipal~~
4. ~~Authentication Handler 등록~~
5. ~~header인증을 위한 BasicAuthenticationFilter 구현체 JwtAuthorizationFilter~~
6. AuthenticationEntryPoint 구현체 CustomAuthenticationEntryPoint
7. ErrorResponseDTO

지금까지 기본 인증 과정을 구현했다. 이제 에러를 핸들링하는 방법에 대해 알아보자.

기본적으로 Security Error를 제외한 나머지 Controller Error들은 AdviceController에서 모든 오류를 한곳에서 처리할 수 있다.
그리고 Security Error도 forward 시키면 Controller 단에서 받아서 처리할 수 있다. 하지만 뭔가 불 필요한 낭비라고 생각하기 때문에
Security Error는 Security 단에서 처리하도록 해보자.

우선 Security filter내에 JwtAuthorizationFilter에서 auth Token값이 이상한 값이 들어올 경우 

 
