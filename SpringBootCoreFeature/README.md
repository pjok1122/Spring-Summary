# 스프링부트 핵심 기능

## SpringApplication

### 기본 로그 레벨 INFO

`application.properties`에 Debug의 값을 True로 설정하면 쉽게 Debug 모드로 설정할 수 있다.

### 배너

resource 폴더 밑에 `banner(.txt | .jpg | .png)` 라는 파일을 만들고 실행하면 Application이 실행될 때 내가 지정한 배너로 변경된다. 위치를 변경하고 싶다면, `application.properties`에 `spring.banner.location` 값을 변경해주면 된다. 배너에서는 \${spring.boot.version} 등의 변수를 사용할 수도 있다.

### EventListener 등록

이미 `ApplicationContextDetail` 부분에서 EventListener를 등록하는 부분을 살펴봤다. `ApplicationListener<T>`를 구현하는 방법과 `@EventListener` 애노테이션을 사용하는 방법이 존재한다.

이 두 방법은 모두 애플리케이션 리스너가 빈으로 등록되어 사용되는 형태다. 하지만, `Application Context`가 생성되기 전에 실행되는 이벤트는 어떨까? 당연히 빈으로 등록이 불가하므로 위 방법으로 동작하지 않는다.

이 경우에는 `SpringApplication` 객체를 커스터 마이징 해야한다.

```java
@SpringBootApplication
public class Ex3SpringInitApplication {

	public static void main(String[] args) {
		SpringApplication app = new SpringApplication(Ex3SpringInitApplication.class);
		app.addListeners(new SampleEventListener());
		app.run(args);
	}
}
```

### WebApplicationType 설정

WebApplicationType은 Spring MVC를 사용한다면 자동으로 Servlet 타입으로 등록된다. 이 값을 변경하고 싶다면 다음과 같이 설정하면 된다.

```java
@SpringBootApplication
public class Ex3SpringInitApplication {

	public static void main(String[] args) {
		SpringApplication app = new SpringApplication(Ex3SpringInitApplication.class);
		app.setWebApplicationType(WebApplicationType.NONE);
		app.run(args);
	}
}
```

또는 application.properties 설정에 다음과 같이 작성.

`spring.main.web-application-type=none`

### Application Arguments 사용

Application Arguments는 `--`를 붙여서 전달하면 된다. (참고로 JVM에 넘겨주는 인자는 `-D`를 붙여서 사용한다.) 이 값을 애플리케이션에서 읽어들이고 싶을 때에는 ApplicationArguments를 사용하면 된다. 이 객체는 빈으로 등록되어 있어 손쉽게 사용이 가능하다.

```
java -jar ex3_SpringInitApplication-0.0.1-SNAPSHOT-Dfoo --bar
```

```java
@Component
public class ApplicationArgumentsReader {
	public ApplicationArgumentsReader(ApplicationArguments args) {
		System.out.println(args.containsOption("foo"));
		System.out.println(args.containsOption("bar"));
	}
}
```

ApplicationArguments가 Bean으로 등록되어있기 때문에 생성자에 주입하여 사용이 가능한 모습이다.

위의 콘솔 명령어에서 -D옵션으로 넘겨준 경우 JVM 인자이므로 Application Arguments가 아니다. 따라서 bar를 포함하고 있냐는 명령어에만 True를 반환한다.

### ApplicationRunner

애플리케이션을 실행한 뒤에 뭔가를 더 실행하고 싶을 때 ApplicationRunner를 구현하고 빈으로 등록하면 된다. 만약 실행하고 싶은 객체가 다양하다면, `@Order`를 통해 순서를 제어할 수 있고, 비동기로 실행하고 싶다면 `@Async`를 사용하면 된다. 단, `@Async`를 사용하기 위해서는 별도의 설정이 더 필요하다.

```java
@Component
@Order(1)
public class AppRunner implements ApplicationRunner {

	@Override
	public void run(ApplicationArguments args) throws Exception {
		System.out.println("ApplicationRunner!!");
		System.out.println(args.containsOption("bar"));
	}
}
```

ApplicationRunner의 run() 메소드는 `ApplicationArguments`를 인자로 받고 있어, 위와 같은 명령어가 가능하다.

[참고자료](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-application)

<hr>

## 외부 설정

애플리케이션의 값들을 외부에서 설정할 수 있는 방법들이 여러가지가 있다. `properties` 파일, `커맨드 라인 아규먼트`, `환경 변수`, `YAML` 정도가 외부에서 설정하는 정보들이다. 이 중에서 특히 `properties`를 가장 많이 사용한다.

### property 우선순위

1. 유저 홈 디렉토리에 있는 spring-boot-dev-tools.properties
2. **테스트에 있는 @TestPropertySource**
3. **@SpringBootTest 애노테이션의 properties 애트리뷰트**
4. **커맨드 라인 아규먼트**
5. SPRING_APPLICATION_JSON (환경 변수 또는 시스템 프로티) 에 들어있는
   프로퍼티
