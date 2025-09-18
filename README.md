# I have a coding interview in Java with Spring, JPA, Hibernate and more and I need to practice. In it, I will be asked to interact with an external API (use `https://jsonplaceholder.typicode.com/posts` as the consumed API), populate objects, filter them by some parameter and store them in a database. Show me how it's done and how do I set up the project. Don't do Unit Testing. Tell me the step-by-step if I create a Maven project in IntelliJ

---

Awesome—this is a classic interview task. Here’s a clean, no-tests, step-by-step way to build a Spring Boot app (Maven) that:

* Calls `https://jsonplaceholder.typicode.com/posts`
* Maps the JSON to Java objects
* Filters by a parameter (e.g., `userId`)
* Persists results with JPA/Hibernate to an H2 database
* Exposes endpoints so you can trigger the sync and query the saved data

---

# 1) Create the project (IntelliJ + Maven)

**Option A — Spring Initializr (easiest)**

1. IntelliJ → **File → New → Project… → Spring Initializr**.
2. **Language**: Java; **Build**: Maven; **Java**: 17 (or 21).
3. Group: `com.example`, Artifact: `posts-demo`.
4. Add dependencies:

   * Spring Web
   * Spring Data JPA
   * H2 Database
5. Finish.

**Option B — Plain Maven**

1. IntelliJ → **File → New → Project… → Maven** → Create from archetype (or empty).
2. Replace the generated `pom.xml` with the one below (then **Reload Maven**).

---

# 2) `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.3</version>
    <relativePath/> <!-- lookup parent from repository -->
  </parent>

  <groupId>com.example</groupId>
  <artifactId>posts-demo</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <name>posts-demo</name>
  <description>Spring Boot demo for external API + JPA</description>

  <properties>
    <java.version>17</java.version>
  </properties>

  <dependencies>
    <!-- Web (MVC + RestTemplate) -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- JPA/Hibernate -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- In-memory DB for quick demo -->
    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <scope>runtime</scope>
    </dependency>

    <!-- Optional: validation for request params -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
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

# 3) Application config

`src/main/resources/application.properties`

```properties
# H2 (in-memory)
spring.datasource.url=jdbc:h2:mem:postsdb;DB_CLOSE_DELAY=-1
spring.datasource.username=sa
spring.datasource.password=
spring.datasource.driver-class-name=org.h2.Driver

# Hibernate/JPA
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# H2 Console (handy for inspection)
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

# Logging
logging.level.org.springframework.web.client.RestTemplate=INFO
logging.level.org.hibernate.SQL=DEBUG
```

---

# 4) Bootstrapping

`src/main/java/com/example/postsdemo/PostsDemoApplication.java`

```java
package com.example.postsdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PostsDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(PostsDemoApplication.class, args);
    }
}
```

Add a `RestTemplate` bean:

`src/main/java/com/example/postsdemo/config/RestClientConfig.java`

```java
package com.example.postsdemo.config;

import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestClientConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.build();
    }
}
```

---

# 5) DTO for the external API

The JSON at `/posts` looks like:

```json
{ "userId": 1, "id": 1, "title": "...", "body": "..." }
```

`src/main/java/com/example/postsdemo/external/PostDto.java`

```java
package com.example.postsdemo.external;

public class PostDto {
    private Long userId;
    private Long id;     // external id from JSONPlaceholder
    private String title;
    private String body;

    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getBody() { return body; }
    public void setBody(String body) { this.body = body; }
}
```

---

# 6) JPA Entity + Repository

We’ll persist with a **surrogate primary key** and keep the remote ID as `externalId` (unique) for idempotent upserts.

`src/main/java/com/example/postsdemo/domain/PostEntity.java`

```java
package com.example.postsdemo.domain;

import jakarta.persistence.*;

@Entity
@Table(name = "posts", uniqueConstraints = {
        @UniqueConstraint(name = "uk_posts_external_id", columnNames = "external_id")
})
public class PostEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; // DB PK

    @Column(name = "external_id", nullable = false)
    private Long externalId; // remote API id

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(length = 500)
    private String title;

    @Column(length = 4000)
    private String body;

    // Getters/setters
    public Long getId() { return id; }
    public Long getExternalId() { return externalId; }
    public void setExternalId(Long externalId) { this.externalId = externalId; }
    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getBody() { return body; }
    public void setBody(String body) { this.body = body; }
}
```

