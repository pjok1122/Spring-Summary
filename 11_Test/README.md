# 테스트

- 웹 서버를 실행한 후, 수동으로 하는 테스트는 테스트 자체도 번거롭고 버그를 수정하기에도 적합하지 않다. 따라서 테스트는 작은 단위에 대해서 진행해야 하는데, 이를 `단위 테스트`라고 부른다.

- `테스트`를 작성할 때에는 부정적인 케이스를 먼저 만드는 습관을 들이는 것이 좋다. 본인이 만든 프로그램에 대한 테스트 코드는 성공할만한 테스트로 작성될 확률이 높기 때문이다.

- `테스트`를 적용할 때마다, 입력 값이나 결과 값을 개발자가 확인해야 한다면 시간에 대한 낭비가 크다. 따라서 입력값은 `테스트` 내에 포함시키는 것이 좋다. 또, JUnit 이라는 자바 테스팅 프레임워크와 `AssertThat`을 사용하면 개발자는 테스트 결과 값도 손쉽게 확인할 수 있다.

- `테스트`는 몇 번을 수행해도 똑같은 결과를 반환해야 한다. 즉, `테스트`에 사용되는 데이터베이스가 존재한다면 데이터베이스의 상태가 항상 일정하게 유지되어야 한다. 그러므로 `테스트`에는 임베디드 데이터베이스를 쓰는게 좀 더 낫지 않을까 생각한다.

- 만들고자 하는 기능의 내용을 담고 있으면서 만들어진 코드를 검증도 해줄 수 있는 테스트 코드를 만들고, 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방법을 `테스트 주도 개발(TDD)`이라고 한다. TDD의 기본 원칙은 `실패한 테스트를 성공시키기 위한 목적의 코드만을 작성한다.` 라고 한다.

## JUnit 동작과정

1. 테스트 클래스에서 `@Test`가 붙은 `public`이고 `void`형이며 파라미터가 없는 테스트 메서드를 모두 찾는다.
2. 테스트 클래스의 오브젝트를 하나 생성한다.
3. `@Before`가 붙은 메소드가 있으면 실행한다.
4. `@Test`가 붙은 메소드를 하나 호출하고 테스트 결과를 저장해둔다.
5. `@After`가 붙은 메소드가 있으면 실행한다.
6. 나머지 테스트 메소드에 대해 2~5번을 반복한다.
7. 모든 테스트 결과를 종합해서 돌려준다.

즉, 테스트 메소드가 n개라면, 테스트 클래스 또한 n번 생성되고, `@Before`과 `@After`도 n번 호출된다. 이렇게 비효율적인 방식을 사용하는 이유는 `JUnit`이 각 테스트가 서로 영향을 주지 않고, 독립적으로 실행됨을 확실히 보장하기 위함이다.

## Spring Test Context

JUnit의 동작과정에서 살펴봤듯, 테스트 메소드가 n개라면 n개의 오브젝트가 생성된다. 따라서 JUnit 코드에 `ApplicationContext`를 생성하는 코드가 포함되어있다면, `ApplicationContext`를 n번 생성하게 된다. `ApplicationContext`를 생성할 때마다 싱글톤 빈을 생성해야 하므로 엄청난 리소스 낭비라고 할 수 있다.

