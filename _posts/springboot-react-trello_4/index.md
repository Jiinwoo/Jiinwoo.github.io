---
title: Spring Boot, React 사용해 Trello 클론코딩하기 ( 4 )
date: 2020-06-03
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
4. Authentication Handler 등록
5. header인증을 위한 BasicAuthenticationFilter 구현체 JwtAuthorizationFilter
6. AuthenticationEntryPoint 구현체 CustomAuthenticationEntryPoint
7. ErrorResponseDTO

### Authentication Handler 
깜빡하고 authentication 과정이후 토큰내려주는 Handler를 안만들었었다.

AbstractAuthenticationProcessingFilter는 내부적으로 인증이 성공하면 successfulAuthentication, 
실패하면 unsuccessfulAuthentication를 호출한다. 그래서 successfulAuthentication메소드안의 내용을 어떻게 커스텀하면
토큰을 내려줄 수 있을 것 같다. 

우선 토큰을 만드는데 있어 JWT 라이브러리를 사용할거니까 추가해주자
```gradle
implementation group: 'com.auth0', name: 'java-jwt', version: '3.1.0'
```
그리고 JWT key값, 유효기간 같은 세부설정 정보를 담는 클래스를 하나만들자
```java
public class JwtProperties {
    public static final String SECRET = "jinwooking";
    public static final int EXPIRATION_TIME = 864000000; // 10 days
    public static final String TOKEN_PREFIX = "Bearer ";
    public static final String HEADER_STRING = "Authorization";
}
```

위에서 successfulAuthentication를 호출한다고 하는데 이는 또 내부적으로 successHandler의 onAuthenticationSuccess를 호출
하는것을 알 수 있는데 이 때 successHandler를 setSuccessHandler 메소드로 바꿔줄 수 있다. AuthenticationSuccessHandler 인터페이스를
구현 한 클래스라면 handler로 등록 가능하다 failureHandler도 마찬가지니까 한꺼번에 SecurityHandler로 작성하겠다.

```java
@Component
public class SecurityHandler implements AuthenticationSuccessHandler, AuthenticationFailureHandler {
    @Autowired
    ObjectMapper objectMapper;
    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        if (exception instanceof AuthenticationCredentialsNotFoundException) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
        } else {
            response.setStatus(HttpStatus.FORBIDDEN.value());
        }
        Map<String, Object> data = new HashMap<>();
        data.put(
                "timestamp",
                Calendar.getInstance().getTime());
        data.put(
                "exception",
                exception.getMessage());


        response.getOutputStream()
                .println(objectMapper.writeValueAsString(data));

    }

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        String token = JWT.create().withSubject(userPrincipal.getEmail())
                .withExpiresAt(new Date(System.currentTimeMillis() + JwtProperties.EXPIRATION_TIME))
                .sign(Algorithm.HMAC512(JwtProperties.SECRET.getBytes()));
        TokenResponseDTO tokenResponseDTO = TokenResponseDTO.builder().token(token).build();
        PrintWriter out = response.getWriter();
        out.print(objectMapper.writeValueAsString(tokenResponseDTO));
        out.flush();
    }
}
```

회원 정보가 있다면 이제 /api/auth/login으로 로그인을 진행하면 토큰을 발급 받을 수 있다. (생각해보니 회원가입도 
아직 안만든 상태이다...)


### header인증 과정
이제 로그인으로 발급 받은 토큰을 가지고 해당 api에 접근 할 수 있는지 authorization 과정을 만들 차례이다.
JWT방식에는 쿠키-세션같은 상태를 가지고 있지 않기 때문에 header의 Authorization 필드의 bearer token값을 매번 검사해야한다.

요청이 들어오면 header 값 검증을 하는 filter를 만들어 보자.

BasicAuthenticationFilter를 상속 받아 만들 예정. 기본적으로 이 필터는 authorization header를 검사하긴 하나 
Basic 방식? 으로 검사를 하기 때문에 우리가 사용하는 Bearer token방식과는 달라 새로 override해서 구현해야한다.

