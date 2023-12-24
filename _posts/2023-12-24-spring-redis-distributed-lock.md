---

layout: post

title: Spring MVC에서 redisson으로 분산락을 구현하는 방법들

tags: [Spring Boot, Redis]

date: 2023-12-24 23:30:00 +0900

categories: [Development]

---



멀티 인스턴스 환경에서 동시성을 해결하는 방법으로 Redis의 이벤트 루프 기반 싱글스레드 특성을 이용해 분산락을 사용해 쉽게 해결할 수 있습니다. 동시 호출은 DB 데이터의 정합이 깨지거나 메시지 이벤트의 중복 발행 등 예상치 못한 동작으로 이어지는 경우가 많아 해결해야 하는 경우가 빈번합니다.

분산락은 동시성을 제어하기 위한 부가 기능으로 비즈니스 로직과 섞이지 않도록 관심사를 잘 분리하는게 좋습니다. 관련해서 스프링은 Dependency Injection, AOP와 같은 여러 기능을 쉽게 사용할 수 있어 여러 구현 방법을 소개해보려 합니다. 구현은 Java, Spring MVC 기반으로 Redisson을 이용해 구현할 예정입니다.

### 요구사항 
- 분산락이 실패하는 경우 null을 리턴하게 됩니다. 
- 레디스가 장애가 발생해도 동작해야 합니다.
- 락 내부 트랜잭션을 사용할 수 있어야 하며 락이 끝나기 전에 트랜잭션이 종료되어야 합니다. 트랜잭션이 종료되어야 하는 이유는 트랜잭션이 종료함으로 DB에 저장된 시점에 락을 해제해야 동시 호출에 대한 온전히 제어할 수 있기 때문입니다. 더 자세한 내용은 [다음 글](https://helloworld.kurly.com/blog/distributed-redisson-lock/#3-%EB%B6%84%EC%82%B0%EB%9D%BD%EC%9D%84-%EB%B3%B4%EB%8B%A4-%EC%86%90%EC%89%BD%EA%B2%8C-%EC%82%AC%EC%9A%A9%ED%95%A0-%EC%88%98%EB%8A%94-%EC%97%86%EC%9D%84%EA%B9%8C)에서 소개하고 있기에 본 글에서는 생략하겠습니다. 
- 락 설정 정보를 하나의 enum으로 관리할 수 있어야 합니다.


### 함수형 인터페이스를 이용한 구현 (template callback 패턴)

락을 잡으려고 하는 구간을 함수형 인터페이스(콜백 함수) 인자로 받아서 처리하는 방법입니다. 트랜잭션을 사용할 수 있으며, 트랜잭션의 원자성을 종료를 보장하기 위해 전파 옵션을 `PROPAGATION_REQUIRES_NEW`로 설정했습니다. 구현 코드 다음과 같습니다. 

**Lock 설정 코드** 
```java
public enum LockConfig {

  PRODUCT_DECREASE("PRODUCT", 1L, 1L, TimeUnit.SECONDS),
  LOGIN_LOCK("LOGIN_LOCK", 2L, 2L, TimeUnit.SECONDS);

  public final Long waitTime;
  public final Long leaseTime;
  public final TimeUnit timeUnit;
  private final String lockPrefix;

  LockConfig(String lockPrefix, Long waitTime, Long leaseTime, TimeUnit timeUnit) {
    this.lockPrefix = lockPrefix;
    this.waitTime = waitTime;
    this.leaseTime = leaseTime;
    this.timeUnit = timeUnit;
  }

  public String generateKey(String key) {
    if (!StringUtils.hasText(key)) {
      throw new IllegalArgumentException("key must not be empty");
    }
    return String.format("%s_%s", lockPrefix, key);
  }

}
```



``` java
@Component  
@Slf4j  
public class RedissonDistributedLockTemplate {  
  
  private final RedissonClient redissonClient;  
  private final TransactionTemplate transactionTemplate;  
  
  public RedissonDistributedLockTemplate(RedissonClient redissonClient, PlatformTransactionManager platformTransactionManager) {  
    this.redissonClient = redissonClient;  
    this.transactionTemplate = new TransactionTemplate(platformTransactionManager);  
    this.transactionTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);  
    this.transactionTemplate.afterPropertiesSet();  
  }  
  
  public void executeWithLock(String key, LockConfig lockConfig, Runnable callback) {  
    executeWithLock(key, lockConfig, toVoidSupplier(callback));  
  }  
  
  public <T> T executeWithLock(String key, LockConfig lockConfig, Supplier<T> callback) {  
    RLock lock = redissonClient.getLock(lockConfig.generateKey(key));  
  
    try {  
      boolean isAcquired = lock.tryLock(lockConfig.waitTime, lockConfig.leaseTime, lockConfig.timeUnit);  
      if (!isAcquired) {  
        log.warn("[lock 획득 실패] {}, key : {}", lockConfig, key);  
        return null;      }  
      return callback.get();  
    } catch (RedisConnectionException redisUnavailableException) {  
      log.warn("", redisUnavailableException);  
      return callback.get();  
    } catch (InterruptedException e) {  
      log.error("", e);  
      Thread.currentThread().interrupt();  
      return null;    } finally {  
      if (lock.isLocked() && lock.isHeldByCurrentThread()) {  
        lock.unlock();  
      }  
    }  
  }  
  
  public void executeWithLockAndTransaction(String key, LockConfig lockConfig, Runnable callback) {  
    executeWithLockAndTransaction(key, lockConfig, toVoidSupplier(callback));  
  }  
  
  public <T> T executeWithLockAndTransaction(String key, LockConfig lockConfig, Supplier<T> callback) {  
    RLock lock = redissonClient.getLock(lockConfig.generateKey(key));  
  
    try {  
      boolean isAcquired = lock.tryLock(lockConfig.waitTime, lockConfig.leaseTime, lockConfig.timeUnit);  
      if (!isAcquired) {  
        log.warn("[lock 획득 실패] {}, key : {}", lockConfig, key);  
        return null;      }  
      return transactionTemplate.execute(status -> callback.get());  
    } catch (RedisConnectionException redisUnavailableException) {  
      log.warn("", redisUnavailableException);  
      return transactionTemplate.execute(status -> callback.get());  
    } catch (InterruptedException e) {  
      log.error("", e);  
      Thread.currentThread().interrupt();  
      return null;    } finally {  
      if (lock.isLocked() && lock.isHeldByCurrentThread()) {  
        lock.unlock();  
      }  
    }  
  }  
  
  private Supplier<Void> toVoidSupplier(Runnable runnable) {  
    return () -> {  
      runnable.run();  
      return null;    };  
  }  
  
}

```

사용하는 방법은 다음과 같습니다. 

``` java

@Service  
@RequiredArgsConstructor  
public class V2ProductService {  
  
  private final ProductRepository productRepository;  
  private final RedissonDistributedLockTemplate redissonDistributedLockTemplate;  
  
  ... 
  
  public Product decreaseWithCallback(Long id, Long quantity) {  
    Product result = redissonDistributedLockTemplate.executeWithLock(id.toString(), PRODUCT_DECREASE, () -> {  
      Product product = productRepository.findById(id).orElseThrow();  
      product.decrease(quantity);  
      return productRepository.save(product);  
    });  
    return result;  
  }  
  
  public Product decreaseWithCallbackTransaction(Long id, Long quantity) {  
    Product result = redissonDistributedLockTemplate.executeWithLockAndTransaction(id.toString(), PRODUCT_DECREASE, () -> {  
      Product product = productRepository.findById(id).orElseThrow();  
      product.decrease(quantity);  
      return product;  
    });  
    return result;  
  }  
  
}

```


테스트 코드

```java
@SpringBootTest
class V2ProductServiceTest {

  @Autowired
  private V2ProductService v2ProductService;

  @Autowired
  private ProductRepository productRepository;

  private Product product;

  @BeforeEach
  void setUp() {
    product = productRepository.save(new Product("description", new BigDecimal(10000), 100L));
  }
  
  ...

  @Test
  void decreaseWithCallback() {
    // given
    int requestCount = 10;

    List<CompletableFuture<?>> futureList = new ArrayList<>();
    for (int i = 0; i < requestCount; i++) {
      CompletableFuture<Object> objectCompletableFuture = CompletableFuture.supplyAsync(() -> {
        v2ProductService.decreaseWithCallback(product.getId(), 1L);
        return null;
      });
      futureList.add(objectCompletableFuture);
    }

    // then
    Product result = productRepository.findById(product.getId()).orElseThrow();
    assertThat(result.getQuantity()).isEqualTo(90L);

  }
}
```


장점 
- 특별한 클래스 분리가 필요 없이 Template 주입만을 통해서 메소드 내부에서 직접 락 구간을 지정할 수 있습니다. 

단점 
- 기존 코드의 depth가 깊거나 콜백(함수형 인터페이스)을 사용하는 코드가 많다면, 콜백 지옥에 빠질 수 있습니다.
- 콜백 메소드 내부에서 값들이 여러개라면 해당 값들을 모두 리턴해야 합니다. 그렇게 되면 이를 위한 객체를 만들어야 합니다. 
- 부가 기능이 테스트에 영향을 주게 됩니다. 콜백 메소드 내부만 테스트하고 싶다면 template에 대한 Mocking혹은 Stubbing 필요합니다.



### AOP(Aspect Oriented Programming)를 이용한 구현 

스프링 부트는 어노테이션 기반 AOP 기능을 손쉽게 사용할 수 있게 제공합니다. AOP를 사용하기 위해서는`@EnableAspectJAutoProxy`설정을 등록해야 합니다. 코드는 다음과 같습니다. 

AOP 구현체
- JoinPointSpELParser는 직접 정의한 spring Expression parser로 `#매개변수명`을 통해 값을 접근할 수 있습니다. `@Cacheable`, `@PreAuthorize`, `@Value` 에서 활용하는 것과 동일하며, 직접 구현해 사용했습니다. 자세한 구현은 코드에서 확인해볼 수 있으며 본 내용에서는 생략하도록 하겠습니다. 

```java
@Component
@Slf4j
@Aspect
@RequiredArgsConstructor
public class DistributedLockAspect {

  private final RedissonClient redissonClient;
  private final JoinPointSpELParser joinPointSpELParser;

  @Around("@annotation(distributedLock)")
  public Object lock(ProceedingJoinPoint pjp, DistributedLock distributedLock) throws Throwable {
    String pa = joinPointSpELParser.parseSpEL(pjp, distributedLock.key());
    final String key = distributedLock.lockConfig().generateKey(pa);
    RLock lock = redissonClient.getLock(key);
    try {
      final boolean isAcquired = lock.tryLock(distributedLock.lockConfig().waitTime, distributedLock.lockConfig().leaseTime,
          distributedLock.lockConfig().timeUnit);
      if (!isAcquired) {
        return null;
      }
      return pjp.proceed();
    } catch (RedisConnectionException redisUnavailableException) {
      log.warn("", redisUnavailableException);
      return pjp.proceed();
    } catch (InterruptedException e) {
      log.error("", e);
      Thread.currentThread().interrupt();
      return null;
    } finally {
      if (lock.isLocked() && lock.isHeldByCurrentThread()) {
        lock.unlock();
      }
    }
  }

}

@Target(value = ElementType.METHOD)
@Retention(value = RetentionPolicy.RUNTIME)
public @interface DistributedLock {

  String key();

  LockConfig lockConfig();

}
```


장점 
- 비즈니스 로직과 부가 기능의 관심사를 완전히 분리할 수 있습니다. 이를 통해 비즈니스 로직 테스트를 작성하기 더 쉬워집니다.
- Spring Expression을 이용해 인자값으로 키값 설정을 간편하게 활용할 수 있습니다. 


단점 
- 타 AOP 어노테이션(`@Transactional`, `@Async` 등)과의 순서(`@Order`) 지정 및 동작 예측이 어려워집니다.
- 고질적인 AOP의 단점인 자기 호출([self-invocation](https://gmoon92.github.io/spring/aop/2019/04/01/spring-aop-mechanism-with-self-invocation.html)) 문제가 존재합니다. 오버로딩을 많이 하는 코드에서는 사용하기 어려울 수 있습니다.


### AOP와 함수형 인터페이스를 이용한 구현 
사실 AOP와 콜백의 장단은 서로 보완관계라고 생각해 상황에 맞게 취사선택하는 방법도 좋은 방법이라고 생각합니다. 사실 AOP 내부에서 함수형 인터페이스를 이용한 구현체를 사용하면 일관된 구현을 통해 락을 활용할 수 있습니다. 이를  `@V2DistributedLock`으로 만들어 구현해보겠습니다. 


```java 

@Component  
@Aspect  
@RequiredArgsConstructor  
public class V2DistributedLockAspect {  
  
  private final JointPointSpELParser jointPointSpELParser;  
  private final RedissonDistributedLockTemplate redissonDistributedLockTemplate;  
  
  @Around("@annotation(v2DistributedLock)")  
  public Object lock(ProceedingJoinPoint pjp, V2DistributedLock v2DistributedLock) {  
    String parsedKey = jointPointSpELParser.parseSpEL(pjp, v2DistributedLock.key());  
    final String key = v2DistributedLock.lockConfig().generateKey(parsedKey);  
  
    if (v2DistributedLock.isTransactionEnabled()) {  
      return redissonDistributedLockTemplate.executeWithLockAndTransaction(key, v2DistributedLock.lockConfig(), proceed(pjp));  
    }  
  
    return redissonDistributedLockTemplate.executeWithLock(key, v2DistributedLock.lockConfig(), proceed(pjp));  
  }  
  
  private Supplier<Object> proceed(ProceedingJoinPoint pjp) {  
    return () -> {  
      try {  
        return pjp.proceed();  
      } catch (Throwable e) {  
        throw new RuntimeException(e);  
      }  
    };  
  }
  
}

@Target(value = ElementType.METHOD)  
@Retention(value = RetentionPolicy.RUNTIME)  
public @interface V2DistributedLock {  
  
  String key();  
  
  LockConfig lockConfig();  
  
  boolean isTransactionEnabled() default false;  
  
}
```

사용 코드 
```java
@Service
@RequiredArgsConstructor
public class V2ProductService {

  private final ProductRepository productRepository;

  @DistributedLock(key = "#id", lockConfig = PRODUCT_DECREASE)
  public Product decreaseWithAOP(Long id, Long quantity) {
    Product product = productRepository.findById(id).orElseThrow();
    product.decrease(quantity);
    return productRepository.save(product);
  }

  @V2DistributedLock(key = "#id", lockConfig = PRODUCT_DECREASE, isTransactionEnabled = true)
  public Product decreaseWithAOPV2(Long id, Long quantity) {
    Product product = productRepository.findById(id).orElseThrow();
    product.decrease(quantity);
    return product;
  }

}

```

### 마무리  
일반적인 상황에서 오버헤드는 많이 발생하지 않지만 트래픽이 많아지거나 redis 지연 발생, 혹은 장애 상황이 발생할 때의 고민도 필요합니다. 또한, DB 작업은 테이블의 데이터양이 변화함에 따라 실행 속도가 일관되지 않을 수 있기 때문에 수행 시간에 대한 모니터링, 타임아웃 발생에 대한 모니터링을 진행해 락 시간의 유동적인 조절이 필요할 수도 있습니다.

번외로 동시성에 대해 해결 방법을 소개했지만 근본적으로 동시성이 어디서, 왜 발생하는가에 대한 근본적인 질문을 해보는 것도 방법입니다. 단순히 분산락은 문제를 해결하기 위한 수단일 뿐 절대적인 해결 방법으로만 고려하는건 적절하지 않다고 생각합니다. 물론 글에서 언급한 경우는 발생할 수 있는 상황이기에 동시성을 제어할 필요가 있습니다.  

하지만 모든 동시성 상황이 선착순이나 재고 관리와 같은 상황이 아닐 수 있습니다. 간혹 동시성이 일어나는 경우 클라이언트의 잘못된 구현으로 동시 호출이 일어날 수도 있고, UX 상으로 동시성이 일어날 수 있는 설계일 수도 있습니다. 특히 클라이언트의 경우 이벤트 기반으로 동작하는 경우가 많아 동시 호출을 놓치는 경우도 종종 있었습니다.

모든 예외는 존재하지만 문제의 근본적인 원인과 그 문제의 임팩트에 대해 고민해보고 상황에 맞는 해결책을 제시하고 실행하는게 가장 중요합니다.  

더 자세한 코드들과 테스트 코드들은 다음 링크에서 확인할 수 있습니다. https://github.com/ChoiEungi/redis-in-actions



참고 링크 
- https://helloworld.kurly.com/blog/distributed-redisson-lock/#3-%EB%B6%84%EC%82%B0%EB%9D%BD%EC%9D%84-%EB%B3%B4%EB%8B%A4-%EC%86%90%EC%89%BD%EA%B2%8C-%EC%82%AC%EC%9A%A9%ED%95%A0-%EC%88%98%EB%8A%94-%EC%97%86%EC%9D%84%EA%B9%8C

