# Spring Security — Complete Interview Reference

---

## 1. Overview

Spring Security is the **de-facto security framework** for Spring-based applications. It provides comprehensive security services for Java EE-based enterprise software, with a heavy focus on two core concerns:

- **Authentication** — Who are you? (identity verification)
- **Authorization** — What are you allowed to do? (permission enforcement)

### Why Does It Exist?
Security is a cross-cutting concern — every endpoint, every service method, and every resource needs to be protected consistently. Rolling your own security is error-prone, especially for concerns like session fixation, CSRF, clickjacking, brute-force attacks, and secure password storage. Spring Security handles all of these out of the box.

### Where It Fits in the Spring Ecosystem
- Plugs into Spring MVC via a `DelegatingFilterProxy` (a Servlet Filter)
- Integrates with Spring Data for method-level security
- Integrates with Spring Boot via auto-configuration
- Supports reactive applications via Spring WebFlux (Spring Security Reactive)
- Supports OAuth2, OpenID Connect, SAML 2.0, LDAP out of the box

---

## 2. Core Concepts & Sub-Topics

### 2.1 Authentication vs Authorization

| Concept        | Question Answered        | Example                                         |
|----------------|--------------------------|------------------------------------------------|
| Authentication | Who are you?             | Login with username/password, JWT token        |
| Authorization  | What can you do?         | Only ADMIN can delete users                    |

### 2.2 Principal
The currently logged-in user. Accessible via `SecurityContextHolder.getContext().getAuthentication().getPrincipal()`.

### 2.3 Authentication Object
Represents an authenticated entity. Contains:
- `principal` — the user identity (usually a `UserDetails` object)
- `credentials` — password or token (cleared after auth)
- `authorities` — the roles/permissions granted
- `isAuthenticated()` — whether authentication was successful

### 2.4 SecurityContext and SecurityContextHolder
- `SecurityContext` holds the `Authentication` object for the current request.
- `SecurityContextHolder` is a thread-local storage that holds the `SecurityContext`.
- By default uses `ThreadLocal` strategy — one `SecurityContext` per thread.
- Can be configured to `InheritableThreadLocal` (child threads inherit context) or `GLOBAL` (all threads share one context).

```
Request Thread
│
├─ SecurityContextHolder (ThreadLocal)
│     └─ SecurityContext
│           └─ Authentication
│                 ├─ principal   (UserDetails)
│                 ├─ credentials (cleared after auth)
│                 └─ authorities [ROLE_USER, ROLE_ADMIN]
```

### 2.5 GrantedAuthority and Roles vs Permissions
- `GrantedAuthority` is a string like `"ROLE_ADMIN"` or `"ORDER_WRITE"`.
- **Roles** are prefixed with `ROLE_` by convention.
- **Permissions/Privileges** are fine-grained strings without the prefix.
- `hasRole("ADMIN")` automatically prepends `ROLE_`; `hasAuthority("ROLE_ADMIN")` does not.

### 2.6 UserDetails and UserDetailsService
- `UserDetails` — interface representing a user (username, password, authorities, account flags).
- `UserDetailsService` — interface with single method `loadUserByUsername(String username)`.
- Your custom implementation queries the DB and returns a `UserDetails` object.
- `UserDetailsManager` extends `UserDetailsService` with CRUD operations.

### 2.7 PasswordEncoder
Hashes passwords before storage and verifies them on login.

| Encoder             | Algorithm       | Notes                                              |
|---------------------|-----------------|----------------------------------------------------|
| `BCryptPasswordEncoder` | BCrypt       | Default recommendation; adaptive cost factor      |
| `Argon2PasswordEncoder` | Argon2       | Memory-hard; winner of Password Hashing Competition|
| `SCryptPasswordEncoder` | SCrypt       | Memory-hard                                        |
| `Pbkdf2PasswordEncoder` | PBKDF2       | FIPS-compliant                                     |
| `NoOpPasswordEncoder`   | Plaintext    | **Never use in production**                        |
| `DelegatingPasswordEncoder` | Multiple | Allows migrating between encoders                 |

`DelegatingPasswordEncoder` is the default since Spring Security 5. Passwords are stored with a prefix like `{bcrypt}$2a$10$...` enabling algorithm migration.

### 2.8 The Security Filter Chain

Spring Security is implemented as a **chain of servlet filters**. The key class is `SecurityFilterChain`. Filters execute in a well-defined order.

```
HTTP Request
     │
     ▼
DelegatingFilterProxy  (Servlet container, bridges to Spring bean)
     │
     ▼
FilterChainProxy (Spring Security's main entry point)
     │
     ▼
SecurityFilterChain #1 → ... → SecurityFilterChain #N
     │
     ▼
[Filter 1] → [Filter 2] → ... → [Filter N] → DispatcherServlet
```

**Key filters in order (abbreviated):**

| Order | Filter                                    | Purpose                                         |
|-------|-------------------------------------------|-------------------------------------------------|
| 100   | `DisableEncodeUrlFilter`                  | Disables URL-encoded session ID                 |
| 200   | `SecurityContextHolderFilter`             | Loads SecurityContext for request               |
| 400   | `LogoutFilter`                            | Handles `/logout`                               |
| 500   | `UsernamePasswordAuthenticationFilter`    | Form login processing                           |
| 600   | `BasicAuthenticationFilter`               | HTTP Basic auth                                 |
| 700   | `BearerTokenAuthenticationFilter`         | OAuth2 Bearer / JWT processing                  |
| 800   | `RequestCacheAwareFilter`                 | Resumes cached request after login redirect     |
| 900   | `SecurityContextHolderAwareRequestFilter` | Wraps request with security methods             |
| 1000  | `AnonymousAuthenticationFilter`           | Sets anonymous token if no auth present         |
| 1100  | `ExceptionTranslationFilter`              | Converts security exceptions to HTTP responses  |
| 1200  | `AuthorizationFilter`                     | Enforces access control rules                   |

### 2.9 Authentication Manager and Authentication Provider

```
Client → AuthenticationManager.authenticate(Authentication)
              │
              ▼
       ProviderManager (iterates list of AuthenticationProviders)
              │
         ┌────┴────────────────────────────┐
         ▼                                 ▼
DaoAuthenticationProvider        JwtAuthenticationProvider
(queries UserDetailsService)     (validates JWT)
```

