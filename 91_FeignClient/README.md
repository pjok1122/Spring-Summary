# FeignClient

- Netflix에서 개발된 Http Client 기술로 손쉽게 외부 API를 호출할 수 있다는 장점이 있다.
- 외부 API의 스펙을 Interface에 정의하고 적절한 annotation만 사용하면 구현체를 만들어준다.
- RestTemplate을 사용했을 때보다 훨씬 생산성 높은 코드를 작성할 수 있다.

```java
// RestTemplate
public String registStore(PosContext context, MerchantStoreRegistRequest merchantStoreRegistRequest) {
    try {
        // Header 생성
        HttpHeaders headers = new HttpHeaders();
        headers.set("Content-Type", "application/json");
        headers.set(ACCESS_TOKEN.getHeaderName(), context.getAccessToken());
        headers.set(CLIENT_INFO.getHeaderName(), context.getClientInfo());
        headers.set(DEVICE_MODEL.getHeaderName(), context.getDeviceModel());
        headers.set(DEVICE_UUID.getHeaderName(), context.getDeviceUuid());

        // URI 생성
        URI uri = UriComponentsBuilder.fromHttpUrl(storeRegistUrl).build().encode().toUri();

        // send HTTP request
        ResponseEntity<String> responseEntity = merchantRestTemplate.exchange(uri,HttpMethod.POST, new HttpEntity<>(merchantStoreRegistRequest, headers), String.class);
        String responseString = responseEntity.getBody();
        
        // String to Object
        ApiResponse response = JsonUtils.fromJson(responseString, ApiResponse.class);
        return response.getUserId();
    
    // Exception Handle
    } catch (Exception e) {
        log.error("[INVESTIGATION] failed to request regist store to Merchant Center.", e);
        throw new LinePosLogicException(ReturnCode.MERCHANT_SERVER_FAILED);
    }
    return;
}

// Feign Client
public String registStore(PosContext context, MerchantStoreRegistRequest merchantStoreRegistRequest) {
    try {
        // Feign client 호출
        ApiResponse resposne = merchantCenterClient.registerStore(context.getAccessToken(), context.getClientInfo(), context.getDeviceModel(), context.getDeviceUuid(), merchantStoreRegisterRequest)
        return response.getUserId();
    } catch (Exception e) {
        log.error("[INVESTIGATION] failed to request regist store to Merchant Center.", e);
        throw new LinePosLogicException(ReturnCode.MERCHANT_SERVER_FAILED);
    }
    return;
}

// FeignClient 정의
@FeignClient(name ="merchant-center-api", url = "${feign.merchant.center.url}")
public interface MerchantCenterClient {

    @PostMapping("/register-store")
    String registerStore(@RequestHeader("x-lpos-access-token") String accessToken, @RequestHeader("x-lpos-client-info") String clientInfo,
                        @RequestHeader("x-lpos-device-model") String deviceModel, @RequestHeader("x-lpos-device-uuid") String deviceUuid, @RequestBody MerchantStoreRegisterRequest request)
}
```

## 의존성 주입

- Spring boot 버전에 맞는 cloud 버전을 선택해야 한다.
- https://spring.io/projects/spring-cloud
  
```
gradle
ext {
    springCloudVersion = 'Finchley.RELEASE'
    ...

}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud"
    }
}

dependencies {
    compile("org.springframework.cloud:spring-cloud-starter-openfeign")
}
```

```
pom.xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <version>3.0.3</version>
</dependency>
```

## Feign 사용 설정하기

- `@EnableFeignClients`
- main에 설정할 경우, main이 포함된 패키지 밑에 있는 모든 `@FeignClient`를 찾아서 관리.
- main이 아닌 경우, 명시적으로 basePackages를 선언하여 사용한다.

```java
//예시1)
@SpringBootApplication
@EnableFeignClients
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}

//예시2)
@Configuration
@EnableFeignClients(basePackages = "com.youngjae.feign.client.demo.feign")
public class FeignConfig {

}

```

## FeignClient 생성

- `EnableFeignClients` 의 basePackages 밑에 @FeignClient 애노테이션이 붙은 인터페이스를 생성.

```java
@FeignClient(name ="localhost-api", url = "${feign.localhost.url}")
public interface FeignLocalClient {
    
}
```

- url은 hostname을 의미하며, properties에 정의된 값을 불러올 수 있음.
- 생성되는 구현체의 bean 이름은 interface 이름과 일치하게 생성된다. (beanName : feignLocalClient)
  - `qualifieres`를 지정하여 bean name 변경 가능.
- 



## FeignClient 문법(?)

### 파라미터가 없는 경우

```java
@RestController
public class ExternalController {
    @GetMapping("/no-params")
    public String noParams() {
        return "noParams!";
    }
}

@FeignClient(name ="external-api", url = "http://localhost:8080")
public interface FeignLocalClient {
    @GetMapping("/no-params")
    String noParams();
}

@Service
public class LocalFeignService {
    @Autowired
    private FeignLocalClient feignLocalClient;

    public String callNoParamAPI() {
        return feignLocalClient.noParams();
    }
}
```


### 파라미터가 있는 경우

