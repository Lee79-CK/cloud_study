# Java 17 & 21 — 기존 버전 대비 개선점과 활용 방안

---

## 0. 들어가며: Java의 릴리스 모델 이해

Java는 2017년(Java 9)부터 **6개월 주기 릴리스**를 채택했습니다. 이 중 **LTS(Long-Term Support)** 버전만 장기 지원을 받습니다.

| 버전 | 출시 | LTS 여부 | 지원 종료(Oracle) |
|------|------|----------|-------------------|
| Java 8 | 2014.03 | ✅ LTS | 2030+ |
| Java 11 | 2018.09 | ✅ LTS | 2032+ |
| Java 17 | 2021.09 | ✅ LTS | 2029+ |
| Java 21 | 2023.09 | ✅ LTS | 2031+ |

현재 기업 환경에서의 마이그레이션 경로는 대부분 **8 → 11 → 17 → 21** 입니다. 이 강의에서는 Java 11 이하를 "기존 버전"으로 놓고, 17과 21에서 무엇이 달라졌는지를 체계적으로 다룹니다.

---

## Part 1. Java 17 — "현대 Java의 시작점"

Java 17은 Java 11 이후 두 번째 LTS로, 12~16에서 인큐베이팅/프리뷰되던 기능들이 **정식 확정(finalize)** 된 버전입니다.

---

### 1.1 Sealed Classes (JEP 409) — 상속 계층 제어

**기존 문제:** Java에서 클래스 상속을 제한하는 방법이 `final`(완전 차단) 아니면 패키지-프라이빗(같은 패키지 내 허용)뿐이었습니다. "이 3개 클래스만 상속 가능"이라는 **중간 단계 제어**가 불가능했습니다.

```java
// Java 17: sealed 클래스
public sealed class Shape 
    permits Circle, Rectangle, Triangle {
    // ...
}

public final class Circle extends Shape {
    private final double radius;
    // Circle은 더 이상 하위 클래스를 가질 수 없음 (final)
}

public non-sealed class Rectangle extends Shape {
    // non-sealed: 자유롭게 확장 가능
}

public sealed class Triangle extends Shape 
    permits EquilateralTriangle {
    // sealed: 지정한 하위 클래스만 확장 가능
}
```

**핵심 키워드 3가지:**
- `sealed`: 이 클래스를 상속할 수 있는 클래스를 `permits`로 명시적 지정
- `final`: 더 이상 상속 불가
- `non-sealed`: 다시 개방 (아무나 상속 가능)

**왜 중요한가?**

컴파일러가 **모든 하위 타입을 알 수 있게** 되므로, 패턴 매칭에서 완전성(exhaustiveness) 검사가 가능해집니다. 이는 Java 21의 `switch` 패턴 매칭과 결합하면 강력한 시너지를 냅니다.

**활용 시나리오 — MLOps 파이프라인 이벤트:**
```java
public sealed interface PipelineEvent 
    permits TrainingStarted, TrainingCompleted, TrainingFailed, ModelDeployed {
}

public record TrainingStarted(String modelId, Instant startTime) 
    implements PipelineEvent {}
public record TrainingCompleted(String modelId, double accuracy) 
    implements PipelineEvent {}
public record TrainingFailed(String modelId, Exception cause) 
    implements PipelineEvent {}
public record ModelDeployed(String modelId, String endpoint) 
    implements PipelineEvent {}
```

---

### 1.2 Records (JEP 395) — 불변 데이터 클래스

**기존 문제:** 단순 데이터 운반 객체(DTO, VO)를 만들려면 `equals()`, `hashCode()`, `toString()`, getter, 생성자를 **일일이 작성**(또는 Lombok 의존)해야 했습니다.

