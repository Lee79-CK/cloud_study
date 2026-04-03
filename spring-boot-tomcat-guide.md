# Spring Boot & 내장 Tomcat 완벽 가이드

> **대상**: 클라우드 엔지니어 및 MLOps/LLMOps 실무자  
> **범위**: 기본 개념 → 고급 설정 → 실전 운영까지

---

## 목차

1. [Part 1: Spring Boot 기본 개념](#part-1-spring-boot-기본-개념)
2. [Part 2: 내장 Tomcat 아키텍처](#part-2-내장-tomcat-아키텍처)
3. [Part 3: 고급 설정과 튜닝](#part-3-고급-설정과-튜닝)
4. [Part 4: 실전 활용 사례](#part-4-실전-활용-사례)
5. [Part 5: 설치 및 운영](#part-5-설치-및-운영)
6. [Part 6: 클라우드 네이티브 운영](#part-6-클라우드-네이티브-운영)
7. [Part 7: 트러블슈팅과 모니터링](#part-7-트러블슈팅과-모니터링)

---

## Part 1: Spring Boot 기본 개념

### 1.1 Spring Boot란 무엇인가

Spring Boot는 Spring Framework를 기반으로 **독립 실행 가능한(stand-alone)** 프로덕션 수준의 애플리케이션을 최소한의 설정으로 만들 수 있게 해주는 프레임워크이다. 기존 Spring의 복잡한 XML 설정과 의존성 관리 문제를 해결하기 위해 2014년에 처음 릴리스되었다.

**핵심 철학**: "Convention over Configuration" (설정보다 관례)

### 1.2 Spring vs Spring Boot 비교

| 구분 | Spring Framework | Spring Boot |
|------|-----------------|-------------|
| 설정 방식 | XML 또는 Java Config 수동 설정 | 자동 설정(Auto-Configuration) |
| 서버 구성 | 외부 WAS(Tomcat, JBoss 등) 별도 설치 | 내장 서버(Tomcat, Jetty, Undertow) 포함 |
| 의존성 관리 | 개별 라이브러리 버전 수동 관리 | Starter 의존성으로 일괄 관리 |
| 배포 방식 | WAR 패키징 → WAS 배포 | JAR 패키징 → `java -jar`로 실행 |
| 초기 구성 시간 | 수 시간~수 일 | 수 분 |

### 1.3 Spring Boot의 4대 핵심 기능

#### 1.3.1 Auto-Configuration (자동 설정)

`@EnableAutoConfiguration` 또는 `@SpringBootApplication` 어노테이션이 활성화되면, Spring Boot는 클래스패스에 존재하는 라이브러리를 감지하여 자동으로 Bean을 구성한다.

```java
@SpringBootApplication  // @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

동작 원리:
- `spring-boot-autoconfigure` 모듈의 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 파일에 자동 설정 클래스 목록이 정의되어 있다.
- `@ConditionalOnClass`, `@ConditionalOnMissingBean` 등의 조건부 어노테이션으로 특정 조건이 충족될 때만 Bean을 생성한다.
- 예를 들어, 클래스패스에 `DataSource` 클래스와 H2 드라이버가 존재하면 자동으로 인메모리 DataSource를 구성한다.

#### 1.3.2 Starter 의존성

Starter는 관련 라이브러리를 묶어놓은 **메타 의존성**이다. 버전 충돌 없이 필요한 라이브러리 세트를 한 번에 가져올 수 있다.

```xml
<!-- 웹 애플리케이션 개발에 필요한 모든 것 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

주요 Starter 목록:

- `spring-boot-starter-web`: Spring MVC + 내장 Tomcat + Jackson
- `spring-boot-starter-data-jpa`: Spring Data JPA + Hibernate
- `spring-boot-starter-security`: Spring Security
- `spring-boot-starter-actuator`: 모니터링 및 관리 엔드포인트
- `spring-boot-starter-test`: JUnit 5 + Mockito + Spring Test
- `spring-boot-starter-webflux`: 리액티브 웹 (Netty 기반)

#### 1.3.3 Externalized Configuration (외부화된 설정)

Spring Boot는 `application.properties` 또는 `application.yml` 파일을 통해 애플리케이션 설정을 외부화한다. 설정 값은 다음과 같은 우선순위로 로딩된다 (높은 순서부터):

1. 커맨드라인 인수 (`--server.port=9090`)
2. OS 환경 변수 (`SERVER_PORT=9090`)
3. `application-{profile}.yml`
4. `application.yml`
5. `@PropertySource`로 지정된 파일
6. 기본값

```yaml
# application.yml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/mydb
    username: ${DB_USER:admin}
    password: ${DB_PASS:secret}
```

#### 1.3.4 Spring Boot Actuator

프로덕션 환경에서 애플리케이션을 모니터링하고 관리하기 위한 내장 엔드포인트를 제공한다.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env
  endpoint:
    health:
      show-details: when-authorized
  metrics:
    export:
      prometheus:
        enabled: true
```

주요 Actuator 엔드포인트:

- `/actuator/health`: 애플리케이션 헬스체크 (DB, 디스크, 커스텀 인디케이터)
- `/actuator/metrics`: Micrometer 기반 메트릭 (JVM, HTTP, 커스텀)
- `/actuator/prometheus`: Prometheus 형식 메트릭 노출
- `/actuator/env`: 환경 변수 및 프로퍼티 조회
- `/actuator/threaddump`: 스레드 덤프
- `/actuator/heapdump`: 힙 덤프 (바이너리)

---

## Part 2: 내장 Tomcat 아키텍처

### 2.1 내장 Tomcat이란

전통적인 Java 웹 개발에서는 Tomcat을 독립적으로 설치하고, 빌드된 WAR 파일을 Tomcat의 `webapps` 디렉토리에 배포하는 방식을 사용했다. Spring Boot는 이 패러다임을 뒤집어 **Tomcat을 애플리케이션 내부에 라이브러리로 포함**시킨다.

```
[전통 방식]                          [Spring Boot 방식]
┌─────────────────┐                 ┌─────────────────┐
│   Tomcat (WAS)  │                 │   Application   │
│  ┌───────────┐  │                 │  ┌───────────┐  │
│  │  WAR App  │  │        →        │  │  Tomcat   │  │
│  └───────────┘  │                 │  │ (embedded) │  │
│  ┌───────────┐  │                 │  └───────────┘  │
│  │  WAR App  │  │                 │                 │
│  └───────────┘  │                 │  → java -jar    │
└─────────────────┘                 └─────────────────┘
   외부 WAS 관리 필요                   단일 JAR로 실행
```

### 2.2 내장 Tomcat의 구조

Tomcat의 아키텍처는 계층적으로 구성된다:

```
Server (최상위 컨테이너)
 └── Service (서비스 그룹)
      ├── Connector (HTTP/AJP 커넥터) ──── 네트워크 요청 수신
      └── Engine (서블릿 엔진)
           └── Host (가상 호스트)
                └── Context (웹 애플리케이션 = 1개의 서블릿 컨텍스트)
                     └── Wrapper (개별 서블릿 래퍼)
                          └── Servlet (DispatcherServlet 등)
```

각 구성 요소의 역할:

- **Server**: Tomcat 인스턴스 전체를 대표하는 최상위 컨테이너. 생명주기 관리를 담당한다.
- **Service**: Connector와 Engine을 묶는 논리적 그룹이다.
- **Connector**: 클라이언트로부터 네트워크 요청을 수신하고, 프로토콜(HTTP/1.1, HTTP/2, AJP)을 처리한다. NIO(Non-blocking I/O) 기반의 `NioEndpoint`가 기본으로 사용된다.
- **Engine**: 모든 요청을 처리하는 서블릿 엔진이다. 요청을 적절한 Host로 라우팅한다.
- **Host**: 가상 호스트 단위를 나타낸다. 내장 Tomcat에서는 보통 `localhost` 하나만 사용한다.
- **Context**: 하나의 웹 애플리케이션을 나타낸다. Spring Boot에서는 DispatcherServlet이 등록된 단일 Context를 사용한다.

### 2.3 요청 처리 흐름 (Request Lifecycle)

HTTP 요청이 Spring Boot 애플리케이션에 도달했을 때의 전체 처리 흐름:

```
1. 클라이언트 HTTP 요청
       │
2. NioEndpoint (Acceptor 스레드가 소켓 수락)
       │
3. Poller 스레드 (NIO Selector로 I/O 이벤트 감지)
       │
4. Worker 스레드풀에서 스레드 할당
       │
5. Http11Processor (HTTP 프로토콜 파싱)
       │
6. CoyoteAdapter (Tomcat 내부 Request/Response 객체 생성)
       │
7. Engine → Host → Context 파이프라인 통과
       │
8. Filter Chain 실행 (CharacterEncodingFilter, CorsFilter 등)
       │
9. DispatcherServlet.doDispatch()
       │
10. HandlerMapping → HandlerAdapter → Controller 메서드 실행
       │
11. ViewResolver 또는 HttpMessageConverter로 응답 생성
       │
12. Filter Chain 역순 통과
       │
13. 클라이언트에 HTTP 응답 전송
```

### 2.4 내장 Tomcat의 스레드 모델

```
┌─────────────────────────────────────────────────┐
│                  NioEndpoint                     │
│                                                  │
│  [Acceptor Thread] ─→ ServerSocketChannel.accept │
│         │                                        │
│  [Poller Thread(s)] ─→ Selector.select()         │
│         │                (NIO 이벤트 루프)         │
│         ▼                                        │
│  ┌──────────────────────────┐                    │
│  │   Worker Thread Pool     │                    │
│  │  (기본: 최소 10, 최대 200)  │                    │
│  │                          │                    │
│  │  Thread-1: 요청 A 처리 중  │                    │
│  │  Thread-2: 요청 B 처리 중  │                    │
│  │  Thread-3: 대기 중         │                    │
│  │  ...                     │                    │
│  └──────────────────────────┘                    │
└─────────────────────────────────────────────────┘
```

- **Acceptor Thread**: 새 TCP 연결을 수락(accept)하고 Poller에게 전달한다. 보통 1개 스레드이다.
- **Poller Thread**: Java NIO Selector를 사용하여 소켓의 읽기/쓰기 가능 여부를 감시한다. I/O 이벤트가 발생하면 Worker 스레드에 작업을 위임한다.
- **Worker Thread Pool**: 실제 HTTP 요청을 처리한다. 기본 최대 200개이며, 모든 스레드가 사용 중이면 요청은 대기 큐(`acceptCount`)에서 대기한다.

### 2.5 내장 서버 비교: Tomcat vs Jetty vs Undertow

| 특성 | Tomcat | Jetty | Undertow |
|------|--------|-------|----------|
| 기본 제공 | Spring Boot 기본 | 수동 교체 필요 | 수동 교체 필요 |
| I/O 모델 | NIO (기본) / NIO2 | NIO / HTTP/2 네이티브 | XNIO (NIO 기반) |
| 메모리 사용량 | 중간 | 낮음 | 가장 낮음 |
| HTTP/2 지원 | 지원 (설정 필요) | 네이티브 지원 | 네이티브 지원 |
| WebSocket | 지원 | 지원 | 지원 |
| 적합한 시나리오 | 범용, 호환성 중시 | 경량 마이크로서비스 | 고성능, 리액티브 |

내장 서버 교체 방법 (Undertow로 변경하는 예시):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

---

## Part 3: 고급 설정과 튜닝

### 3.1 Tomcat 커넥터 튜닝

```yaml
server:
  tomcat:
    # ── 스레드 풀 설정 ──
    threads:
      max: 200          # Worker 스레드 최대 수 (기본: 200)
      min-spare: 10     # 유휴 상태 유지 최소 스레드 수 (기본: 10)
    
    # ── 연결 설정 ──
    max-connections: 8192       # 동시 연결 최대 수 (기본: 8192)
    accept-count: 100           # 대기 큐 크기 (기본: 100)
    connection-timeout: 20000   # 연결 타임아웃 ms (기본: 20000)
    
    # ── Keep-Alive 설정 ──
    keep-alive-timeout: 20000   # Keep-Alive 타임아웃 ms
    max-keep-alive-requests: 100  # Keep-Alive 연결당 최대 요청 수
    
    # ── 요청 크기 제한 ──
    max-http-form-post-size: 2MB
    max-swallow-size: 2MB
    
    # ── 프로토콜 설정 ──
    uri-encoding: UTF-8
    redirect-context-root: false
```

#### 스레드 풀 사이징 공식

```
최적 스레드 수 = (동시 요청 수) × (1 + 대기시간/처리시간)

예시:
- 동시 요청: 100
- 평균 처리 시간: 50ms
- 평균 I/O 대기 시간: 150ms (DB 쿼리, 외부 API 등)
- 최적 스레드 수 = 100 × (1 + 150/50) = 400
```

주의: 스레드 수를 무조건 늘리면 컨텍스트 스위칭 오버헤드가 증가한다. CPU 코어 수, 메모리, 요청 특성을 고려하여 부하 테스트로 최적값을 찾아야 한다.

### 3.2 프로그래밍 방식의 Tomcat 커스터마이징

`application.yml`로 불가능한 세밀한 설정은 `WebServerFactoryCustomizer`를 사용한다.

```java
@Component
public class TomcatCustomizer implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.addConnectorCustomizers(connector -> {
            // HTTP/1.1 프로토콜 핸들러 설정
            Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
            protocol.setMaxThreads(300);
            protocol.setMinSpareThreads(20);
            protocol.setConnectionTimeout(15000);
            protocol.setMaxKeepAliveRequests(200);
            
            // 압축 설정
            connector.setProperty("compression", "on");
            connector.setProperty("compressibleMimeType",
                "text/html,text/xml,text/plain,text/css,application/json,application/javascript");
            connector.setProperty("compressionMinSize", "1024");
        });

        // Access Log 설정
        factory.addContextCustomizers(context -> {
            context.setReloadable(false);
            context.setUseHttpOnly(true);
        });
    }
}
```

### 3.3 SSL/TLS 설정

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASS}
    key-store-type: PKCS12
    key-alias: myapp
    # TLS 버전 제한 (보안 강화)
    protocol: TLS
    enabled-protocols: TLSv1.2,TLSv1.3
    # 약한 암호 스위트 비활성화
    ciphers: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

HTTP → HTTPS 리다이렉트를 위한 추가 커넥터:

```java
@Bean
public WebServerFactoryCustomizer<TomcatServletWebServerFactory> httpsRedirect() {
    return factory -> factory.addAdditionalTomcatConnectors(createHttpConnector());
}

private Connector createHttpConnector() {
    Connector connector = new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
    connector.setScheme("http");
    connector.setPort(8080);
    connector.setSecure(false);
    connector.setRedirectPort(8443);
    return connector;
}
```

### 3.4 HTTP/2 활성화

```yaml
server:
  http2:
    enabled: true   # HTTP/2 활성화 (SSL 필요)
```

HTTP/2의 이점: 멀티플렉싱(단일 TCP 연결에서 여러 요청/응답 동시 처리), 헤더 압축(HPACK), 서버 푸시. 특히 ML 모델 서빙 API에서 다수의 동시 추론 요청을 처리할 때 유리하다.

### 3.5 Graceful Shutdown (우아한 종료)

```yaml
server:
  shutdown: graceful          # 우아한 종료 활성화

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 최대 대기 시간
```

동작 방식:
1. SIGTERM 시그널 수신
2. 새로운 요청 수락 중단 (Connector 일시 중지)
3. 진행 중인 요청이 완료될 때까지 대기 (최대 30초)
4. 타임아웃 초과 시 강제 종료
5. Spring ApplicationContext 종료 (Bean destroy 콜백 실행)

이는 쿠버네티스 환경에서 Rolling Update 시 요청 유실을 방지하는 데 필수적이다.

### 3.6 Gzip 압축

```yaml
server:
  compression:
    enabled: true
    min-response-size: 1024          # 1KB 이상만 압축
    mime-types:
      - application/json
      - application/xml
      - text/html
      - text/xml
      - text/plain
      - text/css
      - application/javascript
```

### 3.7 JVM 튜닝 (Tomcat 성능에 직결)

```bash
java -jar myapp.jar \
  -Xms512m \                    # 초기 힙 크기
  -Xmx2g \                      # 최대 힙 크기
  -XX:+UseG1GC \                 # G1 GC 사용
  -XX:MaxGCPauseMillis=200 \     # GC 일시정지 목표
  -XX:+UseStringDeduplication \  # 문자열 중복 제거
  -XX:MetaspaceSize=256m \       # 메타스페이스 초기 크기
  -XX:MaxMetaspaceSize=512m \    # 메타스페이스 최대 크기
  -Djava.security.egd=file:/dev/./urandom  # 빠른 난수 생성
```

---

## Part 4: 실전 활용 사례

### 4.1 ML 모델 서빙 API

LLMOps/MLOps 환경에서 모델 추론 엔드포인트를 Spring Boot로 구축하는 패턴:

```java
@RestController
@RequestMapping("/api/v1/predict")
public class ModelServingController {

    private final ModelInferenceService inferenceService;

    @PostMapping
    public ResponseEntity<PredictionResponse> predict(
            @Valid @RequestBody PredictionRequest request) {
        
        PredictionResponse response = inferenceService.predict(request);
        return ResponseEntity.ok(response);
    }

    @PostMapping("/batch")
    public ResponseEntity<List<PredictionResponse>> batchPredict(
            @Valid @RequestBody List<PredictionRequest> requests) {
        
        List<PredictionResponse> responses = requests.parallelStream()
            .map(inferenceService::predict)
            .collect(Collectors.toList());
        return ResponseEntity.ok(responses);
    }

    // 비동기 추론 (대용량 모델 또는 긴 처리시간)
    @PostMapping("/async")
    public ResponseEntity<AsyncJobResponse> asyncPredict(
            @Valid @RequestBody PredictionRequest request) {
        
        String jobId = inferenceService.submitAsync(request);
        return ResponseEntity.accepted()
            .body(new AsyncJobResponse(jobId, "/api/v1/predict/status/" + jobId));
    }
}
```

Tomcat 설정 최적화 (모델 서빙 특화):

```yaml
server:
  tomcat:
    threads:
      max: 50          # 모델 추론은 CPU/GPU 집약적이므로 스레드를 줄임
      min-spare: 5
    max-connections: 200
    accept-count: 50
    connection-timeout: 60000   # 추론 시간을 고려해 타임아웃 확대
  servlet:
    encoding:
      charset: UTF-8
      force: true
```

### 4.2 API Gateway 패턴

```java
@Configuration
public class GatewayConfig {

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> gatewayCustomizer() {
        return factory -> {
            factory.addConnectorCustomizers(connector -> {
                Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
                // API Gateway는 많은 동시 연결을 처리해야 함
                protocol.setMaxThreads(500);
                protocol.setMinSpareThreads(50);
                protocol.setMaxKeepAliveRequests(500);
                protocol.setKeepAliveTimeout(30000);
            });
        };
    }
}
```

### 4.3 파일 업로드 서비스

```yaml
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 100MB
      max-request-size: 100MB
      file-size-threshold: 10MB   # 이 크기 초과 시 디스크에 임시 저장

server:
  tomcat:
    max-http-form-post-size: 100MB
    max-swallow-size: -1          # 제한 없음
```

### 4.4 WebSocket을 활용한 실시간 통신

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(modelStatusHandler(), "/ws/model-status")
                .setAllowedOrigins("*");
    }

    @Bean
    public WebSocketHandler modelStatusHandler() {
        return new ModelStatusWebSocketHandler();
    }
}
```

```yaml
server:
  tomcat:
    # WebSocket 연결은 오래 유지되므로 max-connections 확보 필요
    max-connections: 10000
    threads:
      max: 200
```

### 4.5 Multi-Connector 구성 (관리용 포트 분리)

```java
@Configuration
public class MultiPortConfig {

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> multiPort() {
        return factory -> {
            // 메인 커넥터: 8080 (비즈니스 트래픽)
            // 기본 커넥터가 이미 8080으로 설정됨
            
            // 관리용 커넥터: 9090 (내부 네트워크에서만 접근)
            Connector mgmtConnector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
            mgmtConnector.setPort(9090);
            mgmtConnector.setProperty("maxThreads", "10");
            factory.addAdditionalTomcatConnectors(mgmtConnector);
        };
    }
}
```

---

## Part 5: 설치 및 운영

### 5.1 프로젝트 생성

#### Spring Initializr 사용 (권장)

```bash
# curl로 프로젝트 생성
curl https://start.spring.io/starter.tgz \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.3.0 \
  -d baseDir=my-app \
  -d groupId=com.example \
  -d artifactId=my-app \
  -d dependencies=web,actuator,devtools,prometheus \
  -d javaVersion=21 \
  | tar -xzvf -
```

또는 https://start.spring.io 에서 GUI로 생성 가능하다.

#### 최소 pom.xml 구성

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <layers>
                        <enabled>true</enabled>  <!-- Docker 레이어 최적화 -->
                    </layers>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 5.2 빌드와 실행

```bash
# 빌드
./mvnw clean package -DskipTests

# 실행 (기본)
java -jar target/my-app-0.0.1-SNAPSHOT.jar

# 프로필 지정 실행
java -jar target/my-app-0.0.1-SNAPSHOT.jar \
  --spring.profiles.active=prod

# JVM 옵션과 함께 실행
java -Xms512m -Xmx2g \
  -XX:+UseG1GC \
  -jar target/my-app-0.0.1-SNAPSHOT.jar \
  --server.port=9090
```

### 5.3 Profile 기반 환경 분리

```
src/main/resources/
├── application.yml              # 공통 설정
├── application-dev.yml          # 개발 환경
├── application-staging.yml      # 스테이징 환경
├── application-prod.yml         # 운영 환경
└── application-test.yml         # 테스트 환경
```

```yaml
# application-prod.yml
server:
  port: 8080
  tomcat:
    threads:
      max: 200
      min-spare: 20
    max-connections: 8192
    accept-count: 100
  shutdown: graceful
  compression:
    enabled: true

spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}
    hikari:
      maximum-pool-size: 30
      minimum-idle: 10
      connection-timeout: 5000

logging:
  level:
    root: WARN
    com.example: INFO
  file:
    name: /var/log/myapp/application.log
  logback:
    rollingpolicy:
      max-file-size: 100MB
      max-history: 30
```

### 5.4 systemd 서비스 등록 (Linux)

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Spring Boot Application
After=network.target

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/java \
  -Xms512m -Xmx2g \
  -XX:+UseG1GC \
  -jar /opt/myapp/my-app.jar \
  --spring.profiles.active=prod
ExecStop=/bin/kill -TERM $MAINPID
SuccessExitStatus=143
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

# 보안 강화
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/myapp /opt/myapp/data

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
sudo journalctl -u myapp -f   # 로그 확인
```

---

## Part 6: 클라우드 네이티브 운영

### 6.1 Docker 컨테이너화

#### 멀티 스테이지 Dockerfile (최적화)

```dockerfile
# Stage 1: 빌드
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /workspace

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
RUN ./mvnw dependency:go-offline -B

COPY src src
RUN ./mvnw package -DskipTests && \
    java -Djarmode=layertools -jar target/*.jar extract --destination extracted

# Stage 2: 실행 (레이어드 JAR)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# 보안: 비루트 사용자
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 레이어 순서: 변경 빈도가 낮은 것부터 (Docker 캐시 활용)
COPY --from=builder /workspace/extracted/dependencies/ ./
COPY --from=builder /workspace/extracted/spring-boot-loader/ ./
COPY --from=builder /workspace/extracted/snapshot-dependencies/ ./
COPY --from=builder /workspace/extracted/application/ ./

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

핵심 포인트:
- **Layered JAR**: `spring-boot-maven-plugin`의 `layers` 옵션으로 의존성 레이어를 분리하여 Docker 빌드 캐시를 극대화한다. 코드만 변경 시 전체 의존성을 다시 복사하지 않는다.
- **`-XX:+UseContainerSupport`**: JVM이 컨테이너의 CPU/메모리 제한을 인식하도록 한다 (Java 10+에서 기본 활성화).
- **`-XX:MaxRAMPercentage=75.0`**: 컨테이너 메모리 제한의 75%를 힙에 할당한다. 나머지는 메타스페이스, 네이티브 메모리, 스레드 스택 등에 사용된다.

#### Spring Boot의 Cloud Native Buildpacks

Dockerfile 없이 최적화된 컨테이너 이미지를 빌드할 수 있다:

```bash
# Maven
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest

# Gradle
./gradlew bootBuildImage --imageName=myapp:latest
```

### 6.2 Kubernetes 배포

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0     # 무중단 배포
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      terminationGracePeriodSeconds: 60  # graceful shutdown 대기
      containers:
        - name: myapp
          image: myregistry/myapp:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
            - name: JAVA_TOOL_OPTIONS
              value: >-
                -XX:MaxRAMPercentage=75.0
                -XX:+UseG1GC
                -XX:MaxGCPauseMillis=200
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          
          # 스타트업 프로브: 초기 기동 확인
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30     # 최대 160초(10+5×30) 대기
          
          # Liveness 프로브: 애플리케이션 생존 확인
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            periodSeconds: 10
            failureThreshold: 3
          
          # Readiness 프로브: 트래픽 수신 준비 확인
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            periodSeconds: 5
            failureThreshold: 3

          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]  # 서비스 디스커버리 반영 대기

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

#### Kubernetes Health Group 설정 (Spring Boot 측)

```yaml
# application-prod.yml
management:
  endpoint:
    health:
      probes:
        enabled: true     # liveness/readiness 그룹 활성화
      group:
        liveness:
          include: livenessState,ping
        readiness:
          include: readinessState,db,redis
```

### 6.3 HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_server_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

---

## Part 7: 트러블슈팅과 모니터링

### 7.1 자주 발생하는 문제와 해결

#### 문제 1: "Connection reset by peer" / 502 Bad Gateway

원인: Tomcat의 `keep-alive-timeout` < 로드밸런서의 `idle timeout`

```yaml
# Tomcat의 Keep-Alive가 LB보다 길어야 함
# AWS ALB idle timeout이 60초라면:
server:
  tomcat:
    keep-alive-timeout: 65000    # 65초 (LB보다 길게)
```

#### 문제 2: 스레드 고갈 (Thread Exhaustion)

증상: 응답 지연 후 "Connection refused" 발생

```bash
# 실시간 스레드 확인
curl http://localhost:8080/actuator/threaddump | jq '.threads | length'

# Tomcat 스레드풀 메트릭 확인
curl http://localhost:8080/actuator/metrics/tomcat.threads.busy
curl http://localhost:8080/actuator/metrics/tomcat.threads.current
curl http://localhost:8080/actuator/metrics/tomcat.threads.config.max
```

해결: 스레드 수 증가보다 근본 원인(느린 DB 쿼리, 외부 API 타임아웃 미설정)을 먼저 해결해야 한다.

```java
// 외부 API 호출 시 반드시 타임아웃 설정
@Bean
public RestClient restClient(RestClient.Builder builder) {
    return builder
        .requestFactory(new JdkClientHttpRequestFactory(
            HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(3))
                .build()))
        .build();
}
```

#### 문제 3: OOM (Out of Memory)

```bash
# 힙 덤프 자동 생성 설정
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/myapp/heapdump.hprof \
     -jar myapp.jar
```

#### 문제 4: 느린 시작 시간

```yaml
spring:
  jmx:
    enabled: false                    # JMX 비활성화 (불필요 시)
  main:
    lazy-initialization: true         # 지연 초기화 (첫 요청 시 Bean 생성)
```

Spring Boot 3.x에서는 **GraalVM Native Image**로 시작 시간을 수십 ms로 줄일 수 있다:

```bash
./mvnw -Pnative native:compile
./target/my-app   # ~50ms 이내 시작
```

### 7.2 모니터링 스택 구성

```
┌──────────┐     ┌─────────────┐     ┌───────────┐     ┌─────────┐
│ Spring   │────▶│ Prometheus  │────▶│ Grafana   │────▶│  Alert  │
│ Boot App │     │ (수집/저장)  │     │ (시각화)   │     │ Manager │
│ /metrics │     └─────────────┘     └───────────┘     └─────────┘
└──────────┘
     │
     ▼
┌──────────┐     ┌─────────────┐
│ Logback  │────▶│ ELK / Loki  │
│ (로그)    │     │ (로그 집계)   │
└──────────┘     └─────────────┘
```

#### 핵심 커스텀 메트릭 등록

```java
@Component
public class InferenceMetrics {

    private final Counter inferenceCounter;
    private final Timer inferenceTimer;
    private final DistributionSummary batchSizeSummary;

    public InferenceMetrics(MeterRegistry registry) {
        this.inferenceCounter = Counter.builder("model.inference.total")
            .description("Total inference requests")
            .tag("model", "gpt-serving")
            .register(registry);

        this.inferenceTimer = Timer.builder("model.inference.duration")
            .description("Inference latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(registry);

        this.batchSizeSummary = DistributionSummary.builder("model.inference.batch_size")
            .description("Batch size distribution")
            .register(registry);
    }

    public <T> T recordInference(Supplier<T> task) {
        inferenceCounter.increment();
        return inferenceTimer.record(task);
    }
}
```

### 7.3 Grafana 대시보드에서 모니터링할 핵심 지표

JVM 메트릭:
- `jvm_memory_used_bytes` / `jvm_memory_max_bytes` (힙 사용률)
- `jvm_gc_pause_seconds` (GC 일시정지 시간)
- `jvm_threads_live_threads` (활성 스레드 수)
- `process_cpu_usage` (CPU 사용률)

Tomcat 메트릭:
- `tomcat_threads_busy_threads` / `tomcat_threads_config_max_threads` (스레드풀 사용률)
- `tomcat_connections_current_connections` (현재 연결 수)
- `tomcat_sessions_active_current_sessions` (활성 세션 수)

HTTP 메트릭:
- `http_server_requests_seconds_count` (요청 수 - rate)
- `http_server_requests_seconds_sum` (총 처리 시간)
- `http_server_requests_seconds_max` (최대 응답 시간)
- 상태 코드별 분류: `{status="200"}`, `{status="500"}`

### 7.4 운영 체크리스트

```
[사전 배포]
☐ Profile이 prod로 설정되어 있는가
☐ 민감 정보가 환경 변수 또는 Vault로 주입되는가
☐ Graceful shutdown이 활성화되어 있는가
☐ Health check 엔드포인트가 올바르게 구성되어 있는가
☐ 로깅 레벨이 적절한가 (DEBUG → INFO/WARN)
☐ DevTools 의존성이 제거되었는가

[성능]
☐ 스레드풀 크기가 부하 테스트 결과에 맞게 설정되었는가
☐ DB 커넥션 풀이 적절히 설정되었는가
☐ 외부 API 호출에 타임아웃이 설정되어 있는가
☐ Gzip 압축이 활성화되어 있는가
☐ JVM 힙 크기가 컨테이너 메모리 제한에 맞는가

[보안]
☐ Actuator 엔드포인트 접근이 제한되어 있는가
☐ TLS가 활성화되어 있는가 (또는 앞단 LB에서 TLS 종료)
☐ CORS 정책이 올바르게 설정되어 있는가
☐ 보안 헤더가 설정되어 있는가 (HSTS, CSP, X-Frame-Options)
☐ 불필요한 포트/엔드포인트가 비활성화되어 있는가

[모니터링]
☐ Prometheus 메트릭이 노출되고 수집되고 있는가
☐ 로그가 중앙 집계 시스템으로 전송되는가
☐ Alert 규칙이 설정되어 있는가 (5xx 급증, 응답 지연 등)
☐ 힙 덤프 자동 생성이 설정되어 있는가
```

---

## 부록: 빠른 참조

### application.yml 전체 예시 (프로덕션)

```yaml
server:
  port: 8080
  shutdown: graceful
  compression:
    enabled: true
    min-response-size: 1024
    mime-types: application/json,text/html,text/css,application/javascript
  tomcat:
    threads:
      max: 200
      min-spare: 20
    max-connections: 8192
    accept-count: 100
    connection-timeout: 20000
    keep-alive-timeout: 65000
    max-keep-alive-requests: 200
    uri-encoding: UTF-8
    accesslog:
      enabled: true
      directory: /var/log/myapp
      pattern: "%h %l %u %t \"%r\" %s %b %D"
  http2:
    enabled: true

spring:
  application:
    name: my-app
  lifecycle:
    timeout-per-shutdown-phase: 30s
  jackson:
    serialization:
      write-dates-as-timestamps: false
    default-property-inclusion: non_null

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true

logging:
  level:
    root: WARN
    com.example: INFO
    org.springframework.web: WARN
    org.apache.tomcat: WARN
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
```

### 주요 Tomcat 프로퍼티 요약

| 프로퍼티 | 기본값 | 설명 |
|---------|--------|------|
| `server.tomcat.threads.max` | 200 | Worker 스레드 최대 수 |
| `server.tomcat.threads.min-spare` | 10 | 유휴 스레드 최소 수 |
| `server.tomcat.max-connections` | 8192 | 최대 동시 연결 수 |
| `server.tomcat.accept-count` | 100 | 연결 대기 큐 크기 |
| `server.tomcat.connection-timeout` | 20000ms | 연결 타임아웃 |
| `server.tomcat.keep-alive-timeout` | 20000ms | Keep-Alive 타임아웃 |
| `server.tomcat.max-keep-alive-requests` | 100 | Keep-Alive당 최대 요청 |
| `server.tomcat.uri-encoding` | UTF-8 | URI 인코딩 |
| `server.tomcat.max-http-form-post-size` | 2MB | POST 폼 데이터 최대 크기 |

---

> **다음 단계 학습 추천**: Spring WebFlux(리액티브), Spring Cloud(마이크로서비스), GraalVM Native Image(서버리스), Virtual Threads(Java 21 Project Loom)
