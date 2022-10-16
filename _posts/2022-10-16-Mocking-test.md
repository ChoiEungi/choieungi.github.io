---

layout: post

title: "[Testing] Mockito를 이용한 Service Layer Unit Testing"

tags: [Testing]

date: 2022-09-07 02:02:00 +0900

categories: [Testing]

---



# Testing에 관한 고찰

일반적으로 스프링으로 테스트를 작성할 때, `@SpringBootTest`를 이용해 통합테스트와 인수테스트를 진행했습니다. 이를 이용해 테스트를 진행하면, 개별적으로 테스트를 실행하기에도, 전체를 테스트를 실행하기에도 너무 속도가 느리다는 단점이 있었습니다. 뿐만 아니라, 테스트 간의 격리성을 확보하기 위해서 모든 데이터를 지우는 과정에서 `DataIntegrityException` 이 자주 발생해 어려움이 있었습니다. 

 이를 위해서 격리가 쉽고 빠른 테스트를 진행할 수 있는 테스트 방법에 대해 알아보았고, 마침 Mock을 이용한 Service Layer에 대한 단위테스트를 진행해보고 느낀 경험을 공유합니다.

### Mockito를 이용한 Service Layer의 Unit Test

저희의 ArgumentResolver에서 로그인한 유저를 조회하기 위해 사용하는 Service의 메소드 중 하나는 다음과 같습니다. jwtProvider를 통해 decode를 진행한 후에, email을 통해 user를 조회하게 됩니다.

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final JWTProvider jwtProvider;

		...

		public User loginUser(String token) {
		    String email = jwtProvider.decodeJWTToSubject(token);
		returnuserRepository.findUserByEmail(email).orElseThrow(NoSuchRecordException::new);
		}
}
```

이를 `Mockito`를 사용해 Service Layer의 의존성을 격리해 테스팅을 진행하면 코드는 다음과 같습니다.

```java
@ExtendWith(MockitoExtension.class)
public class UserServiceMockTest {

    private static final String ACCESS_TOKEN = "accessToken";
    private static final String EMAIL = "swm.team.goyukkuri@gmail.com";
    private static final String TOKEN = "token";

    @Mock
    private UserRepository userRepository;

    @Mock
    private JWTProvider jwtProvider;

    @InjectMocks
    private UserService userService;

		private User googleUser;

    @BeforeEach
    void setup(){
        googleUser = Fixtures.UserStub.defaultGoogleUser(EMAIL);
    }

		...