```java
// Java 11 이하: 전통적 DTO
public class ServerMetrics {
    private final String hostname;
    private final double cpuUsage;
    private final long memoryBytes;

    public ServerMetrics(String hostname, double cpuUsage, long memoryBytes) {
        this.hostname = hostname;
        this.cpuUsage = cpuUsage;
        this.memoryBytes = memoryBytes;
    }

    public String getHostname() { return hostname; }
    public double getCpuUsage() { return cpuUsage; }
    public long getMemoryBytes() { return memoryBytes; }

    @Override
    public boolean equals(Object o) { /* 15줄의 보일러플레이트 */ }
    @Override
    public int hashCode() { return Objects.hash(hostname, cpuUsage, memoryBytes); }
    @Override
    public String toString() { /* ... */ }
}
```

```java
// Java 17: Record — 한 줄로 끝
public record ServerMetrics(String hostname, double cpuUsage, long memoryBytes) {
    // 컴팩트 생성자로 유효성 검증 가능
    public ServerMetrics {
        if (cpuUsage < 0 || cpuUsage > 100) {
            throw new IllegalArgumentException("CPU usage must be 0-100");
        }
    }
}

// 사용
var metrics = new ServerMetrics("gpu-node-01", 87.5, 34_359_738_368L);
System.out.println(metrics.hostname());    // "gpu-node-01" (getter가 아닌 접근자)
System.out.println(metrics);               // ServerMetrics[hostname=gpu-node-01, cpuUsage=87.5, ...]
```

**Record의 특성:**
- **불변(immutable):** 모든 필드가 `private final`
- **값 기반 동등성:** `equals()`가 모든 컴포넌트의 값으로 비교
- **상속 불가:** 암묵적으로 `final`
- **인터페이스 구현 가능:** `implements Serializable` 등

**Lombok이 필요 없어지는가?**
Record가 커버하는 영역(불변 DTO)에서는 더 이상 Lombok이 필요 없습니다. 다만 Builder 패턴이나 가변 클래스에는 여전히 Lombok이 유용합니다.

---

### 1.3 Pattern Matching for instanceof (JEP 394)

**기존 문제:**
```java
// Java 11: 타입 체크 후 캐스팅 반복
if (obj instanceof String) {
    String s = (String) obj;  // 중복! 이미 String인 걸 아는데
    System.out.println(s.length());
}
```

```java
// Java 17: 패턴 변수 바인딩
if (obj instanceof String s) {
    // s는 이미 String 타입으로 바인딩됨
    System.out.println(s.length());
}

// 가드 조건과 결합
if (obj instanceof String s && s.length() > 10) {
    processLongString(s);
}
```

작아 보이지만, 실무에서 `instanceof` 체크가 빈번한 코드(이벤트 핸들러, 직렬화/역직렬화 로직 등)에서 **가독성이 크게 향상**됩니다.

---

### 1.4 Text Blocks (JEP 378) — 여러 줄 문자열

**기존 문제:** JSON, SQL, YAML, HTML 등 멀티라인 문자열을 다루려면 `+`와 `\n`의 지옥이었습니다.

```java
// Java 11
String query = "SELECT m.model_id, m.accuracy, m.created_at\n" +
               "FROM ml_models m\n" +
               "JOIN experiments e ON m.experiment_id = e.id\n" +
               "WHERE e.status = 'COMPLETED'\n" +
               "  AND m.accuracy > 0.95\n" +
               "ORDER BY m.created_at DESC";
```

```java
// Java 17: Text Block
String query = """
        SELECT m.model_id, m.accuracy, m.created_at
        FROM ml_models m
        JOIN experiments e ON m.experiment_id = e.id
        WHERE e.status = 'COMPLETED'
          AND m.accuracy > %s
        ORDER BY m.created_at DESC
        """.formatted(minAccuracy);
```

**들여쓰기 규칙:** 닫는 `"""`의 위치가 기준점(incidental whitespace 제거)이 됩니다.

**활용 시나리오:** Kubernetes YAML 생성, SQL 쿼리, JSON 페이로드 구성, Dockerfile 템플릿 등 — 클라우드 엔지니어링에서 매우 실용적입니다.

---

### 1.5 Switch Expressions (JEP 361)

