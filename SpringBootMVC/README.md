## HttpMessageConverter

- HTTP 요청 본문을 객체로 변경하거나, 객체를 HTTP 응답 본문으로 변경할 때 사용한다.
- 스프링부트는 `HttpMessageConvertersAutoConfiguration`을 제공하고있어, 별도의 설정없이 HttpMessageConverter를 사용할 수 있다.
- {"username":"youngjae", "password":"1234"} <-> User Object

**예시코드**

```java
public class User {
	private Long id;
	private String username;
	private String password;

    /* Getter & Setter */
}
```

```java
@RestController
public class UserController{
    @GetMapping("/hello")
	public @ResponseBody String hello() {
		return "hello";
    }
}
```

- `@RestController`를 사용하는 경우, return 타입에 `@ResponseBody` 애노테이션을 생략할 수 있다.

- `@ResponseBody`는 return 하는 값이 ViewResolver에 의해 물리적인 View 객체로 변환하는 것이 아니라, 반환하는 값이 String임을 의미한다.

- `String Message Converter`는 ResponseBody에 "hello"라는 값을 담아 전송한다.

<br>

```java
@RestController
public class UserController{
    @PostMapping("/users/create")
	public User user create(@RequestBody User user) {
		return user;
    }
}
```

- `@RequestBody` 애노테이션이 사용되면, `HttpMessageConverter`를 구현한 `JsonMessageConverter`가 request body에 있는 데이터를 `User`라는 객체에 바인딩해준다.

- 리턴 값은 `user`이므로 `HttpMessageConverter`가 클라이언트가 원하는 미디어 타입으로 변환해서 응답한다.

**테스트 코드**

```java
@Test
public void createUser_JSON() throws Exception {
    String userJson="{\"username\":\"youngjae\", \"password\":\"1234\"}";
    mockMvc.perform(post("/users/create")
            .contentType(MediaType.APPLICATION_JSON_UTF8)
            .accept(MediaType.APPLICATION_JSON_UTF8)
            .content(userJson))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.username", is(equalTo("youngjae"))))
    .andExpect(jsonPath("$.password", is(equalTo("1234"))));
}
```

### XML 메시지 컨버터 추가하기

- XML에 대한 컨버팅 기능은 디폴트로 지정되어있지 않다. 따라서 별도의 XML 컨버터를 추가해줘야 한다.

```xml
<dependency>
    <groupId>com.fasterxml.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactid>
    <version>2.9.6</version>
</dependency>
```

### ContentNegotiatingViewResolver

`ContentNegotiatingViewResolver`는 뷰 리졸버를 결정하는 뷰 리졸버의 역할을 수행한다고 했다. 간단히 동작과정을 상기해보자.

- 모든 뷰 리졸버를 호출해서 적용이 가능한 모든 뷰를 찾는다. 그 중에서 Http Request가 원하는 미디어 타입의 뷰를 선택한다.

위에서 살펴본 `HttpMessageConverter`는 이 `ContentNegotiatingViewResolver`의 `미디어 타입 결정 방법`을 사용한다. 즉, `HttpMessageConverter`는 `ContentNegotiatingViewResolver`와 함께 동작한다.

스프링 부트는 `WebMvcAutoConfiguration`을 제공하므로 `ContentNegotiatingViewResolver`를 별도의 설정없이 사용할 수 있다.

#### 미디어 타입 결정

1. URL의 확장자 타입을 이용한다. ex) /hello.json
2. 포맷 파라미터를 확인한다. ex) /hello?format=pdf
3. HTTP Accept 헤더를 확인한다.
4. defaultContentType 프로퍼티에서 설정해준 디폴트 미디어 타입을 사용한다.

여기서 1번은 더 이상 스프링에서 지원하지 않는 방식이다.

<br><hr>

## 정적 리소스 지원

### 기본 리소스 위치

- classpath:/static
- classpath:/public
- classpath:/resources/
- classpath:/META-INF/resources

ex) URL : "/hello.html" 이고, /static/hello.html 에 Resource가 있다면 매핑된다.

- `index.html`는 웰컴 페이지이므로 "/" 에 매핑된다.
- `favicon.ico`을 리소스 핸들러가 관리하는 패스에 저장해두면 파비콘을 변경할 수 있다.

<br>

### 캐싱

정적 리소스는 캐싱이 가능하다. `프록시 서버`에서 하는 캐싱이 있고, `브라우저 내`에서 사용하는 캐싱이 있다.

