---

layout: post

title: 분산 시스템 환경에서 Spring Cloud Bus 없이 Spring Cloud Config 프로퍼티 Refresh하는 방법

tags: [Spring Boot, Spring Cloud Config]

date: 2024-01-21 16:12:00 +0900

categories: [Development]

---



근래에 [요즘 우아한 개발](https://m.yes24.com/Goods/Detail/122535338)이라는 책을 읽으면서 내용 중에 배포 없이 Spring Cloud Config에서 받아오는 프로퍼티를 변경해 서버에 적용하려는 내용을 접했습니다. 
이는 단일 장애 지점(SPOF)을 제거하기 위해 [외부 메시지 플랫폼을 이중화](https://techblog.woowahan.com/7724/)하면서 각 외부 플랫폼 연동에 대한 트래픽 분배를 어플리케이션이 실행 중에도 유연하게 변경하려는 의도였습니다.
또, 최종적으로 선택한 방법이 일반적으로 사용하는 Spring Cloud Bus를 이용하기보다 Spring Boot만을 활용해 프로퍼티를 재배포 없이 수정하는 방법이 흥미롭게 느껴져 호기심에 구현해보게 되었습니다.


### 배포 없이 프로퍼티를 변경
Spring Boot를 이용하면 로깅 레벨, 데이터 소스 정보, 타임아웃 설정, 환경 변수 등 여러 설정 정보(이하 프로퍼티)를 application.yml 혹은 application.properties 파일에 명시해서 외부화(externalize)할 수 있습니다. 이를 통해 다양한 환경(local, dev, prod 등) 설정을 코드가 아닌 외부 파일에 명시함으로 해당 관심사를 코드에서 분리할 수 있습니다.

책에 나온 사례와 같이 외부 시스템 장애와 같은 기민한 대응을 하려면 어플리케이션 배포 없이도 런타임 환경에서 변경할 수 있어야 합니다. 만약 해당 기능이 비즈니스에서 중요한 역할을 하는 부분이라면 더더욱 조심스럽고 기민하게 다뤄야 합니다.

Spring Cloud Config 프로퍼티를 직접 클라우드 서비스에 올려 사용할 수 있도록 해줍니다. 이를 이용하면 Cloud Config 서버를 호출해서 스프링 프로퍼티값을 받아올 수 있습니다. 또 Cloud Config를 이용하면 `@RefreshScope` 빈을 손쉽게 reload할 수 있게 됩니다. 간단한 예로 Spring Cloud Config 서버의 프로퍼티를 변경한 이후에  actuator에서 `/refresh` endpoint를 활성화 한 상태에서 호출하게 되면 변경된 프로퍼티 값을 받아올 수 있습니다. [참고 링크](https://velog.io/@korea3611/Spring-Boot-Spring-Cloud-Config3-Actuator-refresh-MSA6)

일반적으로 트래픽이 많은 서비스의 서버는 멀티 인스턴스 환경에서 운영됩니다. 여기서 모든 인스턴스의 Config 프로퍼티는 동일해야 하며, 이를 api 호출로는 하나의 인스턴스 밖에 Config 프로퍼티 변경이 안되고 이는 서버마다 설정값이 달라지게 됩니다.

![](https://i.imgur.com/n9s2Nsg.png)

이러한 문제를 해결하기 위해 Spring Cloud Bus를 통해 멀티 인스턴스 환경에서도 프로퍼티를 Refresh할 수 있습니다. 간략하게 소개하면 Config Server의 프로퍼티에 변경이 감지되면 Kafka, Redis, AMQP 중 하나를 이용해 변경된 프로퍼티 사용하는 모든 서버에 전달하는 역할을 합니다.

다만 Spring Cloud Bus 라이브러리는 Kafka, Redis, AMQP를 사용해 외부 인프라 의존성을 가지게 되고 해당 인프라 상태가 정상적이지 않을 때 사용하기 어렵습니다.

이를 해결하기 위해서 책에서는 Spring Cloud Config + Scheduled Polling을 이용한 아키텍처를 제안합니다. 외부 인프라를 관리하는 비용이 없다는 장점이 있습니다. 이를 구현해보도록 해보면 다음과 같습니다.
![](https://i.imgur.com/aU4A4Av.png)

이제 코드를 통해 살펴보도록 하겠습니다.


### `@Schedule` 와 ConteextRefresher를 이용해 구현


``` java
@Component
@RequiredArgsConstructor
@ConditionalOnProperty(value = "application.config.refresh.auto.enabled", havingValue = "true", matchIfMissing = true)
@Slf4j
public class ConfigRefreshScheduler {

  private final ContextRefresher contextRefresher;
  private final TestValueProperties testValueProperties;

  @Scheduled(fixedDelay = 5L, timeUnit = TimeUnit.SECONDS)
  @Async("refreshThreadPoolExecutor")
  public void refreshConfig() {
    try {
      Set<String> refreshedKeys = contextRefresher.refreshEnvironment();
      if (!CollectionUtils.isEmpty(refreshedKeys)) {
        log.info("[Refreshed] " + String.join(",", refreshedKeys));
        contextRefresher.refresh();
        log.info("Changed value: {}", testValueProperties.getValue());
      }
    } catch (Exception e) {
      log.error("config refresh failed {}", e);
    }
  }

  @ConditionalOnBean(value = ConfigRefreshScheduler.class)
  @Bean
  public ThreadPoolTaskExecutor refreshThreadPoolExecutor() {
    ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
    threadPoolTaskExecutor.setThreadNamePrefix("refresh");
    threadPoolTaskExecutor.setCorePoolSize(1);
    threadPoolTaskExecutor.setMaxPoolSize(1);
    threadPoolTaskExecutor.setQueueCapacity(Integer.MAX_VALUE);
    threadPoolTaskExecutor.initialize();
    return threadPoolTaskExecutor;
  }

  @ConfigurationProperties(prefix = "my")
  @Configuration
  @Getter
  @Setter
  @RefreshScope
  public static class TestValueProperties {

    // get my.value from config properties
    private String value;

    @PostConstruct
    void init() {
      log.info("-------------------------------- my properties value --------------------------------");
      log.info("{} Bean Loaded", TestValueProperties.class.getName());
      log.info(value);
      log.info("-------------------------------- my properties value --------------------------------");
    }

  }

}
```

해당 코드는 5초마다 Config Server의 프로퍼티와 서버의 프러퍼티를 비교해 변경된 프로퍼티가 있으면 `@RefreshScope` 빈(`TestValueProperties`)을 초기화하는 코드입니다. `TestProperties`는 Config Server에서 `my.value`라는 프로퍼티 값을 바인딩하는 빈입니다.

refresh에서 중요한 역할을 하는 `ContextRefresher`를 간략하게 살펴보면 `refreshEnvironment()`는 빈을 refresh하지 않으며, config 설정만 refresh하고 이로 인해 변경된 config의 key값을 리턴합니다. `refresh()`의 경우 `refreshEnvironment()` 을 먼저 호출해서 config를 spring conetext에 저장하며 그 이후 `@RefreshScope` 빈을 모두 refresh하고 변경된 config의 key값을 리턴합니다.

실제로 서버를 구동하면 다음 값을 가지고 있습니다.
![](https://i.imgur.com/jgKFt6l.png)

이후 config Server 값을 변경하면 다음과 같습니다.
![](https://i.imgur.com/Gd7yzEQ.png)

만약 Config Server의 yml 형식이 잘못되었거나 config 서버에 이상이 생긴다면 다음과 같은 에러 로그가 발생합니다. 이는 프로퍼티를 원활히 받아오지 못했기에 기존에 정상적으로 받아온 프로퍼티로 서비스가 운영되게 됩니다.
![](https://i.imgur.com/6bUmP6y.png)


이제 `ContextRefresher` 내부 코드를 통해 더 구체적인 동작 원리를 알아보겠습니다.


### Context Refresher 원리

 ContextRefresher은 Config 서버에서 받아와 API 서버 프로퍼티를 Refresh하거나 RefreshScope 빈을 Refresh하는 역할을 합니다. ContextRefresher는 Spring Cloud Client 라이브러리 의존성을 추가하면 `RefreshAutoConfiguration`으로 인해 자동으로 빈으로 등록됩니다.
 다음 코드를 보면 알 수 있듯 `spring.cloud.bootstrap.enabled: true` 면 `LegacyContextRefresher`가 빈으로 등록되고 그렇지 않으면 `ConfigDataContextRefresher`가 빈으로 등록됩니다. Spring Cloud에서 bootstrap 기본 옵션은 false이기에 `ConfigDataContextRefresher` 가 빈으로 등록됩니다.
  두 차이를 간략하게 소개하면 내부 코드에서 `ConfigDataContextRefresher`는 빈 후처리기인 `EnvironmentPostProcessor`를 이용해 ApplicationContext를 Refresh하기 때문에 ApplicationContext를 전체를 refresh하는 `LegacyContextRefresher`보다 더 효율적이라고 볼 수 있습니다. 더 자세한 내용은 본 글의 주제와 벗어나 더 다루지는 않겠습니다.

``` java
//package org.springframework.cloud.autoconfigure.RefreshAutoConfiguration;

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RefreshScope.class)
@ConditionalOnProperty(name = RefreshAutoConfiguration.REFRESH_SCOPE_ENABLED, matchIfMissing = true)
@AutoConfigureBefore(HibernateJpaAutoConfiguration.class)
@EnableConfigurationProperties(RefreshAutoConfiguration.RefreshProperties.class)
public class RefreshAutoConfiguration {

	/**
	 * Name of the refresh scope name.
	 */
	public static final String REFRESH_SCOPE_NAME = "refresh";

	/**
	 * Name of the prefix for refresh scope.
	 */
	public static final String REFRESH_SCOPE_PREFIX = "spring.cloud.refresh";

	/**
	 * Name of the enabled prefix for refresh scope.
	 */
	public static final String REFRESH_SCOPE_ENABLED = REFRESH_SCOPE_PREFIX + ".enabled";

	@Bean
	@ConditionalOnMissingBean(RefreshScope.class)
	public static RefreshScope refreshScope() {
		return new RefreshScope();
	}

	...

	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnBootstrapEnabled
	public LegacyContextRefresher legacyContextRefresher(ConfigurableApplicationContext context, RefreshScope scope,
			RefreshProperties properties) {
		return new LegacyContextRefresher(context, scope, properties);
	}

	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnBootstrapDisabled
	public ConfigDataContextRefresher configDataContextRefresher(ConfigurableApplicationContext context,
			RefreshScope scope, RefreshProperties properties) {
		return new ConfigDataContextRefresher(context, scope, properties);
	}

	@Bean
	public RefreshEventListener refreshEventListener(ContextRefresher contextRefresher) {
		return new RefreshEventListener(contextRefresher);
	}
  ...

}
```


`ConfigDataContextRefresher`는 `ContextRefresher`의 구현체입니다. ContextRefresher는 spring web actuator에서 `/refresh` endpoint를 enable할 떄 ContextRefresher를 사용해 `@RefreshScope` 빈을 refresh합니다. 실제로 spirng web actuator의 RefreshEndpoint 클래스는 다음과 같으며, `contextRefresher.refresh()`를 통해 `@RefreshScope` 빈을 refresh해주고 있습니다.

```java
// package org.springframework.cloud.context.refresh.ConfigDataContextRefresher;
public class ConfigDataContextRefresher extends ContextRefresher
		implements ApplicationListener<ApplicationPreparedEvent> {

	private SpringApplication application;

	@Deprecated
	public ConfigDataContextRefresher(ConfigurableApplicationContext context, RefreshScope scope) {
		super(context, scope);
	}

	public ConfigDataContextRefresher(ConfigurableApplicationContext context, RefreshScope scope,
			RefreshAutoConfiguration.RefreshProperties properties) {
		super(context, scope, properties);
	}

	@Override
	public void onApplicationEvent(ApplicationPreparedEvent event) {
		application = event.getSpringApplication();
	}

	@Override
	protected void updateEnvironment() {
	  ...
	}
}
```


```java
// package org.springframework.cloud.endpoint.RefreshEndpoint

@Endpoint(id = "refresh")
public class RefreshEndpoint{

	private ContextRefresher contextRefresher;

	public RefreshEndpoint(ContextRefresher contextRefresher) {
		this.contextRefresher = contextRefresher;
	}

	@WriteOperation
	public Collection<String> refresh() {
		Set<String> keys = this.contextRefresher.refresh();
		return keys;
	}

}
```


ContextRefresher를 조금 더 들어가보겠습니다. 먼저 `refreshEnvironment()`는 빈을 refresh하지 않으며, config 설정만 refresh하고 이로 인해 변경된 config의 key값을 리턴합니다. `refresh()`의 경우 `refreshEnvironment()` 을 먼저 호출해서 config를 spring conetext에 저장하며 그 이후 `@RefreshScope` 빈을 모두 refresh하고 변경된 config의 key값을 리턴합니다.

이를 통해 bean overriding option에 따라 ConextRefresher의 구현체가 달라지는 것은 refresh할 때 빈을 정의하는 방식이 달라지기 때문임을 알 수 있습니다. 이는 `updateEnvironment()`를 구현체를 통해 구현한다는 것으로 이해할 수 있습니다.


```java
// package org.springframework.cloud.context.refresh.ContextRefresher;
public abstract class ContextRefresher {
  
	private ConfigurableApplicationContext context;
	private RefreshScope scope;

	@SuppressWarnings("unchecked")
	protected ContextRefresher(ConfigurableApplicationContext context, RefreshScope scope,
			RefreshAutoConfiguration.RefreshProperties properties) {
		this.context = context;
		this.scope = scope;
		additionalPropertySourcesToRetain = properties.getAdditionalPropertySourcesToRetain();
	}

	public synchronized Set<String> refresh() {
		Set<String> keys = refreshEnvironment();
		this.scope.refreshAll();
		return keys;
	}

	public synchronized Set<String> refreshEnvironment() {
		Map<String, Object> before = extract(this.context.getEnvironment().getPropertySources());
		updateEnvironment();
		Set<String> keys = changes(before, extract(this.context.getEnvironment().getPropertySources())).keySet();
		this.context.publishEvent(new EnvironmentChangeEvent(this.context, keys));
		return keys;
	}

	protected abstract void updateEnvironment();

	...
}
```


이를 통해 서비스 점검 시간 등 잘 변하지 않지만 종종 변경해줘야 값들 등 property에 넣어서 사용하는 값들을 배포 없이 config 서버의 값 변경만으로 준 실시간으로 적용할 수 있게됩니다. 또, 피쳐 플래그, 스프링 스케줄러, 외부 API 등 on/off 관련 기능에도 적용해볼 수 있습니다.

### 주의사항
시스템이 커지다 보면 코드량이 많아지면서 빌드 시간이 늘어나게 되고 스프링의 경우 어플리케이션 빈이 많아지다보면 실행 시간이 늘어나게됩니다. 이에 따라 설정값 변경만으로 인해 다시 배포하는데 시간이 오래 소요됩니다. 그래서 배포 없이 런타임 환경에서 설정 정보를 변경하는건 굉장히 유용하다고 생각합니다. 하지만 은탄환은 존재하지 않습니다.

먼저 `@RefreshScope`를 남용하면 안됩니다. datasource, redis, kafka와 같은 설정 정보들은 빈이 복잡하게 얽혀있고 서비스가 운영되는 중에 refresh를 하면 서버에 부담이 크게 작용할 수 있습니다. 실제로 사내에서 DataourceConfiguration 관련 빈에 RefreshScope을 넣어 서버가 죽은 경험도 있습니다. 차라리 해당 상황의 경우 배포를 다시해서 설정 정보를 초기화 하는게 더 안정적으로 서비스를 제공하는 방법이라고 생각합니다.

또, `ConetxtRefresher`를 활용할 때 ENC(value) 값과 같이 Jasypt를 이용해 암호화된 설정 값을 이용하게 되면 `refreshEnvironment()`에서 복호화된 값과 복호화되기 전의 값을 가져오게 되어 명확하게 변경된 설정값을 가져오지 못하는 현상도 존재합니다. 해당 이슈에 대해서는 다음 글에서 알아보도록 하겠습니다.

실제로 config 폴링에 대해 spring-cloud-commons docs에서는 [폴링을 권장하지 않는 방법이라고](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#http-clients:~:text=Note%20that%20the%20Spring%20Cloud%20Config%20Client%20does%20not%2C%20by%20default%2C%20poll%20for%20changes%20in%20the%20Environment.%20Generally%2C%20we%20would%20not%20recommend%20that%20approach%20for%20detecting%20changes%20(although%20you%20can%20set%20it%20up%20with%20a%20%40Scheduled%20annotation)) 언급합니다.

> Note that the Spring Cloud Config Client does not, by default, poll for changes in the `Environment`. Generally, we would not recommend that approach for detecting changes (although you can set it up with a `@Scheduled` annotation).

폴링의 경우 Config Server에 대한 트래픽이 더 커지게 되고 조직의 규모가 커지고 해당 Config Server를 사용하는 API 서버들이 많아질수록 더 부하가 많이갈 수 있습니다. 이는 예상치 못한 일이 발생할 수 있기에 해당 방법을 차용하게되면 Config Server를 더 세심하게 모니터링해야 합니다. 현재 조직 상황에 맞는 방법을 적절히 사용하는게 가장 중요하다고 생각합니다.
실제로 [Toss Slash 23 영상(20:35~)](https://www.youtube.com/watch?v=Zs3jVelp0L8&t=1235s)에 따르면 운영 환경에서는 사용하지 않고 휴면 에러가 발생할 수 있다는 의견도 있기에 상황에 맞게 적절히 사용해야 합니다.

해당 소스 코드는 다음 [링크](https://github.com/ChoiEungi/spring-cloud-config-refresh)에서 확인해 수 있습니다.


참고 링크
- [https://techblog.woowahan.com/7724/](https://techblog.woowahan.com/7724/)
- [https://www.youtube.com/watch?v=Zs3jVelp0L8&t=1235s](https://www.youtube.com/watch?v=Zs3jVelp0L8&t=1235s)
- [https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#http-clients:~:text=Note%20that%20the%20Spring%20Cloud%20Config%20Client%20does%20not%2C%20by%20default%2C%20poll%20for%20changes%20in%20the%20Environment.%20Generally%2C%20we%20would%20not%20recommend%20that%20approach%20for%20detecting%20changes%20(although%20you%20can%20set%20it%20up%20with%20a%20%40Scheduled%20annotation)](https://docs.spring.io/spring-cloud-commons/docs/current/reference/html/#http-clients:~:text=Note%20that%20the%20Spring%20Cloud%20Config%20Client%20does%20not%2C%20by%20default%2C%20poll%20for%20changes%20in%20the%20Environment.%20Generally%2C%20we%20would%20not%20recommend%20that%20approach%20for%20detecting%20changes%20(although%20you%20can%20set%20it%20up%20with%20a%20%40Scheduled%20annotation))