```java
// Java 11: switch statement (fall-through 위험)
String tier;
switch (statusCode) {
    case 200:
    case 201:
        tier = "SUCCESS";
        break;     // break 빼먹으면 버그
    case 404:
        tier = "NOT_FOUND";
        break;
    default:
        tier = "ERROR";
}
```

```java
// Java 17: switch expression (값 반환, 화살표 문법)
String tier = switch (statusCode) {
    case 200, 201 -> "SUCCESS";
    case 404      -> "NOT_FOUND";
    default       -> "ERROR";
};

// 블록이 필요하면 yield로 값 반환
String message = switch (statusCode) {
    case 200 -> "OK";
    case 500 -> {
        log.error("Internal Server Error");
        yield "SERVER_ERROR";
    }
    default -> "UNKNOWN";
};
```

---

### 1.6 기타 주요 변경사항

**NullPointerException 개선 (JEP 358):**
```
// 기존: java.lang.NullPointerException
// Java 17:
// java.lang.NullPointerException: 
//   Cannot invoke "String.length()" because "user.getAddress().getCity()" is null
```
→ 어떤 참조가 null인지 정확히 알려주므로 **디버깅 시간이 대폭 단축**됩니다.

**ZGC / Shenandoah GC 정식화:**
- ZGC: 대용량 힙(수 TB)에서도 **pause time < 1ms** 목표
- 클라우드 환경에서 GC 튜닝 부담 감소

**Strong Encapsulation of JDK Internals:**
- `sun.misc.Unsafe` 등 내부 API에 대한 접근이 기본 차단
- `--add-opens`로 임시 허용 가능하지만, 장기적으로 공식 API 사용 권장

**Apple Silicon (AArch64) 네이티브 지원:**
- M1/M2 Mac에서 네이티브 성능으로 개발 가능

---

## Part 2. Java 21 — "프로젝트 Loom과 모던 Java의 완성"

Java 21은 역대 LTS 중 가장 변화가 큰 버전입니다. **Virtual Threads**, **완성된 Pattern Matching**, **Sequenced Collections** 등이 정식화되었습니다.

---

### 2.1 Virtual Threads (JEP 444) — 게임 체인저

**이것이 Java 21의 가장 중요한 기능입니다.**

**기존 문제: Platform Thread의 한계**

```
[요청 1] → [Platform Thread 1] → DB I/O 대기... → 응답
[요청 2] → [Platform Thread 2] → API 호출 대기... → 응답
...
[요청 2000] → Thread 생성 실패! (OS 스레드 한계)
```

- Platform Thread는 **OS 커널 스레드와 1:1 매핑**
- 스레드 하나당 약 1MB 스택 메모리 소비
- 수천 개 이상 생성하면 OOM 또는 컨텍스트 스위칭 비용 폭증
- 그래서 **스레드 풀**로 제한하고, I/O 대기 시 스레드가 블록됨 → 처리량 병목

**해결: Virtual Thread**

```
[요청 1] → [Virtual Thread 1] → I/O 대기 시 Platform Thread 반납 → 완료되면 다시 할당
[요청 2] → [Virtual Thread 2] → ...
...
[요청 100만] → Virtual Thread 100만 개 가능! (각각 수 KB)
```

```java
// 1. 가장 기본적인 생성
Thread.startVirtualThread(() -> {
    System.out.println("Hello from virtual thread: " + Thread.currentThread());
});

// 2. ExecutorService 활용 (실무에서 가장 많이 쓰는 패턴)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // 10만 개의 동시 HTTP 요청도 문제없음
    List<Future<String>> futures = IntStream.range(0, 100_000)
        .mapToObj(i -> executor.submit(() -> fetchUrl("https://api.example.com/item/" + i)))
        .toList();

    for (var future : futures) {
        String result = future.get();
        process(result);
    }
}   // try-with-resources로 자동 종료 대기

// 3. Spring Boot 3.2+에서는 설정 한 줄로 적용
// application.yml
// spring:
//   threads:
//     virtual:
//       enabled: true
```

**Virtual Thread의 동작 원리:**

