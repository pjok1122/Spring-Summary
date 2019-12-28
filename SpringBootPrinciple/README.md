# 스프링부트의 원리

- [스프링부트의 원리](#%ec%8a%a4%ed%94%84%eb%a7%81%eb%b6%80%ed%8a%b8%ec%9d%98-%ec%9b%90%eb%a6%ac)
  - [의존성 관리](#%ec%9d%98%ec%a1%b4%ec%84%b1-%ea%b4%80%eb%a6%ac)
  - [@SpringBootApplication](#springbootapplication)
    - [@EnableAutoConfiguration](#enableautoconfiguration)
  - [내장 웹 서버 설정](#%eb%82%b4%ec%9e%a5-%ec%9b%b9-%ec%84%9c%eb%b2%84-%ec%84%a4%ec%a0%95)
    - [웹 서버 변경](#%ec%9b%b9-%ec%84%9c%eb%b2%84-%eb%b3%80%ea%b2%bd)
    - [웹 서버 비활성화](#%ec%9b%b9-%ec%84%9c%eb%b2%84-%eb%b9%84%ed%99%9c%ec%84%b1%ed%99%94)
    - [포트 변경](#%ed%8f%ac%ed%8a%b8-%eb%b3%80%ea%b2%bd)
    - [HTTPS 설정하기](#https-%ec%84%a4%ec%a0%95%ed%95%98%ea%b8%b0)
      - [인증서 만들기](#%ec%9d%b8%ec%a6%9d%ec%84%9c-%eb%a7%8c%eb%93%a4%ea%b8%b0)
      - [인증서 설정](#%ec%9d%b8%ec%a6%9d%ec%84%9c-%ec%84%a4%ec%a0%95)
      - [HTTP Connector 구현](#http-connector-%ea%b5%ac%ed%98%84)
    - [HTTP/2 설정하기](#http2-%ec%84%a4%ec%a0%95%ed%95%98%ea%b8%b0)
      - [HTTP/2 with Undertow](#http2-with-undertow)
      - [Jetty, Tomcat](#jetty-tomcat)
  - [독립적으로 실행가능한 JAR](#%eb%8f%85%eb%a6%bd%ec%a0%81%ec%9c%bc%eb%a1%9c-%ec%8b%a4%ed%96%89%ea%b0%80%eb%8a%a5%ed%95%9c-jar)

## 의존성 관리

Maven을 사용하는 경우 pom.xml에 dependency만 추가해주면 Spring이 버전을 전부 관리해준다. 모든 의존성은 버전을 명시해야 하는데, pom.xml에는 이 부분이 나와있지 않다. 하지만, `<parent></parent>` 부분을 따라가면,
`spring-boot-starter-parent`와 `spring-boot-dependencies` 라는 프로젝트가 있다. 여기서 `<properties>` 부분에 Spring이 관리하는 의존성 패키지의 버전이 명시되어있다. 따라서 우리는 `spring-boot-starter-parent`를 부모 프로젝트로 등록해두면 별도의 의존성 관리없이 패키지를 사용할 수 있다.
만약 버전을 바꾸고 싶은 경우, `pom.xml`에서 `properties` 태그를 만들어 작성하면 오버라이딩된다.

<hr>

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

<hr>

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

</br>

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

</br>

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

</br>

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

<hr>

### HTTP/2 설정하기

HTTP/2 설정은 기본적으로 HTTPS에 대한 설정이 이루어져있어야 한다. 또, 어떤 웹 서버를 사용하느냐에 따라 HTTP/2 설정 방식이 달라진다.

#### HTTP/2 with Undertow

`application.properties` 내에 `server.http2.enabled=True` 속성 하나만 정의해주면 된다.

#### Jetty, Tomcat

Jetty와 Tomcat은 아래의 참고자료를 참고하길 바란다.

[참고자료](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-embedded-web-servers)

<hr>

## 독립적으로 실행가능한 JAR

Spring boot에서 `mvn package` 명령을 수행하면 어플리케이션을 target 디렉터리 밑에 jar파일로 만들어준다. 우리는 만들어진 jar 파일을 `java -jar` 라는 명령어로 손쉽게 실행할 수 있다. 이렇게 jar 파일 하나만 가지고도 어플리케이션을 수행할 수 있도록 만드는 것이 Spring boot의 Goal이라고 볼 수 있다.

어떻게 이런 일이 가능할까?

생성된 jar파일 내부에는 어플리케이션이 의존하고 있는 모듈을 jar파일로 가지고있다. 즉, jar 파일 내부에 jar파일들이 저장되어 있다. 기본적으로 자바에는 내장 JAR를 로딩하는 방법이 없으므로, Spring boot는 jar파일을 읽을 수 있는 별도의 방법을 정의했다. `org.springframework.boot.loader.jar.JarFile` 이라는 클래스가 내장 JAR를 읽을 수 있게 도와주며, `org.springframework.boot.loader.JarLauncher` 라는 클래스는 jar 파일을 실행할 수 있게 도와준다.