클라이언트는 처음 정적 리소스를 받을 때, 리스폰스 헤더에 `Last-Modified` 라는 필드를 받는다. 이 필드에는 마지막으로 리소스가 갱신된 날짜가 적혀있다.

클라이언트가 이후 똑같은 리소스를 요청할 때, 리퀘스트에 `If-Modified-Since` 헤더 필드를 담아 전송한다. 이 헤더 필드는 이 날짜 이후에 데이터가 갱신되었다면 돌려달라는 의미이다. 데이터가 갱신된 경우, 서버는 갱신된 데이터를 다시 전송한다. 하지만 데이터가 갱신되지 않은 경우, 리소스를 전송하지 않고 `응답코드 304 Not Modified`를 전송한다.

### addResourceHandler

기본 리소스 위치 네 가지 말고도 추가적인 리소스 핸들러를 정의할 수 있다.

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer{
	@Override
	public void addResourceHandlers(ResourceHandlerRegistry registry) {
		registry.addResourceHandler("/m/**")
		.addResourceLocations("classpath:/m/")
		.setCachePeriod(0);
	}
}
```

- `addResourceHandlers`는 기존에 정의되어있는 네 가지 핸들러는 그대로 사용하되, 핸들러를 추가할 때 사용한다.

- `"/m"` 으로 시작하는 url에 대해서는 `ResourceLocations`에 명시된 경로에서 찾는다.
-
- `addResourceLocation()`의 인자는 반드시 "/"로 끝내야 한다.
- `setCachePeriod`를 0으로 셋팅하면 캐싱 기능을 사용하지 않는다. 옵션을 주지않을 경우, 마지막 갱신날짜를 가지고 캐싱을 결정한다. 인자의 단위는 초(s)이다.

<br><hr>

## 웹 JAR

`WebJars`는 클라이언트에서 사용되는 웹 라이브러리(jquery, bootstrap, react, Vue)를 JAR 파일 안에 패키징한 것이다.

webjars 를 사용하기 위해서는 http://www.webjars.org/ 접근하여 제공하는 라이브러리를 탐색하여 사용하면 된다. 원하는 라이브러리 이름을 검색하고, Maven이나 Gradle 자신이 사용하는 의존성 관리방법을 선택해 의존성을 추가한다.

**예시코드**

- `pom.xml`에 의존성 추가하기.

```xml
<!-- https://mvnrepository.com/artifact/org.webjars.bower/jquery -->
<dependency>
    <groupId>org.webjars.bower</groupId>
    <artifactId>jquery</artifactId>
    <version>3.4.1</version>
