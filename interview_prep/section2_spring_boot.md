# Section 2: Spring Boot

---

## 2.1 Core Spring Boot Concepts

Spring Boot's auto-configuration scans the classpath and wires beans without XML. The container
manages every bean's lifecycle — creation, injection, destruction. Understanding *how* Spring
creates and connects beans is essential for diagnosing subtle bugs: circular dependencies,
beans not being proxied when they should be, `@Transactional` not working because a method is
called internally. At RBC, field injection caused untestable code in the auth service; moving
to constructor injection fixed it.

**One-sentence interview answer:** "Spring Boot auto-configures beans by scanning classpath
dependencies; the IoC container manages lifecycle; constructor injection is preferred because
it makes dependencies explicit and enables immutable, testable objects."

**Most likely follow-ups:**
1. What is the difference between `@Component`, `@Service`, `@Repository`?
2. What happens if two beans of the same type exist — how does Spring resolve it?
3. What is a circular dependency and how do you fix it?

```java
// ── Auto-configuration ────────────────────────────────────────────────────
// Spring Boot reads META-INF/spring/org.springframework.boot.autoconfigure.
// AutoConfiguration.imports and activates classes annotated @AutoConfiguration
// when their @ConditionalOn* conditions are met.
//
// Example: DataSourceAutoConfiguration activates when:
//   1. HikariCP is on the classpath
//   2. No DataSource bean is already defined
//   3. spring.datasource.url is set in application.properties
//
// You override auto-config by declaring your own bean:
// @Bean DataSource dataSource() { ... }  ← Spring sees this, skips auto-config

// ── Bean Lifecycle ────────────────────────────────────────────────────────
import org.springframework.beans.factory.annotation.*;
import org.springframework.context.annotation.*;
import org.springframework.stereotype.*;
import org.springframework.transaction.annotation.*;
import javax.annotation.*;

@Repository  // = @Component + marks as DAO layer + enables exception translation
             // (DataAccessException wrapping — Spring translates JDBC/JPA exceptions
             //  to its hierarchy so callers don't couple to database vendor exceptions)
public class InvestorRepository {
    // ... JPA operations
    public InvestorEntity findById(String id) { return new InvestorEntity(id); }
    public void save(InvestorEntity entity) {}
}

@Service  // = @Component + semantic marker for business logic layer
          // No functional difference from @Component; use for code clarity
public class InvestorLifecycleService {

    private final InvestorRepository repository;   // constructor injection — preferred

    // Constructor injection advantages:
    // 1. Dependencies visible in constructor signature — immediately clear what's needed
    // 2. final fields — immutable after construction
    // 3. Testable without Spring context — just pass mocks to constructor
    // 4. Fails fast at startup if dependency missing
    @Autowired  // optional when single constructor; explicit for clarity
    public InvestorLifecycleService(InvestorRepository repository) {
        this.repository = repository;
    }

    // @PostConstruct: runs after injection is complete, before bean is used
    // Use for: warm up caches, validate config, establish connections
    @PostConstruct
    public void init() {
        System.out.println("InvestorLifecycleService ready — cache warm-up complete");
    }

    // @PreDestroy: runs before bean is removed from context (app shutdown)
    // Use for: release resources, flush buffers, close connections
    @PreDestroy
    public void cleanup() {
        System.out.println("InvestorLifecycleService shutting down — releasing resources");
    }
}

// ── @Transactional — how it works + pitfalls ──────────────────────────────
// Spring wraps your bean in a proxy. When you call a @Transactional method,
// the proxy begins a transaction, calls your method, then commits or rolls back.
// The proxy only intercepts calls coming FROM OUTSIDE the bean.
//
//  ┌─────────────────────────────────────────────────────────────┐
//  │  Caller → [Spring Proxy] → YourBean.transactionalMethod()  │
//  │                ↑ transaction begins/commits here            │
//  └─────────────────────────────────────────────────────────────┘

@Service
@Transactional(readOnly = true)   // default: read-only for all methods — no accidental writes
public class PortfolioTransactionService {

    private final InvestorRepository repository;

    public PortfolioTransactionService(InvestorRepository repository) {
        this.repository = repository;
    }

    // Overrides class-level readOnly=true for write operations
    @Transactional(
        propagation = Propagation.REQUIRED,    // join existing tx, or create new (default)
        isolation   = Isolation.READ_COMMITTED, // see only committed data (prevents dirty reads)
        rollbackFor = Exception.class          // rollback on ALL exceptions (default: only RuntimeException)
    )
    public void recordTrade(String investorId, double amount) {
        // Both operations in same transaction — either both succeed or both roll back
        InvestorEntity investor = repository.findById(investorId);
        investor.updateBalance(amount);
        repository.save(investor);
    }

    // ── @Transactional PITFALL 1: Self-invocation ──────────────────────────
    // This does NOT start a transaction because the call goes directly to 'this',
    // bypassing the proxy. The @Transactional annotation is ignored.
    public void processPortfolioRebalance(String investorId) {
        this.recordTrade(investorId, 1000.0);  // WRONG: proxy not involved
    }

    // Fix: inject self, or move to separate bean, or use AspectJ weaving
    // Best fix: move recordTrade to a separate @Service so proxy wraps the call

    // ── @Transactional PITFALL 2: Checked exceptions don't rollback by default
    // Spring only rolls back on RuntimeException unless rollbackFor specified above
    // @Transactional without rollbackFor means checked Exception won't rollback!

    // ── @Transactional PITFALL 3: public methods only
    // Spring proxy only intercepts public methods. @Transactional on private/protected = no-op.
}

// ── Spring Profiles ───────────────────────────────────────────────────────
// application.properties            ← always loaded
// application-dev.properties        ← loaded when profile = dev
// application-prod.properties       ← loaded when profile = prod
//
// Activate: SPRING_PROFILES_ACTIVE=prod (env var) or -Dspring.profiles.active=prod
//
// At RBC: dev profile uses local PostgreSQL + mock Interac API
//         prod profile uses RDS + real Interac API + Redis cluster

@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")   // only active in dev
    public DataSource devDataSource() {
        // H2 in-memory for local development
        return DataSourceBuilder.create()
                .url("jdbc:h2:mem:investor_dev")
                .username("sa").password("").build();
    }

    @Bean
    @Profile("prod")  // only active in prod
    public DataSource prodDataSource() {
        // HikariCP connecting to RDS PostgreSQL
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(System.getenv("DB_URL"));           // from ECS task env vars
        config.setUsername(System.getenv("DB_USER"));
        config.setPassword(System.getenv("DB_PASSWORD"));
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(3000);                    // 3s — fail fast
        config.setIdleTimeout(600000);                        // 10 min idle eviction
        return new HikariDataSource(config);
    }
}

// Stubs for compilation
class InvestorEntity {
    InvestorEntity(String id) {}
    void updateBalance(double amount) {}
}

class DataSourceBuilder {
    static DataSourceBuilder create() { return new DataSourceBuilder(); }
    DataSourceBuilder url(String u) { return this; }
    DataSourceBuilder username(String u) { return this; }
    DataSourceBuilder password(String p) { return this; }
    javax.sql.DataSource build() { return null; }
}
class HikariConfig {}
class HikariDataSource extends javax.sql.DataSource { ... }  // stub
```

