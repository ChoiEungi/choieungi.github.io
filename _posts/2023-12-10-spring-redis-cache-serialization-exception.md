---

layout: post

title: “Spring Boot에서 Redis @Cacheable을 사용할 때 주의할 점”

tags: [Spring Boot, Redis]

date: 2023-12-10 23:30:00 +0900

categories: [Development]

---



사내에서 패키지 구조 변경 작업을 하고 배포를 했는데 갑자기 특정 API에서 `transaction silently rolled back`이 발생했었습니다. 관련해서 확인해보니 DB조회 값을  Dto 객체로 변환해 캐싱한 값을 역직렬화하는 과정에서 문제가 발생했었습니다. 해당 캐시는 월마다 한번씩 바뀌는 주기를 갖는 값으로, 조회가 많은 비율을 차지합니다. 캐시로 사용하는 정보가 DB에서 열거형으로 관리되고 있어 이를 자바 Dto 객체로 직렬화해서 redis에 저장해 캐시로 활용하고 있었습니다.

코드를 확인해보면 다음과 같습니다.



### 문제 상황 

설정 값들
``` java 
// package com.example.redisinactions.api;
@Getter  
@AllArgsConstructor(access = AccessLevel.PRIVATE)  
@NoArgsConstructor(access = AccessLevel.PROTECTED)  
public class ProductResponse implements Serializable {  
	private String description;  
	private BigDecimal price;  
  
	private static ProductResponse of(Product product) {  
		return new ProductResponse(product.getDescription(), product.getPrice());  
	}  
  
	public static List<ProductResponse> listOf(List<Product> productList) {  
		return productList.stream()  
		.map(ProductResponse::of)  
		.toList();  
	}  
}

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(60))
                .disableCachingNullValues()
                .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }

    @Bean
    public RedisCacheManagerBuilderCustomizer redisCacheManagerBuilderCustomizer() {
        return (builder) -> builder
                .withCacheConfiguration(PRODUCT_CACHE,
                        RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(10)));
    }

    public static class CacheName {
        public static final String PRODUCT_CACHE = "productCache";
    }
}

```

레디스 캐시를 사용하는 서비스 
```java
@Service
@Slf4j
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;

    @PostConstruct
    void initProducts() {
        productRepository.saveAll(List.of(
                        new Product("box", new BigDecimal(1000)),
                        new Product("snack", new BigDecimal(4000)),
                        new Product("chicken", new BigDecimal(20000))
                )
        );
    }

    @Cacheable(cacheNames = PRODUCT_CACHE, key = "'top10'")
    public List<ProductResponse> getTenProduct() {
        log.warn("NO CACHE - find top 10 products from DB");
        return ProductResponse.listOf(productRepository.findTop10By());
    }

    @CacheEvict(cacheNames = PRODUCT_CACHE, key = "'top10'")
    public void evict() {
        log.warn("Cache Evicted");
    }

}

@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @GetMapping("/products/top10")
    public ResponseEntity<?> getTop10Products() {
        return ResponseEntity.ok(productService.getTenProduct());
    }
}
```

해당 코드에서 getTenProduct()를 먼저 호출하면 다음과 같은 응답이 오며 redis에 잘 쌓이게 됩니다. 

<img src="https://i.imgur.com/yGBPAuN.png" width=400px, height=400px>



<img src="https://i.imgur.com/s6JUDmM.png"  width=400px, height=400px>



해당 상황을 도식화 하면 다음과 같습니다.

![](https://i.imgur.com/YAYsWIq.png)


이후 ProductResponse를 v2 패키지로 변경한 이후 어플리케이션을 재실행해서 동일한 API를 호출하면 SerializationException이 발생합니다.

``` // package com.example.redisinactions.api.v2;
// package com.example.redisinactions.api.v2;
@Getter  
@AllArgsConstructor(access = AccessLevel.PRIVATE)  
@NoArgsConstructor(access = AccessLevel.PROTECTED)  
public class ProductResponse implements Serializable {
   ...
}
```

![](https://i.imgur.com/KlKjpgm.png)

Exception의 cause를 확인해보면 `ClassNotFountException`이 발생합니다. `com.example.redisinactions.api.ProductResponse` 클래스를 역직렬화해야 하는데 해당 클래스가  `com.example.redisinactions.api.v2.ProductResponse`로 변경되어 발생한 현상입니다.
```
Caused by: java.lang.ClassNotFoundException: com.example.redisinactions.api.ProductResponse
	at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:641) ~[na:na]
	...

```

![](https://i.imgur.com/whlF2nJ.png)



### 해결 방법: Cache Key Prefix 변경

해당 문제를 해결하기 위해서 `@Cacheable`에서 키값 deserialization에서 오류가 나는 것이므로 키값을 바꿔줘서 해결할 수 있습니다. 기존에 저장된 캐시를 재사용하는 부분에서 문제가 발생하는 것이기에 새로운 캐시를 다시 저장하고 이를 활용하면 됩니다. 기존 키값에 해당하는 값은 역직렬화할 수 없으므로 자연스럽게 TTL로 인해 사라지게 됩니다. 이를 통해 서비스에 지장 없이 안정적으로 캐시를 변경해서 사용할 수 있습니다. 코드로 나타나면 다음과 같습니다.

```java
@Configuration
@EnableCaching
public class CacheConfig {
   ...

    public static class CacheName {
        public static final String PRODUCT_CACHE = "V2_productCache"; // as-is: productCache
    }
}
```

![](https://i.imgur.com/tm5aDLE.png)

사실 해당 값을 캐싱하는 부분에서 꼭 Redis를 이용해야 하는 부분에 대해서도 고민해볼 필요가 있습니다. Redis가 아니더라도 LocalCache를 이용한다면 빈을 주입할 때 값을 DB에서 조회해서 캐싱해서 사용하는 방법도 좋은 방법이라고 생각합니다.

본래 문제는 이로 인한 트랜잭션의 실패였습니다. 더 생각해볼 점은 `@Cacheable`은 [Cahce aside pattern](https://yearnlune.github.io/general/cache-aside-pattern/#)을 사용하는데 해당 전략은 캐시 조회가 실패한다면 원본 데이터에서 가져오는 전략입니다. 따라서 해당 작업이 트랜잭션에서 캐시 조회에서 오류가 발생한다고 롤백 마크로 인해 전체 트랜잭션이 실패하면 안된다고 생각합니다. 이는 트랜잭션을 사용할 때 두고두고 고민해야 하는 부분이라고 생각합니다.

관련 소스 코드는 다음 링크에서 확인할 수 있습니다.
https://github.com/ChoiEungi/redis-in-actions/tree/feature/redis-cacheable



