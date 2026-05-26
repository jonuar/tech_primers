# Spring Boot Cheatsheet

## Mental Model

Spring Boot is an **opinionated wrapper around the Spring Framework** that eliminates XML configuration and manual bean wiring. The core idea: annotate your classes, and Spring Boot auto-configures everything — web server, DB connections, security, serialization — based on what's on the classpath. The programming model is **dependency injection via annotations**: Spring manages the lifecycle of your objects (beans) and injects them where needed. You write the business logic; Spring wires it together.

---

## Install & Minimal Setup

```bash
# Spring Initializr — generate project scaffold
# https://start.spring.io — pick: Maven/Gradle, Java 21, Spring Boot 3.x
# Common dependencies to add: Spring Web, Spring Data JPA, PostgreSQL Driver,
# Spring Security, Validation, Actuator, Lombok

# Or via CLI
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,postgresql,validation,actuator,lombok \
  -d javaVersion=21 \
  -d groupId=com.example \
  -d artifactId=my-api \
  -o my-api.zip

# Run
./mvnw spring-boot:run
./gradlew bootRun

# Build fat jar
./mvnw package
java -jar target/my-api-0.0.1-SNAPSHOT.jar
```

```java
// Entry point
@SpringBootApplication   // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApiApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApiApplication.class, args);
    }
}
```

---

## Core Concepts

### 1. Dependency Injection & Beans

```java
// @Component — generic Spring-managed bean
@Component
public class EmailService { ... }

// Specialized stereotypes (semantically meaningful, same behavior)
@Service      // business logic layer
@Repository   // data access layer (also translates DB exceptions)
@Controller   // web layer (returns views)
@RestController  // web layer (returns JSON — @Controller + @ResponseBody)

// @Bean — explicit bean declaration in configuration class
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}

// Constructor injection — ALWAYS prefer over field injection
@Service
public class UserService {
    private final UserRepository repo;
    private final EmailService emailService;

    // @Autowired is optional when there's a single constructor (Spring 4.3+)
    public UserService(UserRepository repo, EmailService emailService) {
        this.repo = repo;
        this.emailService = emailService;
    }
}
```

