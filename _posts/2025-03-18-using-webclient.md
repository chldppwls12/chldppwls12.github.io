---
title: WebClient로 HTTPS 통신하기
date: 2025-03-18 17:13:52 +/-TTTT
categories: [Spring Boot]
tags: [WebClient]
---


Spring Boot 기반의 A 서비스와 B 서비스가 통신해야하는 상황이라 `WebClient`를 사용했습니다.
B 서비스가 자체 서명된 인증서를 사용하고 있어, SSL 인증 오류가 발생했습니다.
이를 해결하기 위해 `WebClient`  사용 시, SSL 검증을 우회하는 방법에 대해 설명합니다.

<br/>

## RestTemplate VS WebClient
먼저, Spring Boot에서 서비스 간 통신을 위해서는 `RestTemplate`이나 `WebClient`를 사용할 수 있습니다.

### RestTemplate

Spring 3부터 제공되는 Blocking HTTP Client입니다.

내부적으로 Java Servlet API를 사용해 응답이 올 때까지 해당 스레드는 차단(Block) 상태가 되는 방식입니다.
요청마다 새로운 스레드가 생성되며, 동시 요청이 많아지면 메모리 사용량 증가 및 Context Switching 오버헤드가 발생합니다.

결과적으로 많은 스레드를 생성해 스레드 풀이 고갈되거나, 메모리 사용량이 급격히 증가할 위험이 있습니다.

> 📌 **Spring 5부터는 유지보수 모드로 전환되었으며, WebClient 사용이 권장됩니다.**

### WebClient

Spring 5부터 제공되는 Non-Blocking HTTP Client입니다.
비동기(Asynchronous) 및 논블로킹(Non-Blocking) 방식으로 동작합니다.

내부적으로 Netty 또는 Jetty의 이벤트 루프 모델을 사용합니다.
일반적으로 CPU 코어 수에 비례하는 소수의 스레드만 사용하지만, 각 스레드가 여러 요청을 동시에 처리할 수 있어 높은 성능을 제공합니다.

<br/>

## WebClient 사용해보기

### 종속성 추가

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### 구현
현재 요청 받은 B 서비스는 자체 서명된 인증서를 사용하고 있습니다.  
그렇기 때문에 Spring Boot가 기본적으로 신뢰하는 CA 인증서 목록에 B 서비스의 자체 서명된 인증서가 포함되지 않아, 아래와 같은 SSL 인증 오류가 발생합니다.

```
io.netty.handler.codec.DecoderException: javax.net.ssl.SSLHandshakeException: General OpenSslEngine problem
...
Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```

이 문제를 해결하는 방법은 다음과 같습니다.

✅ **권장 해결 방법**: B 서비스에 **공인된 CA 인증서를 적용**  
🚨 **우회 방법**: SSL 검증을 비활성화 (보안상 위험할 수 있음)  

<br/>

제가 해당 서비스의 인증서를 변경할 수 없는 상황이기에, SSL 검증을 비활성화 하는 방식을 사용했습니다.

```java
@Configuration
public class WebClientConfig {
    // 기본 WebClient
    @Bean
    public WebClient webClient() {
        return WebClient.builder()
                .baseUrl("https://도메인 주소")
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .build();
    }

    // SSL 검증 우회 WebClient
    @Bean(name = "insecureWebClient")
    public WebClient insecureWebClient() throws SSLException {
        // 1. SSL 검증을 우회하는 설정
        SslContext sslContext = SslContextBuilder
                .forClient()
                .trustManager(InsecureTrustManagerFactory.INSTANCE) // 모든 인증서를 신뢰
                .build();
    
        // 2. HTTP 클라이언트 생성 (타임아웃 설정 포함)
        HttpClient httpClient = HttpClient.create()
                .secure(t -> t.sslContext(sslContext))
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000) // 연결 타임아웃 5초
                .responseTimeout(Duration.ofSeconds(30)) // 응답 타임아웃 30초
                .doOnConnected(conn -> conn
                        .addHandlerLast(new ReadTimeoutHandler(30)) // 읽기 타임아웃
                        .addHandlerLast(new WriteTimeoutHandler(30))); // 쓰기 타임아웃
    
        // 3. WebClient 생성
        return WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .build();
    }
}
```

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class TestService {
    private final WebClient webClient;
    private final WebClient insecureWebClient;

    // SSL 검증
    public Mono<String> withSecureWebClient() {
    return webClient.post()
            .uri("/test")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(requestBody)
            .retrieve()
            .bodyToMono(String.class)
            .onErrorResume(e -> {
                e.printStackTrace();
                return Mono.just("Error occurred: " + e.getMessage());
            });
    }

    // SSL 검증 우회
    public Mono<String> withInsecureWebClient() {
    return insecureWebClient.post()
            .uri("https://도메인주소/test")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(requestBody)
            .retrieve()
            .bodyToMono(String.class)
            .onErrorResume(e -> {
                e.printStackTrace();
                return Mono.just("Error occurred: " + e.getMessage());
            });
    }
}
```
SSL 우회 방식을 사용하여 서비스 간 정상적인 통신을 확인했습니다.

하지만 SSL 검증을 우회하는 것은 보안상 권장되지 않으며,  
가능하면 CA 인증서를 적용하여 인증 문제를 해결하는 것이 권장됩니다.

---
참고 자료

- [Spring WebClient vs. RestTemplate](https://www.baeldung.com/spring-webclient-resttemplate)
- [RestTemplate vs WebClient](https://velog.io/@emotional_dev/RestTemplate-vs-WebClient)
- [[WebClient] Spring WebClient로 HTTPS 통신하기(sun.security.provider.certpath.SunCertPathBuilderException 해결)](https://colabear754.tistory.com/177)
