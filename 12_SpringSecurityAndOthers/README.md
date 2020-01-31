## 스프링 REST Client

### RestTemplate

- Blocking I/O 기반의 Synchronous API
- RestTemplateAutoConfiguration
- 프로젝트에 spring-web 모듈이 있다면 RestTemplateBuilder를 빈으로 등록해 줍니다.
- [공식문서 참고](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#rest-client-access)

**예제 코드**

```java
@RestController
public class SampleController {
	
	@GetMapping("/hello")
	public String hello() throws InterruptedException {
		Thread.sleep(5000);
		return "hello";
	}
	
	@GetMapping("/world")
	public String world() throws InterruptedException {
		Thread.sleep(3000);
		return "world";
	}
}
```

```java
@Component
public class RestTemplateRunner implements ApplicationRunner {

	@Autowired
	RestTemplateBuilder builder;
	
	@Override
	public void run(ApplicationArguments args) throws Exception {
		RestTemplate restTemplate = builder.build();

		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		
		String helloResult = restTemplate.getForObject("http://localhost:8080/hello", String.class);
		System.out.println(helloResult);
		String worldResult = restTemplate.getForObject("http://localhost:8080/world", String.class);
		System.out.println(worldResult);
		
		stopWatch.stop();
		System.out.println(stopWatch.prettyPrint());
	}

}
```

예제코드가 수행되면 `Thread.sleep()`이 Synchronous하게 총 8초동안 수행되므로, stopwatch에도 약 8초정도의 시간이 걸린다.

#### RestTemplate Customizing

- RestTemplate은 기본으로 java.net.HttpURLConnection 사용.

**로컬 커스터마이징**

- build()를 수행하기 전에, 커스터마이징을 할 수 있다.

```java
public class RestTemplateRunner implements ApplicationRunner {
	RestTemplate restTemplate;
	
	public RestTemplateRunner(RestTemplateBuilder builder) {
		restTemplate =  builder.setConnectTimeout(Duration.ofMinutes(3)).build();
	}
}
```

**글로벌 커스터마이징**

```java
@Bean
public RestTemplateCustomizer restTemplateCustomizer() {
    return new RestTemplateCustomizer() {
        
        @Override
        public void customize(RestTemplate restTemplate) {
            restTemplate.setRequestFactory(new HttpComponentsClientHttpRequestFactory());
        }
    };
}
```

- 의존성에 apache http Client를 추가하고, 위의 예시처럼 커스터마이징하면 restTemplate을 HttpClient으로 사용할 수 있다.

### WebClient

- Non-Blocking I/O 기반의 Asynchronous API
- WebClientAutoConfiguration
- 프로젝트에 spring-webflux 모듈이 있다면 WebClient.Builder를 빈으로 등록해 줍니다.
- [공식문서 참고](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client)

**예제 코드**

```java
@RestController
public class SampleController {
    //위의 예제와 동일.
}
```

```java
@Component
public class WebClientRunner implements ApplicationRunner {

	@Autowired
	WebClient.Builder builder;

	@Override
	public void run(ApplicationArguments args) throws Exception {
		WebClient webClient = builder.build();
		StopWatch stopWatch = new StopWatch();
		
		stopWatch.start();
		
		Mono<String> helloMono = webClient.get()
		.uri("http://localhost:8080/hello")
		.retrieve()
		.bodyToMono(String.class);
		
		helloMono.subscribe(s->{
			System.out.println(s);
			
			if(stopWatch.isRunning()) {
				stopWatch.stop();
			}
			System.out.println(stopWatch.prettyPrint());
			stopWatch.start();
		});
		
		Mono<String> worldMono = webClient.get()
		.uri("http://localhost:8080/world")
		.retrieve()
		.bodyToMono(String.class);
		
		worldMono.subscribe(s -> {
			System.out.println(s);

			if (stopWatch.isRunning()) {
				stopWatch.stop();
			}
			System.out.println(stopWatch.prettyPrint());
			stopWatch.start();
		});
	}

}
```

webClient는 subscribe()가 호출되었을 때, API 요청을 보내고 그 결과를 비동기적으로 받게 된다. 따라서 `/hello`에 대한 요청을 수행하고, 그 결과를 바로 받는 것이 아니라, `/world`에 대한 요청을 수행한다. `RestTemplate`을 사용했을 때에는 8초라는 시간이 걸렸지만, `WebClient`를 사용했을 때에는 5초 정도의 시간밖에 걸리지 않는다. 

### WebClient Customizing

**로컬 커스터마이징**

```java
public class WebClientRunner implements ApplicationRunner {
	WebClient webClient;
	
	public WebClientRunner(WebClient.Builder builder) {
		webClient = builder.baseUrl("http://localhost:8080").build();
	}
}
```

**글로벌 커스터마이징**

```java
@Bean
public WebClientCustomizer webClientCustomizer() {
    return new WebClientCustomizer() {
        
        @Override
        public void customize(Builder webClientBuilder) {
            webClientBuilder.baseUrl("http://localhost:8080");
        }
    };
}
```
