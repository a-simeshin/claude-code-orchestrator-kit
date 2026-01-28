# Java Antipatterns

## Overview

Critical antipatterns to catch during code review. These cause real bugs, security issues, or maintenance nightmares.

---

## Code Structure

### Arrow Code / Deep Nesting

**Problem:** Multiple levels of nesting make code hard to read and maintain.

```java
// BAD: Arrow code
public void process(Request request) {
    if (request != null) {
        if (request.isValid()) {
            if (hasPermission(request)) {
                if (isNotRateLimited(request)) {
                    // Finally, the actual logic buried here
                    doWork(request);
                }
            }
        }
    }
}

// GOOD: Guard clauses (early return)
public void process(Request request) {
    if (request == null) return;
    if (!request.isValid()) throw new ValidationException("Invalid request");
    if (!hasPermission(request)) throw new AccessDeniedException();
    if (isRateLimited(request)) throw new RateLimitException();

    doWork(request); // Clear, at top level
}
```

---

### God Class

**Problem:** Class doing too many things, hundreds of methods.

```java
// BAD: God class
public class UserManager {
    public void createUser() { }
    public void deleteUser() { }
    public void sendEmail() { }          // Should be EmailService
    public void generateReport() { }      // Should be ReportService
    public void processPayment() { }      // Should be PaymentService
    public void validateAddress() { }     // Should be AddressValidator
    // ... 50 more methods
}

// GOOD: Single responsibility
public class UserService {
    private final EmailService emailService;
    private final UserRepository userRepository;

    public User create(CreateUserRequest request) { }
    public void delete(Long userId) { }
}
```

---

### Anemic Domain Model

**Problem:** Entities are just data containers, all logic in services.

```java
// BAD: Anemic entity
@Entity
public class Order {
    private BigDecimal total;
    private OrderStatus status;
    private List<OrderItem> items;
    // Only getters/setters
}

// Service does everything
public class OrderService {
    public void addItem(Order order, Product product, int qty) {
        order.getItems().add(new OrderItem(product, qty));
        recalculateTotal(order);
    }
}

// GOOD: Rich domain model
@Entity
public class Order {
    private BigDecimal total;
    private OrderStatus status;
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(Product product, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        items.add(new OrderItem(product, quantity));
        recalculateTotal();
    }

    private void recalculateTotal() {
        this.total = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

---

## Spring Framework

### Field Injection

**Problem:** Hidden dependencies, hard to test, no immutability.

```java
// BAD: Field injection
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private PaymentService paymentService;
}

// GOOD: Constructor injection
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = Objects.requireNonNull(orderRepository);
        this.paymentService = Objects.requireNonNull(paymentService);
    }
}
```

---

### @Transactional on Private Methods

**Problem:** Spring proxies don't intercept private methods - transaction is NOT active.

```java
// BAD: Transaction doesn't work!
@Service
public class UserService {

    @Transactional
    private void updateUserInternal(User user) {
        // NO TRANSACTION HERE - private method not proxied
        userRepository.save(user);
    }

    public void updateUser(Long userId, String name) {
        User user = userRepository.findById(userId).orElseThrow();
        user.setName(name);
        updateUserInternal(user); // Transaction not active!
    }
}

// GOOD: @Transactional on public method
@Service
public class UserService {

    @Transactional
    public void updateUser(Long userId, String name) {
        User user = userRepository.findById(userId).orElseThrow();
        user.setName(name);
        userRepository.save(user);
    }
}
```

---

### Missing @Transactional(readOnly = true)

**Problem:** Missing read-only optimization for queries.

```java
// BAD: No readOnly for queries
public List<User> getAllUsers() {
    return userRepository.findAll();
}

// GOOD: readOnly for queries
@Transactional(readOnly = true)
public List<User> getAllUsers() {
    return userRepository.findAll();
}
```

**Benefits of readOnly:**
- Hibernate skips dirty checking
- Some databases route to read replicas
- Prevents accidental modifications

---

### N+1 Query Problem

**Problem:** 1 query for parent + N queries for children.

```java
// BAD: N+1 queries
@Transactional(readOnly = true)
public List<OrderDTO> getOrders() {
    List<Order> orders = orderRepository.findAll(); // 1 query
    return orders.stream()
        .map(order -> new OrderDTO(
            order.getId(),
            order.getUser().getName() // N queries!
        ))
        .collect(Collectors.toList());
}

