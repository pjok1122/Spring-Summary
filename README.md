## [1장. IoC Container & Bean](./01_IoC-Container/README.md)

### 토비의 스프링 3.1 Vol.1 Chapter1. IoC 컨테이너와 빈 요약 정리.

- IoC 컨테이너와 Bean
- 설정 메타정보
- IoC 컨테이너 종류
- IoC 컨테이너 계층구조
- 빈 설정과 DI
- Autowired
- 빈의 스코프
- 프로파일과 프로퍼티
- IoC 요약

## [2장. ApplicationCotext의 다른 기능들](./02_ApplicationContextDetail/README.md)

### 백기선님의 스프링 프레임워크 핵심기술 강좌 참고

- Environment (프로퍼티 등록)
- MessageSource (다국화 기능)
- ResourceLoader (리소스 추상화)
- ApplicationEventPublisher (이벤트 프로그래밍)

## [3장. Validation &amp; DataBinding 추상화](./03_ValidationDataBinding/README.md)

### 백기선님의 스프링 프레임워크 핵심기술 강좌 참고

- Validator (객체 유효성 검사)
- DataBinding
  - PropertyEditor (오래된 버전)
  - Converter
  - Formatter
  - ConversionService

## [4장. Aspect Oriented Programming](./04_AOP/README.md)

### 백기선님의 스프링 프레임워크 핵심기술 강좌 참고

- AOP의 주요 개념
- AOP의 적용 방법
  - 컴파일
  - 로드타임
  - 런타임
- 스프링 AOP의 특징
  - 프록시패턴 AOP
  - 스프링 AOP 사용하기

## [5장. 스프링부트의 원리](./05_SpringBootPrinciple/README.md)

### 백기선님의 스프링 부트 개념과 활용 강좌 참고

- 의존성 관리
- @EnableAutoConfiguration
- 내장 웹 서버 설정
  - Tomcat, Jetty, Undertow
  - HTTPS, HTTP2 설정
- 독립적으로 실행 가능한 JAR 파일

## [6장. 스프링부트의 핵심 기능](./06_SpringBootCoreFeature/README.md)

### 백기선님의 스프링 부트 개념과 활용 강좌 참고

- SpringApplication (배너, 외부인자)
- 외부설정 (property, 우선순위, 컨버젼, 유효성 검사)
- 프로파일
- 로깅
- 테스트 (테스트 코드 작성, 슬라이스 테스트)

## [7장. 스프링 웹 기술과 MVC](./07_SpringMVC/README.md)

### 토비의 스프링 3.1 Vol.2 Chapter 3. Spring MVC 요약정리

- **DispatcherServlet의 동작 과정(Spring MVC 동작 과정)**
- 컨트롤러의 종류와 핸들러 어댑터 (`@Controller와 AnnotationMethodHandlerAdapter` ...)
- 핸들러 매핑 (`BeanNameUrlHandlerMapping` `DefaultAnnotationHandlerMapping` ...)
- 뷰 오브젝트(`InternalResourceView` ...)
- 뷰 리졸버(`InternalResourceViewResolver` ...)

## [7.5장. @MVC](./08_@MVC/README.md)

- @RequestMapping
- @Controller
- @ModelAttribute, BindingResult
- Validation

## [8장. 스프링부트 MVC](./08_SpringBootMVC/README.md)

### 백기선님의 스프링 부트 개념과 활용 강좌 참고

- HttpMessageConverter (`ContentNegotiatingViewResolver`)
- 정적 리소스 지원 (`index.html`, `favicon.ico`)
- 웹 JAR (`jquery`, `vue.js`)
- 템플릿 엔진 (`thymeleaf`)
- HTML 테스트 코드 (`HtmlUnit`)
- ExceptionHandler
- HATEOAS
- CORS

## [9장. 스프링 데이터 액세스 기술](./09_SpringDataAccess/README.md)

### 토비의 스프링 3.1 Vol.2 Chapter 2. 데이터 액세스 기술 정리

- DataSource
- Spring JDBC
- JdbcTemplate API
- JPA는 다른 레포에서 정리.

## [10장. 스프링부트 데이터 액세스 기술](./10_SpringBootDataAccess/README.md)

### 백기선님의 스프링 부트 개념과 활용 강좌 참고

- 인메모리 데이터 베이스 (H2)
- DBCP
- MySQL, MariaDB
- PostgreSQL
- Spring Data JPA
- 데이터 마이그레이션 (Flyway)
- Redis
- MongoDB
- Neo4j

## [11장. 테스트](./11_Test/README.md)

### 토비의 스프링 3.1 Vol.1 Chapter2. 테스트, Vol.2 Chapter6. 테스트 컨텍스트 프레임워크 요약 정리

- 테스트를 하는 이유
- JUnit 동작 과정
- 테스트 컨텍스트 프레임워크
- 슬라이스 테스트 (`@WebMvcTest`, `@DataJpaTest`)