`src/main/java/com/example/postsdemo/domain/PostRepository.java`

```java
package com.example.postsdemo.domain;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Collection;
import java.util.List;
import java.util.Optional;

public interface PostRepository extends JpaRepository<PostEntity, Long> {
    Optional<PostEntity> findByExternalId(Long externalId);
    List<PostEntity> findByExternalIdIn(Collection<Long> externalIds);

    // For querying saved data
    List<PostEntity> findByUserId(Long userId);
    List<PostEntity> findByTitleContainingIgnoreCase(String titlePart);
}
```

---

# 7) Service: fetch, filter, map, upsert

We’ll **filter on the server** by passing `?userId=…` to JSONPlaceholder (more efficient), then upsert into our DB.

`src/main/java/com/example/postsdemo/service/PostService.java`

```java
package com.example.postsdemo.service;

import com.example.postsdemo.domain.PostEntity;
import com.example.postsdemo.domain.PostRepository;
import com.example.postsdemo.external.PostDto;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

@Service
public class PostService {

    private static final String POSTS_URL = "https://jsonplaceholder.typicode.com/posts";
    private final RestTemplate restTemplate;
    private final PostRepository postRepository;

    public PostService(RestTemplate restTemplate, PostRepository postRepository) {
        this.restTemplate = restTemplate;
        this.postRepository = postRepository;
    }

    public List<PostEntity> fetchAndSaveByUserId(long userId) {
        String url = POSTS_URL + "?userId=" + userId;

        PostDto[] response = restTemplate.getForObject(url, PostDto[].class);
        List<PostDto> dtos = response == null ? List.of() : Arrays.asList(response);

        // Defensive: also possible to filter client-side if interviewer asks
        List<PostDto> filtered = dtos.stream()
                .filter(p -> p.getUserId() != null && p.getUserId() == userId)
                .collect(Collectors.toList());

        return upsertPosts(filtered);
    }

    private List<PostEntity> upsertPosts(List<PostDto> dtos) {
        if (dtos.isEmpty()) return List.of();

        // Load existing by externalId in bulk to avoid N+1
        List<Long> extIds = dtos.stream().map(PostDto::getId).filter(Objects::nonNull).toList();
        Map<Long, PostEntity> existingByExtId = postRepository.findByExternalIdIn(extIds).stream()
                .collect(Collectors.toMap(PostEntity::getExternalId, Function.identity()));

        List<PostEntity> toSave = new ArrayList<>();
        for (PostDto dto : dtos) {
            if (dto.getId() == null) continue;

            PostEntity entity = existingByExtId.getOrDefault(dto.getId(), new PostEntity());
            entity.setExternalId(dto.getId());
            entity.setUserId(dto.getUserId());
            entity.setTitle(dto.getTitle());
            entity.setBody(dto.getBody());

            toSave.add(entity);
        }

        return postRepository.saveAll(toSave);
    }
}
```

---

# 8) Controller: endpoints to trigger and query

`src/main/java/com/example/postsdemo/web/PostController.java`

```java
package com.example.postsdemo.web;

import com.example.postsdemo.domain.PostEntity;
import com.example.postsdemo.domain.PostRepository;
import com.example.postsdemo.service.PostService;
import jakarta.validation.constraints.Min;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/posts")
@Validated
public class PostController {

    private final PostService postService;
    private final PostRepository postRepository;

    public PostController(PostService postService, PostRepository postRepository) {
        this.postService = postService;
        this.postRepository = postRepository;
    }

    // 1) Trigger fetch+filter+save (by userId)
    @PostMapping("/sync")
    public List<PostEntity> syncByUser(
            @RequestParam @Min(1) long userId
    ) {
        return postService.fetchAndSaveByUserId(userId);
    }

    // 2) Query what's saved (filter locally)
    @GetMapping
    public List<PostEntity> getSaved(
            @RequestParam(required = false) Long userId,
            @RequestParam(required = false, name = "titleContains") String titleContains
    ) {
        if (userId != null) {
            return postRepository.findByUserId(userId);
        }
        if (titleContains != null && !titleContains.isBlank()) {
            return postRepository.findByTitleContainingIgnoreCase(titleContains);
        }
        return postRepository.findAll();
    }
}
```

