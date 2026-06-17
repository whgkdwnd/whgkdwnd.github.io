---
title: "JWT 인증을 직접 설계하면서 배운 것들 — 이벤트 당일 장애까지"
tags:
  - JWT
  - Spring_Security
  - Redis
  - Backend
date: 2026-06-16
thumbnail: /assets/img/thumbnail/empty.jpg
---

판매 이벤트를 운영하던 중 DM 한 통이 왔다. "채팅이 안 쳐져요."

별다른 에러 알림은 없었다. 로직도 문제없어 보였다. 그래서 로그를 직접 쿼리해봤다.

401이었다.

이 글은 MyFave(인플루언서 쇼핑 서비스)에서 인증 시스템을 단독 설계·구현하면서 내린 결정들, 그리고 실제 운영 중 마주한 장애가 그 결정들을 어떻게 검증했는지에 대한 기록이다.

---

## 세션 대신 JWT를 선택한 이유

사용자가 100명일 때 세션은 괜찮다. 1만 명이 동시에 로그인하면, 그 세션들은 전부 서버 메모리 위에 살아있다.

서버 세션의 구조적 문제는 두 가지다.

**첫째, 서버 메모리 점유.** 동시 접속자가 수천 명이면 세션 객체가 수천 개 메모리에 적재된다. 판매 이벤트처럼 트래픽이 급증하는 상황에서 이는 직접적인 GC 압박과 응답 지연으로 이어진다.

**둘째, 수평 확장 시 세션 공유 문제.** 로드 밸런서가 다음 요청을 다른 서버로 보내는 순간 세션을 찾지 못해 강제 로그아웃된다. Sticky Session은 특정 서버에 트래픽이 편중되고, 외부 세션 저장소를 도입하면 결국 "세션을 쓰면서 Redis도 써야 하는" 상황이 된다.

JWT는 서버가 Signature만 검증하면 어떤 인스턴스도 동일하게 처리할 수 있다. MyFave에서 JWT를 선택한 핵심 이유는 다수 사용자 동시 요청 환경에서 서버 세션 부하를 구조적으로 제거하고 싶었기 때문이다.

### Stateless의 한계 — 즉시 무효화 불가

JWT의 가장 큰 약점은 서명만 유효하면 서버가 거부할 수 없다는 점이다. 이 문제를 수용한 방식은 두 가지다:

1. **Access Token 수명을 짧게 설정** — 탈취되더라도 유효 시간을 최소화
2. **Refresh Token을 Redis에 저장** — 로그아웃 시 즉시 키를 삭제해 무효화 가능

완전한 Stateless가 아니라는 비판이 있을 수 있다. 하지만 Access Token은 Stateless로 유지하면서 Refresh Token만 Redis로 관리하는 혼합 구조가 보안과 성능의 현실적인 균형점이라고 판단했다.

---

## Access Token + Refresh Token 이중 구조 설계

Access Token 하나만 쓰면 가장 단순하다. 그런데 그 토큰이 탈취당했을 때 막을 방법이 없다.

단일 토큰의 딜레마(수명 길면 탈취 위험, 짧으면 UX 저하)를 역할 분리로 해결했다.

| | Access Token | Refresh Token |
|---|---|---|
| 역할 | API 요청 인증 | Access Token 재발급 |
| 수명 | 30분 | 7일 |
| 저장 위치 | 클라이언트 메모리 | Redis |
| 노출 빈도 | 매 요청마다 전송 | 재발급 시에만 전송 |

### Refresh Token을 왜 Redis에 저장했나

Refresh Token을 DB에 저장하는 방법도 있다. 비교해보면 선택 이유가 명확하다.

**DB 저장의 문제:**
- 재발급 요청마다 DB 조회 → I/O 비용
- 만료된 토큰 삭제를 위한 배치 작업 필요

**Redis 저장의 장점:**
1. **TTL 자동 만료**: `EXPIRE` 설정하면 Redis가 자동 삭제. 별도 배치 불필요
2. **O(1) 조회**: 수백만 건에서도 조회 시간 일정
3. **즉시 무효화**: 로그아웃 시 해당 키 삭제로 즉시 처리

```java
// Key: "refresh_token:{userId}", Value: tokenValue, TTL: 7일
redisTemplate.opsForValue().set(
    "refresh_token:" + userId,
    refreshToken,
    7,
    TimeUnit.DAYS
);

// 로그아웃 시
redisTemplate.delete("refresh_token:" + userId);
```

---

## Spring Boot 구현: JWT 필터 체인

### JwtAuthenticationFilter

