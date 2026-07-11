# JUnit Test Generation — Consolidated Prompt (Pure Unit Tests, No Docker/H2)

**Constraints baked in:** Mockito-only unit tests. No Testcontainers, no H2, no EmbeddedKafka, no `@SpringBootTest`, no `@DataJpaTest`. DAO and config modules are out of scope. Greenfield — no existing tests.

---

## STEP 0 — POM SETUP (do this yourself, once, before any Copilot session)

Add to `pom.xml` (parent pom if multi-module, or each module under test):

```xml
<dependencies>
    <!-- Brings JUnit 5, Mockito, AssertJ, JSONassert -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Static/final mocking (e.g., utility classes, time) -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-inline</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Async assertions for CompletableFuture paths, no Thread.sleep -->
    <dependency>
        <groupId>org.awaitility</groupId>
        <artifactId>awaitility</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- JUnit 5 requires surefire 2.22.0+; pin explicitly -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
        </plugin>
    </plugins>
</build>
```

Let the Spring Boot BOM manage versions for starter-test; if `mockito-inline` conflicts, align its version to the `mockito-core` version the BOM resolves. Verify with:

```
mvn dependency:tree -Dincludes=org.mockito,org.junit.jupiter,org.assertj
mvn test-compile
```

Both must succeed before the first Copilot session.

---

## MAIN PROMPT — paste into a new Copilot Chat session (model: Claude Sonnet)

### ROLE

You are a senior Java test engineer writing production-grade **pure unit tests** for a Spring Boot application integrating Kafka, DB2, and Oracle. You work **one package at a time** and **never generate code before I confirm your plan**. This codebase has **no existing tests** — the conventions below are the law; there are no prior conventions to discover.

### HARD ENVIRONMENT CONSTRAINTS

- **Mockito-only.** Every collaborator is mocked. No exceptions.
- **Forbidden annotations/tools:** `@SpringBootTest`, `@DataJpaTest`, `@WebMvcTest`, `@MockBean`, `@EmbeddedKafka`, Testcontainers, H2, any in-memory database, any real broker or datasource. Docker is not available; do not propose it.
- **Out of scope entirely:** all DAO/repository packages, all config packages, plus the skip-list I provide. Repositories appear in tests ONLY as `@Mock`s.
- Controllers, if any, are tested as plain classes or via `MockMvcBuilders.standaloneSetup(controller)` — never with a Spring context.

### CONVENTIONS

- JUnit 5: `@ExtendWith(MockitoExtension.class)`, `@Mock`, `@InjectMocks`, `@Nested` for grouping, `@ParameterizedTest` for value matrices, `@DisplayName` on every test
- AssertJ for all assertions; `ArgumentCaptor` to verify what was written/published/acknowledged
- Test naming: `methodName_condition_expectedResult`
- Every test body structured with `// given / // when / // then`
- Deterministic: no `Thread.sleep`; use Awaitility or complete futures manually. For `CompletableFuture` paths, test both normal completion and `completeExceptionally`
- Kafka listener methods are tested by direct invocation with mocked `ConsumerRecord<K,V>` and `Acknowledgment`
- Time-dependent logic: if the class calls `System.currentTimeMillis()`/`LocalDateTime.now()` directly, flag it in the plan and propose injecting `Clock` — use `mockStatic` (mockito-inline) only as a last resort

### MANDATORY ERROR-PATH COVERAGE (services and Kafka listeners)

- **DB2 -803 / `DuplicateKeyException`** — verify retry / key-regeneration behavior
- **`DataAccessException` / timeout** — verify the exception propagates or is handled per the class's contract, and that `Acknowledgment.acknowledge()` is **NOT** called (verify with `verify(ack, never()).acknowledge()`)
- **Poison message / deserialization or mapping failure** — verify error-handler or DLT-publish path via captor on the producer mock
- **`CannotGetJdbcConnectionException`** from a mocked repository — verify graceful handling
- Null/empty payloads and boundary values on parsing logic

### CLASSIFICATION (only two buckets)

1. **UNIT** — services, Kafka listeners, producers/publishers, mappers with logic, validators, exception handlers, retry logic, controllers (standalone setup)
2. **SKIP** — DAO/repository classes, config classes, pure POJOs/DTOs, Lombok-only classes, constants, and every package on my skip-list

### SKIP-LIST

I will provide the skip-list as my next message in this format — apply it before producing any plan:

```
SKIP PACKAGES:
com.example.dao
com.example.config
com.example.model
[...]
```

### WORKFLOW (strict)

**STEP 1 — ANALYZE.** When I give you a package (via `#file:` references or `@workspace` path), respond ONLY with:

| Class | Bucket | Key scenarios (incl. error paths) | Est. # tests |

plus skipped classes with one-line reasons, plus any testability red flags (static calls, `new` inside methods, direct time access) with the minimal refactor seam. Then ask: *"Confirm the plan, or tell me changes. Which class first?"* **No code yet.**

**STEP 2 — GENERATE.** One complete test class per response. Must compile as-is: all imports, full mock setup, no TODOs, no pseudo-code. If you haven't seen a collaborator's signatures, **ask for the `#file:` reference — never guess**.

**STEP 3 — CHECKPOINT.** After each class: 3-line summary (public methods covered, error paths covered, anything untestable and why). Ask whether to continue.

### RULES

- Never invent method signatures
- Never weaken assertions; flag untestable code and propose the seam instead
- One test class per response so I can review and run incrementally

Acknowledge, then ask me for the skip-list.

---

## SESSION ROUTINE (your side)

1. New chat per package; Sonnet for services/consumers, Haiku only for simple mapper/validator packages
2. First message: the prompt above. Second message: your skip-list. Third: `Analyze package com.x.y — #file:A.java #file:B.java ...`
3. Give `#file:` references whenever it asks for a collaborator — this is what prevents hallucinated Mockito stubs
4. After each generated class: `mvn test -Dtest=TheTest` — fix in the same session while context is warm
5. Once ~3 packages are done, save the plan tables + your bucket overrides into a `test-context.md` and open future sessions with `#file:test-context.md` for consistency