---

## 2.2 REST API Design

A well-designed REST API is predictable, consistent, and handles errors uniformly.
`@ControllerAdvice` + `@ExceptionHandler` gives you a single place to translate domain
exceptions into HTTP responses — without this, every controller has its own try-catch blocks.
At RBC the investor verification controller used `@Valid` on request bodies to catch
malformed requests before they reached the service layer.

**One-sentence interview answer:** "I design REST APIs with consistent URL conventions,
`@Valid` for input validation, `@ControllerAdvice` for centralized error mapping, and
meaningful HTTP status codes that let clients distinguish retriable errors from fatal ones."

**Most likely follow-ups:**
1. What HTTP status code do you return for a validation error vs a server error?
2. What is the difference between `@RequestBody` and `@RequestParam`?
3. How do you version a REST API?

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.http.*;
import org.springframework.validation.*;
import javax.validation.constraints.*;
import javax.validation.*;
import java.util.*;

// ── Request / Response DTOs ───────────────────────────────────────────────
// DTOs (Data Transfer Objects) separate the API contract from the domain model.
// Never expose JPA entities directly — changes to DB schema shouldn't break API.

public record VerificationRequest(
        @NotBlank(message = "investorId is required")
        @Pattern(regexp = "INV-[A-Z0-9]{3,10}", message = "investorId format: INV-XXXXX")
        String investorId,

        @NotBlank(message = "token is required")
        @Size(min = 10, max = 1000, message = "token length invalid")
        String token,

        @NotNull(message = "protocol is required")
        @Pattern(regexp = "REST|GRPC|INTERAC", message = "protocol must be REST, GRPC, or INTERAC")
        String protocol
) {}