```java
@RestController
public class ExternalController {
    @GetMapping("/yes-params")
    public String yesParams(@RequestParam String name,
                            @RequestParam Integer age) {
        return name + " : " + age;
    }
}

@FeignClient(name ="external-api", url = "http://localhost:8080")
public interface FeignLocalClient {
    @GetMapping("/yes-params")
    String yesParams(@RequestParam String name, @RequestParam Integer age);
}

@Service
public class LocalFeignService {
    @Autowired
    private FeignLocalClient feignLocalClient;

    public String callYesParamAPI(String name, Integer age) {
        return feignLocalClient.yesParams(name, age);
    }
}
```

- 파라미터로 받을 값을 `@RequestParam` 애노테이션을 붙여준다. (`org.springframework.web.bind.annotation.RequestParam`)


### 외부 API가 Object를 리턴하는 경우

```java
@RestController
public class ExternalController {
    @GetMapping("/return-object")
    public User returnObject(@RequestParam String name,
                             @RequestParam Integer age) {
        return new User(name, age);
    }
}

@FeignClient(name ="external-api", url = "http://localhost:8080")
public interface FeignLocalClient {
    @GetMapping("/return-object")
    String returnObject(@RequestParam String name, @RequestParam Integer age);

    @GetMapping("/return-object")
    User returnObject(@RequestParam String name, @RequestParam Integer age);
}


@Service
public class LocalFeignService {
    @Autowired
    private FeignLocalClient feignLocalClient;

    public User callReturnObjectAPI(String name, Integer age) {
        return feignLocalClient.returnObject(name, age);
    }

    public String callReturnStringAPI(String name, Integer age) {
        return feignLocalClient.returnString(name, age);
    }
}
```

- `HttpMessageConverters`에 의해 원하는 객체 타입으로 바인딩을 시도. (spring Web에서 사용하는 Converter)

### RequestBody가 있는 경우

```java
@RestController
public class ExternalController {
    @PostMapping("/yes-body")
    public String yesBody(@RequestBody User user) {
        return user.getName();
    }
}

@FeignClient(name ="external-api", url = "http://localhost:8080")
public interface FeignLocalClient {
    @PostMapping("/yes-body")
    String yesBody(@RequestBody User user);
}


@Service
public class LocalFeignService {
    @Autowired
    private FeignLocalClient feignLocalClient;

    public String callYesBodyAPI(User user) {
        return feignLocalClient.yesBody(user);
    }
}
```

- `@RequestBody` 애노테이션으로 Body로 받을 객체를 지정한다. (`org.springframework.web.bind.annotation.RequestBody`)
- `@RequestBody`가 생략되어도 Body로 생각하여 전달됨.

### PathVariable이 있는 경우

```java
@RestController
public class ExternalController {
    @GetMapping("/{user-name}/yes-path-variable")
    public String yesPathVariable(@PathVariable("user-name") String userName) {
        return userName;
    }
}

@FeignClient(name ="external-api", url = "http://localhost:8080")
public interface FeignLocalClient {
    @GetMapping("/{user-name}/yes-path-variable")
    String yesPathVariable(@PathVariable("user-name") String userName);     //value를 반드시 명시적으로 지정해야 한다.
}

@Service
public class LocalFeignService {
    @Autowired
    private FeignLocalClient feignLocalClient;

    public String callYesPathVariableAPI(String userName) {
        return feignLocalClient.yesPathVariable(userName);
    }
}
```

- `@Pathvariable`의 value 값을 반드시 지정해줘야 한다. 변수 이름으로 매칭해주는 기능을 지원하지 않음.


### RequestHeader가 있는 경우

```java
@RestController
public class ExternalController {
    @GetMapping("/yes-header")
    public String yesHeader(HttpServletRequest request) {
        String contentType = request.getHeader("content-type");
        String headerTest = request.getHeader("test-header");

        return "contentType : " + contentType + ", headerTest : " + headerTest;
    }

}

@FeignClient(name ="external-api", url = "http://localhost:8080")
public interface FeignLocalClient {
    @GetMapping(value = "/yes-header", headers = {"content-type=application/json; charset=utf-8", "accept=application/json"})
    String yesDynamicHeader(@RequestHeader("test-header") String testHeader);
}

@Service
public class LocalFeignService {
    @Autowired
    private FeignLocalClient feignLocalClient;

    public String callYesDynamicHeaderAPI(String testHeader) {
        return feignLocalClient.yesDynamicHeader(testHeader);
    }
}
```

- `@RequestHeader`, `headers`를 이용해서 헤더를 지정할 수 있다.
  - `@RequestHeader`는 Map도 지원하기 때문에 많은 헤더가 필요한 경우, `@RequestHeader Map<String, String> headerMap` 처럼 사용이 가능하다.
- 모든 메서드에 대해서 동일한 헤더가 필요한 경우에는 `RequestInterceptor`를 설정해둘 수 있다.
- `@feign.Headers` 애노테이션과 `@ReqeustHeader` 애노테이션은 같이 사용할 수 없음.
  - `@Headers` : Feign Contract
  - `@RequestHeader` : SpringMvc Contract
- `Feign`, `Spring-web` 둘 중 하나에서 제공하는 Annotation만 사용할 것!

위의 모든 코드는 [예시코드](https://github.com/pjok1122/feign-client-example)에 정리되어있습니다.


## FeignClient 설정

- 실무에서 사용하려면 좀 더 세밀한 설정을 하고 사용하는 것이 바람직하다.
- Log, ObjectMapper, Retry, Decoder, connection, compression 등 다양한 설정이 필요.