		@Test
		void user_google_login_test() {
		    when(userRepository.findUserByEmail(anyString())).thenReturn(Optional.of(googleUser));
		    when(jwtProvider.decodeJWTToSubject(TOKEN)).thenReturn(EMAIL);
		
		    User user = userService.loginUser(TOKEN);
				
		    assertThat(user.getEmail()).isEqualTo(EMAIL);
		    verify(userRepository, times(1)).findUserByEmail(anyString());
		    verifyNoMoreInteractions(userRepository);
		}
}
```

`UserService`를 위한 의존관계인 `UserRepository`와 `jwtProvider`를 주입하려면, 먼저 Mocking을 진행해야 합니다. `@Mock`를 통해 가짜 빈을 넣어줄 수 있으며, 실제 구현된 객체와 무관하게 작동하게 됩니다. 다시말해 userRepository의 메서드들이 껍데기만 존재할 뿐, 구현체가 없게 됩니다. 만약 실제 객체를 주입하고 싶으면 `@Spy` 를 이용하면 됩니다. 

 의존 관계를 다 Mocking을 했다면, `@InjectMock` 을 통해 의존관계를 주입할 수 있습니다. 이를 통해 격리의 주체를 userService로만 둘 수 있게 되고, 다른 실제 객체들의 의존성이 모두 제거됩니다.

 이후 `when()` 을 이용해서 Mock 객체의 메소드의 결과를 어떻게 설정할 것인지 정해준 후,  `userService.loginUser` 메소드를 실행하게 됩니다. 만약 `jwtProvider` 혹은 `userRepository` 같은 Mock 객체의 메소드를 정의를 안해준 것이 있다면, 실행할 수 없게 됩니다. 

 이후 `verify` 를 통해 Mock 객체의 행위에 대해 검증해볼 수도 있습니다. 뿐만 아니라, `when()` 으로 설정한 것 이외의 행위가 있었는 지도 `verifyNoMoreInteractions` 를 통해 확인해볼 수 있었습니다.

### Mock Unit Test의 장점

1. 속도

SpringBootTest보다 압도적으로 속도가 빠르다는 점은 비교 불가한 장점이었습니다. 이를 통해 피드백을 더 빨리 받을 수 있었고, 빌드 시간도 단축할 수 있을 것이라는 생각이 들었습니다. 이는 곧 개발 생산성 향상으로 이어질 것입니다.

2. unit testing을 통해 코드의 품질 확인 

 Mock을 사용해 다른 의존성들을 테스트 대역으로 사용하니, 내부 구현에 대해서 모두 Mocking을 진행해야 했습니다. 이는 내부 구현을 한번 더 확인해보게 되었으며, unit testing이 어려워지는 객체는 Mocking을 많이 진행해야 하고 고려해야 할 부분이 많아졌습니다. 이는, 메소드의 결합도가 높아졌다는 것을 알 수 있었습니다. 이는 통합테스트만으로는 알기 어려웠습니다. 

3. 상대적으로 편리한 테스트 격리 

 Mockito를 통해 빈으로 주입받는 의존성을 모두 Mocking을 통해 테스트 대역을 편리하게 만들 수 있었습니다. `@SpringBootTest`도 `@MockBean` 을 통해 가능했지만, 이는 통합테스트 환경에서 특별한 의존성이 아닌 것들을 Mocking을 하는 것은 시스템 간의 상호작용을 확인하기 어려워지기 때문에 적절치 못하다고 생각했습니다.

### Mock Unit test의 아쉬운 점

1. 구현 세부에 대해 굉장히 잘 알아야하며, 유지 보수에 대한 부담이 커집니다.

모든 repository가 무엇이 사용되는 지 알아야 하고 이에 대한 return 값을 일일히 정해줘야 합니다. 이 부분은 관리 포인트가 많아진다는 단점을 지니고 있습니다. 뿐만 아니라, 인가 정책이 바뀌어 jwt가 아닌 session으로 바뀐다면, 테스트에 jwtProvider 절을 변경해야 할 것입니다. 이는 통합테스트에 비해 변경에 취약하게 됩니다.

2. 영속성 전이 테스트의 어려움 

저희 프로젝트의 도메인 서비스 로직에서 프로필 정보를 입력하는 다음과 같은 로직이 존재합니다.

```java
@Transactional
public voidupdateUserProfile(User user, ProfileRequest request) {
    Profile profile =newProfile(request.getNickname(), request.getGender(), request.getAge(), request.getJob(), user);
    profile.addPsychologicalExam(request.getPsychologicalExams());
    user.updateProfile(profile);
}
```

 본 로직은 영속성 전이(CASCADE.ALL)를 통해 프로필 정보에 대한 생명주기를 하나로 뒀습니다. 이는 Repository를 구현해 save를 하는 것은 객체 지향적이지 못하다고 생각해 위와 같이 구현했습니다. 이 같은 경우, 실제 쿼리가 어떻게 나가는 지에 대해 확인이 필요하다고 생각합니다. 다만, Mockito를 통해 unit testing을 진행한다면, 영속성 전이가 잘 이뤄졌는지에 대해서 확인이 어려울 것입니다. 뿐만 아니라, 전반적인 트랜잭션에 대한 테스트도 어려울 듯 싶습니다.

### 결론

결론적으로 은탄환은 없다고, Mocking을 이용한 방법이 속도는 빠르지만 여러 단점이 존재했던 것 같습니다. 결국 각각의 장단점을 명확하게 인식하고 문제 상황에 맞게 조직에서 합의한 기준을 바탕으로 테스트를 잘 작성하는 것이 중요한 것 같습니다. 마지막으로, Mocking에 대한 여러 견해가 존재하는데 특히 테스트의 고전파와 런던파에 대한 [마틴 파울러의 글](https://martinfowler.com/articles/mocksArentStubs.html)(본 글의 테스트는 런던파에 해당합니다)을 읽어보는 것도 좋을 듯 합니다.

Mock Test 작성에 도움이 된 글

[Unit Testing the Service Layer of Spring boot Application](https://1kevinson.com/testing-service-spring-boot/)

책

블라디미르 코리코프, 단위테스트, 생산성과 품질을 위한 단위테스트 원칙과 패턴