따라서 `Test Context Framework`를 사용하는 것이 좋다. 테스트 컨텍스트는 애플리케이션 컨텍스트를 생성하여, 경로가 같은 테스트 코드에서는 이 애플리케이션 컨텍스트를 공유하도록 한다. 클래스가 다르더라도 같은 `@ContextConfiguration`의 경로가 같다면, 같은 컨텍스트를 사용한다.

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/applicationContext.xml")
public class UserDaoTest{
    @Autowired
    private ApplicationContext ac;
    ...
}
```

만약 테스트 코드에서 테스트 컨텍스트 프레임워크의 일부만을 바꿔야 한다면 `@DirtiesContext` 애노테이션을 메서드나 클래스에 붙여주면 된다.

## 슬라이스 테스트

`UserDaoTest`를 만들어 `UserDao`를 테스트하기 위해 `ApplicationContext`의 모든 빈을 등록해주어야 할까? 이런 경우 불필요한 빈들을 등록하기 때문에 테스트 성능이 오히려 나빠질 수 있다. 이런 경우, `슬라이스 테스트`를 사용하면 편리하다.

`@WebMvcTest`, `@DataJpaTest`, `@RestClientTest`, `@JsonTest` 와 같은 애노테이션을 붙여서 작성하면 된다. 애노테이션 각각이 지원하는 기능에 대해서 살펴보자.

### SpringBootTest

`@SpringBootTest`는 애플리케이션 컨텍스트와 동일하게 생성되므로, 여러 단위 테스트를 하나의 통합된 테스트로 수행할 때 적합하다.

여러가지 속성을 제공할 수 있다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest(
        properties = {
                "property.value=propertyTest",
                "value=test"
        },
        classes = {TestApplication.class},
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT
)
public class TestApplicationTests {

    @Value("${value}")
    private String value;

    @Value("${property.value}")
    private String propertyValue;

    @Test
    public void contextLoads() {
        assertThat(value, is("test"));
        assertThat(propertyValue, is("propertyTest"));
    }

}
```

- `@RunWith(SpringRunner.class)`는 테스트 컨텍스트를 사용하기 위해 반드시 붙여줘야 하는 애노테이션이다. `SpringRunner`는 `SpringJUnit4ClassRunner`를 상속받은 클래스이다.

- `properties={}`는 테스트가 실행되기 전에 `key=value` 형식의 프로퍼티를 추가할 수 있다.

- `classes={}`는 애플리케이션에 로드할 클래스를 지정할 때 사용한다. 이 값을 지정하지 않으면 `@SpringBootConfiguration`을 찾아서 로드하게 되는데, 이 경우가 애플리케이션 컨텍스트의 모든 기능을 사용하게 되는 것과 같다.

- `webEnvironment=SpringBootTest.webEnvironment.??`는 애플리케이션이 실행될 웹 환경을 지정할 수 있다. 기본값은 `Mock`인데, `RANDOM_PORT`나 `DEFINE_PORT`를 선택하면 실제 웹 서버가 구동된다.

- 프로파일 환경을 갖는다면, `@ActiveProfiles("test")`과 같은 방식으로도 사용할 수 있다.

<br>

### @WebMvcTest

MVC를 위한 슬라이스 테스트. 웹에서 테스트하기 힘든 컨트롤러를 테스트하기에 적합하다.
그 이유는 `@WebMvcTest` 애노테이션을 사용하면, MVC 관련 설정인 `@Controller`, `@ControllerAdvice`, `@JsonComponent`, `Filter`, `WebMvcConfigurer`, `HandlerMetohdAgumentResolver`가 모두 빈으로 등록되기 때문이다. 또,
`@WebMvcTest`는 `@AutoConfigureMockMvc`를 가지고 있기 때문에 `MockMvc`를 주입받아 사용할 수 있기 때문에 컨트롤러를 테스트하기가 편리하다.

```java
@RunWith(SpringRunner.class)
@WebMvcTest(SampleController.class)
public class SampleControllerTest {
    @Autowired
    MockMvc mockMvc;

    @MockBean
    SampleService mockSampleService;

    @Test
    public void hello() throws Exception {
    	when(mockSampleService.getName()).thenReturn("yj");

		mockMvc.perform(get("/hello"))
		.andExpect(status().isOk())
		.andExpect(content().string("Hello yj"))
		.andDo(print());
    }
}
```

- `@WebMvcTest(SampleController.class)`처럼 특정 클래스만 명시하는 경우에는 명시된 클래스와 관련된 빈들만 등록된다.

- `@MockBean`은 이미 존재하는 `SampleService`를 빈으로 사용하는 것이 아니라, 테스트 코드에서 정의내린(Mocking) 방식으로 동작하길 원할 때 사용한다. 여기서는 `when`을 이용해, `getName()`이 호출되면 `yj`를 리턴하라고 Mocking했다.

- '가상'의 HTTP 요청을 보낼 때에는 `mockMvc.perform()`을 이용한다. `get`이나 `post`에는 체이닝을 이용해 부가정보를 더 넘겨줄 수 있다. (`contentType`,`content` ..)