- `AuthenticationManager` — interface with `authenticate(Authentication)`.
- `ProviderManager` — default implementation; delegates to a list of `AuthenticationProvider`s.
- Each `AuthenticationProvider` supports specific `Authentication` types (e.g., `UsernamePasswordAuthenticationToken`).
- If one provider throws `AuthenticationException`, the next is tried; if all fail, exception is thrown.

### 2.10 Access Decision / Authorization
- In Spring Security 5.x: `AccessDecisionManager` + `AccessDecisionVoter` (legacy).
- In Spring Security 6.x: replaced by `AuthorizationManager` — cleaner, composable.
- `AuthorizationFilter` calls `AuthorizationManager.check(Supplier<Authentication>, HttpServletRequest)`.
- Returns `AuthorizationDecision(true/false)`.

### 2.11 HTTP Security Configuration
`SecurityFilterChain` bean replacing the deprecated `WebSecurityConfigurerAdapter`:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```

### 2.12 CSRF (Cross-Site Request Forgery)
- CSRF attacks trick authenticated users into making unwanted requests.
- Spring Security enables CSRF protection by default for state-changing methods (POST, PUT, DELETE, PATCH).
- Uses a **Synchronizer Token Pattern**: server generates a token stored in session; client must echo it in every state-changing request.
- For stateless REST APIs (JWT), CSRF is typically **disabled** because no cookies carry the session.

```java
http.csrf(csrf -> csrf.disable()); // For stateless APIs
```

### 2.13 CORS (Cross-Origin Resource Sharing)
- Browsers block cross-origin requests by default.
- Spring Security must be configured to allow CORS before its filter chain processes the request.
- Wrong order (security before CORS) causes preflight OPTIONS requests to fail.

```java
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

### 2.14 Session Management
- **Stateful (default)**: session created after authentication, JSESSIONID cookie sent to client.
- **Stateless**: no session created (used with JWT/OAuth2).
- Session fixation attack protection: Spring Security by default changes the session ID after authentication.
- Concurrent session control: limit number of simultaneous sessions per user.

```java
http.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS) // For JWT
    .maximumSessions(1)
    .maxSessionsPreventsLogin(true)
);
```

### 2.15 JWT (JSON Web Token) Authentication
- Stateless token-based authentication.
- JWT = `Base64(Header)`.`Base64(Payload)`.`HMAC-SHA256(Header+Payload+Secret)` or RSA signature.
- Server validates signature, no session lookup needed.
- Access token (short-lived, ~15m) + Refresh token (long-lived, ~7d).

```
Client                         Server
  │── POST /login ─────────────►│
  │◄─ { accessToken, refreshToken }─│
  │                              │
  │── GET /orders (Bearer token)─►│
  │   (server validates JWT)      │
  │◄─ 200 OK ─────────────────── │
```

### 2.16 OAuth2 and OpenID Connect (OIDC)
- **OAuth2**: authorization framework — allows third-party apps to access resources on behalf of a user without sharing credentials.
- **OpenID Connect**: identity layer on top of OAuth2 — adds ID tokens for authentication.
- **Roles**: Resource Owner (user), Client (your app), Authorization Server (Google/Okta), Resource Server (your API).
- **Grant Types**: Authorization Code (most secure), Client Credentials (machine-to-machine), Implicit (deprecated), Password (deprecated).

### 2.17 Method-Level Security

```java
@EnableMethodSecurity // Spring Security 6 (replaces @EnableGlobalMethodSecurity)
```

| Annotation       | Source           | Description                                                  |
|------------------|------------------|--------------------------------------------------------------|
| `@PreAuthorize`  | Spring Security  | SpEL before method; most flexible                            |
| `@PostAuthorize` | Spring Security  | SpEL after method; useful to check return value              |
| `@Secured`       | Spring Security  | Role-based, no SpEL                                          |
| `@RolesAllowed`  | JSR-250          | Role-based (standard annotation)                             |
| `@PreFilter`     | Spring Security  | Filters input collection before method                       |
| `@PostFilter`    | Spring Security  | Filters returned collection                                  |

---

## 3. How It Works Internally

### 3.1 Full Request Authentication Flow (Form Login)

```
1. HTTP POST /login arrives at server
2. DelegatingFilterProxy → FilterChainProxy
3. UsernamePasswordAuthenticationFilter matches /login
4. Extracts username/password from request
5. Creates UsernamePasswordAuthenticationToken(username, password) [unauthenticated]
6. Calls AuthenticationManager.authenticate(token)
   └── ProviderManager iterates providers
       └── DaoAuthenticationProvider:
           a. calls UserDetailsService.loadUserByUsername(username)
           b. UserDetails retrieved from DB
           c. PasswordEncoder.matches(rawPassword, storedHash)
           d. Checks account flags: enabled, non-locked, non-expired
           e. Returns fully-populated Authentication object
7. Authentication stored in SecurityContextHolder
8. SessionAuthenticationStrategy creates/updates session
9. AuthenticationSuccessHandler called → redirects or returns 200
```

### 3.2 JWT Authentication Flow

```
1. HTTP POST /login with credentials
2. Custom filter or endpoint validates credentials
3. On success: JwtUtil.generateToken(userDetails) → signed JWT string
4. Client stores JWT (localStorage or httpOnly cookie)

--- Subsequent requests ---
5. Client sends: Authorization: Bearer <jwt>
6. BearerTokenAuthenticationFilter extracts token
7. JwtAuthenticationProvider.authenticate():
   a. Parses JWT
   b. Validates signature with secret/public key
   c. Checks expiration
   d. Loads UserDetails (optional — can be in JWT claims)
   e. Creates JwtAuthenticationToken
8. Authentication stored in SecurityContextHolder (no DB lookup)
9. AuthorizationFilter checks roles/permissions
10. Request proceeds to controller
```

### 3.3 Method Security (AOP Proxy)

```
Client calls orderService.deleteOrder(id)
     │
     ▼
CGLIB proxy wraps real bean
     │
     ▼
MethodSecurityInterceptor (AOP advice)
     │
     ▼
@PreAuthorize("hasRole('ADMIN')") evaluated via SpEL
     │
  [GRANTED] ──────────────────────────────────► real method executes
  [DENIED]  → AccessDeniedException thrown → ExceptionTranslationFilter → 403
```