// GOOD: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.user")
List<Order> findAllWithUser(); // 1 query
```

---

## Exception Handling

### Catching Exception/Throwable

**Problem:** Catches everything including NPE, OutOfMemoryError.

```java
// BAD: Too broad
try {
    processOrder(order);
} catch (Exception e) { // Catches NPE, ClassCastException...
    log.error("Error", e);
}

// BAD: Even worse
try {
    processOrder(order);
} catch (Throwable t) { // Catches OutOfMemoryError!
    log.error("Error", t);
}

// GOOD: Specific exceptions
try {
    processOrder(order);
} catch (PaymentException e) {
    log.error("Payment failed for order: {}", order.getId(), e);
    throw new OrderProcessingException("Payment failed", e);
} catch (InventoryException e) {
    log.error("Inventory check failed", e);
    throw new OrderProcessingException("Out of stock", e);
}
```

---

### Empty Catch Block

**Problem:** Silently swallows exceptions - debugging nightmare.

```java
// BAD: Silent failure
try {
    sendEmail(user);
} catch (Exception e) {
    // Swallowed - who knows what happened?
}

// GOOD: At minimum log it
try {
    sendEmail(user);
} catch (EmailException e) {
    log.warn("Failed to send email to {}: {}", user.getEmail(), e.getMessage());
    // Decide: rethrow, return error, or continue
}
```

---

## Data Handling

### Mutable DTOs

**Problem:** DTOs can be modified after creation.

```java
// BAD: Mutable DTO
public class UserDTO {
    private String name;
    private String email;

    // Getters and setters
    public void setName(String name) {
        this.name = name; // Can be modified anywhere
    }
}

// GOOD: Immutable record
public record UserDTO(String name, String email) {}

// GOOD: Builder for complex cases
@Builder
public record CreateUserRequest(
    String name,
    String email,
    List<String> roles
) {}
```

---

### Primitive Obsession

**Problem:** Using primitives for domain concepts.

```java
// BAD: Primitives everywhere
public void createUser(String email, String phone, int age) {
    // No validation, can pass any string
}

// GOOD: Value objects
public record Email(String value) {
    public Email {
        if (!value.matches("^[\\w.-]+@[\\w.-]+\\.[a-z]{2,}$")) {
            throw new IllegalArgumentException("Invalid email: " + value);
        }
    }
}

public record PhoneNumber(String value) {
    // Validation in constructor
}

public void createUser(Email email, PhoneNumber phone, int age) {
    // Type-safe, validated
}
```

---

## Security

### Hardcoded Credentials

**Problem:** Secrets in code = leaked secrets.

```java
// BAD: Hardcoded
private static final String API_KEY = "sk-abc123secret";
private static final String DB_PASSWORD = "admin123";

// GOOD: Externalized
@Value("${app.api.key}")
private String apiKey;

// BETTER: ConfigurationProperties
@ConfigurationProperties(prefix = "app.api")
public record ApiConfig(String key, String secret) {}
```

---

### SQL Injection via String Concatenation

**Problem:** User input directly in SQL.

```java
// BAD: SQL injection vulnerability
public User findByName(String name) {
    String sql = "SELECT * FROM users WHERE name = '" + name + "'";
    // Input: "'; DROP TABLE users; --"
    return jdbcTemplate.queryForObject(sql, ...);
}

// GOOD: Parameterized query
public User findByName(String name) {
    String sql = "SELECT * FROM users WHERE name = ?";
    return jdbcTemplate.queryForObject(sql, User.class, name);
}

// GOOD: Spring Data
User findByName(String name); // Auto-parameterized
```

---

## Quick Antipattern Checklist

- [ ] No arrow code (deep nesting) - use early return
- [ ] No God classes - single responsibility
- [ ] No field injection - constructor only
- [ ] No @Transactional on private methods
- [ ] @Transactional(readOnly = true) for queries
- [ ] No N+1 queries - use JOIN FETCH
- [ ] No catching Exception/Throwable - specific types
- [ ] No empty catch blocks
- [ ] No mutable DTOs - use records
- [ ] No hardcoded credentials
- [ ] No string concatenation in SQL