(Optional) Simple error handler:

`src/main/java/com/example/postsdemo/web/GlobalExceptionHandler.java`

```java
package com.example.postsdemo.web;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleValidation(MethodArgumentNotValidException ex) {
        return Map.of("error", "Invalid request", "details", ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Map<String, String> handleGeneric(Exception ex) {
        return Map.of("error", "Unexpected error", "details", ex.getMessage());
    }
}
```

---

# 9) Run it

From IntelliJ **(green play button)** or terminal:

```bash
./mvnw spring-boot:run
```

Open the H2 console (optional):
`http://localhost:8080/h2-console`

* JDBC URL: `jdbc:h2:mem:postsdb`
* User: `sa` (no password)

---

# 10) Try the endpoints

**Sync & save posts for `userId=1`:**

```bash
curl -X POST "http://localhost:8080/api/posts/sync?userId=1"
```

**List everything saved so far:**

```bash
curl "http://localhost:8080/api/posts"
```

**Filter saved posts by `userId`:**

```bash
curl "http://localhost:8080/api/posts?userId=1"
```

**Filter saved posts by title substring:**

```bash
curl "http://localhost:8080/api/posts?titleContains=ipsum"
```

---

## What interviewers often look for (and you can mention)

* **Separation of concerns**: DTO (external shape) vs Entity (DB shape).
* **Idempotency**: use `externalId` as a unique key to upsert.
* **Efficiency**: bulk read existing records with `findByExternalIdIn` to avoid N+1 queries.
* **Validation**: basic request param validation (optional).
* **Observability**: enable SQL logs, or add logs around the fetch/transform/save path.
* **Extensibility**: add more filters (e.g., body contains), pagination on GET, switch H2 → Postgres by changing JDBC URL & driver.

If you want, I can also show a **WebClient** version (reactive) or swap H2 for Postgres with a Docker compose file.

---

# Use Lombok

---

Perfect! Let’s simplify the boilerplate by adding **Lombok**. It helps reduce getters, setters, constructors, and logging clutter, which is something interviewers often like because it shows you know how to keep code concise.

---

## 1) Add Lombok dependency

Update your `pom.xml`:

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.32</version>
    <scope>provided</scope>
</dependency>
```

➡ After adding, in IntelliJ:

* Enable **Annotation Processing** (`Settings → Build, Execution, Deployment → Compiler → Annotation Processors → Enable annotation processing`).

---

## 2) Update your classes with Lombok

### `PostDto.java`

```java
package com.example.postsdemo.external;

import lombok.Data;

@Data
public class PostDto {
    private Long userId;
    private Long id;     // external id
    private String title;
    private String body;
}
```

`@Data` gives you:

* Getters & setters
* `toString()`
* `equals()` & `hashCode()`

---

### `PostEntity.java`

```java
package com.example.postsdemo.domain;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "posts", uniqueConstraints = {
        @UniqueConstraint(name = "uk_posts_external_id", columnNames = "external_id")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PostEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id; // DB PK

    @Column(name = "external_id", nullable = false)
    private Long externalId;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(length = 500)
    private String title;

    @Column(length = 4000)
    private String body;
}
```

Here:

* `@Getter` / `@Setter` → eliminates boilerplate
* `@NoArgsConstructor` / `@AllArgsConstructor` → convenient for JPA & testing
* `@Builder` → makes object construction cleaner in services/tests

---

### Optional: Use Lombok logging

If you want logging in your service:

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class PostService {
    ...
    public List<PostEntity> fetchAndSaveByUserId(long userId) {
        log.info("Fetching posts for userId={}", userId);
        ...
    }
}
```

---

## 3) Example usage with Lombok

Because `@Builder` is enabled:

```java
PostEntity entity = PostEntity.builder()
        .externalId(dto.getId())
        .userId(dto.getUserId())
        .title(dto.getTitle())
        .body(dto.getBody())
        .build();
```

This is cleaner than manually calling setters everywhere.

---

✅ Now your project is leaner, easier to read, and interviewer-friendly.

Would you like me to also **refactor the `PostService` upsert method** to take advantage of `@Builder` and Lombok instead of manual setters?

---

Do you want me to also show you how to **enforce uniqueness** on `external_id` at the JPA level so you can guarantee idempotent saves (no duplicates)?