---

## 4. Key APIs — Annotations / Classes / Interfaces / Methods

### Annotations

| Annotation                    | Purpose                                                                          |
|-------------------------------|----------------------------------------------------------------------------------|
| `@EnableWebSecurity`          | Activates Spring Security's web configuration (auto-applied in Spring Boot)      |
| `@EnableMethodSecurity`       | Enables method-level security annotations (`@PreAuthorize`, etc.)                |
| `@PreAuthorize`               | SpEL expression evaluated before method execution                                |
| `@PostAuthorize`              | SpEL expression evaluated after method execution                                 |
| `@Secured`                    | List of roles that can access a method (no SpEL)                                 |
| `@RolesAllowed`               | JSR-250 equivalent of `@Secured`                                                 |
| `@WithMockUser`               | Test annotation — populates SecurityContext with a mock user                     |
| `@WithUserDetails`            | Test annotation — loads a real `UserDetails` for tests                           |

### Core Classes & Interfaces

| Class/Interface                        | Type       | Purpose                                                                   |
|----------------------------------------|------------|---------------------------------------------------------------------------|
| `SecurityFilterChain`                  | Interface  | Defines which filters apply to which requests                             |
| `FilterChainProxy`                     | Class      | Spring Security's main servlet filter; delegates to `SecurityFilterChain` |
| `DelegatingFilterProxy`                | Class      | Bridge between Servlet container and Spring context                       |
| `AuthenticationManager`                | Interface  | `authenticate(Authentication)` — entry point for authentication           |
| `ProviderManager`                      | Class      | Default `AuthenticationManager`; delegates to `AuthenticationProvider`s   |
| `AuthenticationProvider`               | Interface  | Performs actual authentication for a specific `Authentication` type       |
| `DaoAuthenticationProvider`            | Class      | Uses `UserDetailsService` + `PasswordEncoder`                             |
| `UserDetails`                          | Interface  | User info: username, password, authorities, account flags                 |
| `UserDetailsService`                   | Interface  | `loadUserByUsername(String)` — load user from store                       |
| `SecurityContext`                       | Interface  | Holds the `Authentication` for the current thread                        |
| `SecurityContextHolder`                | Class      | ThreadLocal storage for `SecurityContext`                                 |
| `Authentication`                       | Interface  | Token for auth request or authenticated user                              |
| `UsernamePasswordAuthenticationToken`  | Class      | Concrete `Authentication` for username/password                           |
| `GrantedAuthority`                     | Interface  | A single permission string                                                |
| `SimpleGrantedAuthority`               | Class      | Simple string-based `GrantedAuthority`                                    |
| `PasswordEncoder`                      | Interface  | `encode(raw)` and `matches(raw, encoded)`                                 |
| `BCryptPasswordEncoder`                | Class      | BCrypt implementation of `PasswordEncoder`                                |
| `HttpSecurity`                         | Class      | DSL for configuring web-based security                                    |
| `ExceptionTranslationFilter`           | Class      | Handles `AuthenticationException` → 401, `AccessDeniedException` → 403   |
| `AccessDeniedException`                | Exception  | Thrown when authenticated user lacks permission                           |
| `AuthenticationException`              | Exception  | Base for all auth failures                                                |
| `AuthorizationManager`                 | Interface  | Spring Security 6 authorization abstraction                               |
| `OncePerRequestFilter`                 | Class      | Base class for custom filters guaranteed to run once per request          |

---

## 5. Configuration & Setup

### Maven Dependency

```xml
<!-- Spring Boot Starter Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- For JWT (JJWT library) -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>

<!-- For testing -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

### application.properties

```properties
# Default in-memory user (dev only)
spring.security.user.name=admin
spring.security.user.password=secret
spring.security.user.roles=ADMIN

# JWT properties (custom)
app.jwt.secret=your-256-bit-secret-key-here-minimum-32-chars
app.jwt.expiration-ms=900000
app.jwt.refresh-expiration-ms=604800000
```

### Minimal Security Configuration (Spring Boot 3 / Spring Security 6)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)           // Stateless API
            .cors(cors -> cors.configurationSource(corsSource()))
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 6. Real-World Production Code Example

**Domain: E-Commerce Platform — JWT-based stateless authentication + role-based authorization**

### Entity

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;  // Stored as BCrypt hash

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "user_roles", joinColumns = @JoinColumn(name = "user_id"))
    @Column(name = "role")
    private Set<String> roles = new HashSet<>();

    private boolean enabled = true;

    // getters/setters
}
```

### Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
```

### UserDetailsService Implementation

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found: " + username));  // Generic message avoids username enumeration

        // Convert roles to GrantedAuthority (Spring expects ROLE_ prefix)
        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
            .collect(Collectors.toList());

        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),
            user.isEnabled(),
            true,  // accountNonExpired
            true,  // credentialsNonExpired
            true,  // accountNonLocked
            authorities
        );
    }
}
```

### JWT Utility

```java
@Component
public class JwtUtil {

    @Value("${app.jwt.secret}")
    private String jwtSecret;

    @Value("${app.jwt.expiration-ms}")
    private long jwtExpirationMs;

    // Use HMAC-SHA256 with a strong secret key
    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(jwtSecret);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList()));

        return Jwts.builder()
            .claims(claims)
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
            .signWith(getSigningKey())
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean validateToken(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    private <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        Claims claims = Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
        return claimsResolver.apply(claims);
    }
}
```

### JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final CustomUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        // If no Bearer token, continue chain (anonymous request)
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String jwt = authHeader.substring(7);
        String username;

        try {
            username = jwtUtil.extractUsername(jwt);
        } catch (JwtException e) {
            // Invalid token — let the request proceed as unauthenticated
            filterChain.doFilter(request, response);
            return;
        }

        // Only authenticate if not already authenticated
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            if (jwtUtil.validateToken(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,               // credentials cleared after auth
                        userDetails.getAuthorities()
                    );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                // Populate SecurityContext — this is what grants access
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

### Auth Controller

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final CustomUserDetailsService userDetailsService;
    private final JwtUtil jwtUtil;
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        // Throws AuthenticationException if credentials are invalid
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(),
                request.getPassword()
            )
        );

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        String jwt = jwtUtil.generateToken(userDetails);

        return ResponseEntity.ok(new AuthResponse(jwt));
    }

    @PostMapping("/register")
    public ResponseEntity<String> register(@Valid @RequestBody RegisterRequest request) {
        if (userRepository.findByUsername(request.getUsername()).isPresent()) {
            return ResponseEntity.badRequest().body("Username already taken");
        }

        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword()));  // ALWAYS hash
        user.setRoles(Set.of("USER"));
        userRepository.save(user);

        return ResponseEntity.status(HttpStatus.CREATED).body("User registered");
    }
}
```

### Order Controller with Method Security

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    // Any authenticated user can view their own orders
    @GetMapping
    public List<Order> getMyOrders(Authentication authentication) {
        return orderService.findByUsername(authentication.getName());
    }

    // Only ADMIN can delete any order
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        orderService.delete(id);
        return ResponseEntity.noContent().build();
    }

    // User can only update their own order (using SpEL with method arguments)
    @PutMapping("/{id}")
    @PreAuthorize("@orderSecurityService.isOwner(authentication, #id)")
    public Order updateOrder(@PathVariable Long id, @RequestBody OrderRequest request) {
        return orderService.update(id, request);
    }
}
```

### Custom Security Evaluator (SpEL Bean)

```java
@Component("orderSecurityService")
@RequiredArgsConstructor
public class OrderSecurityService {

    private final OrderRepository orderRepository;

    public boolean isOwner(Authentication authentication, Long orderId) {
        return orderRepository.findById(orderId)
            .map(order -> order.getUsername().equals(authentication.getName()))
            .orElse(false);
    }
}
```

### Global Exception Handler for Security

```java
@RestControllerAdvice
public class SecurityExceptionHandler {

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ErrorResponse("Access denied", "FORBIDDEN"));
    }
}
```

---

## 7. Production Use-Cases & Best Practices