```java
public class JwtAuthorizationFilter extends BasicAuthenticationFilter {

    CustomUserDetailsService customUserDetailsService;

    public JwtAuthorizationFilter(AuthenticationManager authenticationManager,CustomUserDetailsService customUserDetailsService ) {
        super(authenticationManager);
        this.customUserDetailsService = customUserDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        String header = request.getHeader(JwtProperties.HEADER_STRING);

        if(header == null || !header.startsWith(JwtProperties.TOKEN_PREFIX)){
            chain.doFilter(request,response);
            return;
        }
        Authentication authentication = getUsernamePasswordAuthentication(request);
        SecurityContextHolder.getContext().setAuthentication(authentication);
        chain.doFilter(request,response);
    }
    private Authentication getUsernamePasswordAuthentication(HttpServletRequest request) throws UnsupportedEncodingException {
        String token = request.getHeader(JwtProperties.HEADER_STRING).replace(JwtProperties.TOKEN_PREFIX,"");
        String username = JWT.require(Algorithm.HMAC512(JwtProperties.SECRET.getBytes()))
                .build()
                .verify(token)
                .getSubject();
        UserDetails userDetails = customUserDetailsService.loadUserByUsername(username);
        UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
        return authentication;
    }
}
```
간단히 말하면 header의 bearer 토큰을 꺼내 파싱 후 db에서 해당하는 유저가있는지 조회하는 로직을 가지고 있다. 
 
여기까지 만들었으면 기본적인 JWT 인증과정을 구현이 끝났다.
간단히 회원가입 api와 hello api를 만들어보자

회원가입 요청, 응답 DTO
```java
@Getter
@NoArgsConstructor
public class SignupRequestDTO {
    @Valid
    private String email;
    @NotEmpty
    private String username;
    @NotEmpty
    private String password;

    @Builder
    public SignupRequestDTO(String email, String username, String password) {
        this.email = email;
        this.username = username;
        this.password = password;
    }

    public User toEntity(PasswordEncoder passwordEncoder) {
        return User.builder()
                .email(this.email)
                .password(passwordEncoder.encode(this.password))
                .username(this.username)
                .build();
    }
}
////////
@Getter
@Setter
@NoArgsConstructor
public class SignupResponseDTO {
    @Valid
    private String email;
    private String username;

    public SignupResponseDTO(User user) {
        this.email = user.getEmail();
        this.username = user.getUsername();
    }
}
```
그리고 회원가입시켜줄 UserService 
```java
@RequiredArgsConstructor
@Service
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Transactional
    public SignupResponseDTO signup (SignupRequestDTO signupRequestDTO) {
        //중복체크
//        User user = userRepository.findByEmail(signupRequestDTO.getEmail());
//        if(user != null) {
//            throw new EmailDuplicationException(user.getEmail().getValue());
//        }
        return new SignupResponseDTO(userRepository.save(signupRequestDTO.toEntity(passwordEncoder)));

    }

}
```
중간의 저 주석은 security Error 핸들링할 때 같이 처리해줄 예정이다.

그리고 api controller 부분
```java
/// AuthController
@RequiredArgsConstructor
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final UserService userService;

    @PostMapping("/users")
    @ResponseStatus(HttpStatus.CREATED)
    SignupResponseDTO signup (@RequestBody @Valid SignupRequestDTO signupRequestDTO) {
        return userService.signup(signupRequestDTO);
    }
}
/// HelloController
@RestController
@RequestMapping("/hello")
public class HelloController {

    @GetMapping("/")
    public String hello () {
        return "hello";
    }
}
```
여기까지 작성하면 로그인 및 회원가입이 가능하고 helloController로는 토큰이 없으면 401에러를 반환할 것이다.
다음 포스팅때 security와 controller단 error handling을 해보자.

> AuthorizationFilter내부에서 token으로 db를 한번씩 조회하는데 이 과정을 redis로 처리해보고싶다. 언젠간 해야징