```
Virtual Thread (수백만 개 가능)
    ↓ 스케줄링
Carrier Thread (= Platform Thread, ForkJoinPool)
    ↓ 매핑
OS Kernel Thread
```

- Virtual Thread가 I/O(소켓 읽기, DB 쿼리 등)에서 블록되면, **Carrier Thread를 즉시 반납**
- I/O가 완료되면 사용 가능한 Carrier Thread에 다시 마운트
- 개발자는 기존 블로킹 코드를 그대로 쓰면 됨 → **리액티브 프로그래밍 불필요**

**주의사항:**
```java
// ❌ 피해야 할 패턴: synchronized 블록 내 블로킹 I/O
synchronized (lock) {
    // Virtual Thread가 Carrier Thread를 "pin"하여 반납 불가
    resultSet = statement.executeQuery(sql);
}

// ✅ 대안: ReentrantLock 사용
private final ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    resultSet = statement.executeQuery(sql);
} finally {
    lock.unlock();
}
```

**활용 시나리오 — ML 모델 서빙 게이트웨이:**
```java
// 수천 개의 동시 추론 요청을 처리하는 게이트웨이
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var futures = inferenceRequests.stream()
        .map(req -> executor.submit(() -> {
            // 각 요청이 별도 Virtual Thread에서 실행
            var features = featureStore.fetch(req.userId());   // I/O: Redis
            var prediction = modelClient.predict(features);     // I/O: gRPC
            metricsStore.record(req.modelId(), prediction);     // I/O: DB
            return prediction;
        }))
        .toList();
    // ...
}
```

---

### 2.2 Pattern Matching for switch (JEP 441) — 완전한 패턴 매칭

Java 17의 Sealed Classes + Record + instanceof 패턴 매칭이 **switch 문에서 완성**됩니다.

```java
// Java 21: 타입 패턴 + 가드 조건
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0  -> "양의 정수: " + i;
        case Integer i             -> "정수: " + i;
        case String s when s.isBlank() -> "빈 문자열";
        case String s              -> "문자열(길이=" + s.length() + ")";
        case int[] arr             -> "int 배열(크기=" + arr.length + ")";
        case null                  -> "null!";
        default                    -> "기타: " + obj.getClass().getSimpleName();
    };
}
```

**Sealed Class와의 시너지 — exhaustive switch:**
```java
// PipelineEvent가 sealed이므로 컴파일러가 모든 케이스를 검증
String handleEvent(PipelineEvent event) {
    return switch (event) {
        case TrainingStarted e   -> "학습 시작: " + e.modelId();
        case TrainingCompleted e -> "학습 완료 (정확도: " + e.accuracy() + ")";
        case TrainingFailed e    -> "학습 실패: " + e.cause().getMessage();
        case ModelDeployed e     -> "배포 완료: " + e.endpoint();
        // default 불필요! 컴파일러가 모든 케이스가 커버됨을 보장
        // 새 하위 타입 추가 시 컴파일 에러 → 누락 방지
    };
}
```

이것은 함수형 언어(Scala, Kotlin, Rust 등)의 **ADT(Algebraic Data Type) + 패턴 매칭**과 동등한 표현력입니다.

---

### 2.3 Record Patterns (JEP 440) — 구조 분해

```java
// Record 내부 컴포넌트를 직접 분해
record Point(int x, int y) {}
record Line(Point start, Point end) {}

void printLine(Object obj) {
    // 중첩 구조 분해! (Destructuring)
    if (obj instanceof Line(Point(var x1, var y1), Point(var x2, var y2))) {
        System.out.printf("(%d,%d) -> (%d,%d)%n", x1, y1, x2, y2);
    }
}

// switch에서도 동일하게 사용
String classify(Shape shape) {
    return switch (shape) {
        case Circle(var r) when r > 100 -> "큰 원";
        case Circle(var r)              -> "작은 원 (반지름=" + r + ")";
        case Rectangle(var w, var h)    -> "직사각형 " + w + "x" + h;
        default -> "기타 도형";
    };
}
```

---

