---

layout: post

title: "[Spring] Spring boot에서 Domain Event 활용해 도메인 간의 결합도 낮추기"

tags: [Spring, SW 마에스트로]

date: 2022-11-13 12:55:00 +0900

categories: [Development]

---



# Spring boot에서 Domain Event 활용해 도메인 간의 결합도 낮추기

### Goal

- 업적 달성 시, 데이터베이스에 유저의 업적 달성을 기록한다.
- 업적 달성은 편지 송신, 좋아요와 같은 유저의 행위가 발생한 후 조건을 만족하면 이뤄진다.
- 본 글에서는 유저가 편지를 작성할 떄, 이벤트를 발행해 유저의 업적을 달성하는 상황에 대한 코드를 다룹니다.

### Solution

- 도메인 이벤트를 활용해 업적 조건을 만족시키는 것을 이벤트 리스너로 추적한다. 이를 통해 서로 다른 도메인의 결합도를 낮춘다.

### 문제 상황

업적 달성은 데이터베이스의 유저의 정보 변경이 일어날 때 발생한다. 이는 곧, 유저 테이블에서 Insert와 Update 작업이 발생할 때, 업적 달성이 이뤄진다. 그렇기에 이 작업들에 대해 추적을 해야할 것이다. 여러가지 방법이 존재하겠지만, 이를 코드에서 추적할 수 있다면 좋겠다고 생각했고, 본 프로젝트에서는 Spring data를 사용하고 있어 관련된 해결책을 모색했다. 

 가장 첫 번째로 생각이 들었던건 단순히 비즈니스 로직을 처리하는 서비스 레이어에서 업적이 일어날 때마다 분기를 넣어 해결하는 방법인데, 이는 서로 다른 도메인의 강결합이 일어나는 문제가 존재했다. 편지 서비스 계층에서 편지 도메인의 편지라는 엔티티가 생성될 때,  유저 도메인의 유저 업적 엔티티를 생성하게 된다면 편지 도메인과 유저 업적 도메인이 강결합을 갖게 된다.  이렇게 다른 컨텍스트에 존재하는 도메인의 결합이 된다면, 추후 편지 작성이 아닌 다른 도메인에 해당하는 행위에 대한 업적을 생성할 때도 해당하는 엔티티와 유저 업적 엔티티는 강결합을 갖게 된다. 이는 극단적으로 유저 도메인과 다른 모든 도메인이 의존관계를 갖게 되는 잠재적 위험성이 있다. 

그렇기 때문에 이러한 결합을 끊고, 스프링에서 제공하는 좋은 기능인  Domain Event를 발행해 해결한 사례를 공유합니다. 

### Domain Event란?

도메인 객체에서 어떤 작업이 실행됬을 때, 발행할 수 있는 이벤트를 의미한다. 이를 통해 객체의 생성이나 변경을 다른 객체와 결합 없이 Event Linstener를 통해 추적할 수 있다. 이를 통해 얻을 수 있는 장점은 다음과 같다.