public record VerificationResponse(
        String investorId,
        boolean verified,
        String verificationId,
        long processingTimeMs
) {}

public record ErrorResponse(
        int status,
        String error,
        String message,
        Map<String, String> fieldErrors,    // null for non-validation errors
        long timestamp
) {}

// ── Global Exception Handler ──────────────────────────────────────────────
// @ControllerAdvice intercepts exceptions thrown by ANY controller in the app.
// Without this: each controller repeats try-catch; inconsistent error formats.
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handles @Valid failures — returns 400 with per-field error details
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex) {

        Map<String, String> fieldErrors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
                .forEach(e -> fieldErrors.put(e.getField(), e.getDefaultMessage()));

        return ResponseEntity.badRequest().body(new ErrorResponse(
                400, "Validation Failed", "Request contains invalid fields",
                fieldErrors, System.currentTimeMillis()));
    }

    @ExceptionHandler(InvestorNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(InvestorNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorResponse(
                404, "Not Found", ex.getMessage(), null, System.currentTimeMillis()));
    }

    @ExceptionHandler(InvestorVerificationException.class)
    public ResponseEntity<ErrorResponse> handleVerificationFailure(
            InvestorVerificationException ex) {
        // 401 Unauthorized — client should prompt for re-authentication
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(new ErrorResponse(
                401, "Verification Failed", ex.getMessage(), null, System.currentTimeMillis()));
    }

    @ExceptionHandler(InteracApiException.class)
    public ResponseEntity<ErrorResponse> handleInteracFailure(InteracApiException ex) {
        // 502 Bad Gateway — upstream (Interac) failed, client can retry
        return ResponseEntity.status(HttpStatus.BAD_GATEWAY).body(new ErrorResponse(
                502, "Upstream Error", "Interac authentication service unavailable",
                null, System.currentTimeMillis()));
    }

    // Catch-all — never expose internal details to the client
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex) {
        // Log full stack trace internally but return generic message externally
        // (security: don't leak stack traces or database errors to clients)
        return ResponseEntity.internalServerError().body(new ErrorResponse(
                500, "Internal Server Error", "An unexpected error occurred",
                null, System.currentTimeMillis()));
    }
}

// ── Controller ────────────────────────────────────────────────────────────
@RestController
@RequestMapping("/api/v1/investors")
@Validated  // enables @PathVariable / @RequestParam validation (not just @RequestBody)
public class InvestorVerificationController {

    private final InvestorVerificationService verificationService;
    private final InvestorRepository investorRepository;

    // Constructor injection — no @Autowired needed with single constructor in Spring 4.3+
    public InvestorVerificationController(InvestorVerificationService verificationService,
                                          InvestorRepository investorRepository) {
        this.verificationService = verificationService;
        this.investorRepository  = investorRepository;
    }

    // POST /api/v1/investors/verify
    // @Valid triggers Bean Validation on the request body — handled by GlobalExceptionHandler
    @PostMapping("/verify")
    public ResponseEntity<VerificationResponse> verify(
            @Valid @RequestBody VerificationRequest request) throws InteracApiException {

        long start = System.currentTimeMillis();
        boolean verified = verificationService.verifyWithAudit(
                request.investorId(), request.token());

        VerificationResponse response = new VerificationResponse(
                request.investorId(),
                verified,
                UUID.randomUUID().toString(),
                System.currentTimeMillis() - start
        );

        // Return 200 OK with result — don't use 401 here since 401 is for HTTP auth failure
        // The verification result (true/false) is business logic, not HTTP auth
        return ResponseEntity.ok(response);
    }

