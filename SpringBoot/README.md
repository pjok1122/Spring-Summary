# 스프링부트의 원리

## 의존성 관리

Maven을 사용하는 경우 pom.xml에 dependency만 추가해주면 Spring이 버전을 전부 관리해준다. 모든 의존성은 버전을 명시해야 하는데, pom.xml에는 이 부분이 나와있지 않다. 하지만, `<parent></parent>` 부분을 따라가면,
`spring-boot-starter-parent`와 `spring-boot-dependencies` 라는 프로젝트가 있다. 여기서 `<properties>` 부분에 Spring이 관리하는 의존성 패키지의 버전이 명시되어있다. 따라서 우리는 `spring-boot-starter-parent`를 부모 프로젝트로 등록해두면 별도의 의존성 관리없이 패키지를 사용할 수 있다.
만약 버전을 바꾸고 싶은 경우, `pom.xml`에서 `properties` 태그를 만들어 작성하면 오버라이딩된다.

## @SpringBootApplication

`@SpringBootApplication`은 `Configuration`, `ComponentScan`, `EnableAutoConfiguration` 라는 세 애노테이션을 합친 것으로 볼 수 있다. 먼저 `Configuration`은 알다시피 빈 설정파일임을 명시하는 애노테이션이고, `ComponentScan`은 해당 패키지 아래에 있는 모든 패키지에 대해 컴포넌트를 스캔하도록 지정하는 애노테이션이다.

그렇다면 `EnableAutoConfiguration`은 뭘까? 이 애노테이션도 빈을 찾아 등록하는 애노테이션이다. 즉 빈을 등록하는 단게는 사실 2단계에 걸쳐서 일어난다. 처음에는 `@ComponentScan`이며 두 번째는 `@EnableAutoConfiguration` 이다.

### @EnableAutoConfiguration

`@EnableAutoConfiguration`은 `spring-boot-autoconfiugre.xxx.jar` 라는 프로젝트에 있는 대부분의 `Configuration`을 `bean`으로 등록해준다. 실제로 프로젝트가 있는 jar파일로 이동하여 `META-INF > spring.factories` 파일을 열어보면 다음과 같이 작성되어있다.

```xml
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
...
```

즉, 나열되어있는 모든 클래스들이 Auto Configuration의 대상이다. 이 클래스 중 하나를 열어보면 `@Component` 라는 애노테이션을 가지고 있는 것을 볼 수 있다. `@ConditionalOnXxxYyyZzz()` 라는 애노테이션도 보이는데, 이 애노테이션은 조건을 만족하는 경우에만 빈으로 등록하라는 조건부 빈 등록 애노테이션이다.

## 내장 웹 서버 설정

스프링 부트는 `Tomcat` 이라는 기본 내장 웹 서버를 이용한다. 즉, 우리가 아무런 설정을 해주지 않아도 자동으로 `톰캣 객체 생성`, `포트 설정`, `톰캣에 컨텍스트 추가`, `서블릿 만들기`, `톰캣에 서블릿 추가`, `컨텍스트에 서블릿 맵핑` 등 다양한 일을 처리해준다.

- ServletWebServerFactoryAutoConfiguration : 서블릿 웹 서버 생성
- DispatcherServletAutoConfiguration : 서블릿 만들고 등록

절차가 이렇게 둘로 구분되는 이유는 어떤 서블릿 컨테이너를 사용하던지 간에, DispatcherServlet을 만들어야 하기 때문이다.

### 웹 서버 변경

기본 웹 서버인 tomcat을 exclusion 시키고, 추가하고싶은 웹 서버를 `dependency`에 추가한다.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

<br>

### 웹 서버 비활성화

```java
public static void main(String[] args){
    SpringApplication springApplication = new SpringApplication(ServeltContainerApplication.class);
    springApplication.setWebApplicationType(WebApplicationType.NONE);
    springApplication.run(args);
}
```

또는 `application.properties`에 다음과 같이 설정한다.

```xml
spring.main.web-application-type=none
```

<br>

### 포트 변경

`application.properties`에 다음과 같이 설정한다. port 번호를 0으로 주면 사용 가능한 랜덤 포트를 사용한다.

```xml
server.port=8080
```

현재 설정된 포트 번호를 확인하려면 `ApplicationListener<ServletWebServerInitializedEvent>` 를 구현하면 된다.

```java
@Component
public class PortListener implements ApplicationListener<ServletWebServerInitializedEvent> {

	@Override
	public void onApplicationEvent(ServletWebServerInitializedEvent event) {
		ServletWebServerApplicationContext ac = event.getApplicationContext();
		WebServer webServer = ac.getWebServer();
		System.out.println(webServer.getPort());
	}
}
```

### HTTPS 설정하기

HTTPS는 HTTP에 `SSL(Secure Socket Layer)` 라는 프로토콜을 사용하여 `암호화`, `인증`을 보장받는 프로토콜이다.

HTTPS에 대해 자세히 모른다면 이전에 작성했던 글을 참고하자.

[Network 참고자료](https://github.com/pjok1122/Interview_Question_for_Beginner/blob/master/Network/README.md)

#### 인증서 만들기

HTTPS 설정을 하기 위해서는 인증서가 필요하다.

프로젝트 내에 커맨드 라인으로 다음과 같은 명령어를 입력하자.

```command
keytool -genkey
-alias tomcat
-storetype PKCS12
-keyalg RSA
-keysize 2048
-keystore keystore.p12
-validity 4000
```

물어보는 질문에 대답만 열심히 하다보면, `인증서(keystore.p12)`가 생성된다. 이제 HTTPS 요청이 들어오면, 우리는 클라이언트의 브라우저에 인증서를 전달한다.

#### 인증서 설정

인증서 설정은 아주 간단하다. `application.properties` 파일에 몇 가지 설정만 추가하면 된다.

```xml
server.ssl.key-store=keystore.p12
server.ssl.key-store-password=123456
server.ssl.key-store-type=PKCS12
server.ssl.key-alias=tomcat
server.port=8443
```

#### HTTP Connector 구현

위 설정을 그대로 진행하면, HTTPS는 정상 작동하지만 `HTTP connector`가 작동하지 않는다. Spring boot는 `application.properties`를 이용해서는 1가지 connector 밖에 구성할 수 없기 때문이다. 따라서 HTTP에 대한 커넥터는 프로그래밍을 통해 구현해야 한다.

```java
@Bean
public ServletWebServerFactory serveltWebServerFactory() {
    TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
    tomcat.addAdditionalTomcatConnectors(createStandardConnector());
    return tomcat;
}

private Connector createStandardConnector() {
    Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
    connector.setPort(8080);
    return connector;
}
```

### HTTP/2 설정하기

HTTP/2 설정은 기본적으로 HTTPS에 대한 설정이 이루어져있어야 한다. 또, 어떤 웹 서버를 사용하느냐에 따라 HTTP/2 설정 방식이 달라진다.

[참고자료](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-embedded-web-servers)
