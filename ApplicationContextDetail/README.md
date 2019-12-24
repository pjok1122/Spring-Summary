## ApplicationContext의 다른 기능

- [Environment (프로퍼티 등록)](#런타임환경과-프로퍼티)
- [MessageSource (다국화 기능)](#messagesource)
- [ResourceLoader (리소스 추상화)](#resourceloader)
- [ApplicationEventPublisher (이벤트 프로그래밍)](#applicationeventpublisher)

### 런타임환경과 프로퍼티

프로퍼티란 키-밸류 형식의 데이터를 의미한다. JDBC나 DataSource를 사용할 때 설정했던 정보들은 보통 프로퍼티로 저장해둔다. 스프링이 제공해주는 application.properties에 프로퍼티를 생성하는 방법 외에도, 다양한 프로퍼티 지정이 가능하다. 참고만 하자.

- 환경변수 (OS)
- 시스템 프로퍼티 (JVM) : -Dkey="value"
- JDNI : java:comp/env/
- ServletContext 매개변수
- ServletConfig 매개변수

별도의 properties 파일을 생성하는 경우, `@PropertySource`를 이용해 프로퍼티 파일을 등록할 수 있다.

```java
@Configuration
@PropertySource("classpath:/database.properties")
public class AppConfig{
```

<hr>

### MessageSource

애플리케이션 컨텍스트는 MessageSource를 extends하고 있다. MessageSource는 `국제화 기능(다국어)`을 제공하는 인터페이스라고 생각하면 된다. 스프링 부트를 사용한다면 별다른 설정없이 messages.properties를 사용할 수 있다.

- messages.properties
- messages_ko_kr.properties
- ...

#### 사용방법

```java
@Autowired
MessageSource messageSource;

public void run(ApplicationArguments args) throws Exception{
    // messageSource.getMessage(String code, Object[] args, String, default, Locale, loc)
    messageSource.getMessage("greeting", new String[] {"yj"}, Locale.KOREA);
    messageSource.getMessage("greeting", new String[] {"yj"}, Locale.getDefault());
}
```

#### 릴로딩이 있는 메시지소스 사용하기

```java

@Bean
public MessageSource messageSource(){
    var messageSource = new ReloadableResourceBundleMessageSource();
    messageSource.setBasename("classpath:/messages");
    messageSource.setDefaultEncoding("UTF-8");
    messageSource.setCacheSeconds(3);
    return messageSource;
}
```

_cf) var는 Java 10 이상에서 지원하는 문법입니다._

<hr>

### ResourceLoader

리소스를 읽어오는 기능을 제공하는 인터페이스다. 이 인터페이스의 핵심 메서드는 `getResource` 하나이다.

### Resource

스프링은 자바에 존재하는 일관성없는 리소스 접근 API를 추상화해서 Resource라는 추상화 인터페이스를 정의했다. (java.net.URL을 추상화)

- 스프링에서 외부의 리소스가 필요할 때, 대부분 이 `Resource` 추상화를 사용한다.
- Resource는 스프링에서 빈이 아니라 값으로 취급된다.
- Resource 타입은 `<property>` 태그의 value를 이용해 문자열로 값을 넣는데, 이 문자열로 된 리소스 정보를 Resource 오브젝트로 변환한다.

### Resource 인터페이스

Resource에는 다음과 같은 주요 메서드가 있다.

- getInputStream()
- exist()
- isOpen()
- getDescription() : 전체 경로 포함한 파일 이름 또는 실제 URL

#### Resource 구현체

Resource의 구현체로는 UrlResource, ClassPathResource, FileSystemResource, ServletContextResource 등이 있다.

### 리소스 읽어오기

Resource 타입은 location 문자열과 ApplicationContext의 타입에 따라 결정된다.

- `ClassPathXmlApplicationContext("location")` -> `ClassPathResource`
- `FileSystemXmlApplicationContext("location")` -> `FileSystemResource`
- `WebApplicationContext("location")` -> `ServeltContextResource`

ApplicationContext의 타입에 상관없이 **리소스 타입을 강제하려면 java.net.URL 접두어(+ classpath:) 중 하나를 선택하여 사용할 수 있다. (추천방법)**

- 접두어

  - file:///
  - classpath:
  - http:

- 예시
  - file:///some/resource/path/config.xml -> FileSystemResource
  - classpath:spring/yj/config.xml -> ClassPathResource
  - http://www.myserver.com/test.dat

<hr>

### ApplicationEventPublisher

ApplicationContext가 제공하는 또 다른 기능 중 하나로, 이벤트 프로그래밍에 필요한 인터페이스를 제공한다. (옵저버 패턴 구현체)

#### 이벤트 만들기

- ApplicationEvent 상속
- 스프링 4.2부터는 이 클래스를 상속받지 않아도 된다.

```java
public class Event {
	private int data;
	private String alias;

	public Event(int data, String alias) {
		this.data = data;
		this.alias = alias;
	}
```

#### 이벤트 발생시키기

- ApplicationEventPublisher.publishEvent(ApplicationEvent event)

```java
@Autowired
ApplicationEventPublisher applicationEventPublisher;

@Override
public void run(ApplicationArguments args) throws Exception {
    applicationEventPublisher.publishEvent(new Event(10, "홍길동"));
}
```

#### 이벤트 처리하기

- `ApplicationListener<이벤트>`를 구현한 클래스를 만들어 빈으로 등록한다.
- 스프링 4.2부터는 `@EventListener`를 사용해서 빈의 메서드에 사용할 수 있다.
- 기본적으로는 synchronized로 구현되어있지만, @Async를 사용할 수 있다.
- 순서를 정해주고 싶은 경우에는 @Order와 함께 사용한다.

```java
@Component
public class EventHandler {

	@EventListener
	public void handle(Event event) {
		System.out.println(Thread.currentThread().toString());
		System.out.println("event Handler1 : " + event.toString());
	}
}
```

#### 스프링이 제공하는 이벤트

- ContextRefreshedEvent : ApplicationContext가 초기화됐거나 리프레시 했을 때 발생.
- ContextStaredEvent : ApplicationContext를 start()하여 라이프사이클 빈들이 시작 신호를 받은 시점에 발생
- ContextStoppedEvent : ApplicationContext를 stop()하여 라이플사이클 빈들이 정지 신호를 받은 시점에 발생
- ContextClosedEvent : ApplicationContext를 close()하여 싱글톤 빈들이 소멸되는 시점에 발생.
- RequestHandledEvent : HTTP 요청을 처리했을 때 발생

스프링부트는 더 다양한 이벤트를 제공해주고 있다.