### 2.4 Sequenced Collections (JEP 431) — 컬렉션 API 정비

**기존 문제:** "첫 번째 요소"와 "마지막 요소"에 접근하는 방법이 컬렉션마다 달랐습니다.

```java
// Java 11: 일관성 없는 API
// List
list.get(0);                          // 첫 번째
list.get(list.size() - 1);            // 마지막

// Deque
deque.getFirst();                     // 첫 번째
deque.getLast();                       // 마지막

// SortedSet
sortedSet.first();                    // 첫 번째
sortedSet.last();                     // 마지막

// LinkedHashSet → 마지막 요소? 방법 없음! 전체 순회 필요
```

```java
// Java 21: 통일된 인터페이스
// SequencedCollection: List, Deque, LinkedHashSet, SortedSet 모두 구현
SequencedCollection<String> seq = ...;
seq.getFirst();         // 첫 번째
seq.getLast();          // 마지막
seq.addFirst("new");    // 맨 앞에 추가
seq.addLast("end");     // 맨 뒤에 추가
seq.reversed();         // 역순 뷰 반환 (새 컬렉션 아님)

// SequencedMap도 추가
SequencedMap<String, Integer> map = new LinkedHashMap<>();
map.firstEntry();       // 첫 번째 엔트리
map.lastEntry();        // 마지막 엔트리
map.pollLastEntry();    // 마지막 제거 후 반환
map.reversed();         // 역순 뷰
```

---

### 2.5 Structured Concurrency (JEP 453, Preview) — 구조화된 동시성

Virtual Thread와 함께 사용하면 강력한 **동시 작업 관리** 도구입니다.

```java
// 여러 API를 동시 호출하되, 하나라도 실패하면 전체 취소
ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

ModelServingResponse serve(InferenceRequest req) throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        // 세 작업을 동시에 시작 (각각 Virtual Thread)
        Subtask<Features> features = scope.fork(() -> 
            featureStore.getFeatures(req.userId()));
        Subtask<ModelConfig> config = scope.fork(() -> 
            configService.getModelConfig(req.modelId()));
        Subtask<RateLimitResult> rateLimit = scope.fork(() -> 
            rateLimiter.check(req.clientId()));

        scope.join();              // 모든 작업 완료 대기
        scope.throwIfFailed();     // 하나라도 실패 시 예외 전파

        // 모든 결과를 안전하게 사용
        return modelEngine.predict(
            features.get(), config.get(), rateLimit.get()
        );
    }
    // scope 종료 시 미완료 작업은 자동 취소됨
}
```

**기존 CompletableFuture 대비 장점:**
- 작업 간 **부모-자식 관계**가 명확
- 자식 실패 시 형제 작업 **자동 취소** (리소스 누수 방지)
- 스레드 덤프에서 **작업 계층 구조** 확인 가능 (디버깅 용이)

---

### 2.6 기타 주요 변경사항

**String 신규 메서드들 (Java 12~21 누적):**
```java
// 들여쓰기
"hello".indent(4);        // "    hello\n"

// 변환
"hello".transform(s -> s.toUpperCase() + "!");  // "HELLO!"

// stripIndent, translateEscapes 등
```

**Generational ZGC (JEP 439):**
- ZGC에 세대별 GC 개념 추가 → young/old generation 분리
- 처리량(throughput) 대폭 향상, 메모리 오버헤드 감소
- 클라우드 환경에서 컨테이너 리소스 효율성 개선

**Key Encapsulation Mechanism API (JEP 452):**
- 포스트 양자 암호화를 위한 기반 API
- 향후 보안 표준 대응에 중요

**Unnamed Patterns and Variables (JEP 443):**
```java
// 사용하지 않는 변수를 _로 표시
try {
    // ...
} catch (Exception _) {
    // 예외 객체를 사용하지 않음을 명시
    log.warn("작업 실패");
}

// 패턴에서도
if (obj instanceof Point(var x, _)) {
    // y 좌표는 불필요
    System.out.println("x = " + x);
}
```

---

