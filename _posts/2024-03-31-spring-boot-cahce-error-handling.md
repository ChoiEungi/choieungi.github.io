---

layout: post

title: Spring Boot에서 CacheManager를 에러 핸들링하는 방법

tags: [Spring Boot, Redis, Caching]

date: 2024-03-31 11:22:00 +0900

categories: [Development]

---



본 글은 아래 링크의 글의 내용과 이어집니다.

- [https://choieungi.github.io/posts/spring-redis-cache-serialization-exception](https://choieungi.github.io/posts/spring-redis-cache-serialization-exception)

Spring에서 제공하는 `@Cacheable`을 이용하면 캐쉬를 AOP 기반으로 쉽게 사용할 수 있습니다. [이전 글](https://choieungi.github.io/posts/spring-redis-cache-serialization-exception/)에서 다뤘듯, 기본적으로 `@Cacheable` 을 실행하는 Aspect 메소드에서 예외가 발생하면 전체 메소드 자체가 실패하게 됩니다. 해당 상황에서 에러 핸들링을 통해 특정 상황에서 Exception이 발생하지 않도록 변경하는 방법을 소개합니다.



### `@Cacheable` 의 `CacheErrorHandler` 동작 원리

[Spring 문서](https://docs.spring.io/spring-framework/docs/4.1.x/spring-framework-reference/html/cache.html#:~:text=By%20default%2C%20any%20exception%20throw%20during%20a%20cache%20related%20operations%20are%20thrown%20back%20at%20the%20client)에 따르면 Error Handler는 `SimpleCacheErrorHandler`를 기본 값으로 사용하고 있으며 기본적으로 에러를 클라이언트에 직접 반환합니다.`SimpleCacheErrorHandler` 코드는 다음과 같습니다.

```java
// package org.springframework.cache.interceptor;
public class SimpleCacheErrorHandler implements CacheErrorHandler {

	@Override
	public void handleCacheGetError(RuntimeException exception, Cache cache, Object key) {
		throw exception;
	}

	@Override
	public void handleCachePutError(RuntimeException exception, Cache cache, Object key, @Nullable Object value) {
		throw exception;
	}

	@Override
	public void handleCacheEvictError(RuntimeException exception, Cache cache, Object key) {
		throw exception;
	}

	@Override
	public void handleCacheClearError(RuntimeException exception, Cache cache) {
		throw exception;
	}
}
```

해당 ErrorHandler를 `CachingConfigurer` 에 대한 구현체 중 `errorHandler()` 의 리턴값으로 등록하면 됩니다.

```java
// package org.springframework.cache.annotation;

public interface CachingConfigurer {

  ...
  
	@Nullable
	default CacheManager cacheManager() {
		return null;
	}
	
	...

	@Nullable
	default CacheErrorHandler errorHandler() {
		return null;
	}

}
```

`CachingConfigurer` 를 빈으로 등록하면 spring-context에서에서 기본 빈으로 등록되는 `AbstractCachingConfiguration` 로 인해 등록되게 됩니다. 코드는 다음과 같습니다.

```java
// package org.springframework.cache.annotation;

@Configuration(proxyBeanMethods = false)
public abstract class AbstractCachingConfiguration implements ImportAware {

	@Nullable
	protected AnnotationAttributes enableCaching;

	@Nullable
	protected Supplier<CacheManager> cacheManager;

	@Nullable
	protected Supplier<CacheResolver> cacheResolver;

	@Nullable
	protected Supplier<KeyGenerator> keyGenerator;

	@Nullable
	protected Supplier<CacheErrorHandler> errorHandler;

  ...

	@Autowired
	void setConfigurers(ObjectProvider<CachingConfigurer> configurers) {
		Supplier<CachingConfigurer> configurer = () -> {
			List<CachingConfigurer> candidates = configurers.stream().collect(Collectors.toList());
			if (CollectionUtils.isEmpty(candidates)) {
				return null;
			}
			if (candidates.size() > 1) {
				throw new IllegalStateException(candidates.size() + " implementations of " +
						"CachingConfigurer were found when only 1 was expected. " +
						"Refactor the configuration such that CachingConfigurer is " +
						"implemented only once or not at all.");
			}
			return candidates.get(0);
		};
		useCachingConfigurer(new CachingConfigurerSupplier(configurer));
	}

```

실제로 `setConfigurers()`에 `@Autowired` 를 이용해 setter Injection을 통해 의존성 주입을 진행해주게 됩니다.여기서 `CachingConfigurer`를 추가로 빈으로 등록하지 않는다면, `@Cacheable` 이 실제로 동작하는 `CacheAspectSupport` 에서 초기화 할 때 기본값인 `SimpleCacheErrorHandler`가 등록되게 됩니다. 관련 코드는 다음과 같습니다.

```java
// package org.springframework.cache.interceptor;
public abstract class CacheAspectSupport extends AbstractCacheInvoker
		implements BeanFactoryAware, InitializingBean, SmartInitializingSingleton {
		
		...
		
		public void configure(
					@Nullable Supplier<CacheErrorHandler> errorHandler, @Nullable Supplier<KeyGenerator> keyGenerator,
					@Nullable Supplier<CacheResolver> cacheResolver, @Nullable Supplier<CacheManager> cacheManager) {
		
				this.errorHandler = new SingletonSupplier<>(errorHandler, SimpleCacheErrorHandler::new);
				this.keyGenerator = new SingletonSupplier<>(keyGenerator, SimpleKeyGenerator::new);
				this.cacheResolver = new SingletonSupplier<>(cacheResolver,
						() -> SimpleCacheResolver.of(SupplierUtils.resolve(cacheManager)));
			}
}
```



`CacheAspectSupport` 는 `ProxyCachingConfiguration`에서 빈으로 등록합니다. `AspectJCachingConfiguration`는 앞서 설정 값(`CachingConfigurer`)들을 setter injection으로 의존성 주입을 받은 `AbstractCachingConfiguration`를 상속받으며, 이 주입받은 값들을 이용해 빈으로 등록하게 됩니다. 해당 코드는 다음과 같으며 등록하는 빈인 `AnnotationCacheAspect`는 `CacheAspectSupport`를 상속받습니다.


```java
// package org.springframework.cache.annotation;
@Configuration(proxyBeanMethods = false)  
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)  
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

	@Bean(name = CacheManagementConfigUtils.CACHE_ASPECT_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public AnnotationCacheAspect cacheAspect() {
		AnnotationCacheAspect cacheAspect = AnnotationCacheAspect.aspectOf();
		cacheAspect.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
		return cacheAspect;
	}

}

// package org.springframework.cache.aspectj;
@Aspect
public class AnnotationCacheAspect extends AbstractCacheAspect {
	 ...
}
```



### 구현 코드

내부 원리를 확인해봤으니 실제 코드를 작성해보겠습니다. 문제 상황은 캐쉬 조회(`CacheGet`) 시 SerializationException이 발생하는 경우이기에, 이를 상속받아 원하는 대로 동작하도록 변경하면 다음과 같습니다. `SerializationException` 가 발생했을 때 로그만 남기고 예외를 무시하도록 처리했습니다

```java
@Slf4j
public class CustomCacheErrorHandler extends SimpleCacheErrorHandler {

    @Override
    public void handleCacheGetError(RuntimeException exception, Cache cache, Object key) {
        if (exception instanceof SerializationException) {
            log.warn("Failed to deserialize cache value for key: {}", key, exception);
            return;
        }

        super.handleCacheGetError(exception, cache, key);
    }

}
```

캐시 설정은 다음과 같습니다. 전체 캐시(Spring Boot Cache)에 대한 책임을 `CacheConfig`에 두었고 레디스 캐시 설정에 대한 책임을 `RedisCacheConfig`에 뒀습니다,

```java
@Configuration(proxyBeanMethods = false)
@EnableCaching
@RequiredArgsConstructor
public class CacheConfig implements CachingConfigurer {

    private final CacheManager redisCacheManager;

    @Override
    @Bean
    @Primary
    public CacheManager cacheManager() {
        return redisCacheManager;
    }

    @Override
    @Bean
    public CacheErrorHandler errorHandler() {
        return new CustomCacheErrorHandler();
    }

}

@Configuration(proxyBeanMethods = false)
public class RedisCacheConfig{

    @Bean
    public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        return RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(cacheConfiguration())
            .withCacheConfiguration(PRODUCT_CACHE,
                RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(10)))
            .build();
    }

    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(60))
            .disableCachingNullValues()
            .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }

    public static class CacheName {
        public static final String PRODUCT_CACHE = "productCache_V2";
    }
}
```

캐시를 이용해 비즈니스 로직을 처리하는 서비스 코드는 다음과 같습니다. `@Cacheable`을 이용해 해당 key에 대한 값이 캐시에  존재하면 캐시에서 조회하고 그렇지 않으면 캐시를 등록하는 코드입니다.

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

위의 `CustomCacheErrorHandler`와 `CachingConfigurer` 를 설정해주지 않은 상태에서`api.ProductResponse` 패키지 경로로 해당 객체의 캐시를 만들고 `api.v2.ProductResponse` 패키지로 옮긴 다음에 동일한 키로 조회를 하면 `SerializationException`이 발생하면서 ExceptionHandler를 통한 에러 응답이 오게됩니다.

위의 코드를 적용하고 동일한 상황을 재현해보면 warn 로그가 찍히고 응답이 정상적으로 가는 것을 볼 수 있습니다. 이로 인해 새로운 버전의 클래스로 캐시를 덮어쓰게 되면서 이후 요청에 대해서는 정상적으로 캐시를 조회하게 됩니다.

```java
2024-03-31 20:13:28.248  WARN 43092 --- [nio-8080-exec-1] c.e.r.config.CustomCacheErrorHandler     : Failed to deserialize cache value for key: top10

org.springframework.data.redis.serializer.SerializationException: Cannot deserialize; nested exception is org.springframework.core.serializer.support.SerializationFailedException: Failed to deserialize payload. Is the byte array a result of corresponding serialization for DefaultDeserializer?; nested exception is org.springframework.core.NestedIOException: Failed to deserialize object type; nested exception is java.lang.ClassNotFoundException: com.example.redisinactions.api.v2.ProductResponse
	at org.springframework.data.redis.serializer.JdkSerializationRedisSerializer.deserialize(JdkSerializationRedisSerializer.java:84) ~[spring-data-redis-2.7.2.jar:2.7.2]
	at org.springframework.data.redis.serializer.DefaultRedisElementReader.read(DefaultRedisElementReader.java:49) ~[spring-data-redis-2.7.2.jar:2.7.2]
	at org.springframework.data.redis.serializer.RedisSerializationContext$SerializationPair.read(RedisSerializationContext.java:272) ~[spring-data-redis-2.7.2.jar:2.7.2]
	at org.springframework.data.redis.cache.RedisCache.deserializeCacheValue(RedisCache.java:298) ~[spring-data-redis-2.7.2.jar:2.7.2]
	at org.springframework.data.redis.cache.RedisCache.lookup(RedisCache.java:95) ~[spring-data-redis-2.7.2.jar:2.7.2]
	at org.springframework.cache.support.AbstractValueAdaptingCache.get(AbstractValueAdaptingCache.java:58) ~[spring-context-5.3.22.jar:5.3.22]
	at org.springframework.cache.interceptor.AbstractCacheInvoker.doGet(AbstractCacheInvoker.java:73) ~[spring-context-5.3.22.jar:5.3.22]
	...
2024-03-31 20:13:28.250  WARN 43092 --- [nio-8080-exec-1] c.e.redisinactions.api.ProductService    : NO CACHE - find top 10 products from DB
2024-03-31 20:13:28.298 DEBUG 43092 --- [nio-8080-exec-1] org.hibernate.SQL                        : select product0_.id as id1_0_, product0_.description as descript2_0_, product0_.price as price3_0_, product0_.quantity as quantity4_0_ from product product0_ limit ?
Hibernate: select product0_.id as id1_0_, product0_.description as descript2_0_, product0_.price as price3_0_, product0_.quantity as quantity4_0_ from product product0_ limit ?
2024-03-31 20:13:28.311 DEBUG 43092 --- [nio-8080-exec-1] o.s.w.s.m.m.a.HttpEntityMethodProcessor  : Using 'application/json', given [*/*] and supported [application/json, application/*+json, application/json, application/*+json]
2024-03-31 20:13:28.311 DEBUG 43092 --- [nio-8080-exec-1] o.s.w.s.m.m.a.HttpEntityMethodProcessor  : Writing [[com.example.redisinactions.api.ProductResponse@3c54979a, com.example.redisinactions.api.ProductResp (truncated)...]
2024-03-31 20:13:28.317 DEBUG 43092 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed 200 OK
2024-03-31 20:14:47.332 DEBUG 43092 --- [nio-8080-exec-3] o.s.web.servlet.DispatcherServlet        : GET "/api/v1/products/top10", parameters={}
2024-03-31 20:14:47.332 DEBUG 43092 --- [nio-8080-exec-3] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped to com.example.redisinactions.api.ProductController#getTop10Products()
2024-03-31 20:14:47.336 DEBUG 43092 --- [nio-8080-exec-3] o.s.w.s.m.m.a.HttpEntityMethodProcessor  : Using 'application/json', given [*/*] and supported [application/json, application/*+json, application/json, application/*+json]
2024-03-31 20:14:47.336 DEBUG 43092 --- [nio-8080-exec-3] o.s.w.s.m.m.a.HttpEntityMethodProcessor  : Writing [[com.example.redisinactions.api.ProductResponse@39bba9c2, com.example.redisinactions.api.ProductResp (truncated)...]
2024-03-31 20:14:47.337 DEBUG 43092 --- [nio-8080-exec-3] o.s.web.servlet.DispatcherServlet        : Completed 200 OK

```



하지만 소개한 해결 방법은 배포 전략에 따라 문제가 될 수 있습니다. 단번에 배포를 변경하는 Blue-Green 전략의 경우 롤백하지 않는다면 새로 배포된 인스턴스는 새로운 캐시만 바라보게 되므로 문제가 되지 않습니다.

하지만 만약 캐시에 해당하는 클래스의 패키지를 변경하고 카나리 배포를 진행하고 구 버전과 신 버전의 인스턴스가 동시에 떠있는 상태라면 CacheName은 동일하기에 새로운 버전의 인스턴스(`api.v2.ProductResponse`)와 구 버전의 인스턴스(`api.ProductResponse`)에 대한 캐시 쓰기가 반복될 수 있습니다. 이 경우 트래픽이 많은 서비스라면 캐시 저장소에 대한 부하가 커질 수 있다는 단점이 존재합니다. 따라서 카나리의 경우는 이전에 제시한 해결책인 새로운 CacheName을 가진 캐시를 새로 만드는게 더 안전한 방법일 수 있습니다. 따라서 본 글의 목적인 상황에 맞게 캐시의 에러 핸들링 전략을 취하면 적절할 것 같습니다.



코드는 다음 링크에서 확인해볼 수 있습니다.

- [https://github.com/ChoiEungi/redis-in-actions](https://github.com/ChoiEungi/redis-in-actions)