- 검증 할 때는 `.andExpect`를 사용한다. json타입을 검증할 때에는 `jsonPath("id", is("yj"))`와 같은 방식으로 검증이 가능하다.

<br>

### @DataJpaTest

`@DataJpaTest` 애노테이션은 JPA 관련된 빈들만 등록한다. `@Entity` 클래스를 스캔해서 임베디드 데이터베이스에 저장하는 것이 `default`다. `@DataJpaTest` 애노테이션을 붙이면 `DataSource`, `JdbcTemplate`, `xxxRepository`를 DI받아 사용할 수 있다.

```java
@RunWith(SpringRunner.class)
@DataJpaTest
public class UserRepositoryTest {

	@Autowired
	UserRepository userRepository;

	@Autowired
	JdbcTemplate jdbcTemplate;

	@Test
	public void addAndGet() {
		User user = new User();
		user.setName("yj");

		userRepository.save(user);
		Optional<User> byId = userRepository.findByName(user.getName());
		assertThat(byId).isNotEmpty();
		assertThat(byId.get().getId()).isEqualTo(user.getId());
	}
}
```

<br>

### @RestClientTest, @JsonTest

추후에 정리 예정.

## 학습 테스트

`학습 테스트`란 자신이 만들지 않은 프레임워크나 라이브러리 등에 대해서 테스트를 작성하며 자신이 사용할 프레임워크나 API의 사용 방법을 익히는 것이다.

- 학습 테스트를 이용하면 다양한 조건에 따라 기능이 어떻게 동작하는지 빠르게 확인할 수 있다.

- 학습 테스트 코드를 개발 중에 참고할 수 있다.

- 프레임워크나 제품을 업그레이드할 때 호환성을 검증할 수 있다.

- 테스트 작성에 대한 좋은 훈련이 된다.

- 새로운 기술을 공부하는 과정이 지루하지 않다.

**JUnit 학습 테스트**

```java
public class JUnitTest{
    static JUnitTest testObj;

    @Test public void test1(){
        assertThat(this, is(not(sameInstance(testObj))));
        testObj = this;
    }
    @Test public void test2(){
        assertThat(this, is(not(sameInstance(testObj))));
        testObj = this;
    }
    @Test public void test3(){
        assertThat(this, is(not(sameInstance(testObj))));
        testObj = this;
    }
}
```

## 버그 테스트

`버그 테스트`란 코드에 오류가 있을 때 그 오류를 가장 잘 드러내줄 수 있는 테스트를 말한다. 무턱대고 코드를 수정하려고 하기보다는 먼저 버그 테스트를 만들어보는 편이 유용하다.

## 요약 정리

- `테스트는 자동화`돼야 하고, 빠르게 실행할 수 있어야 한다.
- main() 테스트 대신 JUnit 프레임워크를 이용한 테스트 작성이 편리하다.
- `테스트 결과는 일관성`이 있어야 한다. 환경이나 테스트 실행 순서에 따라서 결과가 달라지면 안된다.
- 테스트는 포괄적으로 작성해야 한다. `충분한 검증`을 하지 않은 테스트는 테스트를 안하는 것보다 나쁘다.
- 코드 작성과 테스트 수행의 간격이 짧을수록 효과적이다.
- 테스트하기 쉬운 코드가 곧 좋은 코드다.
- 테스트를 먼저 작성하고 테스트를 성공시키는 코드를 만들어가는 `테스트 주도 개발` 방법도 유용하다.
- 테스트 코드도 `적절한 리팩토링`이 필요하다.
- `@Before`, `@After`를 사용해서 테스트 메소드들의 공통 작업을 수행할 수 있다.
- 스프링 테스트 컨텍스트 프레임워크를 사용하면 테스트 성능을 향상시킬 수 있다.
- 동일한 설정파일을 사용하는 테스트는 하나의 애플리케이션 컨텍스트를 공유한다.
- `@Autowired`를 이용하면 컨텍스트의 빈을 테스트 오브젝트에 DI할 수 있다.
- 기술의 사용 방법을 이기고 이해를 돕기 위해 `학습 테스트`를 작성하자.
- 오류가 발견될 경우 그에 대한 버그 테스트를 만들어두면 유용하다.