### 7.1 Stateless JWT for REST APIs
Use `SessionCreationPolicy.STATELESS` + JWT. Never store sensitive data in JWT payload (it's base64-encoded, not encrypted unless using JWE). Sign with RS256 (asymmetric) in production so only the auth server needs the private key; resource servers need only the public key.

### 7.2 Refresh Token Rotation
Access tokens expire quickly (15 min). Refresh tokens are stored in DB with a one-time use mechanism. On refresh: issue new access + refresh token, invalidate old refresh token. On suspicious reuse of a refresh token (token theft), invalidate the entire refresh token family.

### 7.3 Password Storage
Always use `BCryptPasswordEncoder` (strength 10–12) or `Argon2PasswordEncoder`. Use `DelegatingPasswordEncoder` so you can migrate algorithms without forcing password resets.

### 7.4 Role Hierarchy
For `ROLE_ADMIN > ROLE_MANAGER > ROLE_USER`:
```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.withDefaultRolePrefix()
        .role("ADMIN").implies("MANAGER")
        .role("MANAGER").implies("USER")
        .build();
}
```
Without this, you must explicitly assign all roles.

### 7.5 Rate Limiting on Auth Endpoints
Implement rate limiting on `/login` and `/register` to prevent brute-force and credential stuffing attacks. Use Bucket4j or a reverse proxy (Nginx, AWS WAF).

### 7.6 Secure Cookie Flags
When using cookies (session or refresh token), always set:
- `HttpOnly` — prevents JavaScript access
- `Secure` — HTTPS only
- `SameSite=Strict` or `Lax` — prevents CSRF

### 7.7 Least Privilege Principle
Define fine-grained permissions (`ORDER_READ`, `ORDER_WRITE`) not just coarse roles. Use `@PreAuthorize("hasAuthority('ORDER_WRITE')")` for method security. Map permissions to roles in a `GrantedAuthority` hierarchy.

### 7.8 CORS Configuration
Always explicitly configure allowed origins — never use `*` with `allowCredentials(true)` (browsers block it anyway). In production, whitelist only known frontend origins.

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.mycompany.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

---

## 8. Scenario Walkthroughs

### Scenario 1: A JWT token is stolen — what happens?

**Setup:** User A's JWT (valid for 15 min) is stolen via XSS. Attacker uses it to call `/api/orders`.

**What happens step by step:**
1. Attacker sends `Authorization: Bearer <stolen_jwt>`.
2. `JwtAuthenticationFilter` extracts and validates the token — **signature is valid, token is not expired**.
3. SecurityContext is populated with User A's identity.
4. Attacker gets User A's orders.

**Why this is a problem:** JWTs are **self-contained and stateless** — the server has no way to invalidate a specific token before expiry.

**How to handle it correctly:**
- Keep access token lifetime short (5–15 min).
- Maintain a **token blacklist/denylist** in Redis: on logout or suspected theft, add the JWT ID (`jti` claim) to the denylist. Check the denylist in the JWT filter.
- Alternatively, use **opaque tokens** (random strings) with a token store — stateful but revocable instantly.
- Prevent XSS that steals tokens: use `HttpOnly` cookies for JWT storage instead of localStorage.

---

### Scenario 2: `@PreAuthorize` not working — method called but no security check

**Setup:** You added `@PreAuthorize("hasRole('ADMIN')")` to a service method, but any user can still call it.

**Root cause walkthrough:**
1. Spring Security method security works via **AOP proxies**.
2. If you call the method **from within the same class** (self-invocation), the call bypasses the proxy — goes directly to `this.method()`, not through the CGLIB/JDK proxy.
3. If `@EnableMethodSecurity` is missing, no AOP interceptor is registered.
4. If the bean is created outside the Spring container (e.g., `new MyService()`), it has no proxy wrapper.

**How to fix:**
- Add `@EnableMethodSecurity` to a configuration class.
- Inject the service into itself (`@Lazy @Autowired private MyService self`) and call `self.securedMethod()` — not `this.securedMethod()`.
- Better: refactor so the secured method is in a different bean.

---

### Scenario 3: CORS preflight failing for authenticated endpoints

**Setup:** Vue.js frontend on `localhost:3000` calls Spring Boot API on `localhost:8080`. GET requests work but POST fails with CORS error.

**Root cause walkthrough:**
1. Browser sends `OPTIONS` preflight to check CORS headers.
2. Spring Security's filter chain intercepts the `OPTIONS` request.
3. Since `OPTIONS` has no `Authorization` header, it hits `AnonymousAuthenticationFilter`.
4. `AuthorizationFilter` denies the anonymous preflight → 403 before CORS headers are added.
5. Browser sees 403, blocks the actual POST.

**How to fix:**
```java
http.cors(cors -> cors.configurationSource(corsConfigurationSource()))
    // Permit all OPTIONS requests (they never carry credentials anyway)
    .authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
        .anyRequest().authenticated()
    );
```
OR: configure CORS at the `CorsConfigurationSource` bean level, which Spring Security respects before authorization checks.

---

## 9. Common Pitfalls & Gotchas

### Pitfall 1: `WebSecurityConfigurerAdapter` deprecated in Spring Security 5.7+
**Problem:** Old tutorials use `extends WebSecurityConfigurerAdapter` — this is removed in Spring Security 6 (Spring Boot 3).  
**Fix:** Use `@Bean SecurityFilterChain` and `@Bean AuthenticationManager` pattern.

---

### Pitfall 2: `hasRole("ADMIN")` vs `hasAuthority("ROLE_ADMIN")`
**Problem:** Storing `ROLE_ADMIN` in DB but checking with `hasRole("ROLE_ADMIN")` — double-prepends to `ROLE_ROLE_ADMIN`.  
**Fix:** `hasRole("ADMIN")` adds `ROLE_` prefix automatically. `hasAuthority("ROLE_ADMIN")` uses exact string. Pick one convention and be consistent.

---

### Pitfall 3: `@PreAuthorize` ignored due to self-invocation
**Problem:** `orderService.createOrder()` calls `this.validateOrder()` which has `@PreAuthorize` — security is bypassed.  
**Fix:** Move `validateOrder()` to a separate bean, or use `ApplicationContext.getBean()` / self-injection.

---

### Pitfall 4: Password stored in plaintext
**Problem:** `user.setPassword(request.getPassword())` — classic beginner mistake.  
**Fix:** Always `passwordEncoder.encode(rawPassword)` before saving.

---

### Pitfall 5: CSRF disabled universally
**Problem:** Some developers disable CSRF for all endpoints including traditional server-rendered forms.  
**Fix:** Only disable CSRF for true stateless REST APIs. For form-based or cookie-session apps, keep CSRF enabled.

---

### Pitfall 6: JWT secret too short or hardcoded
**Problem:** `String secret = "secret123"` — weak key, stored in code.  
**Fix:** Use a minimum 256-bit secret for HMAC-SHA256. Load from environment variable or secrets manager (Vault, AWS Secrets Manager).

---

### Pitfall 7: Not clearing credentials from `Authentication` object
**Problem:** The raw password is stored in `UsernamePasswordAuthenticationToken.credentials` after authentication.  
**Fix:** `DaoAuthenticationProvider` automatically erases credentials by default. If you build a custom provider, call `((UsernamePasswordAuthenticationToken) auth).eraseCredentials()`.

---

### Pitfall 8: `SecurityContextHolder` in async code
**Problem:** `@Async` method — new thread doesn't inherit `SecurityContext` (default `ThreadLocal` strategy).  
**Fix:** Use `SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL)` or configure `DelegatingSecurityContextAsyncTaskExecutor`:
```java
@Bean
public Executor taskExecutor() {
    return new DelegatingSecurityContextAsyncTaskExecutor(
        new ThreadPoolTaskExecutor());
}
```

---

### Pitfall 9: `@Secured` vs `@PreAuthorize` confusion
**Problem:** `@Secured({"ROLE_ADMIN"})` requires exact role string with prefix. `@PreAuthorize("hasRole('ADMIN')")` prepends `ROLE_` automatically.  
**Fix:** Prefer `@PreAuthorize` — it's more flexible and the most commonly used in modern Spring Security.

---

### Pitfall 10: Not handling `AuthenticationException` vs `AccessDeniedException` separately
**Problem:** Single exception handler catches all exceptions — returns 500 for both unauthorized and forbidden.  
**Fix:** `AuthenticationException` → 401 (not authenticated). `AccessDeniedException` → 403 (authenticated but not authorized). Spring Security's `ExceptionTranslationFilter` does this automatically, but custom handlers must differentiate.

---

### Pitfall 11: CORS vs Security filter order
**Problem:** CORS configuration on Spring MVC `@CrossOrigin` or `WebMvcConfigurer` runs after Spring Security filter chain — OPTIONS preflights fail.  
**Fix:** Configure CORS within `HttpSecurity.cors()` so it runs inside the Security filter chain, before authorization checks.

---

### Pitfall 12: Method security on `@Configuration` classes
**Problem:** `@PreAuthorize` on methods in a `@Configuration` class — can cause proxy issues since `@Configuration` classes are already CGLIB-proxied.  
**Fix:** Avoid securing configuration methods; place security annotations on `@Service` / `@Component` beans.

---

## 10. Miscellaneous & Advanced Details

### Spring Security 5 vs Spring Security 6 (Spring Boot 3)

| Feature                        | Spring Security 5.x                          | Spring Security 6.x (Boot 3)                         |
|--------------------------------|----------------------------------------------|------------------------------------------------------|
| Configuration base class       | `WebSecurityConfigurerAdapter`               | Removed — use `@Bean SecurityFilterChain`            |
| Method security annotation     | `@EnableGlobalMethodSecurity`                | `@EnableMethodSecurity` (default prePostEnabled=true)|
| Authorization architecture     | `AccessDecisionManager` + Voters             | `AuthorizationManager` (simpler, composable)         |
| Request matcher                | `antMatchers()`, `mvcMatchers()`             | `requestMatchers()` (auto-selects best matcher)      |
| `authorizeRequests()`          | Available                                    | Deprecated; use `authorizeHttpRequests()`            |
| Lambda DSL                     | Optional                                     | Preferred/enforced style                             |

### Proxy Mechanism: JDK Dynamic Proxy vs CGLIB

- If your bean implements an interface → **JDK Dynamic Proxy** by default.
- If your bean is a concrete class → **CGLIB proxy**.
- Spring Security method security uses whichever Spring AOP uses.
- Implication: if you inject the interface type, JDK proxy works. If you inject the concrete type, must use CGLIB.
- `@EnableMethodSecurity(proxyTargetClass = true)` forces CGLIB for all.

### Multiple SecurityFilterChains (Spring Security 6)

You can have multiple `SecurityFilterChain` beans for different URL patterns:

```java
@Bean
@Order(1)
public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}

@Bean
@Order(2)
public SecurityFilterChain webSecurityFilterChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/**")
        .formLogin(Customizer.withDefaults())
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
    return http.build();
}
```

### OAuth2 Resource Server (JWT validation built-in)

Spring Security has built-in OAuth2 Resource Server support:

```java
http.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
```

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=https://accounts.google.com
```

Automatically validates JWT signature using the authorization server's JWK Set URI.

### SecurityContextPersistenceFilter vs SecurityContextHolderFilter
- Spring Security 5: `SecurityContextPersistenceFilter` loads context from `HttpSession` for stateful apps.
- Spring Security 6: Replaced by `SecurityContextHolderFilter` — does NOT save context automatically; the filter that created it is responsible.

### Content Security Policy & Security Headers
Spring Security adds sensible default security headers:

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())
    .contentSecurityPolicy(csp ->
        csp.policyDirectives("default-src 'self'; script-src 'self'"))
    .httpStrictTransportSecurity(hsts ->
        hsts.includeSubDomains(true).maxAgeInSeconds(31536000))
);
```

---

## 11. Interview Questions — Standard

**Q1: What is the difference between authentication and authorization in Spring Security?**

Authentication is the process of verifying who you are — it involves presenting credentials (username/password, JWT, API key) and having them validated against a user store. The result is a populated `Authentication` object in the `SecurityContextHolder`. Authorization, on the other hand, determines what an authenticated user is allowed to do — it happens after authentication and involves checking `GrantedAuthority` objects against the required permissions for the requested resource. In Spring Security, authentication is handled by `AuthenticationManager` → `AuthenticationProvider`, while authorization is handled by `AuthorizationFilter` → `AuthorizationManager`.

---

**Q2: What is the SecurityContextHolder and how does it store the security context?**

`SecurityContextHolder` is a static utility class that stores the `SecurityContext` for the currently executing thread. By default, it uses a `ThreadLocal` storage strategy, meaning each thread has its own isolated `SecurityContext`. This works correctly for traditional servlet-based applications where one request is processed by one thread from start to finish. The strategy can be changed to `InheritableThreadLocal` (child threads inherit parent's context) or `GLOBAL` (single context for all threads — rarely used). After each request, Spring Security clears the `SecurityContextHolder` to prevent leaking authentication data to the next request processed by the same thread from a thread pool.

---

**Q3: How does `DaoAuthenticationProvider` work?**

`DaoAuthenticationProvider` is the most commonly used `AuthenticationProvider`. It receives a `UsernamePasswordAuthenticationToken`, calls `UserDetailsService.loadUserByUsername()` to retrieve the stored user, then uses `PasswordEncoder.matches(rawPassword, storedHash)` to verify the password. It also checks the four account flags on `UserDetails`: `isEnabled()`, `isAccountNonExpired()`, `isCredentialsNonExpired()`, and `isAccountNonLocked()`. If all checks pass, it returns a fully authenticated `UsernamePasswordAuthenticationToken` with the user's authorities. If any check fails, it throws an appropriate `AuthenticationException` subclass (`BadCredentialsException`, `DisabledException`, `LockedException`, etc.).

---

**Q4: What is the difference between `@PreAuthorize` and `@Secured`?**

Both are method-level security annotations, but they differ in capability. `@Secured` accepts a list of role strings (e.g., `@Secured("ROLE_ADMIN")`) with no SpEL support — it can only check exact role names. `@PreAuthorize` supports the full Spring Expression Language (SpEL) and can express complex conditions: `@PreAuthorize("hasRole('ADMIN') and #orderId > 0")` or `@PreAuthorize("@mySecurityBean.canAccess(authentication, #id)")`. `@PreAuthorize` is evaluated before the method executes. Both require the corresponding feature to be enabled: `@EnableMethodSecurity` for `@PreAuthorize`/`@PostAuthorize`, with `securedEnabled = true` to enable `@Secured`. In modern applications, `@PreAuthorize` is strongly preferred.

---

**Q5: How does Spring Security handle CSRF protection?**

CSRF (Cross-Site Request Forgery) attacks trick authenticated users into sending unwanted requests. Spring Security protects against this using the **Synchronizer Token Pattern**: on each session, a unique random CSRF token is generated and stored in the HTTP session. For every state-changing request (POST/PUT/DELETE/PATCH), the token must be included either as a request parameter (`_csrf`) or header (`X-CSRF-TOKEN`). The `CsrfFilter` compares the submitted token with the session-stored token; if they don't match, the request is rejected with 403. For stateless REST APIs that use JWT (no cookies/sessions), CSRF is typically disabled since CSRF attacks exploit session cookies — without them, the attack vector doesn't exist.

---

**Q6: Why is JWT stateless and what are the trade-offs?**

A JWT is **self-contained**: the payload carries the user identity and roles, and the signature ensures it hasn't been tampered with. The server validates the signature without any database lookup or session store — this is why it's stateless and scales horizontally with no shared state. Trade-offs: **Pro** — no server-side session storage, works across microservices, scales easily. **Con** — tokens cannot be invalidated before expiry (logout doesn't truly log out), the payload is visible (base64-encoded, not encrypted), and large payloads add overhead to every request. Mitigations include short expiry (15 min), refresh tokens with revocation lists in Redis, and using JWE for sensitive claims.

---

**Q7: What is the difference between `FilterChainProxy` and `DelegatingFilterProxy`?**

`DelegatingFilterProxy` is a standard Servlet `Filter` registered in the servlet container (web.xml or Spring Boot auto-registration). Its sole purpose is to bridge the Servlet world to the Spring ApplicationContext — it looks up a Spring bean by name and delegates filter calls to it. The bean it delegates to is `FilterChainProxy`, which is a Spring-managed bean. `FilterChainProxy` is the real entry point for Spring Security — it holds all the `SecurityFilterChain` instances and routes each request to the appropriate chain based on the URL pattern. This separation allows Spring Security's filters to be Spring beans with all Spring capabilities (DI, AOP, etc.).

---

**Q8: How does Spring Security's password encoding work and what is `DelegatingPasswordEncoder`?**

`PasswordEncoder` provides two methods: `encode(rawPassword)` for hashing and `matches(rawPassword, encodedPassword)` for verification. Since Spring Security 5, the default encoder is `DelegatingPasswordEncoder`, which stores a prefix with each hash indicating the algorithm used: `{bcrypt}$2a$10$...`. On verification, it reads the prefix and delegates to the appropriate encoder. This allows you to migrate from one algorithm to another: existing users have `{sha256}` hashes, new users get `{bcrypt}` hashes, and the system handles both transparently. To upgrade a user's hash on login, implement `PasswordEncoderUpgradeManager` or check if re-encoding is needed after successful authentication.

---

**Q9: How do you implement role hierarchy in Spring Security?**

Without role hierarchy, assigning `ROLE_ADMIN` doesn't automatically grant `ROLE_USER` permissions — you'd have to assign all roles explicitly. Spring Security supports `RoleHierarchy`:

```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.withDefaultRolePrefix()
        .role("ADMIN").implies("MANAGER")
        .role("MANAGER").implies("USER")
        .build();
}

@Bean
public DefaultWebSecurityExpressionHandler webSecurityExpressionHandler() {
    DefaultWebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy());
    return handler;
}
```

With this, a user with `ROLE_ADMIN` automatically satisfies `hasRole("USER")` and `hasRole("MANAGER")` checks.

---

**Q10: What security headers does Spring Security add by default?**

Spring Security automatically adds several HTTP security headers: `X-Content-Type-Options: nosniff` (prevents MIME-type sniffing), `X-Frame-Options: DENY` (prevents clickjacking), `X-XSS-Protection: 0` (deprecated in modern browsers but still sent), `Cache-Control: no-cache, no-store, max-age=0, must-revalidate` (prevents caching of authenticated responses), `Pragma: no-cache`, and `Expires: 0`. In HTTPS environments, it also adds `Strict-Transport-Security` (HSTS). These defaults can be customized or disabled via `http.headers()`.

---

**Q11: What happens when an `AccessDeniedException` is thrown?**

The `ExceptionTranslationFilter` intercepts it. If the current user is **anonymous** (not authenticated), it redirects to the login page (for form login) or sends a 401 response with a `WWW-Authenticate` header (for HTTP Basic/Bearer). If the user is **authenticated** but lacks permission, it delegates to the configured `AccessDeniedHandler`, which by default sends a 403 Forbidden response. This is why `ExceptionTranslationFilter` sits near the end of the filter chain — it needs to process any exceptions thrown by the `AuthorizationFilter` and earlier authentication-requiring filters.

---

**Q12: How does Spring Security integrate with Spring Boot?**

Spring Boot auto-configures Spring Security via `SecurityAutoConfiguration` and `SpringBootWebSecurityConfiguration`. When `spring-boot-starter-security` is on the classpath, Spring Boot: (1) creates a default `InMemoryUserDetailsManager` with a random password printed at startup, (2) enables HTTP Basic auth for all endpoints, (3) enables CSRF, (4) enables standard security headers. You override these defaults by providing your own `@Bean SecurityFilterChain`. The auto-configuration backs off entirely when it detects a user-defined `SecurityFilterChain` bean.

---

## 12. Interview Questions — Tricky / Scenario-Based

**Q1: You added `@PreAuthorize("hasRole('ADMIN')")` to a service method, but it has no effect — any user can call it. Why?**

The most likely cause is **self-invocation**. If another method in the same class calls the secured method using `this.securedMethod()`, the call bypasses the Spring AOP proxy — it goes directly to the target object, skipping the security interceptor. The `@PreAuthorize` annotation only works when the call goes through the Spring proxy. Other causes: `@EnableMethodSecurity` is missing from configuration; the bean was instantiated manually (`new MyService()`) outside the Spring container; or the method is `private`/`final` (CGLIB cannot proxy these). Fix by ensuring the call goes through an injected bean reference, not `this`.

---

**Q2: What's the difference between `STATELESS` session policy and actually being stateless? Does JWT always mean stateless?**

`SessionCreationPolicy.STATELESS` tells Spring Security to **never create or use an HTTP session** for storing the `SecurityContext`. However, this alone doesn't make your application stateless — if your application code or other filters create sessions (e.g., Spring Session, custom code), sessions will still be created. JWT authentication is stateless in the sense that the server validates the token without looking up a session store. But if you maintain a **token blacklist/denylist in Redis for revocation**, you've introduced state — just a different kind. "Stateless JWT" in pure form means no server-side state per user at all; the server trusts the token's signature and expiry completely.

---

**Q3: In a microservices architecture, Service A calls Service B internally. How should Service B authenticate Service A?**

Service-to-service calls are **machine-to-machine**, not user-to-service. Options: (1) **OAuth2 Client Credentials flow** — Service A requests a token from the Authorization Server using its client ID/secret, then passes the Bearer token to Service B. Service B validates it as an OAuth2 Resource Server. This is the enterprise standard. (2) **Propagate the user JWT** — Service A forwards the user's JWT in the `Authorization` header to Service B. Service B validates the same JWT. Simple but couples services to user context. (3) **Mutual TLS (mTLS)** — both services authenticate each other via client certificates. High security, complex setup. In practice, OAuth2 Client Credentials is the recommended approach.

---

**Q4: A user logs out. The JWT (15-min expiry) is now in the hands of an attacker who captured it. It's been 5 minutes. Are you safe?**

No. JWTs are stateless — the server has no way to distinguish valid tokens from revoked ones unless you maintain server-side state. After logout, the token is still cryptographically valid for 10 more minutes. **Safe only if** you have a token blacklist: on logout, add the `jti` (JWT ID) claim of the token to a Redis set with a TTL matching the token expiry. Your JWT filter checks the blacklist on every request. This reintroduces a small amount of state but enables instant revocation. Alternatively, use very short-lived tokens (5 min) to minimize the attack window. This is a deliberate trade-off in JWT-based auth: convenience vs. instant revocation.

---

**Q5: Why does CSRF protection need to be disabled for REST APIs but not for traditional web apps?**

CSRF attacks exploit the fact that browsers automatically include session cookies with every request to a domain. A malicious site can trick a user's browser into sending a POST to your site, and the browser automatically attaches the session cookie — making it appear as a legitimate request. For REST APIs using JWT in the `Authorization` header: browsers do NOT automatically send `Authorization` headers cross-origin (this requires JavaScript, which same-origin policy restricts). So the CSRF attack vector doesn't apply. However, if your REST API uses cookies to store the JWT or session ID, CSRF protection should remain enabled or you should use `SameSite=Strict` cookies. The real rule is: "Disable CSRF only if you're truly not using cookies for authentication state."

---

**Q6: You have `@PreAuthorize("hasRole('ADMIN')")` on a service method. A user with `ROLE_SUPER_ADMIN` is denied. Why?**

Without role hierarchy configuration, `ROLE_SUPER_ADMIN` and `ROLE_ADMIN` are completely separate, unrelated roles. Spring Security doesn't infer any hierarchy — it does exact string matching. `hasRole('ADMIN')` checks for the string `ROLE_ADMIN` in the user's authorities. `ROLE_SUPER_ADMIN` is a different string. To fix: configure `RoleHierarchy` with `SUPER_ADMIN implies ADMIN`, or grant both roles to super admins, or change the check to `hasAnyRole('ADMIN', 'SUPER_ADMIN')`.

---

**Q7: Two requests come in simultaneously for the same user. Is there a race condition with `SecurityContextHolder`?**

No, for typical servlet-based applications. `SecurityContextHolder` uses `ThreadLocal` by default, so each thread has its own isolated `SecurityContext`. Thread A's context for Request 1 and Thread B's context for Request 2 are completely separate. They cannot interfere with each other. A race condition would only be possible if you used `MODE_GLOBAL` (single shared context) — which is an unusual configuration. However, with `InheritableThreadLocal` mode, a child thread spawned from a request thread does inherit the parent's `SecurityContext`, which could be a concern if the child thread outlives the request cleanup.

---

**Q8: Your Spring Security filter is being called twice per request. Why?**

The most common cause: the custom filter is both (1) registered as a `@Component` (auto-detected by Spring Boot and registered in the root application filter chain) AND (2) added to the `SecurityFilterChain` via `http.addFilterBefore(...)`. Spring Boot automatically registers any `@Component` bean that extends `OncePerRequestFilter`/`Filter`, and then Spring Security adds it again. **Fix:** Remove `@Component` from the filter class and instead only add it via `http.addFilterBefore(new MyFilter(...), ...)`, OR keep `@Component` but add:

```java
@Bean
public FilterRegistrationBean<JwtAuthenticationFilter> jwtFilterRegistration(
        JwtAuthenticationFilter filter) {
    FilterRegistrationBean<JwtAuthenticationFilter> registration =
        new FilterRegistrationBean<>(filter);
    registration.setEnabled(false); // Disable auto-registration
    return registration;
}
```

---

**Q9: How does Spring Security handle the "remember me" feature, and what are the security implications?**

Spring Security's "remember me" creates a persistent login token stored in the DB and sent as a cookie. On subsequent visits, the `RememberMeAuthenticationFilter` reads the cookie, looks up the token in the DB, validates it, and authenticates the user. **Security implications:** (1) The cookie, if stolen, allows impersonation until it expires or is revoked — often days or weeks. (2) Always use `HttpOnly` + `Secure` cookie flags. (3) Spring Security 5+ uses `PersistentTokenRepository` (DB-backed) which invalidates the old token on each use (token rotation), mitigating token theft. (4) For sensitive operations (password change, payment), re-authentication should be required even with a valid remember-me session (`isFullyAuthenticated()` vs `isAuthenticated()`).

---

**Q10: What happens if you put `@Transactional` and `@PreAuthorize` on the same method?**

Both annotations work via **AOP proxies**. The order in which the advice executes depends on `@Order` on the aspects. By default, `@PreAuthorize` (from `MethodSecurityInterceptor`) runs **before** `@Transactional` (from `TransactionInterceptor`), which is the correct and safe order — security check happens first, and if denied, no transaction is opened. If the order were reversed (transaction opens, then security check fails), the transaction would be rolled back cleanly, but it wastes resources and could expose timing information. In Spring Security 6, the default order of `MethodSecurityInterceptor` is 100, while `TransactionInterceptor` defaults to `Integer.MAX_VALUE - 5` (~lowest priority), so security runs first.

---

## 13. Quick Revision Summary

- **Authentication** = Who are you? (`AuthenticationManager` → `AuthenticationProvider` → `UserDetailsService`). **Authorization** = What can you do? (`AuthorizationFilter` → `AuthorizationManager`).

- `SecurityContextHolder` stores the `SecurityContext` per thread via `ThreadLocal`. It must be cleared after each request (Spring does this automatically). For `@Async` methods, use `DelegatingSecurityContextAsyncTaskExecutor`.

- The filter chain: `DelegatingFilterProxy` (servlet) → `FilterChainProxy` (Spring bean) → `SecurityFilterChain` (ordered list of filters). Key filters: `UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter`, `ExceptionTranslationFilter`, `AuthorizationFilter`.

- `hasRole("ADMIN")` automatically prepends `ROLE_`; `hasAuthority("ROLE_ADMIN")` uses the exact string. Mixing these up is a very common bug.

- `@PreAuthorize` uses SpEL and is the preferred method-security annotation. It only works via Spring AOP proxy — self-invocation bypasses it entirely.

- JWT is stateless and cannot be revoked before expiry by default. Use short expiry (5–15 min) + refresh tokens + a Redis-backed denylist (`jti` blacklist) for revocation.

- Disable CSRF only for true stateless APIs (JWT in `Authorization` header). Keep it enabled for cookie/session-based authentication.

- Configure CORS inside `HttpSecurity.cors()`, not just at the MVC layer — otherwise OPTIONS preflights fail because Spring Security denies them before CORS headers are added.

- `WebSecurityConfigurerAdapter` is removed in Spring Security 6 (Spring Boot 3). Use `@Bean SecurityFilterChain` pattern.

- `DaoAuthenticationProvider` checks 5 things: password match + 4 account flags (enabled, non-expired, credentials-non-expired, non-locked). Each failure throws a specific `AuthenticationException` subclass.

- `DelegatingPasswordEncoder` stores algorithm prefix (`{bcrypt}`) with every hash — enables algorithm migration without forcing password resets.

- For `@Async` or child thread access to security context: set `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` or use `DelegatingSecurityContextAsyncTaskExecutor`. Default `ThreadLocal` mode gives child threads an empty context.
