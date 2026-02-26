# Java 고급 개념 완전 정복
### Senior Engineer를 위한 박사급 심화 학습 가이드 & FAANG 면접 대비

---

## 목차

1. [JVM 아키텍처 심층 분석](#1-jvm-아키텍처-심층-분석)
2. [메모리 모델 및 가비지 컬렉션](#2-메모리-모델-및-가비지-컬렉션)
3. [동시성과 병렬 프로그래밍](#3-동시성과-병렬-프로그래밍)
4. [Java 타입 시스템 고급](#4-java-타입-시스템-고급)
5. [함수형 프로그래밍과 Stream API](#5-함수형-프로그래밍과-stream-api)
6. [리플렉션, 어노테이션, 메타프로그래밍](#6-리플렉션-어노테이션-메타프로그래밍)
7. [성능 최적화 및 프로파일링](#7-성능-최적화-및-프로파일링)
8. [모듈 시스템(JPMS)과 클래스로더](#8-모듈-시스템jpms과-클래스로더)
9. [Project Loom과 가상 스레드](#9-project-loom과-가상-스레드)
10. [해외 대기업 면접 Q&A 30선](#10-해외-대기업-면접-qa-30선)

---

## 1. JVM 아키텍처 심층 분석

### 1.1 JVM 실행 엔진 (Execution Engine)

JVM은 단순한 인터프리터가 아니라 복잡한 계층 구조를 갖는 런타임 환경입니다. 실행 엔진은 크게 인터프리터, JIT(Just-In-Time) 컴파일러, 그리고 GC로 구성됩니다.

HotSpot JVM의 JIT는 **C1(Client Compiler)**과 **C2(Server Compiler)**의 계층적 컴파일 방식인 **Tiered Compilation**을 사용합니다. 코드는 처음엔 인터프리터로 실행되다가 호출 빈도(invocation count)와 루프 반복 횟수(back-edge count)를 추적하여 "핫스팟"이 감지되면 C1 → C2 순으로 점진적으로 최적화됩니다.

C2 컴파일러는 **Sea-of-Nodes IR(Intermediate Representation)**을 사용하며 다음과 같은 고급 최적화를 수행합니다.

- **Escape Analysis**: 객체가 메서드 범위를 벗어나지 않는다고 판단되면 힙 대신 스택에 할당(**Stack Allocation**)하거나 아예 객체를 분해해 스칼라 값으로 대체(**Scalar Replacement**)합니다. 이는 GC 압력을 획기적으로 줄입니다.
- **Loop Unrolling**: 루프를 물리적으로 펼쳐 분기 비용을 제거합니다.
- **Inlining**: 메서드 호출을 호출 지점에 직접 삽입하여 스택 프레임 생성 비용을 제거하고, 인라이닝 후 추가 최적화가 가능해집니다. 이는 JIT 최적화에서 가장 중요한 단계입니다.
- **Speculative Optimization + Deoptimization**: 단형성(monomorphic) 가상 호출 등을 가정하고 공격적으로 최적화한 뒤, 가정이 무너지면 **uncommon trap**을 통해 인터프리터 모드로 되돌아갑니다.

```java
// Escape Analysis 예제: 아래 Point 객체는 스택에 할당될 수 있음
public int sumCoords() {
    Point p = new Point(3, 4); // 이 메서드를 벗어나지 않음
    return p.x + p.y;          // JIT가 new Point() 자체를 제거 가능
}
```

### 1.2 클래스 로딩 서브시스템

클래스 로딩은 **Loading → Linking(Verification → Preparation → Resolution) → Initialization** 순으로 이루어집니다.

**Lazy Initialization**이 기본 원칙으로, 클래스는 최초 능동적 사용(Active Use) 시점에만 초기화됩니다. 능동적 사용이란 인스턴스 생성, static 필드 접근, static 메서드 호출, 리플렉션, 서브클래스 초기화, 진입점 클래스 실행 중 하나입니다.

**Preparation 단계**에서는 static 변수에 기본값이 할당되고, **Initialization 단계**에서 `<clinit>` 메서드가 실행되어 실제 초기화 코드가 동작합니다. JVM은 `<clinit>` 실행을 스레드 안전하게 보장합니다. 이를 이용한 것이 바로 **Initialization-on-demand Holder** 패턴입니다.

```java
// 스레드 안전하고 지연 초기화되는 싱글턴
public class Singleton {
    private Singleton() {}

    // Holder 클래스는 getInstance() 최초 호출 시점에만 로딩됨
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## 2. 메모리 모델 및 가비지 컬렉션

### 2.1 Java Memory Model (JMM)

JMM은 JLS(Java Language Specification) 17장에 명세된 공식 메모리 일관성 모델입니다. JMM은 하드웨어의 재정렬(reordering)과 캐시 비일관성 문제를 추상화하고, **happens-before** 관계라는 편리한 개념으로 가시성 보장을 명시합니다.

핵심 happens-before 규칙은 다음과 같습니다. **프로그램 순서 규칙**(단일 스레드 내 순서), **모니터 잠금 규칙**(unlock → lock), **volatile 변수 규칙**(volatile 쓰기 → volatile 읽기), **스레드 시작 규칙**(start() 이전 → 스레드 내 모든 동작), **스레드 종료 규칙**(join() 완료 → join() 이후)이 있습니다. Happens-before는 **이행적(transitive)**이므로 A→B, B→C이면 A→C가 보장됩니다.

`volatile` 키워드는 **가시성(visibility)**과 **순서 보장(ordering)**을 제공하지만 **원자성(atomicity)**은 보장하지 않습니다. 64비트 `long`/`double`의 읽기·쓰기 원자성도 `volatile` 없이는 보장되지 않습니다(JLS §17.7).

```java
// Double-Checked Locking 올바른 구현 — volatile 없으면 JVM 재정렬로 깨짐
public class DCLSingleton {
    private volatile static DCLSingleton instance; // volatile 필수!

    public static DCLSingleton getInstance() {
        if (instance == null) {                    // 1차 체크: 빠른 경로
            synchronized (DCLSingleton.class) {
                if (instance == null) {            // 2차 체크: 스레드 안전
                    instance = new DCLSingleton(); // volatile 쓰기 → happens-before 보장
                }
            }
        }
        return instance;
    }
}
```

`new` 연산은 내부적으로 ① 메모리 할당, ② 생성자 실행, ③ 참조 할당의 3단계인데, JIT가 ①③②로 재정렬할 수 있습니다. `volatile`은 이 재정렬을 막습니다.

### 2.2 가비지 컬렉션 알고리즘 심화

**G1GC (Garbage First)**는 힙을 동일 크기의 **Region**(1~32MB)으로 분할하고, 각 Region에 동적으로 Eden/Survivor/Old/Humongous 역할을 부여합니다. GC의 주요 원칙은 가장 많은 가비지가 있는 Region을 우선 수집한다는 것입니다(hence "Garbage First"). Mixed GC를 통해 Young + 선택된 Old Region을 함께 수집하여 STW(Stop-The-World) 시간을 예측 가능하게 제어합니다.

**ZGC (Z Garbage Collector)**는 Java 15부터 Production-ready로 지정된 Low-Latency GC입니다. **Colored Pointers**(참조 포인터 상위 비트에 GC 메타데이터 인코딩)와 **Load Barriers**를 사용하여 GC 작업의 대부분을 애플리케이션 스레드와 동시에 수행합니다. STW 시간은 힙 크기와 무관하게 수 밀리초 이내입니다. Java 21의 ZGC는 **Generational ZGC**로 발전하여 Young/Old Generation을 분리 관리합니다.

**Shenandoah**는 Red Hat이 개발한 또 다른 Low-Latency GC로 **Brooks Pointers**(간접 참조 레이어)를 이용한 동시 압축(Concurrent Compaction)이 특징입니다.

```bash
# GC 옵션 설정 예시
-XX:+UseG1GC -XX:MaxGCPauseMillis=200   # G1GC, 200ms 목표 정지 시간
-XX:+UseZGC -XX:+ZGenerational           # Generational ZGC (Java 21+)
-Xlog:gc*:file=gc.log:time,uptime,level  # GC 로그 상세 출력
```

**GC 튜닝 방법론**: GC 로그를 분석할 때는 Throughput(처리량), Latency(지연 시간), Footprint(메모리 사용량)의 세 축을 동시에 최적화할 수 없다는 **GC Trilemma**를 인식해야 합니다. Humongous Allocation(Region 50% 이상 차지하는 객체 직접 Old Gen 배치)은 G1에서 예상치 못한 Full GC의 주요 원인입니다.

---

## 3. 동시성과 병렬 프로그래밍

### 3.1 Java Concurrency Utilities 심화

`java.util.concurrent` 패키지는 단순한 유틸리티 모음이 아니라 정교한 동시성 이론의 구현체입니다.

**`AbstractQueuedSynchronizer (AQS)`**는 `ReentrantLock`, `Semaphore`, `CountDownLatch`, `CyclicBarrier`, `ReentrantReadWriteLock` 등 대부분의 동기화 프리미티브의 근간이 되는 프레임워크입니다. AQS는 `int` 타입의 **state** 변수와 **CLH 큐(Craig, Landin, and Hagersten Queue)**의 변형인 FIFO 대기 큐를 사용합니다. 서브클래스는 `tryAcquire()`/`tryRelease()`를 구현하여 state의 의미를 정의합니다.

```java
// 커스텀 이진 래치 (AQS 직접 활용)
public class BinaryLatch extends AbstractQueuedSynchronizer {
    // state == 0: open, state == 1: closed
    public void open() { setState(0); releaseShared(0); }

    @Override
    protected int tryAcquireShared(int ignore) {
        return getState() == 0 ? 1 : -1; // 1: 성공, -1: 대기
    }

    @Override
    protected boolean tryReleaseShared(int ignore) { return true; }

    public void await() throws InterruptedException {
        acquireSharedInterruptibly(0);
    }
}
```

**Lock-Free 알고리즘과 CAS**: `AtomicInteger` 등의 Atomic 클래스는 하드웨어의 **Compare-And-Swap(CAS)** 인스트럭션(`LOCK CMPXCHG` on x86)을 직접 활용합니다. CAS는 낙관적(optimistic) 동시성의 핵심으로, 예상값과 현재값이 같을 때만 업데이트합니다.

**ABA 문제**: CAS는 A→B→A 변경을 감지하지 못합니다. `AtomicStampedReference`가 이를 해결하며 버전 태그를 추가로 확인합니다.

**`StampedLock`**: Java 8에서 도입된 낙관적 읽기(Optimistic Read)를 지원하는 고성능 락입니다. 읽기가 압도적으로 많은 워크로드에서 `ReentrantReadWriteLock`보다 훨씬 우수합니다. 단, 재진입(reentrant)을 지원하지 않으며 잘못 사용하면 스레드 기아(starvation)가 발생할 수 있습니다.

```java
// StampedLock 낙관적 읽기 패턴
private final StampedLock sl = new StampedLock();
private double x, y;

public double distanceFromOrigin() {
    long stamp = sl.tryOptimisticRead();    // 락 없이 스탬프 획득
    double curX = x, curY = y;
    if (!sl.validate(stamp)) {              // 읽는 사이 쓰기 발생?
        stamp = sl.readLock();             // 그렇다면 읽기 락으로 재시도
        try { curX = x; curY = y; }
        finally { sl.unlockRead(stamp); }
    }
    return Math.sqrt(curX * curX + curY * curY);
}
```

### 3.2 ForkJoinPool과 Work-Stealing

`ForkJoinPool`은 분할정복(divide and conquer) 알고리즘에 최적화된 스레드 풀입니다. **Work-Stealing** 알고리즘이 핵심으로, 각 스레드는 자신의 deque에서 작업을 가져가며(LIFO, 캐시 친화적), 다른 스레드의 deque 끝에서 훔쳐올 수도 있습니다(FIFO, 경합 최소화). `RecursiveTask<V>`(결과 반환)와 `RecursiveAction`(결과 없음)을 통해 활용합니다.

`parallelStream()`은 내부적으로 공통 `ForkJoinPool`을 사용합니다. CPU-bound 작업에는 적합하지만 IO-bound 작업에는 부적합하며, 공통 풀이 블로킹되면 다른 곳의 병렬 스트림도 영향받습니다.

### 3.3 CompletableFuture 심화

`CompletableFuture`는 Java 8에서 도입된 비동기 파이프라인 구성 도구입니다. 내부적으로 단방향 링크드 리스트 형태의 Completion 체인을 구성합니다.

```java
// 비동기 처리 파이프라인 구성
CompletableFuture
    .supplyAsync(() -> fetchUserData(userId), ioExecutor) // IO 전용 풀 사용
    .thenApplyAsync(this::enrichWithProfile, cpuExecutor)  // CPU 풀로 전환
    .thenCombine(
        CompletableFuture.supplyAsync(() -> fetchPermissions(userId), ioExecutor),
        (enriched, perms) -> merge(enriched, perms)        // 두 CF 결합
    )
    .exceptionally(ex -> handleError(ex))                  // 예외 처리
    .orTimeout(5, TimeUnit.SECONDS)                        // Java 9+: 타임아웃
    .thenAccept(result -> sendResponse(result));
```

`thenApply` vs `thenApplyAsync`의 차이는 실행 컨텍스트입니다. `thenApply`는 이전 단계를 완료한 스레드에서 동기적으로 실행되는 반면, `thenApplyAsync`는 (기본값으로) `ForkJoinPool.commonPool()`에 별도 제출합니다. IO-bound와 CPU-bound 작업을 분리된 Executor로 라우팅하는 것이 프로덕션 코드의 모범 사례입니다.

---

## 4. Java 타입 시스템 고급

### 4.1 제네릭스 (Generics)의 이면

Java 제네릭은 **타입 소거(Type Erasure)** 방식으로 구현됩니다. 컴파일 타임에 타입 안전성을 검사한 뒤, 바이트코드에서는 타입 파라미터가 상한 경계(또는 `Object`)로 대체됩니다. 이는 C++ 템플릿과 근본적으로 다른 접근입니다.

**공변성(Covariance), 반공변성(Contravariance), 불변성(Invariance)**은 제네릭에서 가장 이해하기 어려운 개념입니다.

Java 배열은 공변적(`String[] is-a Object[]`)이지만 제네릭은 불변적(`List<String> is-NOT-a List<Object>`)입니다. 이는 배열의 공변성이 런타임 `ArrayStoreException`이라는 대가를 치르기 때문입니다.

**PECS(Producer Extends, Consumer Super)** 원칙은 와일드카드를 올바르게 사용하는 지침입니다.

```java
// PECS 원칙 구현
// 생산자(Producer)는 extends: T를 꺼내 쓸 때
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    // src에서 읽고(extends), dest에 쓴다(super)
    for (T item : src) dest.add(item);
}

// 함수형 관점의 공변/반공변
// Function<T, R>에서 입력(T)은 반공변, 출력(R)은 공변
// Kotlin의 in/out이 이를 명시적으로 표현
```

**재귀적 타입 경계(Recursive Type Bound)**는 `Comparable`과 같이 자기 자신과 비교할 때 나타납니다.

```java
// T는 자기 자신과 비교 가능해야 한다
public static <T extends Comparable<T>> T max(List<T> list) { ... }

// 빌더 패턴에서 자기 반환 타입 (Curiously Recurring Template Pattern)
public abstract class Builder<T extends Builder<T>> {
    public T withName(String name) { this.name = name; return self(); }
    protected abstract T self();
}
```

### 4.2 Records, Sealed Classes, Pattern Matching (Java 14~21+)

**Records(Java 16 GA)**는 불변 데이터 집합체의 보일러플레이트를 제거합니다. 컴파일러가 생성자, getter, `equals`, `hashCode`, `toString`을 자동 생성합니다. `@Override`로 compact canonical constructor를 이용해 유효성 검사를 간결하게 추가할 수 있습니다.

**Sealed Classes(Java 17 GA)**는 허용된 서브클래스 집합을 컴파일 타임에 제한합니다. Algebraic Data Type(ADT)의 sum type에 해당하며 Pattern Matching과 결합하면 타입 안전한 exhaustive dispatch가 가능합니다.

```java
// Sealed class + Record + Pattern Matching의 결합
public sealed interface Shape permits Circle, Rectangle, Triangle {}
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}

// Java 21 switch expression with pattern matching (JEP 441)
public double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t  -> 0.5 * t.base() * t.height();
        // sealed이므로 default 불필요 — exhaustiveness 컴파일 시 검증
    };
}
```

이 패턴은 기존 `instanceof` 체인과 비교했을 때 OCP(Open/Closed Principle)를 의도적으로 반전시킨 것입니다. 도메인이 닫혀 있고(새 타입 추가가 드물고) 연산이 열려 있는(새 연산 추가가 잦은) 경우 sealed + pattern matching이, 반대의 경우 다형성(인터페이스)이 적합합니다.

---

## 5. 함수형 프로그래밍과 Stream API

### 5.1 Stream 내부 구현 원리

Stream은 **지연 평가(Lazy Evaluation)**와 **파이프라인 퓨전(Pipeline Fusion)**을 통해 중간 컬렉션 생성 없이 데이터를 처리합니다. 내부적으로 `Spliterator`를 기반으로 하며 `ReferencePipeline` 연산 체인이 구성됩니다.

**Spliterator**는 병렬 스트림의 핵심입니다. `trySplit()`을 통해 데이터 소스를 재귀적으로 분할하고, `SIZED`, `ORDERED`, `SORTED`, `DISTINCT`, `NONNULL`, `IMMUTABLE`, `CONCURRENT`, `SUBSIZED` 등의 특성(characteristics)을 ForkJoinPool에 제공하여 최적화를 가이드합니다.

**Short-circuit 연산** `findFirst()`, `anyMatch()`, `limit()` 등은 전체 소스를 소비하지 않고 조기 종료하므로 무한 스트림(`Stream.iterate`, `Stream.generate`)도 처리 가능합니다.

```java
// 커스텀 Spliterator 구현 (대용량 파일 병렬 처리)
public class LineSpliterator implements Spliterator<String> {
    private final List<String> lines;
    private int current, end;

    @Override
    public Spliterator<String> trySplit() {
        int mid = (current + end) >>> 1; // overflow 안전한 중간값
        if (mid == current) return null; // 너무 작아서 분할 불필요
        LineSpliterator prefix = new LineSpliterator(lines, current, mid);
        current = mid;
        return prefix;
    }

    @Override
    public long estimateSize() { return end - current; }

    @Override
    public int characteristics() {
        return ORDERED | SIZED | SUBSIZED | IMMUTABLE;
    }
}
```

### 5.2 Collector 심화

`Collector`는 `supplier`, `accumulator`, `combiner`, `finisher`, `characteristics` 다섯 요소로 정의됩니다. 병렬 스트림에서 `combiner`는 부분 결과를 병합하는 역할을 담당합니다.

```java
// 커스텀 통계 컬렉터: 평균과 분산을 단일 패스로 계산 (Welford's online algorithm)
Collector<Double, ?, double[]> statsCollector = Collector.of(
    () -> new double[]{0, 0, 0},           // [count, mean, M2]
    (a, v) -> {                            // accumulator: Welford update
        a[0]++;
        double delta = v - a[1];
        a[1] += delta / a[0];
        a[2] += delta * (v - a[1]);
    },
    (a, b) -> {                            // combiner: parallel 병합
        double count = a[0] + b[0];
        double delta = b[1] - a[1];
        double mean = (a[0]*a[1] + b[0]*b[1]) / count;
        double m2 = a[2] + b[2] + delta * delta * (a[0]*b[0]/count);
        return new double[]{count, mean, m2};
    }
);
```

---

## 6. 리플렉션, 어노테이션, 메타프로그래밍

### 6.1 리플렉션의 비용과 대안

리플렉션은 JIT 최적화(인라이닝, 이스케이프 분석)를 방해하고, `AccessibleObject.setAccessible(true)` 호출은 JDK 9+에서 모듈 캡슐화를 우회하므로 강한 캡슐화 위반입니다. 리플렉션 비용을 줄이는 전략으로 **`MethodHandle`(Java 7+)**과 **`LambdaMetafactory`**를 사용하면 JIT가 직접 호출처럼 최적화할 수 있습니다.

```java
// MethodHandle을 통한 리플렉션 대체 (JIT-friendly)
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodType mt = MethodType.methodType(String.class, int.class);
MethodHandle mh = lookup.findVirtual(String.class, "substring", mt);

// invokeExact: 타입 정확히 일치해야 하지만 가장 빠름
String result = (String) mh.invokeExact("Hello, World!", 7);
```

### 6.2 Annotation Processing (APT)

`javax.annotation.processing.AbstractProcessor`를 확장하여 **컴파일 타임**에 코드를 분석하고 소스 파일을 생성합니다. Lombok, MapStruct, Dagger, AutoValue가 이 메커니즘을 사용합니다. Annotation Processing은 리플렉션 없이 타입 안전한 코드를 생성하므로 런타임 오버헤드가 없습니다.

**`@Retention` 정책**: `SOURCE`(컴파일 후 삭제), `CLASS`(바이트코드 포함, 런타임 불가), `RUNTIME`(런타임 리플렉션 접근 가능)로 구분됩니다.

---

## 7. 성능 최적화 및 프로파일링

### 7.1 JMH (Java Microbenchmark Harness)

마이크로벤치마크는 JIT 워밍업, Dead Code Elimination, Constant Folding 등의 JVM 최적화 때문에 잘못된 결과를 내기 쉽습니다. JMH는 이를 방지하는 표준 벤치마크 프레임워크입니다.

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(2)  // JVM 프로세스를 새로 띄워 이전 JIT 상태와 격리
@State(Scope.Thread)
public class StringConcatBenchmark {

    private String str = "hello";

    @Benchmark
    public String stringPlus(Blackhole bh) {
        // Blackhole: JIT가 결과를 버리지 않도록 소비
        return str + " world";
    }

    @Benchmark
    public String stringBuilder() {
        return new StringBuilder(str).append(" world").toString();
    }
}
```

Java 9+에서 `+` 연산자는 `invokedynamic` + `StringConcatFactory`를 통해 최적화되므로 단순 연결에서 `StringBuilder`보다 빠르거나 같을 수 있습니다.

### 7.2 메모리 레이아웃과 캐시 친화성

JVM 객체 헤더는 **Mark Word(8바이트)** + **Class Pointer(4/8바이트)** = 최소 12~16바이트입니다. 필드 접근 패턴을 캐시 라인(64바이트) 단위로 정렬하면 성능이 크게 향상됩니다.

**False Sharing**: 서로 다른 스레드가 같은 캐시 라인을 공유하는 변수를 각각 업데이트하면 불필요한 캐시 무효화가 발생합니다. `@Contended` 어노테이션(JDK 내부용, `-XX:-RestrictContended`로 활성화)을 사용하거나 패딩(padding)으로 해결합니다. `LongAdder`가 이 기법을 사용하여 `AtomicLong`보다 고경합 시나리오에서 월등히 빠릅니다.

### 7.3 Off-Heap 메모리: `sun.misc.Unsafe`와 Foreign Memory API

**Foreign Memory API (Java 22 GA, JEP 454)**는 GC 관리 밖의 메모리를 안전하게 제어할 수 있는 공식 API입니다. `MemorySegment`와 `MemoryLayout`을 통해 C 구조체와 호환되는 메모리 레이아웃을 정의하고, `MemoryLayout.PathElement`로 필드에 접근합니다. 대용량 캐시, 네이티브 라이브러리 연동, Parquet/Arrow 파일 직접 처리 등에 활용됩니다.

---

## 8. 모듈 시스템(JPMS)과 클래스로더

### 8.1 Java Platform Module System (Java 9+)

JPMS(Project Jigsaw)는 `module-info.java`를 통해 모듈 간 의존성과 패키지 가시성을 명시적으로 선언합니다.

```java
// module-info.java
module com.example.service {
    requires com.example.api;           // 의존 모듈
    requires transitive com.example.model; // 전이적 의존
    exports com.example.service.api;    // 외부에 노출할 패키지
    opens com.example.service.impl to  // 리플렉션 허용 (특정 모듈에만)
        com.fasterxml.jackson.databind;
    provides com.example.api.ServiceIF  // ServiceLoader SPI
        with com.example.service.impl.ServiceImpl;
    uses com.example.plugin.PluginIF;   // SPI 소비
}
```

`--add-exports`, `--add-opens` JVM 플래그는 모듈 경계를 런타임에 우회하는 이행 조치이며, 장기적으로는 `module-info.java`에 명시하는 것이 권장됩니다.

### 8.2 클래스로더 계층과 위임 모델

Java 9 이전의 **Bootstrap → Extension → Application** 3계층에서 JPMS 이후 **Bootstrap → Platform → Application**으로 변경되었습니다. 클래스로더 위임 모델은 기본적으로 Parent-First지만 OSGi, 웹 컨테이너(Tomcat), 플러그인 시스템에서는 Child-First로 역전시켜 격리(Isolation)를 구현합니다.

```java
// 동적 격리 클래스로더 (플러그인 시스템)
public class PluginClassLoader extends URLClassLoader {
    public PluginClassLoader(URL[] urls, ClassLoader parent) {
        super(urls, parent);
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException {
        // 플러그인 전용 패키지는 parent보다 먼저 자신에서 찾기
        if (name.startsWith("com.example.plugin.")) {
            Class<?> c = findLoadedClass(name);
            if (c == null) c = findClass(name); // parent 위임 전에 자신 탐색
            if (resolve) resolveClass(c);
            return c;
        }
        return super.loadClass(name, resolve); // 나머지는 parent 위임
    }
}
```

---

## 9. Project Loom과 가상 스레드

### 9.1 Virtual Thread (Java 21 GA, JEP 444)

가상 스레드는 JVM이 관리하는 경량 스레드입니다. OS 스레드(Platform Thread)에 1:1로 매핑하는 기존 방식과 달리, 가상 스레드는 `ForkJoinPool` 기반의 **캐리어 스레드(Carrier Thread)** 풀에 M:N으로 다중화됩니다. 가상 스레드가 블로킹 IO 등으로 대기할 때 JVM이 자동으로 언마운트(unmount)하고 캐리어 스레드는 다른 가상 스레드를 처리합니다.

기존 비동기 코드(`CompletableFuture`, 리액티브)와의 차이는 **구조적 단순성**입니다. 가상 스레드를 사용하면 동기적 스타일로 작성하면서도 비동기 수준의 확장성을 얻을 수 있습니다.

```java
// 가상 스레드 생성 방법
Thread vt = Thread.ofVirtual().name("vt-1").start(() -> {
    // 블로킹 IO도 OK — 캐리어 스레드를 블로킹하지 않음
    String result = httpClient.send(request, BodyHandlers.ofString()).body();
});

// ExecutorService를 통한 대규모 동시 처리
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 1_000_000).forEach(i ->
        executor.submit(() -> processRequest(i))
    );
} // AutoCloseable: 모든 가상 스레드 완료 대기
```

**주의 사항**: `synchronized` 블록에서 블로킹 IO가 발생하면 캐리어 스레드까지 핀닝(pinning)될 수 있습니다. 이 경우 `ReentrantLock`으로 교체해야 합니다. `ThreadLocal` 남용은 메모리 이슈가 될 수 있으며, 수백만 개의 가상 스레드가 각각 `ThreadLocal` 값을 가질 때는 **`ScopedValue`(Java 21 Preview)**가 권장됩니다.

### 9.2 Structured Concurrency (Java 21 Preview, JEP 453)

구조적 동시성은 "관련된 가상 스레드들이 단일 논리 작업 단위를 형성한다"는 원칙입니다. `StructuredTaskScope`를 통해 여러 서브태스크를 병렬 실행하고 하나라도 실패하면 나머지를 자동 취소하는 패턴을 표현적으로 작성할 수 있습니다.

```java
// ShutdownOnFailure: 하나 실패 시 나머지 취소
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User>    user    = scope.fork(() -> fetchUser(userId));
    Subtask<Orders>  orders  = scope.fork(() -> fetchOrders(userId));
    Subtask<Profile> profile = scope.fork(() -> fetchProfile(userId));

    scope.join().throwIfFailed(); // 모두 완료 또는 예외 전파

    return new UserDashboard(user.get(), orders.get(), profile.get());
}
```

---

## 10. 해외 대기업 면접 Q&A 30선

> **대상**: Amazon, Apple, Nvidia, Google, Microsoft, Tesla  
> **레벨**: Senior / Staff Engineer  
> **형태**: 기술 심화 질문 + 모범 답안 + 핵심 설명

---

### Q1. `volatile`과 `synchronized`의 차이를 JMM 관점에서 설명하세요.

**답**: `volatile`은 가시성과 순서 보장을 제공하지만 원자성을 보장하지 않습니다. 단일 변수의 읽기/쓰기는 volatile 쓰기 이전의 모든 동작이 volatile 읽기 이후의 동작에 가시적임을 보장(happens-before)합니다. `synchronized`는 상호 배제(Mutual Exclusion)와 함께 모니터 진입/탈출 시 완전한 메모리 펜스를 형성하여 원자성, 가시성, 순서 모두를 보장합니다. 따라서 복합 연산(check-then-act, read-modify-write)은 반드시 `synchronized` 또는 Atomic 클래스를 사용해야 합니다.

---

### Q2. `HashMap`과 `ConcurrentHashMap`의 내부 구현 차이를 설명하고, Java 8의 개선 사항을 말하세요.

**답**: `HashMap`은 비스레드 안전합니다. 동시 수정 시 무한 루프가 발생할 수 있습니다(Java 7 이하 resize 시). Java 8에서는 버킷 내 연결 리스트가 8개 이상 쌓이면 **Red-Black Tree**로 변환되어 최악 탐색 복잡도가 O(n)에서 O(log n)으로 개선되었습니다.

`ConcurrentHashMap`은 Java 7까지는 세그먼트(Segment) 기반 분할 락을 사용했으나, Java 8부터는 세그먼트를 제거하고 버킷별로 `synchronized` 또는 CAS를 사용합니다. 빈 버킷에 첫 삽입은 CAS로, 충돌 버킷 업데이트는 헤드 노드를 모니터로 `synchronized`합니다. `size()` 계산은 `LongAdder`와 유사한 분산 카운터(CounterCell)로 구현되어 경합을 최소화합니다.

---

### Q3. Java의 `String` 불변성(immutability)이 가져다 주는 구체적인 이점은 무엇인가요?

**답**: 첫째, **스레드 안전성**이 자동으로 확보됩니다. 불변 객체는 공유되더라도 동기화가 필요 없습니다. 둘째, **String Pool(Interning)**이 가능합니다. JVM은 힙 내 String Pool에 리터럴을 캐싱하고, `intern()`을 통해 동일 내용 문자열을 공유하여 메모리를 절약합니다. 셋째, **HashCode 캐싱**이 가능합니다. `String`은 `hashCode`를 최초 계산 후 필드에 캐싱하여 `HashMap` 키로서 효율적입니다. 넷째, **보안**입니다. 클래스 로딩 경로, 파일 경로, 네트워크 연결 등 민감한 문자열이 변경되지 않음을 보장합니다.

---

### Q4. `equals`와 `hashCode`의 계약(contract)을 설명하고, 위반 시 어떤 문제가 발생하나요?

**답**: 계약의 핵심은 `a.equals(b) == true`이면 `a.hashCode() == b.hashCode()`여야 한다는 것입니다(역은 성립 안 해도 됨). 위반 시 `HashMap`, `HashSet` 등 해시 기반 컬렉션이 오동작합니다. `equals`가 true인 두 객체가 다른 버킷에 배치되어 `contains`, `get`이 false/null을 반환합니다. `hashCode`는 구현했지만 `equals`를 구현하지 않은 경우(기본 참조 동등성 유지) 논리적으로 같은 객체가 중복 삽입됩니다. Java의 `Objects.hash(field1, field2, ...)` 또는 Guava `Objects.hashCode`를 활용하거나, IDE/Lombok의 생성 코드를 활용하는 것이 안전합니다.

---

### Q5. Java 8의 `Optional`을 올바르게 사용하는 방법과 안티패턴을 설명하세요.

**답**: `Optional`의 의도는 메서드 반환값이 "있을 수도 없을 수도 있음"을 타입 시스템으로 표현하여 NPE를 방지하는 것입니다. 올바른 사용은 반환 타입으로만 사용하고 `map`, `flatMap`, `filter`, `ifPresentOrElse` 등의 함수형 API로 처리하는 것입니다.

안티패턴으로는 `optional.get()` 직접 호출(확인 없이), 메서드 파라미터나 필드 타입으로 사용(직렬화 불가, 성능 저하), `isPresent()`+`get()` 조합(null 체크와 동일), 컬렉션을 `Optional<List<T>>`로 감싸기(빈 컬렉션이 이미 "없음"을 표현) 등이 있습니다.

---

### Q6. `try-with-resources`의 내부 동작과 예외 억제(suppressed exception)를 설명하세요.

**답**: 컴파일러는 `try-with-resources`를 `try-finally`로 변환합니다. 본문과 `close()` 양쪽에서 예외가 발생하면 본문 예외가 주 예외로 전파되고, `close()` 예외는 `Throwable.addSuppressed()`로 주 예외에 첨부됩니다. `Throwable.getSuppressed()`로 확인할 수 있습니다. 이는 Java 7 이전의 `finally`에서 예외가 발생하면 원본 예외가 소멸되던 문제를 해결합니다. `AutoCloseable`과 `Closeable`의 차이는 `close()` 선언 예외 종류입니다(`Exception` vs `IOException`).

---

### Q7. Java의 메모리 영역(Heap, Stack, Metaspace 등)을 설명하고 각 영역의 특성과 OOM 원인을 설명하세요.

**답**: **Heap**은 GC 관리 대상으로 Young(Eden + S0 + S1) + Old로 구성됩니다. OOM 원인은 메모리 누수, 대용량 객체 생성입니다. **Stack**은 스레드별 프레임(지역 변수, 오퍼랜드 스택)을 저장합니다. StackOverflowError는 과도한 재귀, OutOfMemoryError는 스레드 수 과다(스택 합산 초과) 시 발생합니다. **Metaspace**(Java 8 이후 PermGen 대체)는 클래스 메타데이터를 OS 네이티브 메모리에 저장합니다. OOM 원인은 동적 클래스 생성 누수(CGLib, Groovy, 동적 프록시)입니다. **Code Cache**는 JIT 컴파일된 코드를 저장합니다. 꽉 차면 JIT가 비활성화되어 성능이 급락합니다.

---

### Q8. 인터페이스의 `default` 메서드가 Java 8에서 도입된 이유와 다중 상속 충돌 해결 규칙을 설명하세요.

**답**: 도입 이유는 기존 인터페이스(`Collection`, `List` 등)에 `stream()`, `forEach()` 등 새 메서드를 추가하면서 구현체의 하위 호환성을 유지하기 위함입니다. 다중 충돌 해결 규칙은 세 가지입니다. 첫째, **클래스/수퍼클래스 구현 우선**: 클래스나 수퍼클래스에 동일 시그니처 구현이 있으면 인터페이스 `default`를 이깁니다. 둘째, **서브 인터페이스 우선**: 더 구체적인 인터페이스(상속 계층 하위)가 우선합니다. 셋째, **명시적 선택**: 두 인터페이스가 동등하게 충돌하면 구현 클래스가 반드시 `InterfaceA.super.method()`로 명시적 선택을 해야 합니다. 컴파일 에러가 발생하여 강제됩니다.

---

### Q9. `ThreadLocal`의 메모리 누수 메커니즘을 설명하고 안전하게 사용하는 방법을 제시하세요.

**답**: `Thread` 객체는 `ThreadLocalMap`을 소유하며, 이 맵의 키는 `ThreadLocal`에 대한 **WeakReference**입니다. `ThreadLocal` 객체가 GC되면 키가 null이 되지만, 값(Value)은 강참조로 남아 있습니다. Thread가 종료되지 않는 스레드 풀 환경에서 이 엔트리가 누적되어 누수가 발생합니다. 안전한 사용 패턴은 `try-finally`에서 `threadLocal.remove()`를 호출하는 것입니다. 가상 스레드 환경에서는 수백만 개의 가상 스레드 각각이 `ThreadLocal` 값을 가질 수 있어 메모리 문제가 심각해지므로 `ScopedValue`로의 마이그레이션을 권장합니다.

---

### Q10. Java에서 `Comparable`과 `Comparator`의 차이와 올바른 사용 지침을 설명하세요.

**답**: `Comparable`은 클래스 자신이 자연 순서(Natural Ordering)를 정의하며 `compareTo` 구현이 `equals`와 일관되어야 합니다(일관되지 않으면 `TreeSet`과 `equals` 기반 컬렉션의 동작이 달라집니다). `Comparator`는 외부에서 다양한 정렬 기준을 주입할 때 사용합니다. Java 8에서 `Comparator.comparing()`, `thenComparing()`, `reversed()`, `nullsFirst/Last()`로 선언적 체인 구성이 가능해졌습니다. `compareTo` 구현 시 `substraction(a.val - b.val)` 방식은 정수 오버플로 위험이 있으므로 `Integer.compare(a.val, b.val)`을 사용해야 합니다.

---

### Q11. JVM에서 객체 생성 비용을 낮추는 방법을 설명하세요. (Object Pooling, Value Types 등)

**답**: 객체 풀링은 생성/소멸 비용이 높은 객체(DB 연결, 스레드, 대형 버퍼)에 유효하지만, JVM에서 일반 소형 객체 풀링은 GC 효율(Young Gen 컬렉션이 매우 빠름)보다 풀 관리 비용이 더 클 수 있습니다. 효과적인 방법은 불필요한 boxing/unboxing 제거(Primitive Stream 활용), StringBuilder 재사용, `ThreadLocal` 버퍼 활용입니다.

향후 **Project Valhalla**의 **Value Types(Value Classes)**는 Java에 원시 타입처럼 배열/스택에 인라인 저장되는 사용자 정의 타입을 도입합니다. `int[]`처럼 `Point[]`를 연속 메모리에 배치하면 캐시 효율이 극적으로 향상됩니다.

---

### Q12. Java의 예외 계층 구조를 설명하고 checked/unchecked 예외를 언제 사용해야 하는지 논하세요.

**답**: `Throwable` 하위에 `Error`(JVM 수준 심각한 오류, 복구 불가)와 `Exception`이 있습니다. `Exception` 중 `RuntimeException` 및 그 하위는 Unchecked, 나머지는 Checked입니다.

Checked 예외는 "호출자가 합리적으로 복구할 수 있는 조건"에 사용합니다(`IOException`, `SQLException`). Unchecked는 "프로그래밍 오류"(`NullPointerException`, `IllegalArgumentException`)에 사용합니다. 실무에서는 프레임워크(Spring, Hibernate 등)가 Checked를 Unchecked로 래핑하는 것이 일반화되어 있고, 함수형 인터페이스(`Function<T, R>`)는 체크 예외를 허용하지 않아 스트림/람다와 Checked 예외의 궁합이 나쁩니다. API 설계 시 Checked 남용은 "예외 소화(catch and ignore)"를 유발하므로 절제가 필요합니다.

---

### Q13. `finalize()` 메서드의 문제점과 `Cleaner` API를 설명하세요.

**답**: `finalize()`는 GC가 도달 불가 객체를 처리할 때 JVM 내부 Finalizer 스레드가 호출합니다. 문제점은 다음과 같습니다. 실행 시점이 보장되지 않으며, Finalizer 스레드가 객체를 강참조하여 부활(resurrection)시킬 수 있어 GC를 방해합니다. 또한 Finalizer 큐가 처리 속도보다 쌓이는 속도가 빠르면 OutOfMemoryError가 발생하고, 보안 취약점(Finalizer Attack)도 존재합니다. Java 9에서 `@Deprecated`로 지정되었습니다.

대안으로 Java 9의 **`java.lang.ref.Cleaner`**는 별도 Cleaner 스레드에서 `PhantomReference`를 통해 정리 동작을 수행하며, 객체 부활이 불가능합니다. 그러나 가장 올바른 방법은 `try-with-resources`와 `AutoCloseable`을 통한 명시적 자원 해제입니다.

---

### Q14. Java NIO와 전통적인 IO의 차이, 그리고 Selector를 활용한 Non-Blocking IO를 설명하세요.

**답**: 전통 `java.io`는 스트림 기반, 블로킹, 단방향입니다. `java.nio`는 버퍼(Buffer) 기반, 채널(Channel) 중심, Non-Blocking 지원입니다. `Selector`는 단일 스레드로 다수의 `SelectableChannel`을 멀티플렉싱합니다. `select()` 호출 시 OS의 `epoll`(Linux)/`kqueue`(macOS)/`IOCP`(Windows)에 위임하여 I/O 준비 상태의 채널만 반환합니다. 이것이 Netty, Vert.x 등의 기반입니다.

가상 스레드의 등장으로 블로킹 IO도 확장 가능해졌으나, `Selector` 기반 이벤트 루프는 더 낮은 시스템 콜 오버헤드로 초고성능 네트워크에서 여전히 우위가 있습니다.

---

### Q15. Java 직렬화(Serialization)의 문제점과 대안을 설명하세요.

**답**: 직렬화의 문제점은 보안 취약점(역직렬화 도중 임의 코드 실행 가능, CVE 다수), 성능 저하, 버전 관리 어려움(`serialVersionUID`), 바이트 스트림 형식의 이식성 부족입니다. Josh Bloch은 "Effective Java"에서 새 시스템에 Java 직렬화를 사용하지 말 것을 권고했습니다.

대안으로는 JSON(Jackson, Gson), Protocol Buffers, Avro, Thrift, MessagePack 등이 있습니다. Java 14부터 `ObjectInputFilter`로 역직렬화 필터를 설정하고, JEP 290을 통해 JVM 수준 필터링이 가능합니다. `Serializable` 대신 `Externalizable`로 직렬화를 직접 제어하거나, 직렬화 프록시 패턴을 통해 불변식을 강제할 수 있습니다.

---

### Q16. `java.lang.invoke.MethodHandle` vs 리플렉션 `Method.invoke()`의 성능 차이 이유를 설명하세요.

**답**: `Method.invoke()`는 매 호출마다 접근 검사(access check)를 수행하고(캐싱 가능하지만 제한적), 인수가 `Object[]`로 박싱되며, JIT가 최적화하기 어렵습니다. `MethodHandle`은 접근 검사를 생성 시점(lookup 시)에 한 번만 수행하고, 타입이 명확하여 JIT가 인라이닝 포함 최적화를 수행할 수 있습니다. `MethodHandle`의 `invokeExact`는 원시 타입을 박싱 없이 처리하여 최고 성능을 냅니다. 더 나아가 `LambdaMetafactory`는 `MethodHandle`을 기반으로 첫 호출 시 바이트코드를 동적으로 생성하여 이후 호출이 직접 정적 디스패치처럼 동작합니다.

---

### Q17. Java에서 불변 객체(Immutable Object)를 올바르게 설계하는 방법을 설명하세요.

**답**: 불변 객체의 설계 원칙은 다음과 같습니다. 클래스를 `final`로 선언하거나 생성자를 private으로 만들어 상속을 막습니다. 모든 필드를 `private final`로 선언합니다. 가변 객체를 참조하는 필드는 **방어적 복사(Defensive Copy)**를 통해 반환합니다. 생성자에서도 입력 가변 객체를 복사합니다. 클래스 내에서도 필드를 변경하지 않습니다. `java.time` 패키지의 `LocalDate`, `LocalDateTime` 등이 훌륭한 예시입니다. Record는 이 원칙을 자동으로 적용합니다.

---

### Q18. `ForkJoinPool.commonPool()`을 IO 작업에 사용하면 안 되는 이유를 설명하세요.

**답**: `commonPool`의 스레드 수는 기본적으로 CPU 코어 수 - 1로 고정됩니다. IO 블로킹 작업이 들어오면 스레드가 차단되어 CPU-bound 작업도 처리할 스레드가 없어집니다. 또한 `parallelStream()`도 `commonPool`을 공유하므로 한 IO 작업이 전체 병렬 스트림 처리를 지연시킵니다. 해결책은 IO 전용 Executor(`Executors.newCachedThreadPool()` 또는 커스텀 `ThreadPoolExecutor`)를 별도 생성하고, 또는 Java 21의 가상 스레드 기반 Executor를 사용하는 것입니다.

---

### Q19. `WeakReference`, `SoftReference`, `PhantomReference`의 차이를 설명하고 각각의 활용 사례를 제시하세요.

**답**: **강참조(Strong Reference)**는 GC 대상이 되지 않습니다. **`WeakReference`**는 강참조가 없으면 GC 즉시 수집 대상이 됩니다. `WeakHashMap`(키가 WeakReference)의 내부 구현이나 캐시(키가 사라지면 자동 제거)에 활용됩니다. **`SoftReference`**는 메모리가 부족할 때 GC가 수집합니다. 힙 공간을 사용하는 LRU 캐시, 이미지 캐시에 적합합니다. **`PhantomReference`**는 `get()`이 항상 null을 반환하며 GC가 객체를 수집한 후 `ReferenceQueue`에 등록됩니다. `Cleaner`가 이를 사용하여 파이널라이저 없이 자원 정리를 수행합니다.

---

### Q20. 생산자-소비자 패턴을 Java로 구현하는 방법을 여러 가지 제시하고 각 접근법의 트레이드오프를 설명하세요.

**답**: 가장 기본적인 구현은 `wait()`/`notifyAll()`이며 이해와 유지보수가 어렵습니다. `BlockingQueue`(`ArrayBlockingQueue`, `LinkedBlockingQueue`, `SynchronousQueue`)를 활용하는 것이 실무 표준입니다. `ArrayBlockingQueue`는 고정 크기 배열 기반으로 유한 버퍼를 보장하고, `LinkedBlockingQueue`는 기본 `Integer.MAX_VALUE` 크기라 사실상 무한 버퍼(OOM 위험)이므로 반드시 capacity를 명시해야 합니다. `SynchronousQueue`는 버퍼 크기 0으로 생산자·소비자가 직접 만나 핸드오프합니다(Executor의 기본 큐). 고성능이 필요하면 **Disruptor**(LMAX)의 Ring Buffer를 고려합니다. 리액티브 스트림 환경에서는 `Flux`/`Mono`의 Backpressure 메커니즘이 적합합니다.

---

### Q21. Java에서 동시성 버그(Race Condition, Deadlock, Livelock, Starvation)를 탐지하고 예방하는 방법을 설명하세요.

**답**: **Race Condition**은 두 스레드가 공유 상태를 비원자적으로 읽고 씁니다. 예방은 동기화, Atomic 클래스, 불변 객체입니다. **Deadlock**은 상호 락 대기 사이클입니다. 예방은 전역 락 순서 강제, `tryLock(timeout)` 사용, 락 범위 최소화입니다. **ThreadMXBean**으로 런타임에 데드락을 탐지할 수 있습니다. **Livelock**은 스레드들이 서로의 상태를 관찰하며 무한 재시도합니다. **Starvation**은 우선순위가 낮은 스레드가 자원을 얻지 못합니다. 공정 락(`new ReentrantLock(true)`)으로 해결 가능하지만 성능 저하가 따릅니다. 탐지 도구로는 ThreadDump 분석, JConsole, Java Flight Recorder, async-profiler가 있습니다.

---

### Q22. Java 9의 `Flow` API(Reactive Streams 표준)를 설명하고 Backpressure의 의미를 논하세요.

**답**: `java.util.concurrent.Flow`는 Reactive Streams 사양을 JDK에 포함시킨 것으로 `Publisher`, `Subscriber`, `Subscription`, `Processor` 인터페이스로 구성됩니다. Backpressure는 소비자가 자신이 처리할 수 있는 양만큼만 요청(`subscription.request(n)`)하여 생산자 속도를 제어하는 메커니즘입니다. 이는 버퍼 오버플로와 OOM을 방지합니다. `Flow` API 자체는 인터페이스만 정의하고, RxJava 2+, Project Reactor, Akka Streams 등의 구현체가 실제 연산자를 제공합니다. Spring WebFlux는 Project Reactor 기반으로 구축되어 있습니다.

---

### Q23. GraalVM Native Image가 전통적 JVM과 다른 점과 Java 개발에 주는 영향을 설명하세요.

**답**: GraalVM Native Image는 AOT(Ahead-of-Time) 컴파일로 Java 프로그램을 네이티브 실행 파일로 변환합니다. **Closed-World Assumption**을 전제로 하여 도달 가능한 코드만 포함합니다. 이로 인해 기동 시간이 수십 밀리초로 단축되고 메모리 사용량이 감소합니다. 그러나 리플렉션, 동적 프록시, 리소스 로딩을 빌드 시점에 `reflect-config.json` 등으로 명시해야 합니다. Spring Framework는 Spring AOT 엔진으로 이를 자동화합니다. JIT 최적화가 없으므로 장기 실행 워크로드의 최대 처리량은 JVM보다 낮을 수 있습니다. 서버리스, CLI 도구, 클라우드 네이티브 환경에서 특히 유리합니다.

---

### Q24. `instanceof`의 Pattern Matching(Java 16+)이 기존 방식에 비해 갖는 이점을 설명하세요.

**답**: 기존 `if (obj instanceof String) { String s = (String) obj; ... }` 패턴은 타입 체크와 캐스팅을 두 번 작성해야 합니다. Java 16의 패턴 매칭 `instanceof`는 `if (obj instanceof String s) { ... }`로 체크와 바인딩을 단일 표현식으로 처리합니다. 컴파일러가 범위 흐름 분석(Flow Analysis)을 통해 패턴 변수(`s`)의 범위를 결정합니다. Java 21의 switch 패턴 매칭과 결합하면 sealed class의 완전성 검사와 함께 visitor 패턴을 대체하는 강력한 타입 안전 분기가 가능합니다.

---

### Q25. Java에서 람다 표현식이 내부적으로 어떻게 구현되는지 설명하세요. (vs Anonymous Class)

**답**: 람다는 익명 클래스로 컴파일되지 않습니다. 컴파일러는 람다 바디를 클래스 내의 **private 정적(또는 인스턴스) 메서드**로 디슈거링하고, 호출 지점에 `invokedynamic` 명령을 삽입합니다. JVM이 `invokedynamic`을 처음 실행하면 `LambdaMetafactory`의 부트스트랩 메서드가 호출되어 실제 함수형 인터페이스 구현체를 런타임에 생성합니다. 이 방식은 익명 클래스 대비 클래스 파일 생성을 줄이고, JVM이 최적화 전략을 유연하게 선택할 수 있게 합니다. Capturing lambda(자유 변수를 포획하는 람다)는 포획 변수별 인스턴스를 생성하고, Non-capturing lambda는 싱글턴처럼 재사용될 수 있습니다.

---

### Q26. `Executor` 프레임워크에서 `ThreadPoolExecutor` 파라미터를 올바르게 설정하는 방법을 논하세요.

**답**: `ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler)`의 각 파라미터 상호작용이 중요합니다. 작업이 들어오면 ① corePool이 차지 않으면 새 스레드 생성, ② corePool이 차면 Queue에 삽입 시도, ③ Queue도 차면 maximumPool까지 스레드 생성, ④ maximumPool도 차면 RejectedExecutionHandler 호출 순입니다. 큐 선택이 핵심입니다. `LinkedBlockingQueue(unbounded)`는 3단계로 가지 않아 maximumPool이 의미없고, `SynchronousQueue`는 항상 3단계로 이동합니다. CPU-bound 작업은 `corePoolSize = N_CPU`, IO-bound 작업은 `Little's Law(λ * W)`로 계산합니다. `Executors.newFixedThreadPool()`의 내부는 `LinkedBlockingQueue`로 큐가 무한 증가할 수 있으므로 프로덕션에서 직접 `ThreadPoolExecutor`를 생성하는 것이 안전합니다.

---

### Q27. Java에서 힙 덤프(Heap Dump)를 분석하여 메모리 누수를 찾는 방법론을 설명하세요.

**답**: 힙 덤프는 `jmap -dump:format=b,file=heap.hprof <pid>`, OOM 시 `-XX:+HeapDumpOnOutOfMemoryError` 플래그, JConsole/VisualVM으로 획득합니다. 분석 도구로는 Eclipse MAT(Memory Analyzer Tool), VisualVM, YourKit이 있습니다. MAT의 **Leak Suspects** 리포트는 유지된 힙(Retained Heap)이 큰 객체를 우선 표시합니다. **Dominator Tree**로 어떤 객체가 가장 많은 메모리를 지배(유지)하는지 확인합니다. 전형적인 누수 패턴으로는 static 컬렉션 무한 증가, 이벤트 리스너 미등록 해제, ClassLoader 누수(Metaspace 증가), ThreadLocal 미제거, 캐시에 만료 정책 없음 등이 있습니다.

---

### Q28. Java의 `interface`와 `abstract class`를 언제 사용해야 하는지 설계 원칙과 함께 설명하세요.

**답**: **인터페이스**는 "무엇을 할 수 있는가(capability/role)"를 표현하고 다중 구현이 가능합니다. 타입을 최대한 인터페이스로 선언(Program to an interface)하면 의존성 역전(DIP)이 용이하고 목(Mock) 교체가 쉬워집니다. `default` 메서드로 선택적 구현을 제공합니다.

**추상 클래스**는 "무엇인가(is-a)"를 표현하고 공통 상태와 구현 공유에 적합합니다. 템플릿 메서드 패턴의 기반입니다. Java에서 단일 상속만 허용되므로 상속 계층 설계를 신중하게 해야 합니다.

현대적 설계에서는 **Composition over Inheritance** 원칙에 따라 추상 클래스보다 인터페이스 + 컴포지션을 선호합니다. 추상 클래스는 공통 구현이 실질적으로 중복을 크게 줄이는 경우에만 사용합니다.

---

### Q29. 가상 스레드(Virtual Thread)가 기존 Thread Pool 방식과 비교해 IO-intensive 애플리케이션에서 왜 유리한지 설명하세요.

**답**: 기존 Platform Thread는 OS 스레드와 1:1 매핑되어 스레드당 약 1MB 스택 메모리와 OS 컨텍스트 스위칭 비용을 부담합니다. IO 블로킹 시 스레드가 대기하므로 동시 IO 요청 수 = 스레드 수로 제한됩니다.

가상 스레드는 JVM 스택(최초 ~100바이트, 동적 확장)을 사용하고 수백만 개 생성이 가능합니다. IO 블로킹 시 캐리어 스레드에서 언마운트되고 다른 가상 스레드가 탑재됩니다. 따라서 동시 IO 요청 수가 CPU 코어 수나 스레드 풀 크기에 제한되지 않습니다. Little's Law에 따라 처리량이 `동시요청수 / 평균응답시간`에 비례하므로, 가상 스레드는 분모(응답시간)는 유지하면서 분자(동시요청수)를 수직 확장하는 효과를 냅니다. 단, CPU-bound 작업에서는 코어 수 이상의 가상 스레드가 실제 병렬 실행되지 않으므로 이점이 없습니다.

---

### Q30. Java 21까지의 LTS 버전에서 가장 중요한 JEP(Java Enhancement Proposal)들을 꼽고, 각각이 실제 개발에 미친 영향을 설명하세요.

**답**: Java 8의 **JEP 126(Lambda & Stream API)**은 Java 생태계 전체를 함수형 스타일로 전환하는 가장 중요한 변화였습니다. Java 9의 **JEP 261(JPMS)**은 대규모 애플리케이션의 캡슐화와 보안을 강화했습니다. Java 11의 **JEP 321(HTTP Client)**은 표준 비동기 HTTP API를 제공했습니다. Java 14의 **JEP 359(Records Preview)**와 Java 17의 **JEP 409(Sealed Classes)**는 데이터 중심 모델링을 획기적으로 단순화했습니다. Java 17의 **JEP 356(Enhanced Pseudo-Random Number Generators)**은 알고리즘 교체 가능한 PRNG 인터페이스를 도입했습니다. Java 21의 **JEP 444(Virtual Threads)**는 블로킹 IO 모델을 유지하면서 비동기 수준의 확장성을 제공하는 패러다임 전환입니다. **JEP 441(Pattern Matching for switch)**은 sealed class와 결합해 함수형 언어 수준의 타입 안전 분기를 가능하게 했습니다. **JEP 453(Structured Concurrency Preview)**은 멀티스레드 코드의 가독성과 에러 처리를 혁신적으로 개선합니다.

---

*이 문서는 2024~2025년 Java 생태계 기준으로 작성되었습니다. 면접 준비 시 실제 코드를 직접 작성하고 JVM 플래그를 실험해보며 체화하는 것을 강력히 권장합니다.*