Spring Security 필터 체인에 커스텀 필터를 삽입해 JWT를 검증했다. 요청 헤더에서 토큰을 꺼내 검증하고, 유효하면 SecurityContext에 인증 정보를 등록한다.

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        String token = resolveToken(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            String userId = jwtTokenProvider.getUserId(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(userId);

            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities()
                );
            authentication.setDetails(
                new WebAuthenticationDetailsSource().buildDetails(request)
            );
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### SecurityConfig — 채팅 엔드포인트 접근 정책 분리

MyFave에서 채팅 조회는 비인증 사용자도 허용했다. 상품 Q&A를 보려는 사용자를 로그인으로 막으면 UX가 나빠지고, 조회 자체는 보안 민감 작업이 아니기 때문이다. 이 정책을 SecurityConfig에서 명시적으로 분리했다.

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .exceptionHandling(exception ->
                exception.authenticationEntryPoint(jwtAuthenticationEntryPoint)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                // 채팅 조회는 비인증 허용 — 공개 정보로 간주
                .requestMatchers(HttpMethod.GET, "/api/chat/**").permitAll()
                // 채팅 작성은 인증 필요
                .requestMatchers(HttpMethod.POST, "/api/chat/**").authenticated()
                .anyRequest().authenticated()
            )
            .addFilterBefore(
                jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class
            );

        return http.build();
    }
}
```

### JwtTokenProvider

```java
@Component
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.access-token-expiration}")
    private long accessTokenExpiration; // 30분 = 1800000ms

    @Value("${jwt.refresh-token-expiration}")
    private long refreshTokenExpiration; // 7일 = 604800000ms

    public String createAccessToken(String userId, String role) {
        Date now = new Date();
        return Jwts.builder()
            .setSubject(userId)
            .claim("role", role)
            .setIssuedAt(now)
            .setExpiration(new Date(now.getTime() + accessTokenExpiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    public String createRefreshToken(String userId) {
        Date now = new Date();
        return Jwts.builder()
            .setSubject(userId)
            .setIssuedAt(now)
            .setExpiration(new Date(now.getTime() + refreshTokenExpiration))
            .signWith(getSigningKey(), SignatureAlgorithm.HS256)
            .compact();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (ExpiredJwtException e) {
            return false;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    public String getUserId(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }

    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

---

## 운영 중 마주한 장애 — 쿼리 한 줄로 시작된 디버깅

판매 이벤트를 운영하는 도중 DM이 왔다. 채팅이 안 쳐진다는 내용이었다.

별다른 알림은 없었다. 채팅 서버도 떠 있었고, 눈에 띄는 에러도 없었다. 로그를 직접 쿼리해봤다.

```sql
SELECT * FROM logs WHERE endpoint LIKE '%/api/chat%' ORDER BY created_at DESC LIMIT 50;
```

결과에 401이 찍혀 있었다.

```
[ERROR] 401 Unauthorized - /api/chat
[ERROR] 401 Unauthorized - /api/chat
```

채팅 POST 엔드포인트는 인증이 필요한 API였다. 즉, 해당 사용자의 Access Token이 만료된 상태에서 채팅 전송을 시도했고, 서버는 401을 반환했으며, 클라이언트는 그 상태를 사용자에게 제대로 알려주지 않고 있었다.

**원인은 두 곳에 있었다:**

1. Access Token이 만료됐을 때 클라이언트에서 자동 재발급 처리 없이 요청이 그냥 실패하고 있었다
2. 실패 시 사용자에게 명확한 안내가 없어 "채팅이 안 된다"는 경험으로만 남았다

**조치:**

해당 사용자에게 재로그인을 안내했고, 채팅은 정상화됐다. 근본 원인인 클라이언트 자동 재발급 처리는 이후 개선 과제로 등록했다.

---

## 이 경험이 남긴 것

DM 한 통으로 시작된 이 디버깅이 인상적이었던 이유는 결과가 아니라 과정이었다.

에러 알림이 없었다. 사용자가 "채팅이 안 된다"고 했을 때 어디서부터 봐야 할지 막막했다. 결국 로그를 직접 쿼리해서 401을 찾아냈지만, 이 과정이 너무 수동이었다.

토큰 만료는 인증 시스템에서 예측 가능한 이벤트다. 그런데 그걸 사용자 DM으로 알게 됐다는 건, 모니터링이 제대로 안 되고 있다는 뜻이었다.

이후 Grafana에 401 응답을 엔드포인트별로 분리해서 추적하기 시작했다. 특히 `ExpiredJwtException`과 `SignatureException`을 구분했다. 전자는 수명 정책 문제고, 후자는 변조 시도다. 같은 401이라도 의미가 다르다.

---

## 아직 해결하지 못한 것

이 글에서 구현한 것들을 솔직하게 정리하면, 남은 문제가 보인다.

**Access Token 무효화 창**: Refresh Token을 Redis에서 삭제해도, 이미 발급된 Access Token은 만료(30분)까지 유효하다. 탈취 감지 후 즉시 차단하려면 Access Token 블랙리스트가 필요한데 현재는 미구현이다.

**클라이언트 자동 재발급**: 401 수신 시 Refresh Token으로 자동 재발급 후 원래 요청을 재시도하는 Axios interceptor가 필요하다. 장애 이후 중기 조치로 식별했지만 아직 완료하지 못했다.

**Redis 단일 장애점**: Redis가 다운되면 Refresh Token 재발급이 전부 막힌다. Redis Sentinel이나 Cluster 구성이 프로덕션에서는 필요하다는 것을 알고 있지만 현재는 단일 인스턴스로 운영 중이다.

---

## 정리

인증은 기능이 아니라 신뢰다. 로그인이 풀리는 순간, 사용자는 서비스 전체를 의심하기 시작한다.

| 결정 | 선택 | 이유 |
|---|---|---|
| 인증 방식 | JWT | 수평 확장 시 서버 세션 부하 제거 |
| Refresh Token 저장 | Redis | TTL 자동 만료, O(1) 조회, 즉시 무효화 |
| Access Token 수명 | 30분 | 실제 사용자 세션 22분 + 장애 경험 반영 |
| 채팅 GET | 비인증 허용 | 공개 정보 접근성과 보안 분리 |

지금 당신의 서비스에 JWT가 붙어 있다면, 한 번만 물어보자.

> "Refresh Token 갱신이 실패하면, 사용자는 어떤 경험을 하는가?"

그 질문에 바로 답할 수 있다면, 당신의 인증 설계는 충분히 단단한 것이다.
