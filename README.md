# Goal

A realistic, end‑to‑end practice project you can build quickly: call a public REST API, map the JSON to Java DTOs, filter the data, and persist it with Spring Data JPA + Hibernate in an H2 database. Includes setup, code, and tests.

---

## 0) Prereqs

* Java 17+ (use `java -version`)
* Maven 3.9+ (or Gradle—Maven shown here)
* An IDE (IntelliJ/Eclipse/VS Code)

---

## 1) Generate the project

Using Spring Initializr via CLI:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,h2,validation,lombok \
  -d bootVersion=3.3.3 \
  -d javaVersion=17 \
  -d type=maven-project \
  -d groupId=com.example \
  -d artifactId=api2db \
  -o api2db.zip
unzip api2db.zip && cd api2db
```

> If you prefer without Lombok, omit it and write getters/setters/constructors manually.

---

## 2) `pom.xml`

(If you used Initializr, you already have this. Keep an eye on versions.)

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>api2db</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <name>api2db</name>
  <description>Practice: External API → Filter → JPA/Hibernate</description>
  <properties>
    <java.version>17</java.version>
    <spring-boot.version>3.3.3</spring-boot.version>
  </properties>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <!-- For HTTP stubbing in tests (optional but useful) -->
    <dependency>
      <groupId>com.github.tomakehurst</groupId>
      <artifactId>wiremock-jre8</artifactId>
      <version>2.35.2</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

---

## 3) Configuration (`src/main/resources/application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:api2db;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
    username: sa
    password:
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: update # For dev only (creates/updates tables automatically)
    show-sql: true
    properties:
      hibernate.format_sql: true
  h2:
    console:
      enabled: true
      path: /h2-console

app:
  external:
    base-url: https://jsonplaceholder.typicode.com
    timeout-ms: 5000
```

---

## 4) Domain & Persistence

We’ll fetch *Posts* from a public API (`/posts`), then persist a subset. You can swap the API later—pattern stays the same.

### 4.1) DTO for incoming JSON

`src/main/java/com/example/api2db/external/PostDto.java`

```java
package com.example.api2db.external;

public record PostDto(Long userId, Long id, String title, String body) {}
```

### 4.2) JPA Entity

`src/main/java/com/example/api2db/post/Post.java`

```java
package com.example.api2db.post;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity
@Table(name = "posts", uniqueConstraints = {
    @UniqueConstraint(name = "uk_posts_external_id", columnNames = "external_id")
})
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "external_id", nullable = false)
    private Long externalId; // from the API

    private Long userId;

    @Column(length = 200)
    private String title;

    @Column(length = 4000)
    private String body;

    @Builder.Default
    private Instant importedAt = Instant.now();
}
```

### 4.3) Repository

`src/main/java/com/example/api2db/post/PostRepository.java`

```java
package com.example.api2db.post;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface PostRepository extends JpaRepository<Post, Long> {
    Optional<Post> findByExternalId(Long externalId);
}
```

---

## 5) HTTP Client Setup

Use Spring’s `WebClient` (reactive, lightweight) even in MVC apps.

### 5.1) Config

`src/main/java/com/example/api2db/config/HttpClientConfig.java`

```java
package com.example.api2db.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.ExchangeStrategies;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.netty.http.client.HttpClient;
import java.time.Duration;

@Configuration
public class HttpClientConfig {
    @Bean
    WebClient externalApiClient(
            @Value("${app.external.base-url}") String baseUrl,
            @Value("${app.external.timeout-ms}") long timeoutMs
    ) {
        HttpClient httpClient = HttpClient.create()
                .responseTimeout(Duration.ofMillis(timeoutMs));

        return WebClient.builder()
                .baseUrl(baseUrl)
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .exchangeStrategies(ExchangeStrategies.builder()
                        .codecs(c -> c.defaultCodecs().maxInMemorySize(2 * 1024 * 1024))
                        .build())
                .build();
    }
}
```

---

## 6) Service: Fetch → Filter → Map → Upsert

`src/main/java/com/example/api2db/post/PostImportService.java`

```java
package com.example.api2db.post;

import com.example.api2db.external.PostDto;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.reactive.function.client.WebClient;

import java.util.List;
import java.util.Objects;

@Service
@RequiredArgsConstructor
public class PostImportService {
    private final WebClient externalApiClient;
    private final PostRepository repo;

    /**
     * Import posts and store only those whose title contains the given keyword (case-insensitive).
     */
    @Transactional
    public List<Post> importFilteredPosts(String keyword) {
        String k = Objects.requireNonNullElse(keyword, "").toLowerCase();

        List<PostDto> dtos = externalApiClient.get()
                .uri("/posts")
                .retrieve()
                .bodyToFlux(PostDto.class)
                .collectList()
                .block();

        if (dtos == null || dtos.isEmpty()) return List.of();

        return dtos.stream()
                .filter(d -> k.isBlank() || (d.title() != null && d.title().toLowerCase().contains(k)))
                .map(this::toEntity)
                .map(this::upsertByExternalId)
                .toList();
    }

    private Post toEntity(PostDto d) {
        return Post.builder()
                .externalId(d.id())
                .userId(d.userId())
                .title(d.title())
                .body(d.body())
                .build();
    }

