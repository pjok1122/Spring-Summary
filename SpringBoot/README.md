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
