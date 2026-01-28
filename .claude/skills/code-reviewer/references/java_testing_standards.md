# Java Testing Standards

## Overview

Testing patterns for Java projects using JUnit 5, Mockito, Testcontainers, and Allure reporting.

---

## Allure Annotations

### Hierarchy Structure

| Annotation | Level | Description |
|------------|-------|-------------|
| `@Epic` | Class | High-level functionality (e.g., "User Management") |
| `@Feature` | Class | Feature area (e.g., "User Registration") |
| `@Story` | Nested class | User story (e.g., "Successful Registration") |
| `@Severity` | Method | Priority level |
| `@Description` | Method | Detailed description (Russian) |
| `@DisplayName` | Method | Short name (English) |

### Severity Levels

| Level | When to Use |
|-------|-------------|
| `BLOCKER` | Page doesn't load, app crashes |
| `CRITICAL` | Core functionality broken |
| `NORMAL` | Standard features |
| `MINOR` | Secondary functionality |
| `TRIVIAL` | Cosmetic issues |

### Example Test Class

```java
@Epic("User Management")
@Feature("User Registration")
@DisplayName("User Registration E2E Tests")
class UserRegistrationIT extends BaseE2ETest {

    @Nested
    @Story("Successful Registration")
    @DisplayName("Successful Registration Flow")
    class SuccessfulRegistration {

        @Test
        @Severity(SeverityLevel.CRITICAL)
        @Description("Проверяет успешную регистрацию пользователя с валидными данными")
        @DisplayName("should register user with valid data")
        void shouldRegisterUserWithValidData() {
            // Arrange
            final var request = new RegisterRequest("user@test.com", "password123");

            // Act
            final var response = restTemplate.postForEntity("/api/register", request, UserDTO.class);

            // Assert
            assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
            assertThat(response.getBody()).isNotNull();
            assertThat(response.getBody().email()).isEqualTo("user@test.com");
        }
    }

    @Nested
    @Story("Registration Validation")
    @DisplayName("Validation Errors")
    class ValidationErrors {

        @Test
        @Severity(SeverityLevel.NORMAL)
        @Description("Проверяет ошибку валидации при невалидном email")
        @DisplayName("should return error for invalid email")
        void shouldReturnErrorForInvalidEmail() {
            // Test implementation
        }
    }
}
```

---

## Unit Tests (JUnit 5 + Mockito)

### Structure

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentService paymentService;

    @InjectMocks
    private OrderService orderService;

    @Test
    @DisplayName("should create order when payment succeeds")
    void shouldCreateOrderWhenPaymentSucceeds() {
        // Arrange
        final var request = new CreateOrderRequest(1L, List.of(
            new OrderItemRequest(100L, 2)
        ));
        final var expectedOrder = Order.builder()
            .id(1L)
            .status(OrderStatus.CREATED)
            .build();

        when(paymentService.process(any())).thenReturn(PaymentResult.success());
        when(orderRepository.save(any())).thenReturn(expectedOrder);

        // Act
        final Order result = orderService.createOrder(request);

        // Assert
        assertThat(result.getStatus()).isEqualTo(OrderStatus.CREATED);
        verify(orderRepository).save(any(Order.class));
        verify(paymentService).process(any());
    }

    @Test
    @DisplayName("should throw exception when payment fails")
    void shouldThrowExceptionWhenPaymentFails() {
        // Arrange
        final var request = new CreateOrderRequest(1L, List.of());
        when(paymentService.process(any())).thenReturn(PaymentResult.failed("Declined"));

        // Act & Assert
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(PaymentException.class)
            .hasMessageContaining("Declined");

        verify(orderRepository, never()).save(any());
    }
}
```

### Best Practices

- Use `@ExtendWith(MockitoExtension.class)` instead of `@RunWith`
- `@Mock` for dependencies, `@InjectMocks` for SUT
- Arrange-Act-Assert structure
- One assertion concept per test
- Descriptive `@DisplayName`

---

## Integration Tests (Testcontainers)

### Base Test Configuration

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
abstract class BaseIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(final DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    protected TestRestTemplate restTemplate;
}
```