    // GET /api/v1/investors/{investorId}/portfolio
    @GetMapping("/{investorId}/portfolio")
    public ResponseEntity<InvestorPortfolioDto> getPortfolio(
            @PathVariable @Pattern(regexp = "INV-[A-Z0-9]+") String investorId) {

        // Service throws InvestorNotFoundException → GlobalExceptionHandler → 404
        InvestorEntity investor = investorRepository.findById(investorId);
        return ResponseEntity.ok(toDto(investor));
    }

    // DELETE /api/v1/investors/{investorId} — admin operation
    @DeleteMapping("/{investorId}")
    public ResponseEntity<Void> deactivate(@PathVariable String investorId) {
        // 204 No Content — success with no response body (idiomatic for DELETE)
        investorRepository.findById(investorId);  // throws 404 if not found
        return ResponseEntity.noContent().build();
    }

    private InvestorPortfolioDto toDto(InvestorEntity entity) {
        return new InvestorPortfolioDto(/* map fields */);
    }
}

record InvestorPortfolioDto() {}  // stub
```

---

## 2.3 Spring Security + JWT

JWT (JSON Web Token) is stateless authentication: the token contains signed claims (investorId,
roles, expiry) so the server never needs to look up a session. The Spring Security filter chain
intercepts every request; the JWT filter validates the token *before* the controller runs. At
RBC the token refresh logic had a bug where expired tokens triggered an infinite refresh loop —
fixed by adding a `isRefreshing` flag in the refresh request path.

**One-sentence interview answer:** "JWT authentication works by issuing a signed token on login;
subsequent requests include the token in the Authorization header; a filter validates the
signature and claims before the request reaches any controller."

**Most likely follow-ups:**
1. Where do you store the JWT secret in production?
2. What is the difference between access token and refresh token?
3. How do you revoke a JWT before it expires?

```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.security.authentication.*;
import org.springframework.security.config.annotation.web.builders.*;
import org.springframework.security.config.http.*;
import org.springframework.security.core.*;
import org.springframework.security.core.userdetails.*;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.filter.OncePerRequestFilter;
import javax.crypto.SecretKey;
import javax.servlet.http.*;
import java.util.*;

// ── JWT Token Service ─────────────────────────────────────────────────────
@Service
public class JwtTokenService {

    // In production: load from AWS Secrets Manager / Azure Key Vault, NOT from env var
    // never hardcode secrets in source code
    private final SecretKey signingKey;
    private final long accessTokenTtlMs  = 15 * 60 * 1000L;   // 15 minutes
    private final long refreshTokenTtlMs = 7 * 24 * 60 * 60 * 1000L;  // 7 days

    public JwtTokenService(@Value("${jwt.secret}") String secret) {
        // Key must be at least 256 bits for HS256
        this.signingKey = Keys.hmacShaKeyFor(secret.getBytes());
    }