- 서로 다른 도메인 로직이 섞일 일이 없다.
- 확장할 때 발행한 이벤트에 대해 추적하는 Listener만 추가해주면 되기 때문에 확장에 용이하다.
- 이벤트 발생 이후의 작업(이벤트 리스너의 작업)을 비동기로 처리할 수 있다.

 스프링에서 기본적으로 제공하는 [ApplicationContext는 이벤트를 발행할 수 있기 때문에](https://www.baeldung.com/spring-events#:~:text=And%20like%20many%20other%20things%20in%20Spring%2C%20event%20publishing%20is%20one%20of%20the%20capabilities%20provided%20by%20ApplicationContext.), 이를 활용해 만들어진 도메인 이벤트를 활용하는데 어렵지 않게 사용할 수 있다. 

## 코드

### AggregateRoot<A>

먼저, spring data에서 제공하는 `AbstractAggregateRoot<A>` 를 활용한다면, 도메인 이벤트를 어렵지 않게 구현할 수 있다. `AbstractAggregateRoot` 는 domain event를 간편하게 발행할 수 있도록 만든 모듈이다. 이는 이름에서 볼 수 있듯, 도메인 이벤트를 발행하는 주체는 DDD의 AggregateRoot가 된다는 의미를 내포하고 있다.

 Aggregate는 관련 객체를 하나로 묶은 군집을 의미하며, AggregateRoot는 군집 내에서 여러 객체들을 관리하는 루트 엔티티이다. 일반적으로 하나의 엔티티와 여러 개의 Value Object(값 객체)를 지니고 있으며, 하나의 Aggregate에 속한 객체는 같은 라이프사이클을 지닌다. Aggregate를 통해 간의 관계를 확인한다면, 더 상위 수준에서 도메인 간의 관계를 파악하는데 수월해진다.

 본론으로  `AbstractAggregateRoot<A>` 를 이벤트를 발행할 Entity에 상속시키면 된다. 여기서 제너릭 타입(`<A>`)은 Entity의 타입이 된다.  이를 통해 `registerEvent(event)` 메소드를 상속받을 수 있으며 이 메소드를 통해 이벤트를 등록할 수 있다. 

```java
public class AbstractAggregateRoot<A extends AbstractAggregateRoot<A>> {

	private transient final @Transient List<Object> domainEvents = new ArrayList<>();

	/**
	 * Registers the given event object for publication on a call to a Spring Data repository's save methods.
	 *
	 * @param event must not be {@literal null}.
	 * @return the event that has been added.
	 * @see #andEvent(Object)
	 */
	protected <T> T registerEvent(T event) {

		Assert.notNull(event, "Domain event must not be null!");

		this.domainEvents.add(event);
		return event;
	}

	...
}
```

이를 통해 편지 도메인 코드에 반영을 하면, 다음과 같다.

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class Letter extends AbstractAggregateRoot<Letter> {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Lob
    private String content;

    private LocalDate sendDate;

    private boolean isRead;

    @ManyToOne(fetch = FetchType.LAZY)
    private User sender;

		...

		@PostPersist
    public void created() {
        this.registerEvent(new LetterCreatedEvent(this.id, this.sender.getId(), this.sender.getUserAchievement().getSendLetterCountValue(), this.receiver.getReceiveCount()));
    }
}
```

여기서 `@PostPersist` 를 통해 이벤트를 발행했는데, 이는 JPA 엔티티의 라이프사이클에서 영속화가 된 이후에 이벤트를 등록하려 했기 때문에 다음과 같이 진행했다. 이는 서비스 계층에서 특별히 호출을 안해도 될 뿐더러, PK값 정책이 IDENTITY이기 때문에 영속화가 된 이후에 키 값을 받은 상태로 이벤트를 발행할 수 있다. 

### @EventListener

`@EventListener`를 통해 선언적으로 이벤트를 처리할 수 있다. 이를 통해 이벤트를 받는 코드를 작성하면 다음과 같다. 뿐만 아니라, 특정 조건을 만족하려 한다면 `@EventListener(condition)` 을 사용한다면 간편하게 활용할 수 있다.

```java
@Component
@RequiredArgsConstructor
public class AchievementPolicy {

    private final UserRepository userRepository;
    private final AchievementRepository achievementRepository;
    private final LetterRepository letterRepository;

    ...

    @EventListener
    public void achieveLevelTwo(LetterCreatedEvent letterCreatedEvent) {
        Long userId = letterCreatedEvent.getUserId();
        if (userRepository.existsById(userId) && letterRepository.existsById(letterCreatedEvent.getId())) {
            achievementRepository.save(new Achievement(LEVEL_TWO.getLevel(), LEVEL_TWO.getName(), LEVEL_TWO.getTag(), userId));
            userRepository.increaseUserPoint(userId, LEVEL_TWO.getPoint());
        }
    }

    @EventListener(condition = "#letterReadEvent.read == true")
    public void achieveLevelThree(LetterReadEvent letterReadEvent) {
        Long userId = letterReadEvent.getUserId();
        if (userRepository.existsById(userId) && letterRepository.existsById(letterReadEvent.getId())) {
            achievementRepository.save(new Achievement(LEVEL_THREE.getLevel(), LEVEL_THREE.getName(), LEVEL_THREE.getTag(), userId));
            userRepository.increaseUserPoint(userId, LEVEL_THREE.getPoint());
        }
    }
}
```

### 테스트

이벤트 발행에 대한 테스트는 `@*RecordApplicationEvents*`  옵션을 활용할 수 있다. 이는 단일 테스트 실행 시 발행되는 어플리케이션 이벤트를  `ApplicationEvents` 라는 객체에 저장된다. [공식 문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/context/event/RecordApplicationEvents.html)에는 다음과 같이 나타나 있다.

> @RecordApplicationEvents is a class-level annotation that is used to instruct the Spring TestContext Framework to record all application events that are published in the ApplicationContext during the execution of a single test.
> The recorded events can be accessed via the ApplicationEvents API within your tests.

이에 대한 코드를 작성하면 다음과 같다. 

```java
@RecordApplicationEvents
@SpringBootTest
@ActiveProfiles("test")
public class LetterEventTest {

    @Autowired
    private ApplicationEvents applicationEvents;

    @Autowired
    private LetterService letterService;

    @Autowired
    private UserRepository userRepository;

    private User sender;
    private User receiver;

    @BeforeEach
    void setup(){
        sender = userRepository.save(Fixtures.UserStub.defaultGoogleUser("gmail@gmail.com"));
        receiver = userRepository.save(Fixtures.UserStub.defaultGoogleUser("receiver@gmail.com"));
    }

    @Test
    void letter_created_event_test(){
        letterService.writeLetter(new LetterRequest("content", List.of()), sender);
        letterService.writeLetter(new LetterRequest("content", List.of()), sender);
        assertThat(applicationEvents.stream(LetterCreatedEvent.class).count()).isEqualTo(2);
    }
}
```

하나의 편지를 작성할 떄(DB에 편지를 저장할 떄) 정상적으로 이벤트가 발행되는 것을 볼 수 있다.



### 아쉬운 점

현재 EventListener 내 작업은 트렌젝션이 보장되어야 한다. 이에 대해 이벤트 리스너의 트랜잭션을 어떻게 처리할 지 알아보면 좋을 듯 싶다.

### 레퍼런스

[](https://www.baeldung.com/spring-data-ddd)

[](https://www.baeldung.com/jpa-entity-lifecycle-events)