### Kafka Testcontainer

```java
@Testcontainers
class KafkaIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );

    @DynamicPropertySource
    static void kafkaProperties(final DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Test
    void shouldConsumeMessage() {
        // Test Kafka producer/consumer
    }
}
```

### Cassandra Testcontainer

```java
@Testcontainers
class CassandraIntegrationTest {

    @Container
    static CassandraContainer<?> cassandra = new CassandraContainer<>("cassandra:4.1")
        .withExposedPorts(9042);

    @DynamicPropertySource
    static void cassandraProperties(final DynamicPropertyRegistry registry) {
        registry.add("spring.cassandra.contact-points",
            () -> cassandra.getHost() + ":" + cassandra.getMappedPort(9042));
        registry.add("spring.cassandra.local-datacenter", () -> "datacenter1");
    }
}
```

---

## MockMvc for REST API Tests

### Configuration

```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    @WithMockUser(roles = "USER")
    @DisplayName("should return user by id")
    void shouldReturnUserById() throws Exception {
        // Arrange
        final var user = new UserDTO(1L, "John", "john@test.com");
        when(userService.findById(1L)).thenReturn(Optional.of(user));

        // Act & Assert
        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@test.com"));
    }

    @Test
    @DisplayName("should return 404 when user not found")
    void shouldReturn404WhenUserNotFound() throws Exception {
        when(userService.findById(999L)).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound());
    }

    @Test
    @DisplayName("should create user with valid data")
    void shouldCreateUser() throws Exception {
        final var request = """
            {
                "name": "John",
                "email": "john@test.com",
                "password": "password123"
            }
            """;

        final var created = new UserDTO(1L, "John", "john@test.com");
        when(userService.create(any())).thenReturn(created);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(request))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.id").value(1));
    }
}
```

---

## Maven Configuration

### Surefire Plugin (Unit Tests)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>
            @{argLine}
            -XX:+EnableDynamicAgentLoading
            -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
        </argLine>
        <systemPropertyVariables>
            <allure.results.directory>target/allure-results</allure.results.directory>
        </systemPropertyVariables>
        <includes>
            <include>**/*Test.java</include>
        </includes>
        <excludes>
            <exclude>**/*IT.java</exclude>
        </excludes>
    </configuration>
    <dependencies>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>${aspectj.version}</version>
        </dependency>
    </dependencies>
</plugin>
```

### Failsafe Plugin (Integration Tests)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <configuration>
        <includes>
            <include>**/*IT.java</include>
        </includes>
        <systemPropertyVariables>
            <allure.results.directory>target/allure-results</allure.results.directory>
        </systemPropertyVariables>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### Allure Plugin

```xml
<plugin>
    <groupId>io.qameta.allure</groupId>
    <artifactId>allure-maven</artifactId>
    <version>2.15.2</version>
    <configuration>
        <reportVersion>${allure-bom.version}</reportVersion>
        <resultsDirectory>${project.build.directory}/allure-results</resultsDirectory>
    </configuration>
</plugin>
```

### Profile for Integration Tests

```xml
<profile>
    <id>integration-tests</id>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</profile>
```

---

## Commands

```bash
# Unit tests only
mvn test

# Integration tests
mvn verify -Pintegration-tests

# Generate Allure report
mvn allure:serve

# Generate report without server
mvn allure:report
open target/site/allure-maven-plugin/index.html
```

---

## Quick Testing Checklist

### Unit Tests
- [ ] `@ExtendWith(MockitoExtension.class)` for Mockito
- [ ] Constructor injection mocked via `@InjectMocks`
- [ ] Arrange-Act-Assert structure
- [ ] Descriptive `@DisplayName`

### Integration Tests
- [ ] `@SpringBootTest` with `RANDOM_PORT`
- [ ] `@Testcontainers` for external services
- [ ] `@DynamicPropertySource` for container properties
- [ ] `*IT.java` naming convention

### Allure
- [ ] `@Epic` and `@Feature` on class
- [ ] `@Story` on nested classes
- [ ] `@Severity` on each test method
- [ ] `@Description` in Russian
- [ ] `@DisplayName` in English