## Part 3. 마이그레이션 전략과 실무 적용

### 3.1 버전별 마이그레이션 로드맵

```
Java 8/11 → Java 17 (안정화된 LTS, 생태계 호환성 높음)
    ↓ 안정화 후
Java 17 → Java 21 (Virtual Threads 필요 시)
```

**단계별 체크리스트:**

1단계 — 빌드 및 의존성 점검:
- Gradle 7.3+ / Maven 3.8+ 업그레이드
- 모든 라이브러리의 Java 17/21 호환 버전 확인
- `--add-opens` 필요 여부 확인 (내부 API 사용 코드)

2단계 — 코드 현대화 (점진적):
- IDE의 "Migrate to newer Java" 인스펙션 활용
- DTO → Record 전환 (Lombok 의존성 줄이기)
- instanceof 패턴 매칭 적용
- Text Block 적용 (SQL, JSON, YAML 리터럴)

3단계 — 런타임 최적화:
- GC를 G1 → ZGC 로 전환 검토 (저지연 요구 시)
- Virtual Thread 적용 (Java 21, I/O 바운드 서비스)

### 3.2 Spring Boot와의 호환성

| Spring Boot | 최소 Java | 권장 Java |
|-------------|-----------|-----------|
| 2.7.x | 8 | 17 |
| 3.0~3.1 | 17 | 17 |
| 3.2+ | 17 | 21 |
| 3.4+ | 17 | 21 |

Spring Boot 3.2부터 Virtual Thread를 **설정 한 줄**로 활성화할 수 있고, Spring Framework 6.1은 Virtual Thread를 1급 시민으로 지원합니다.

### 3.3 컨테이너/클라우드 환경 최적화

```dockerfile
# 멀티스테이지 빌드 예시
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY . .
RUN ./gradlew bootJar

FROM eclipse-temurin:21-jre
COPY --from=builder /app/build/libs/*.jar app.jar

# Java 21 + 컨테이너 최적화 JVM 옵션
ENTRYPOINT ["java", \
  "-XX:+UseZGC", \
  "-XX:+ZGenerational", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseContainerSupport", \
  "-jar", "app.jar"]
```

**핵심 JVM 플래그:**
- `-XX:+UseContainerSupport` (기본 활성): 컨테이너 메모리/CPU 제한 자동 인식
- `-XX:MaxRAMPercentage=75.0`: 컨테이너 메모리의 75%를 힙으로 사용
- `-XX:+UseZGC -XX:+ZGenerational`: Java 21의 세대별 ZGC

---

## Part 4. 정리 — 한눈에 보는 핵심 비교

| 영역 | Java 8/11 | Java 17 | Java 21 |
|------|-----------|---------|---------|
| 데이터 클래스 | 보일러플레이트 or Lombok | **Record** | Record |
| 상속 제어 | final or open | **Sealed Classes** | Sealed + 패턴매칭 |
| 패턴 매칭 | 없음 | instanceof 패턴 | **switch 패턴 + Record 패턴** |
| 문자열 | `+`와 `\n` | **Text Block** | Text Block |
| switch | statement (fall-through) | **expression** | expression + 패턴 |
| 동시성 | Thread Pool + CompletableFuture | 동일 | **Virtual Threads** |
| GC | G1 (기본) | ZGC 정식 | **Generational ZGC** |
| NPE | 위치 불명 | **Helpful NPE** | Helpful NPE |
| 컬렉션 | 비일관적 API | 동일 | **Sequenced Collections** |

**최종 권장:**

현시점에서 신규 프로젝트는 **Java 21**로 시작하는 것이 가장 합리적입니다. Virtual Threads 하나만으로도 I/O 바운드 서비스의 아키텍처가 극적으로 단순화됩니다. 리액티브 스택(WebFlux, Reactor)의 복잡성 없이 높은 동시성을 달성할 수 있다는 점은, 특히 ML 서빙이나 데이터 파이프라인처럼 다수의 외부 I/O 호출이 필요한 시스템에서 큰 이점입니다.