6. ServletConfig 파라미터
7. ServletContext 파라미터
8. java:comp/env JNDI 애트리뷰트
9. System.getProperties() 자바 시스템 프로퍼티
10. OS 환경 변수
11. RandomValuePropertySource
12. JAR 밖에 있는 특정 프로파일용 application properties
13. JAR 안에 있는 특정 프로파일용 application properties
14. JAR 밖에 있는 application properties
15. **JAR 안에 있는 application properties**
16. @PropertySource
17. 기본 프로퍼티 (SpringApplication.setDefaultProperties)

#### 예시

- 우선순위 3

```java
@RunWith(SpringRunner.class)
@SpringBootTest(properties="user.name=youngjae")
```

- 우선순위 2

```java
@RunWith(SpringRunner.class)
@TestPropertySource(properties="user.name=youngjae")
@SpringBootTest
```

- 우선순위 2 (별도의 properties 파일 설정)

```java
@RunWith(SpringRunner.class)
@TestPropertySource(locations="classpath:/test.properties")
@SpringBootTest
```

### application.properties의 우선순위

1. file:./config/
2. file:./
3. classpath:/config/
4. classpath:/

기본적으로 생성되는 application.properties는 4번에 해당한다.

_cf) test 폴더 밑에 `resource/application.properties`를 생성하는 경우, jar 파일 생성할 때 src/`main/resource/application.properties`를 덮어써서 빌드한다. 자주 등장하는 버그.._

### application.properties 사용

단순히 key=value 형태로 작성하면 된다. 아무런 설정없이 `random` 과 관련된 함수를 사용할 수 있으며, 이전에 설정한 프로퍼티를 재사용할 수 있는 **플레이스 홀더** 기능도 있다.

```
user.name = Youngjae
user.favorite-number = ${random.int(0,10)}
user.full-name = ${user.name} Park
```

<hr>

### 타입-세이프 프로퍼티 @ConfigurationProperties

`application.properties`에서 여러 프로퍼티를 묶어서 다른 빈에 주입할 수 있다.

```*
user.name = Youngjae
user.favorite-number = ${random.int(0,10)}
user.full-name = ${user.name} Park
```

이렇게 정의된 properties를 `UserProperties`라는 빈으로 바인딩 시킬 수 있다는 의미이다. `@Component`와 `@ComfigurationProperties("prefix")`를 애노테이션으로 붙이고, field의 이름을 application.properties와 동일하게 작성하면 된다.

_cf) application properties에서는 카멜, 스네이크, 케밥 어떤 표기법을 사용하더라도 상관없이 데이터 바인딩 된다._

```java
@Component
@ConfigurationProperties("user")
public class UserProperties {

	private String name;
	private int favoriteNumber;
	private String fullName;

    // ... (Getter & Setter)
```

### 컨버젼과 유효성 검사

application.properties는 전부 String 형태로 작성되어있는데, 별도의 작업없이 UserProperties의 int 타입으로 데이터 바인딩된다는 것을 확인할 수 있다. 따라서 내부적으로 `ConversionService`가 작동하고 있음을 알 수 있다.

스프링부트는 추가적으로 시간에 대한 컨버팅 기능을 제공한다. `@DurationUnit(ChronoUnit.SECONDS)` 같은 애노테이션을 붙여줘서 컨버팅할 수도 있지만, `application.properties`에 suffix를 붙이는 방법이 훨씬 쉽다.

```
user.session-timeout=60s
```

지원하는 suffix로는 `ns`, `us`, `ms`, `s`, `m`, `h`, `d` 이 있다.

_\* 참고) nano second부터 day를 의미한다._

[참고 자료](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-external-config)

<hr>

## 프로파일

프로파일은 IoC-Container에서도 한 번 살펴봤다. [참고](../IoC-Container/README.md)

프로파일은 런타임 환경마다 빈 설정파일을 다르게 줄 수 있는 기능이다.

### @Profile

특정 프로파일에만 실행되도록 설정할 때 `@Profile("이름")` 애노테이션을 붙여주기만 하면 된다. 이 애노테이션을 사용하는 곳은 `@Configuration` 과 `@Component`가 붙은 곳이다. 즉, 컴포넌트 스캔 대상에다가 붙여 사용한다.

### 프로파일 활성화

프로파일 활성화 하는 방법은 `spring.profiles.active` 라는 프로퍼티를 활용하면 된다.

- `application.properties`에 프로퍼티 설정하는 방법
- `-Dspring.profiles.active=prod`
- `--spring.profiles.active=prod`

### 프로파일 Include

프로파일 내에서 프로파일을 추가할 수 있다.

`spring.profiles.include=test` 라고 프로퍼티를 줬을 때, `resource/application-test.properties`가 추가된다.
test 프로파일의 빈 또한 사용 가능하다.

### 프로파일용 Properties

프로파일마다 Properties를 적용할 수 있다. 예를 들어, 현재 활성된 프로파일이 `prod`라고 하자. 이때 `application-prod.properties`가 정의되어있다면, 이 프로퍼티를 사용한다. 만약 application.properties와 application-prod.properties에 동일한 프로퍼티가 정의되어있다면 prod 프로퍼티가 우선순위가 더 높다.

- `application-{profile}.properties`

<hr>

## 로깅

## 테스트

## Spring-Dev-Tools
