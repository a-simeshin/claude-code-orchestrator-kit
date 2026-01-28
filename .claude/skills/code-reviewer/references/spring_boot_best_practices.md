# Spring Boot Best Practices

## Overview

Critical Spring Boot patterns and anti-patterns. Focus on security, performance (N+1), and maintainability issues.

---

## Dependency Injection

### Constructor Injection Only
```java
// GOOD: Constructor injection (immutable, testable)
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public OrderService(final OrderRepository orderRepository,
                        final PaymentService paymentService) {
        this.orderRepository = Objects.requireNonNull(orderRepository);
        this.paymentService = Objects.requireNonNull(paymentService);
    }
}

// BAD: Field injection (hidden dependencies, hard to test)
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private PaymentService paymentService;
}
```

---

## Spring Data JPA

### N+1 Query Problem - The Most Critical Issue

```java
// BAD: N+1 problem - 1 query for orders + N queries for users
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    private User user;
}

// Repository
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    order.getUser().getName(); // Each call = 1 query!
}

// GOOD: JOIN FETCH in query
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatusWithUser(@Param("status") OrderStatus status);

// GOOD: EntityGraph
@EntityGraph(attributePaths = {"user", "items"})
List<Order> findByStatus(OrderStatus status);
```

### @Transactional Boundaries

```java
// GOOD: Read-only for queries (performance)
@Transactional(readOnly = true)
public List<UserDTO> getAllUsers() {
    return userRepository.findAll().stream()
        .map(this::toDto)
        .collect(Collectors.toList());
}

// GOOD: Write transaction with proper boundary
@Transactional
public Order createOrder(final CreateOrderRequest request) {
    final Order order = new Order();
    // ... set fields
    return orderRepository.save(order);
}

// BAD: @Transactional on private method (DOESN'T WORK!)
@Transactional
private void updateInternal() { // Proxy doesn't intercept private methods
    // Transaction not active!
}

// BAD: Missing @Transactional for write operations
public void updateUser(User user) { // No transaction boundary
    userRepository.save(user);
}
```

### LazyInitializationException

```java
// BAD: Accessing lazy collection outside transaction
@Transactional(readOnly = true)
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
}
// Later in controller:
user.getOrders().size(); // LazyInitializationException!

// GOOD: Fetch what you need within transaction
@Transactional(readOnly = true)
public UserWithOrdersDTO getUserWithOrders(final Long id) {
    final User user = userRepository.findByIdWithOrders(id)
        .orElseThrow(() -> new NotFoundException("User: " + id));
    return new UserWithOrdersDTO(user, user.getOrders());
}

// GOOD: Use DTO projections
public interface UserSummary {
    Long getId();
    String getName();
    int getOrderCount();
}

@Query("SELECT u.id as id, u.name as name, SIZE(u.orders) as orderCount FROM User u")
List<UserSummary> findAllSummaries();
```

### Pagination

```java
// GOOD: Always paginate large result sets
@Transactional(readOnly = true)
public Page<UserDTO> getUsers(final Pageable pageable) {
    return userRepository.findAll(pageable)
        .map(this::toDto);
}

// Controller
@GetMapping("/users")
public Page<UserDTO> getUsers(
        @RequestParam(defaultValue = "0") final int page,
        @RequestParam(defaultValue = "20") final int size) {
    return userService.getUsers(PageRequest.of(page, size));
}

// BAD: Loading all records
public List<User> getAllUsers() {
    return userRepository.findAll(); // OOM on large tables
}
```

---

## REST API

### Proper Response Handling

```java
// GOOD: Explicit ResponseEntity
@GetMapping("/{id}")
public ResponseEntity<UserDTO> getUser(@PathVariable final Long id) {
    return userService.findById(id)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}

// GOOD: Created with location header
@PostMapping
public ResponseEntity<UserDTO> createUser(@Valid @RequestBody final CreateUserRequest request) {
    final UserDTO created = userService.create(request);
    final URI location = URI.create("/api/users/" + created.id());
    return ResponseEntity.created(location).body(created);
}

// BAD: Throwing exception for normal flow
@GetMapping("/{id}")
public UserDTO getUser(@PathVariable Long id) {
    return userService.findById(id)
        .orElseThrow(() -> new ResponseStatusException(NOT_FOUND)); // 404 is not exceptional
}
```

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(final NotFoundException e) {
        final var error = new ErrorResponse("NOT_FOUND", e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(final ValidationException e) {
        final var error = new ErrorResponse("VALIDATION_ERROR", e.getMessage());
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(final MethodArgumentNotValidException e) {
        final String message = e.getBindingResult().getFieldErrors().stream()
            .map(err -> err.getField() + ": " + err.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(new ErrorResponse("VALIDATION_ERROR", message));
    }
}

public record ErrorResponse(String code, String message) {}
```

### Request Validation

```java
// GOOD: Validation annotations
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    String name,

    @Email(message = "Invalid email format")
    @NotBlank(message = "Email is required")
    String email,

    @Size(min = 8, message = "Password must be at least 8 characters")
    String password
) {}

// Controller
@PostMapping
public ResponseEntity<UserDTO> create(@Valid @RequestBody final CreateUserRequest request) {
    return ResponseEntity.ok(userService.create(request));
}
```

---

## Spring Security

### SecurityFilterChain Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(final HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable()) // Disable for REST API with JWT
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

### CORS Configuration

```java
// GOOD: Explicit CORS config
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    final var config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://myapp.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);

    final var source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}

// BAD: Allow everything
config.setAllowedOrigins(List.of("*")); // Security risk
config.setAllowCredentials(true); // Doesn't work with "*"
```

### Method-Level Security

```java
@Service
public class OrderService {

    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOwner(#orderId)")
    public Order getOrder(final Long orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order: " + orderId));
    }
}

@Component("orderSecurity")
public class OrderSecurityEvaluator {

    public boolean isOwner(final Long orderId) {
        final String currentUser = SecurityContextHolder.getContext()
            .getAuthentication().getName();
        return orderRepository.isOwnedBy(orderId, currentUser);
    }
}
```

---

## Configuration

### @ConfigurationProperties Over @Value

```java
// GOOD: Type-safe configuration
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(
    String host,
    int port,
    String username,
    @DefaultValue("false") boolean ssl
) {}

// application.yml
app:
  mail:
    host: smtp.example.com
    port: 587
    username: noreply@example.com
    ssl: true

// Usage
@Service
public class MailService {
    private final MailProperties mailProperties;

    public MailService(final MailProperties mailProperties) {
        this.mailProperties = mailProperties;
    }
}

// BAD: @Value scattered across classes
@Value("${app.mail.host}")
private String mailHost;

@Value("${app.mail.port}")
private int mailPort;
```

---

## Quick Checklist

### JPA/Database
- [ ] No N+1 queries (use `JOIN FETCH` or `@EntityGraph`)
- [ ] `@Transactional(readOnly = true)` for read operations
- [ ] `@Transactional` on public methods only (not private)
- [ ] Pagination for large result sets
- [ ] DTO projections for read-only data

### REST API
- [ ] `@Valid` on request body
- [ ] Global `@RestControllerAdvice` for exceptions
- [ ] Proper HTTP status codes (201 for create, 404 for not found)
- [ ] Location header for created resources

### Security
- [ ] Explicit CORS configuration (no wildcards with credentials)
- [ ] Method-level security where needed
- [ ] Passwords encoded with BCrypt

### Configuration
- [ ] `@ConfigurationProperties` for groups of related properties
- [ ] Constructor injection only (no `@Autowired` on fields)
