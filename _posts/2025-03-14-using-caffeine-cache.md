---
title: Spring Boot에서 Caffeine Cache 사용해보기
date: 2025-03-14 09:23:52 +/-TTTT
categories: [Spring Boot]
tags: [Cache]
---


## 왜 캐싱이 필요한가?

현재 프로젝트에서 특정 지표를 기반으로 Bing Maps API를 사용해 지도 이미지를 가져오는 로직이 있습니다. 동일한 지표로 반복 요청이 들어올 경우, 매번 API를 호출해 이미지를 가져오는건 비효율적입니다.
API 호출 횟수를 줄이고 응답 시간 개선을 위해 캐싱 전략을 도입해야합니다.

### Caffeine Cache 선택 이유
해당 서비스는 단일 인스턴스 환경에서 처리되기에 로컬 인메모리 캐시를 사용하는 것이 적합합니다.
Spring Boot에서 로컬 캐시로는 `EhCache`, `Caffeine`이 많이 사용되는 것 같은데, [성능 측면](https://gosunaina.medium.com/cache-redis-ehcache-or-caffeine-45b383ae85ee)에서 `Caffeine`이 더 낫다고 해서 선택했습니다.


<br>

## 캐싱 적용해보기
### 의존성 추가
```
implementation 'org.springframework.boot:spring-boot-starter-cache'
implementation 'com.github.ben-manes.caffeine:caffeine'
```

### 설정값 추가
```java
@Getter
@RequiredArgsConstructor
public enum CacheType {
    BING_MAP_IMAGE("bing-map-image", 1, 1000);

    private final String cacheName;
    private final int expiredAfterWrite;
    private final int maxiumSize;
}
```

### 설정값 넣기
캐시가 여러개일 경우 캐시 설정을 개별적으로 할 수 있도록 Config를 사용합니다.
현재는 BING_MAP_IMAGE만 설정했기 때문에 해당 키 값만 사용됩니다.
- `expreAfterWrite`: 캐시 생성 후 또는 가장 최근에 바뀐 후 특정 기간이 지나면 각 항목이 캐시에서 자동으로 제거합니다.
- `maxiumSize`: 최대 N개 항목만 캐시에 저장 가능합니다. 초과 시에 LRU(Least Recently Used)에 따라 가장 오래된 항목부터 제거합니다.

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public List<CaffeineCache> caffeineCaches() {
        return Arrays.stream(CacheType.values())
                .map(cacheType -> new CaffeineCache(cacheType.getCacheName(), Caffeine.newBuilder()
                        .expireAfterWrite(cacheType.getExpiredAfterWrite(), TimeUnit.MINUTES)    // 캐시 만료 시간
                        .maximumSize(cacheType.getMaxiumSize())   // 최대 캐시 항목 수
                        .recordStats()  // 캐시 통계 활성화
                        .build()
                ))
                .toList();
    }

    @Bean
    public CacheManager cacheManager(List<CaffeineCache> caffeineCaches) {
        SimpleCacheManager cacheManager = new SimpleCacheManager();
        cacheManager.setCaches(caffeineCaches);

        return cacheManager;
    }
}
```

### 캐시 적용하기
`@Cacheable`는 메소드가 호출될 때 캐시를 먼저 확인하고, 캐시에 값이 있으면 메소드 실행 없이 캐시된 결과를 반환합니다. 캐시에 값이 없는 경우에만 메소드를 실행하고 그 결과를 캐시에 저장합니다.
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MapService {
    private final BingMapMapper bingMapMapper;

      @Cacheable(value = "bing-map-image", key = "#reqDto.postSeq")
      public String getBingMapImage(BingMapReqDto reqDto) {
          // 캐시 미스 시에만 실행되는 로직
          return bingMapMapper.getBingMapImage(reqDto);
      }
}
```

## 결과 확인해보기
동일한 파라미터를 가지고 메서드를 호출했을 경우, DB 접근(bingMapMapper)은 한 번만 호출되는 것을 확인할 수 있다.
```java
@SpringBootTest
public class MapServiceCacheTest {

    @Autowired
    private MapService mapService;

    @MockitoBean
    private BingMapMapper bingMapMapper;

    @Test
    @DisplayName("getBingMapImage() 메서드를 같은 파라미터로 여러 번 호출해도 mapper는 한 번만 호출된다")
    void getBingMapImageUsedCache() {
        // given
        BingMapReqDto reqDto = new BingMapReqDto();
        reqDto.setProjectSeq(1);

        // when
        IntStream.range(0, 100).forEach(i -> {
            try {
                mapService.getBingMapImage(reqDto);
            } catch (JsonProcessingException e) {
                throw new RuntimeException(e);
            }
        });

        // then
        verify(bingMapMapper, times(1)).getBingMapImage(reqDto);
    }
}
```
![cache-test.png](/assets/img/using-caffeine-cache/cache-test.png)

---
참고 자료

- [[Spring Boot] Caffeine Cache를 활용하여 부하를 줄이고, 성능을 개선하자](https://velog.io/@komment/%EC%BA%90%EC%8B%9C%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%98%EC%97%AC-%EB%B6%80%ED%95%98%EB%A5%BC-%EC%A4%84%EC%9D%B4%EA%B3%A0-%EC%84%B1%EB%8A%A5%EC%9D%84-%EA%B0%9C%EC%84%A0%ED%95%98%EC%9E%90)