### 2. REST Controllers

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<UserResponse> listUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size
    ) {
        return userService.findAll(page, size);
    }

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@RequestBody @Valid UserCreateRequest request) {
        return userService.create(request);
    }

    @PutMapping("/{id}")
    public UserResponse updateUser(
        @PathVariable Long id,
        @RequestBody @Valid UserUpdateRequest request
    ) {
        return userService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### 3. Request / Response DTOs with Validation

```java
import jakarta.validation.constraints.*;

// Request DTO
public record UserCreateRequest(
    @NotBlank @Size(min = 2, max = 100) String name,
    @Email @NotBlank                    String email,
    @Min(18) @Max(120)                  int age,
    @NotNull                            Role role
) { }

// Response DTO
public record UserResponse(
    Long id,
    String name,
    String email,
    Role role,
    LocalDateTime createdAt
) { }

// Common validation annotations
@NotNull       // not null
@NotBlank      // not null, not empty, not whitespace (Strings)
@NotEmpty      // not null, not empty (String, Collection, Array)
@Size(min, max)
@Min / @Max
@Email
@Pattern(regexp = "...")
@Positive / @PositiveOrZero
@Past / @Future  // for dates
```

### 4. Spring Data JPA

```java
// Entity
@Entity
@Table(name = "users")
@Getter @Setter          // Lombok
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @Enumerated(EnumType.STRING)
    private Role role;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}

// Repository
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // Derived queries — Spring Data generates the SQL
    Optional<User> findByEmail(String email);
    List<User> findByRoleAndActiveTrue(Role role);
    boolean existsByEmail(String email);
    long countByRole(Role role);

    // JPQL query
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name% AND u.active = true")
    List<User> searchByName(@Param("name") String name);

    // Native SQL
    @Query(value = "SELECT * FROM users WHERE created_at > :date", nativeQuery = true)
    List<User> findCreatedAfter(@Param("date") LocalDateTime date);

    // Pagination
    Page<User> findAll(Pageable pageable);
    Page<User> findByRole(Role role, Pageable pageable);
}
```

### 5. Service Layer

```java
@Service
@Transactional(readOnly = true)   // default to read-only; override per method
public class UserService {

    private final UserRepository userRepo;

    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }

    public List<UserResponse> findAll(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return userRepo.findAll(pageable)
            .map(this::toResponse)
            .getContent();
    }

    public UserResponse findById(Long id) {
        return userRepo.findById(id)
            .map(this::toResponse)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    @Transactional   // write operation — override read-only
    public UserResponse create(UserCreateRequest req) {
        if (userRepo.existsByEmail(req.email())) {
            throw new EmailAlreadyExistsException(req.email());
        }
        var user = User.builder()
            .name(req.name())
            .email(req.email())
            .role(req.role())
            .build();
        return toResponse(userRepo.save(user));
    }

    private UserResponse toResponse(User u) {
        return new UserResponse(u.getId(), u.getName(), u.getEmail(), u.getRole(), u.getCreatedAt());
    }
}
```

### 6. Exception Handling

```java
// Custom exceptions
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User not found: " + id);
    }
}

// Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(UserNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return new ErrorResponse("VALIDATION_FAILED", errors.toString());
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex) {
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}

public record ErrorResponse(String code, String message) { }
```

### 7. Configuration — application.yml

```yaml
spring:
  application:
    name: my-api

  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}             # from env variable
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2

  jpa:
    hibernate:
      ddl-auto: validate             # validate | update | create | create-drop
    show-sql: false
    properties:
      hibernate.format_sql: true

  jackson:
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus

# Custom properties
app:
  jwt-secret: ${JWT_SECRET}
  token-expiration-ms: 86400000
```

```java
// Bind custom properties to a bean
@ConfigurationProperties(prefix = "app")
@Component
public record AppProperties(String jwtSecret, long tokenExpirationMs) { }
```

### 8. Spring Security (JWT)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/users/**").hasRole("USER")
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 9. Testing

```java
// Unit test — slice, no Spring context
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock UserRepository userRepo;
    @InjectMocks UserService userService;

    @Test
    void findById_returnsUser_whenExists() {
        var user = User.builder().id(1L).name("Joshua").build();
        when(userRepo.findById(1L)).thenReturn(Optional.of(user));

        var result = userService.findById(1L);

        assertThat(result.name()).isEqualTo("Joshua");
    }

    @Test
    void findById_throws_whenNotFound() {
        when(userRepo.findById(99L)).thenReturn(Optional.empty());
        assertThatThrownBy(() -> userService.findById(99L))
            .isInstanceOf(UserNotFoundException.class);
    }
}

// Integration test — full Spring context
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {

    @Autowired MockMvc mvc;
    @Autowired UserRepository repo;

    @Test
    void getUser_returns200() throws Exception {
        mvc.perform(get("/api/v1/users/1")
            .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Joshua"));
    }
}
```

---

## Project Structure

```
src/main/java/com/example/myapi/
├── MyApiApplication.java
├── config/
│   ├── SecurityConfig.java
│   └── AppProperties.java
├── controller/
│   └── UserController.java
├── service/
│   └── UserService.java
├── repository/
│   └── UserRepository.java
├── entity/
│   └── User.java
├── dto/
│   ├── UserCreateRequest.java
│   └── UserResponse.java
└── exception/
    ├── UserNotFoundException.java
    └── GlobalExceptionHandler.java
```

---

## Gotchas

- **Constructor injection over `@Autowired` on fields** — field injection makes testing harder and hides dependencies. Always inject via constructor.
- **`@Transactional` on private methods doesn't work** — Spring proxies can't intercept private methods. Put `@Transactional` on public methods.
- **N+1 query problem** — `@OneToMany` with `FetchType.LAZY` (default) causes one query per parent when iterating children. Use `JOIN FETCH` in JPQL or `@EntityGraph`.
- **`@SpringBootTest` is slow** — loads the full context. Use slices (`@WebMvcTest`, `@DataJpaTest`) for focused tests.
- **Lombok + records** — Lombok doesn't work on records. Use records for DTOs (Java 16+) and Lombok for JPA entities (which can't be records because they need a no-arg constructor).
- **`ddl-auto: update` in production** — never use `update` or `create` in production. Use `validate` and manage schema with Flyway or Liquibase.

---

## Quick Links

- [Spring Boot Docs](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Initializr](https://start.spring.io)
- [Baeldung Spring](https://www.baeldung.com/spring-tutorial) — best practical resource
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Flyway](https://flywaydb.org) — DB migrations for Spring Boot
- [Testcontainers](https://testcontainers.com) — real DBs in integration tests