    /** Upsert by externalId to avoid duplicates on re-import. */
    private Post upsertByExternalId(Post incoming) {
        return repo.findByExternalId(incoming.getExternalId())
                .map(existing -> {
                    existing.setTitle(incoming.getTitle());
                    existing.setBody(incoming.getBody());
                    existing.setUserId(incoming.getUserId());
                    return existing; // dirty-checked
                })
                .orElseGet(() -> repo.save(incoming));
    }
}
```

---

## 7) Controller: trigger import & expose data

`src/main/java/com/example/api2db/post/PostController.java`

```java
package com.example.api2db.post;

import jakarta.validation.constraints.Size;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/posts")
@RequiredArgsConstructor
@Validated
public class PostController {
    private final PostRepository repo;
    private final PostImportService importer;

    @PostMapping("/import")
    public ResponseEntity<ImportResponse> importPosts(@RequestParam(defaultValue = "") @Size(max=100) String keyword) {
        List<Post> saved = importer.importFilteredPosts(keyword);
        return ResponseEntity.ok(new ImportResponse(saved.size()));
    }

    @GetMapping
    public List<Post> list() {
        return repo.findAll();
    }

    public record ImportResponse(int importedCount) {}
}
```

---

## 8) Application class

`src/main/java/com/example/api2db/Api2dbApplication.java`

```java
package com.example.api2db;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Api2dbApplication {
    public static void main(String[] args) {
        SpringApplication.run(Api2dbApplication.class, args);
    }
}
```

---

## 9) Run it

```bash
./mvnw spring-boot:run
```

Then in another terminal:

```bash
# Import all posts with titles containing "qui"
curl -X POST "http://localhost:8080/api/posts/import?keyword=qui"

# List stored posts
curl "http://localhost:8080/api/posts"
```

Open H2 console at [http://localhost:8080/h2-console](http://localhost:8080/h2-console) (JDBC URL: `jdbc:h2:mem:api2db`).

---

## 10) Integration Test (with HTTP stubbing)

`src/test/java/com/example/api2db/post/PostImportServiceTest.java`

```java
package com.example.api2db.post;

import com.github.tomakehurst.wiremock.WireMockServer;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.util.TestPropertyValues;
import org.springframework.context.ApplicationContextInitializer;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.test.context.ContextConfiguration;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ContextConfiguration(initializers = PostImportServiceTest.Init.class)
class PostImportServiceTest {
    static WireMockServer wm = new WireMockServer(0);

    static class Init implements ApplicationContextInitializer<ConfigurableApplicationContext> {
        @Override public void initialize(ConfigurableApplicationContext ctx) {
            wm.start();
            String base = "http://localhost:" + wm.port();
            TestPropertyValues.of(
                    "app.external.base-url=" + base,
                    "app.external.timeout-ms=2000"
            ).applyTo(ctx);
        }
    }

    @Autowired PostImportService service;
    @Autowired PostRepository repo;

    @BeforeAll static void beforeAll() {
        wm.stubFor(get(urlEqualTo("/posts")).willReturn(okJson("""
            [
              {"userId":1,"id":101,"title":"hello world","body":"x"},
              {"userId":2,"id":102,"title":"QUI venture","body":"y"}
            ]
        """)));
    }

    @AfterAll static void afterAll() { wm.stop(); }

    @Test
    void importsOnlyMatchingKeyword_caseInsensitive() {
        var saved = service.importFilteredPosts("qui");
        assertThat(saved).hasSize(1);
        assertThat(repo.findByExternalId(102L)).isPresent();
    }
}
```

---

## 11) Common interview discussion points

* **Idempotency & upserts:** Unique constraint on `external_id` + `findByExternalId` → update or insert.
* **Transactions:** Service is `@Transactional` to ensure atomicity across save operations.
* **Validation:** Request param has `@Size`. You can add Bean Validation on DTOs if you accept input.
* **Error handling:** Wrap `WebClient` calls with `.onStatus(...)` and map to custom exceptions; expose via `@ControllerAdvice`.
* **Retries/Backoff:** Add Spring Retry or Resilience4j for flaky APIs.
* **Mappings:** For more complex transformations, use MapStruct.
* **Pagination:** Handle external API pagination; stream pages and persist batch-by-batch with `saveAll`.
* **N+1 queries:** If you add relationships, use `fetch = LAZY` wisely and design endpoints to avoid N+1.

---

## 12) Variations to practice

1. **Paginated API**: Suppose the external API has `/posts?page=1&size=50`; loop pages until empty.
2. **Filtering rules**: Filter by `userId`, title length, or body regex.
3. **Schema changes**: Split `Post` into `User` (1‑N). Import users first, then posts.
4. **Switch DB**: Replace H2 with PostgreSQL (change JDBC URL/driver/dependency).
5. **Batching**: Use `repo.saveAll()` in chunks of 100 for large imports.
6. **Caching**: Cache external calls with Caffeine/Spring Cache.

---

## 13) Quick PostgreSQL config (optional)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/api2db
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: validate # use Flyway/Liquibase for prod
```

Add dependency:

```xml
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <scope>runtime</scope>
</dependency>
```

---

## 14) What to say in the interview

* Walk through the **flow**: HTTP → DTO → filter → map → upsert (idempotent) → return summary.
* Mention **testing** with WireMock and **H2** for fast feedback.
* Call out **transaction boundaries**, **error handling**, and **retries**.
* If asked about scalability, discuss **pagination**, **batch inserts**, and **backpressure** with reactive streams.

Good luck—build it, run it, and tweak the variations until it’s muscle memory! 🚀
