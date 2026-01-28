# Java Code Review Checklist

## Overview

Critical code review points for Java 17+ projects. Focus on issues that cause real bugs, security vulnerabilities, or maintainability problems.

---

## Java 17 Features

### Use Records for Immutable Data
```java
// GOOD: Record for DTO
public record UserDTO(Long id, String name, String email) {}

// BAD: Mutable class with boilerplate
public class UserDTO {
    private Long id;
    private String name;
    // getters, setters, equals, hashCode, toString...
}
```

### Pattern Matching in instanceof
```java
// GOOD: Pattern matching
if (obj instanceof String s) {
    return s.length();
}

// BAD: Old style with cast
if (obj instanceof String) {
    String s = (String) obj;
    return s.length();
}
```

### Text Blocks for Multi-line Strings
```java
// GOOD: Text block for SQL
String sql = """
    SELECT u.id, u.name
    FROM users u
    WHERE u.active = true
    ORDER BY u.created_at DESC
    """;

// BAD: String concatenation
String sql = "SELECT u.id, u.name " +
    "FROM users u " +
    "WHERE u.active = true";
```

---

## Immutability & Final

### Final for All Non-Reassigned Variables
```java
// GOOD: final everywhere possible
public ResponseEntity<UserDTO> getUser(final Long id) {
    final User user = userRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("User not found: " + id));
    final UserDTO dto = mapper.toDto(user);
    return ResponseEntity.ok(dto);
}

// BAD: Variables without final
public ResponseEntity<UserDTO> getUser(Long id) {
    User user = userRepository.findById(id).orElseThrow(...);
    UserDTO dto = mapper.toDto(user);
    return ResponseEntity.ok(dto);
}
```

### Final for Method Parameters
```java
// GOOD
public void processOrder(final Order order, final boolean sendNotification) {
    // order and sendNotification cannot be reassigned
}

// BAD
public void processOrder(Order order, boolean sendNotification) {
    order = null; // Possible reassignment - confusing
}
```

---

## No-Nest / Early Return

### Use Guard Clauses to Reduce Nesting
```java
// GOOD: Early return (guard clauses)
public void processRequest(final Request request) {
    if (request == null) {
        log.warn("Request is null");
        return;
    }
    if (!request.isValid()) {
        throw new ValidationException("Invalid request");
    }
    if (!securityService.hasPermission(request)) {
        throw new AccessDeniedException("No permission");
    }

    // Main logic - no nesting
    final Result result = businessService.process(request);
    notificationService.send(result);
}

// BAD: Arrow code / deep nesting
public void processRequest(Request request) {
    if (request != null) {
        if (request.isValid()) {
            if (securityService.hasPermission(request)) {
                Result result = businessService.process(request);
                notificationService.send(result);
            } else {
                throw new AccessDeniedException("No permission");
            }
        } else {
            throw new ValidationException("Invalid request");
        }
    } else {
        log.warn("Request is null");
    }
}
```

### Early Return in Loops
```java
// GOOD: continue for guard, early return when found
public Optional<User> findActiveAdmin(final List<User> users) {
    for (final User user : users) {
        if (!user.isActive()) continue;
        if (!user.hasRole(Role.ADMIN)) continue;

        return Optional.of(user);
    }
    return Optional.empty();
}
```

---

## Null Safety

### Optional for Return Types (Never for Parameters)
```java
// GOOD: Optional return type
public Optional<User> findByEmail(final String email) {
    return userRepository.findByEmail(email);
}

// BAD: Optional parameter
public void process(Optional<String> name) { // Never do this
}

// BAD: Returning null
public User findByEmail(String email) {
    return userRepository.findByEmail(email).orElse(null); // Null leaks
}
```

### Never Call get() Without Check
```java
// GOOD: Use orElseThrow with context
final User user = userRepository.findById(id)
    .orElseThrow(() -> new NotFoundException("User not found: " + id));

// GOOD: Use orElse for defaults
final String name = optionalName.orElse("Anonymous");

// BAD: get() without check
final User user = userRepository.findById(id).get(); // NoSuchElementException
```

### Objects.requireNonNull in Constructors
```java
// GOOD: Fail fast with clear message
public UserService(final UserRepository userRepository,
                   final NotificationService notificationService) {
    this.userRepository = Objects.requireNonNull(userRepository, "userRepository");
    this.notificationService = Objects.requireNonNull(notificationService, "notificationService");
}
```

---

## Exception Handling

### Custom Exceptions with Context
```java
// GOOD: Custom exception with context
public class OrderNotFoundException extends RuntimeException {
    private final Long orderId;

    public OrderNotFoundException(final Long orderId) {
        super("Order not found: " + orderId);
        this.orderId = orderId;
    }

    public Long getOrderId() {
        return orderId;
    }
}

// Usage
throw new OrderNotFoundException(orderId);
```

### Never Empty Catch Blocks
```java
// BAD: Silent failure
try {
    riskyOperation();
} catch (Exception e) {
    // Swallowed - debugging nightmare
}

// GOOD: At minimum log it
try {
    riskyOperation();
} catch (Exception e) {
    log.error("Failed to perform risky operation", e);
    throw new ServiceException("Operation failed", e);
}
```

### Catch Specific Exceptions
```java
// BAD: Too broad
try {
    processFile(path);
} catch (Exception e) { // Catches everything including NullPointerException
    handleError(e);
}

// GOOD: Specific exceptions
try {
    processFile(path);
} catch (IOException e) {
    log.error("IO error reading file: {}", path, e);
    throw new FileProcessingException("Cannot read file", e);
}
```

---

## Stream API

### Prefer Method References
```java
// GOOD: Method reference
users.stream()
    .map(User::getName)
    .filter(Objects::nonNull)
    .collect(Collectors.toList());

// OK but verbose: Lambda
users.stream()
    .map(user -> user.getName())
    .filter(name -> name != null)
    .collect(Collectors.toList());
```

### Avoid Side Effects in Streams
```java
// BAD: Side effect in forEach
final List<String> results = new ArrayList<>();
users.stream()
    .map(User::getName)
    .forEach(results::add); // Side effect

// GOOD: Collect
final List<String> results = users.stream()
    .map(User::getName)
    .collect(Collectors.toList());
```

---

## Resources

### Try-with-Resources for All Closeable
```java
// GOOD: Auto-close
try (final var connection = dataSource.getConnection();
     final var statement = connection.prepareStatement(sql);
     final var resultSet = statement.executeQuery()) {
    // Process results
}

// BAD: Manual close (error-prone)
Connection conn = null;
try {
    conn = dataSource.getConnection();
    // Process
} finally {
    if (conn != null) conn.close(); // Can throw, masking original exception
}
```

---

## Quick Checklist

- [ ] All non-reassigned variables are `final`
- [ ] Method parameters are `final`
- [ ] Records used for immutable data classes
- [ ] Pattern matching for `instanceof`
- [ ] Guard clauses instead of deep nesting (no arrow code)
- [ ] `Optional` for return types, never for parameters
- [ ] No `get()` without `isPresent()` check (use `orElseThrow`)
- [ ] `Objects.requireNonNull()` in constructors
- [ ] No empty catch blocks
- [ ] Specific exception types caught
- [ ] Try-with-resources for all `Closeable`
- [ ] No side effects in streams