</dependency>
```

- `webjars`로부터 jquery 로드하기. (매핑 : "`/webjars/**`")

```html
<body>
  Hello Static Resource!
  <script src="webjars/jquery/3.4.1/dist/jquery.min.js"></script>
  <script>
    $(function() {
      alert("Hi jQuery!");
    });
  </script>
</body>
```

### webjars-locator-core 의존성 추가

위의 예시를 보면 알겠지만, 사용하고자 하는 라이브러리의 `version`을 직접 명시하고 있다. 만약, 사용하는 라이브러리의 버전이 바뀌면 어떨까? 일일이 타이핑해서 수정해야하기 때문에 상당히 곤욕스럽다.

이를 대신해주는 jar 파일이 `webjars-locator-core` 다.

`mvnrepository.com`에 접속해서 위의 예시처럼 의존성을 찾고 추가하면 된다.

**예시코드**

- pom.xml에 의존성 추가하기.

```xml
<!-- https://mvnrepository.com/artifact/org.webjars/webjars-locator-core -->
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>webjars-locator-core</artifactId>
    <version>0.43</version>
</dependency>
```

- 버전을 생략하고 사용할 수 있다. (동작 과정은 잘 모르겠다.. 추후에 필요하면 찾아볼 예정.)

```html
<script src="webjars/jquery/dist/jquery.min.js"></script>
```

<br><hr>

## 템플릿 엔진

템플릿 엔진은 주로 뷰를 만드는 데에 사용한다. 하지만 그 기능 외에도 코드 제너레이션과 같은 곳에서도 사용이 가능하다.

- FreeMarker
- Groovy
- **Thymeleaf**
- Mustache

### JSP를 권장하지 않는 이유

- JSP는 JAR 패키징 할 때는 동작하지 않고, WAR 패키징을 해야 한다. 이는 SpringBoot가 추구하는 `독립적인 실행`과 대립된다.
- 비교적 최신 서블릿 컨테이너인 Undertow는 JSP를 지원하지 않는다.
- JSP는 서블릿 엔진이 템플릿을 생성하기 때문에 웹 서버를 띄우지 않고서는 테스트하기가 어렵다. 즉, 테스트코드에서 템플릿 본문을 확인하기가 어렵다.

### Thymeleaf 사용하기

- 의존성 추가

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

<br>

- 템플릿 파일 위치 : `</src/main/resources/template/>`

<br>

- xml namespace 지정한 후 html 파일 작성.

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
  <head>
    <meta charset="UTF-8" />
    <title>Insert title here</title>
  </head>
  <body>
    <h1 th:text="${name}">Name</h1>
  </body>
</html>
```

Thymeleaf에 대한 문법은 블로그에 정리할 예정. (정리가 끝나면 링크 첨부)

<br>

### HTML 테스트 코드

- `Thymeleaf`를 MockMvc로 테스트하는 코드

```java
@RunWith(SpringRunner.class)
@WebMvcTest(SampleController.class)
public class SampleControllerTest {

	@Autowired
	MockMvc mockMvc;

	@Test
	public void hello() throws Exception {
		// 요청 : "/hello"
		// 응답
		// - 모델 name : youngjae
		// - 뷰 name : hello
		mockMvc.perform(get("/hello"))
		.andExpect(status().isOk())
		.andExpect(model().attribute("name", is("youngjae")))
		.andExpect(view().name("hello"))
		.andExpect(content().string(containsString("youngjae")));
	}
}
```

`HtmlUnit`을 사용하면 템플릿 뷰 테스트를 보다 전문적으로 수행할 수 있다.

### HtmlUnit 테스트 코드

- 의존성 추가

```xml
<dependency>
  <groupId> org.seleniumhq.selenium </groupId>
  <artifactId> htmlunit-driver </artifactId>
  <scope> test </scope>
</dependency>
<dependency>
  <groupId> net.sourceforge.htmlunit </groupId>
  <artifactId> htmlunit </artifactId>
  <scope> test </scope>
</dependency>
```

**테스트 코드**

```java
@RunWith(SpringRunner.class)
@WebMvcTest(SampleController.class)
public class SampleControllerTest {

	@Autowired
	WebClient webClient;

	@Test
	public void hello() throws Exception {
		HtmlPage page = webClient.getPage("/hello");
		HtmlHeading1 h1 = page.getFirstByXPath("//h1");
		assertThat(h1.getTextContent()).isEqualToIgnoringCase("youngjae");
	}
}
```

<br><hr>

## ExceptionHandler

스프링 MVC의 기본 예외 처리 방법은 `@ControllerAdvice` 애노테이션을 사용하는 것이다.

**예시코드**

```java
@ControllerAdvice(basePackageClasses = AcmeController.class)
public class AcmeControllerAdvice extends ResponseEntityExceptionHandler {

    @ExceptionHandler(YourException.class)
    @ResponseBody
    ResponseEntity<?> handleControllerException(HttpServletRequest request, Throwable ex) {
        HttpStatus status = getStatus(request);
        return new ResponseEntity<>(new CustomErrorType(status.value(), ex.getMessage()), status);
    }

    private HttpStatus getStatus(HttpServletRequest request) {
        Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
        if (statusCode == null) {
            return HttpStatus.INTERNAL_SERVER_ERROR;
        }
        return HttpStatus.valueOf(statusCode);
    }

}
```

만약 `Controller` 클래스 내에서 발생한 에러를 처리하기 위해서는 `@Controller` 애노테이션이 붙은 클래스 밑에 `@ExceptionHandler("Exception")`가 붙은 메서드를 정의하면 된다.

```java
@Controller
public class SampleController {
  @GetMapping("/hello")
	public String hello() {
		throw new MyException();
	}

	@ExceptionHandler(MyException.class)
	public @ResponseBody AppError myError(MyException e) {
		AppError appError = new AppError();
		appError.setMessage(e.getMessage());
		appError.setReason("I Don't Know");
		return appError;
	}
}
```

### BasicExceptionHandler

스프링부트에서 제공해주는 기본 예외 처리기는 `BasicExceptionHandler`이다. 이 객체는 `Controller`이다. 이 객체에는 에러 메시지를 `HTML` 또는 `JSON`으로 응답해주는 기능이 있다.

### 커스텀 에러 페이지

상태 코드에 따라서 보여주는 뷰 페이지를 변경할 수 있다.

`src/main/resources/static/error` `src/main/resources/template/error` 밑에 `404.html`, `5xx.html`과 같이 상태코드로 지정이 가능하다.

매핑이 복잡한 경우에는 `ErrorViewResolver`를 직접 구현하는 Bean을 추가할 수도 있다.

**ErrorViewResolver**

```java
public class MyErrorViewResolver implements ErrorViewResolver {

    @Override
    public ModelAndView resolveErrorView(HttpServletRequest request,
            HttpStatus status, Map<String, Object> model) {
        // Use the request or status to optionally return a ModelAndView
        return ...
    }
}
```

<br><hr>

## HATEOAS

`Hypermedia As The Engine Of Application State`의 약자.

- 서버 : 현재 리소스와 연관된 링크 정보들을 클라이언트에게 제공한다.
- 클라이언트 : 연관된 링크 정보를 바탕으로 리소스에 접근한다.
- 연관된 링크 정보란 `Relation`과 `Hypertext Reference`를 의미한다.

**HATEOAS가 적용된 JSON**

```json
{
  "content": "Hello, World!",
  "_links": {
    "self": {
      "href": "http://localhost:8080/greeting?name=World"
    }
  }
}
```

HATEOAS를 적용하는 것은 굉장히 귀찮은 일이지만, 엄밀히 말해서 `REST API`라고 불리우기 위해서는 `HATEOAS`를 만족해야 한다.

### Spring HATEOAS

Spring boot는 `spring-boot-starter-hateoas`를 추가하면 `ObjectMapper`, `LinkDiscovers`를 제공한다.

- ObjectMapper : 객체를 json으로 바꿔주는 기능을 제공
- LinkDiscovers : 클라이언트 쪽에서 링크 정보를 Rel 이름으로 찾을 때 사용할 수 있는 XPath 확장 클래스

**예시 코드**

- 의존성 추가

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

- Entity 생성할 때, `RepresentationModel<Hello>`를 상속받아 구현한다.

```java
public class Hello extends RepresentationModel<Hello> {

	private String msg;
	private String name;

	public String getMsg() {
		return msg;
	}
	public void setMsg(String msg) {
		this.msg = msg;
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}

}
```

- Controller에서 \_link 필드를 삽입한다.

```java
@RestController
public class SampleController {

	@GetMapping("/hello")
	public Hello hello() {
		Hello hello = new Hello();
		hello.setMsg("hi~");
		hello.setName("youngjae");
		hello.add(linkTo(methodOn(SampleController.class).hello()).withSelfRel());
		return hello;
	}
}
```

HATEOAS를 구현하는 방법은 다양하나, 쉬운 방법을 선택해 구현했다. `linkTo` 메서드가 존재하는 `org.springframework.hateoas.server.mvc.WebMvcLinkBuilder` 클래스가 `Deprecated` 됐으므로 이제는 다른 방법으로 구현해야 할 것같다.

## CORS

`CORS(Cross Origin Resource Sharing)`는 Origin이 다른 서버에서 자원을 요청하고 공유할 수 있도록 하는 표준 기술이다.

### Origin

여기서 말하는 `Origin`이란 `스키마(http) + 호스트명(localhost) + 포트번호(8080)`를 조합한 것이다. 예를 들어, `http://localhost:8080`은 하나의 Origin이 된다.

### Single Origin Policy

`동일 출처 정책(Single Origin Policy)`는 CORS 같은 상황이 발생했을 때, 외부서버에 요청한 데이터를 브라우저에서 보안 목적으로 차단하는 정책이다. 즉, 보안 상의 이유로 서로 다른 `Origin`에서는 데이터를 받아올 수 없도록 브라우저에서 차단하고 있다. 만약 서로 다른 `Origin`에서 데이터를 주고 받고 싶다면, `Access-Control-Allow-Origin` 헤더를 설정해야 한다.

### @MVC @CrossOrigin

스프링 MVC는 CORS를 손쉽게 다룰 수 있는 많은 지원을 하고 있으며, 스프링 부트에서는 `@CrossOrigin` 애노테이션 하나로도 CORS 허용이 가능하다. `@CrossOrigin`은 `@Controller`, `@RequestMapping`과 함께 쓸 수 있으며, `WebMvcConfigurer`를 구현해, 글로벌 설정도 가능하다.

**예시 코드**

- `@GetMapping`과 함께 쓰는 경우

```java
@RestController
public class SampleController {
	@GetMapping("/hello")
	@CrossOrigin(origins = "http://localhost:8081")
	public String hello() {
		return "hello!";
	}
}
```

- `WebMvcConfigurer`를 이용한 설정

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
	@Override
	public void addCorsMappings(CorsRegistry registry) {
		registry.addMapping("/**")
		.allowedOrigins("http://localhost:8081");
	}
}
```