    public String generateAccessToken(String investorId, List<String> roles) {
        return Jwts.builder()
                .subject(investorId)
                .claim("roles", roles)
                .claim("type", "ACCESS")
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + accessTokenTtlMs))
                .signWith(signingKey)
                .compact();
    }

    public String generateRefreshToken(String investorId) {
        return Jwts.builder()
                .subject(investorId)
                .claim("type", "REFRESH")
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + refreshTokenTtlMs))
                .signWith(signingKey)
                .compact();
    }

    public Claims validateAndExtract(String token) throws JwtException {
        // Throws ExpiredJwtException, MalformedJwtException, SignatureException etc.
        return Jwts.parser()
                .verifyWith(signingKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    public boolean isTokenExpired(String token) {
        try {
            return validateAndExtract(token).getExpiration().before(new Date());
        } catch (ExpiredJwtException e) {
            return true;
        }
    }
}

// ── JWT Authentication Filter ─────────────────────────────────────────────
// OncePerRequestFilter guarantees exactly one invocation per request,
// even if the filter is registered multiple times (can happen in Spring Boot).
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenService jwtService;
    private final UserDetailsService userDetailsService;
    private final TokenBlacklistCache blacklist;

    public JwtAuthenticationFilter(JwtTokenService jwtService,
                                   UserDetailsService userDetailsService,
                                   TokenBlacklistCache blacklist) {
        this.jwtService         = jwtService;
        this.userDetailsService = userDetailsService;
        this.blacklist          = blacklist;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        // Pass through if no Bearer token (public endpoints, health checks)
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);

        // Check blacklist first — revoked tokens (logout) must not authenticate
        if (blacklist.isBlacklisted(token)) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Token revoked");
            return;
        }

        try {
            Claims claims = jwtService.validateAndExtract(token);
            String investorId = claims.getSubject();

            // Only set authentication if not already set (avoid double-processing)
            if (investorId != null &&
                SecurityContextHolder.getContext().getAuthentication() == null) {

                UserDetails userDetails = userDetailsService.loadUserByUsername(investorId);

                UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                                userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                // Setting this in SecurityContext makes the user "authenticated" for this request
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }

        } catch (ExpiredJwtException e) {
            // Don't set authentication — let the security config return 401
            // The client should use the refresh token to get a new access token
            response.setHeader("X-Token-Expired", "true");  // hint for client
        } catch (JwtException e) {
            // Invalid signature, malformed token — do not authenticate
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid token");
            return;
        }

        filterChain.doFilter(request, response);
    }
}

// ── Security Configuration ────────────────────────────────────────────────
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // enables @PreAuthorize on methods
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtFilter) {
        this.jwtFilter = jwtFilter;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(csrf -> csrf.disable())  // JWT is stateless — CSRF not needed
                                               // (CSRF protects session-cookie auth)
                .sessionManagement(session ->
                        session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/v1/auth/**").permitAll()       // login, refresh
                        .requestMatchers("/actuator/health").permitAll()       // health checks
                        .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                // Insert JWT filter BEFORE UsernamePasswordAuthenticationFilter
                // so token is validated before Spring tries password auth
                .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
                .build();
    }
}

// ── Token Refresh with Retry — the RBC Bug Fix Story ─────────────────────
// Bug: expired access token → client called refresh → refreshService called
// generateAccessToken → old token still in request → filter saw expired token →
// tried to refresh again → infinite loop.
// Fix: refresh endpoint is in permitAll() so filter doesn't require auth on it,
// and refresh logic checks token type claim ("REFRESH" vs "ACCESS").
@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    private final JwtTokenService jwtService;
    private final TokenBlacklistCache blacklist;

    public AuthController(JwtTokenService jwtService, TokenBlacklistCache blacklist) {
        this.jwtService = jwtService;
        this.blacklist  = blacklist;
    }

    public record RefreshRequest(@NotBlank String refreshToken) {}
    public record TokenPair(String accessToken, String refreshToken) {}

    @PostMapping("/refresh")
    public ResponseEntity<TokenPair> refresh(@Valid @RequestBody RefreshRequest request) {
        try {
            Claims claims = jwtService.validateAndExtract(request.refreshToken());

            // Validate it IS a refresh token — prevents using access tokens here
            if (!"REFRESH".equals(claims.get("type", String.class))) {
                return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
            }

            String investorId = claims.getSubject();

            // Blacklist the old refresh token (rotation — prevents reuse)
            blacklist.blacklist(request.refreshToken());

            // Issue new token pair
            String newAccess  = jwtService.generateAccessToken(investorId, List.of("ROLE_INVESTOR"));
            String newRefresh = jwtService.generateRefreshToken(investorId);

            return ResponseEntity.ok(new TokenPair(newAccess, newRefresh));

        } catch (ExpiredJwtException e) {
            // Refresh token also expired — force full re-login
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(
            @RequestHeader("Authorization") String authHeader,
            @Valid @RequestBody RefreshRequest request) {

        // Blacklist both tokens on logout
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            blacklist.blacklist(authHeader.substring(7));
        }
        blacklist.blacklist(request.refreshToken());
        return ResponseEntity.noContent().build();
    }
}
```

---

## 2.4 Spring Data + PostgreSQL

Spring Data JPA generates repository implementations at runtime from interface method names,
eliminating boilerplate DAO code. Connection pooling (HikariCP) is critical — without it,
every database operation opens a new TCP connection (expensive); with it, connections are
reused from a pool. At RBC, the investor portfolio service used HikariCP with `maxPoolSize=20`
matching the PostgreSQL `max_connections` setting to prevent connection exhaustion.

**One-sentence interview answer:** "Spring Data generates repository implementations from
interface method names; JPA entities map to tables; HikariCP pools connections for reuse;
proper indexing on foreign keys and query columns is essential for portfolio-scale data."

**Most likely follow-ups:**
1. What is the N+1 query problem and how do you fix it?
2. What is the difference between `@OneToMany(fetch=LAZY)` and `EAGER`?
3. How does `@Transactional(readOnly=true)` improve performance?

```java
import org.springframework.data.jpa.repository.*;
import org.springframework.data.repository.query.*;
import javax.persistence.*;
import java.math.BigDecimal;
import java.time.*;
import java.util.*;

// ── JPA Entity ────────────────────────────────────────────────────────────
@Entity
@Table(
    name = "investor_portfolio",
    indexes = {
        // Composite index: queries filtering by (investor_id, portfolio_type) hit this
        @Index(name = "idx_portfolio_investor_type", columnList = "investor_id, portfolio_type"),
        // Separate index for date range queries on rebalanced_at
        @Index(name = "idx_portfolio_rebalanced_at", columnList = "rebalanced_at")
    }
)
public class InvestorPortfolioEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "investor_id", nullable = false, length = 20)
    private String investorId;

    @Column(name = "portfolio_type", nullable = false, length = 20)
    private String portfolioType;   // "EQUITY", "FIXED_INCOME", "BALANCED"

    // Use BigDecimal for money — never double/float (floating-point precision errors)
    @Column(name = "total_value", nullable = false, precision = 18, scale = 2)
    private BigDecimal totalValue;

    @Column(name = "cash_balance", precision = 18, scale = 2)
    private BigDecimal cashBalance;

    @Column(name = "rebalanced_at")
    private LocalDateTime rebalancedAt;

    @Version  // optimistic locking: prevents lost updates in concurrent modifications
    private Long version;

    // LAZY loading: holdings are NOT fetched unless explicitly accessed.
    // EAGER: always fetched with the portfolio — causes N+1 problems if not careful.
    // Use LAZY + explicit JOIN FETCH in queries when you need holdings.
    @OneToMany(
        mappedBy = "portfolio",
        cascade = CascadeType.ALL,
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    private List<HoldingEntity> holdings = new ArrayList<>();

    // JPA requires no-arg constructor
    protected InvestorPortfolioEntity() {}

    public InvestorPortfolioEntity(String investorId, String portfolioType, BigDecimal totalValue) {
        this.investorId    = investorId;
        this.portfolioType = portfolioType;
        this.totalValue    = totalValue;
        this.cashBalance   = BigDecimal.ZERO;
    }

    // Convenience method: adds holding and maintains both sides of the relationship
    public void addHolding(HoldingEntity holding) {
        holdings.add(holding);
        holding.setPortfolio(this);  // must set both sides to keep JPA in sync
    }

    // Getters omitted for brevity
    public String getInvestorId() { return investorId; }
    public BigDecimal getTotalValue() { return totalValue; }
    public void setTotalValue(BigDecimal v) { this.totalValue = v; }
    public void setRebalancedAt(LocalDateTime t) { this.rebalancedAt = t; }
}

@Entity
@Table(name = "holding")
class HoldingEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "portfolio_id")
    private InvestorPortfolioEntity portfolio;

    @Column(name = "ticker", length = 10)
    private String ticker;

    @Column(name = "quantity", precision = 18, scale = 6)
    private BigDecimal quantity;

    @Column(name = "market_value", precision = 18, scale = 2)
    private BigDecimal marketValue;

    protected HoldingEntity() {}
    HoldingEntity(String ticker, BigDecimal quantity, BigDecimal marketValue) {
        this.ticker = ticker; this.quantity = quantity; this.marketValue = marketValue;
    }
    void setPortfolio(InvestorPortfolioEntity p) { this.portfolio = p; }
}

// ── Repository ────────────────────────────────────────────────────────────
public interface InvestorPortfolioRepository
        extends JpaRepository<InvestorPortfolioEntity, Long> {

    // Spring Data derives query from method name — no SQL needed
    // Generates: SELECT * FROM investor_portfolio WHERE investor_id = ?
    Optional<InvestorPortfolioEntity> findByInvestorId(String investorId);

    // Derived query with multiple conditions
    List<InvestorPortfolioEntity> findByPortfolioTypeAndTotalValueGreaterThan(
            String portfolioType, BigDecimal minValue);

    // @Query for complex JPQL — when method name becomes unreadable
    // JOIN FETCH eagerly loads holdings in a single query — prevents N+1
    @Query("""
           SELECT p FROM InvestorPortfolioEntity p
           LEFT JOIN FETCH p.holdings h
           WHERE p.investorId = :investorId
           """)
    Optional<InvestorPortfolioEntity> findWithHoldingsByInvestorId(
            @Param("investorId") String investorId);

    // Native SQL for DB-specific functions or complex aggregations
    @Query(value = """
           SELECT p.portfolio_type,
                  COUNT(*)              AS portfolio_count,
                  SUM(p.total_value)    AS total_aum,
                  AVG(p.total_value)    AS avg_portfolio_value
           FROM investor_portfolio p
           WHERE p.rebalanced_at > :since
           GROUP BY p.portfolio_type
           ORDER BY total_aum DESC
           """, nativeQuery = true)
    List<Object[]> aggregateByPortfolioType(@Param("since") LocalDateTime since);

    // @Modifying: required for UPDATE/DELETE queries — marks as write operation
    // clearAutomatically: clears JPA first-level cache after update to avoid stale reads
    @Modifying(clearAutomatically = true)
    @Query("UPDATE InvestorPortfolioEntity p SET p.totalValue = :value, " +
           "p.rebalancedAt = :now WHERE p.investorId = :investorId")
    int updatePortfolioValue(@Param("investorId") String investorId,
                             @Param("value") BigDecimal value,
                             @Param("now") LocalDateTime now);

    // Top-N query: useful for risk reporting
    List<InvestorPortfolioEntity> findTop10ByPortfolioTypeOrderByTotalValueDesc(
            String portfolioType);

    // Existence check — generates efficient SELECT 1 rather than SELECT *
    boolean existsByInvestorId(String investorId);
}

// ── Service using Repository ──────────────────────────────────────────────
@Service
@Transactional
public class PortfolioService {

    private final InvestorPortfolioRepository portfolioRepo;

    public PortfolioService(InvestorPortfolioRepository portfolioRepo) {
        this.portfolioRepo = portfolioRepo;
    }

    // readOnly=true: Spring hints to DB driver this is a read — some drivers
    // route to read replica; Hibernate skips dirty-checking (performance win)
    @Transactional(readOnly = true)
    public InvestorPortfolioEntity getPortfolioWithHoldings(String investorId) {
        return portfolioRepo.findWithHoldingsByInvestorId(investorId)
                .orElseThrow(() -> new InvestorNotFoundException(investorId));
    }

    public void rebalancePortfolio(String investorId, BigDecimal newValue) {
        // optimistic lock: if concurrent update happened, @Version field mismatch
        // throws OptimisticLockException — let Spring retry or return 409 Conflict
        int updated = portfolioRepo.updatePortfolioValue(investorId, newValue, LocalDateTime.now());
        if (updated == 0) throw new InvestorNotFoundException(investorId);
    }

    public InvestorPortfolioEntity createPortfolio(String investorId, String type,
                                                   BigDecimal initialValue) {
        if (portfolioRepo.existsByInvestorId(investorId)) {
            throw new IllegalStateException("Portfolio already exists for " + investorId);
        }
        InvestorPortfolioEntity portfolio = new InvestorPortfolioEntity(investorId, type, initialValue);
        return portfolioRepo.save(portfolio);
    }
}
```

---
