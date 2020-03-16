# Annotation 기반의 Spring MVC

## @RequestMapping

### URL 패턴

```java
@RequestMapping("/hello")               // /hello와 매핑
@RequestMapping("/main*")               //와일드 카드 사용 ex) /mainPage, /main
@RequestMapping("/view.*")              //와일드 카드 사용 ex) /view.do, /view.abc
@RequestMapping("/admin/**/user")       //와일드 카드 사용. ex) /admin/a/bc/user
@RequestMapping("/user/{userid}")       //path variable
@RequestMapping({"/hello", "/hi"})      //하나 이상의 URL 패턴 사용
```

- URL 패턴에서 기억해야 할 것은 `디폴트 접미어 패턴(default suffix pattern)`이다.
- 디폴트 접미어 패턴은 `@RequestMapping("/hello")`에 대해서 `@Requestmapping({"/hello", "/hello/", "/hello.*"})`와 동일한 결과를 갖도록 해준다.

### 요청 파라미터

```java
@RequestMapping(value="/user/edit", parmas="type=admin")
@RequestMapping(value="/user/edit")
```

- `params="value=key"` 형태로 작성하며, 요청할 때 넘겨지는 파라미터를 확인한다.
- `/user/eidt?type=admin`은 첫 번째 매핑이 적용된다.
- `POST /user/edit`에 `<input name="type" value="admin">` 정보를 넘기는 경우에도 첫 번째 매핑이 적용된다.
- `/user/eidt?type=admin`은 첫 번째 두 번째 매핑 조건에 모두 해당한다. 이런 경우 보다 정확한 매핑 조건을 만족하고 있는 메서드가 호출된다.

<hr><br>

## @Controller

`@Controller`를 담당하는 핸들러 어댑터는 상당히 복잡하다. 사용할 수 있는 파라미터나 리턴 값도 다양해서 처음에는 당황스러울 수 있다. 하지만 익숙해지면 가장 최적화된 구조의 깔끔하고 직관적인 컨트롤러를 만들 수 있다.

`@MVC` `@Controller` 개발 방법은 스프링 역사상 가장 획기적인 변신이며, 유연성을 강조하는 스프링에서 조차 권고하는 개발 방법이다.

### 메소드 파라미터 종류

#### `HttpServletRequest` `HttpServletResponse`

대개는 상세한 정보를 담은 파라미터를 활용하지만, 원한다면 위와 같은 파라미터도 사용이 가능하다.

#### HttpSession

HttpSession은 서버에 따라서 멀티스레드 환경에서 안전성이 보장되지 않는다. HttpSession을 안전하게 사용하려면 핸들러 어댑터의 synchronizedOnSession 프로퍼티를 true로 설정해야 한다.

#### WebRequest, NativeWebRequest

`WebRequest`는 `HttpServletRequest`의 요청정보를 대부분 그대로 갖고 있는, 서블릿 API에 종속적이지 않은 오브젝트 타입이다. 스프링 서블릿/MVC의 컨트롤러에서라면 꼭 필요하지 않다.

#### Locale

`DispatcherServlet`의 지역정보 리졸버가 결정한 `Locale` 오브젝트를 받을 수 있다.

#### Reader , Writer

입력 스트림이나 출력 스트림을 받을 수 있다.

#### @PathVariable

URL에 {}로 들어가는 패스 변수를 받을 수 있다.

```java
@RequestMapping("/member/{membercode}/order/{orderId}")
public String lookup(@PathVariable("membercode") String code,
    @PathVariable("orderId") int orderId)
```

Convert 될 수 없는 타입이 전달될 경우 HTTP 400 응답 코드가 전달된다.

#### @RequestParam

단일 HTTP 요청 파라미터를 메소드 파라미터에 넣어주는 애노테이션이다.

```java
public String view(@RequestParam("id") int id,
            @RequestParam("name") String name,
            @Requestparam("file") MultipartFile file)
```

#### @CookieValue

쿠키 값을 메소드 파라미터에 넣어 받을 수 있다.

```java
public String check(@CookieValue(value="auth", required=false, defaultValue="NONE") String auth)
```

#### @RequestHeader

요청 헤더정보를 메소드 파라미터에 넣어주는 애노테이션이다.

```java
public void header(@RequestHeader("Host") String host,
    @RequestHeader("Keep-Alive") long keepAlive)
```

#### Map, Model, ModelMap

모델을 담을 맵은 메소드 내에서 직접 생성할 수도 있지만 그보다는 파라미터로 정의해서 핸들러 어댑터에서 미리 만들어 제공해주는 것을 사용하면 편리하다.

```java
public void hello(ModelMap model){
    User user = new User();
    model.addAttribute(user);     // addAttribute("user", user) 와 동일하다.
}
```

Collections에 담긴 모든 오브젝트를 모델로 추가하려면 addAllAttribute()를 사용한다.

#### @ModelAttribute

`@ModelAttribute`는 여러 가지 기능을 제공한다.

1. 요청 파라미터를 도메인 객체나 DTO 프로퍼티에 바인딩 해줄 수 있다. (`Command`)
2. 모델에 추가해주지 않아도 자동으로 모델에 추가된다.
3. BindingResult 또는 Errors와 함께 사용되어 예외를 처리한다.

**예시 코드**

```java
@RequestMapping("/user/add", method=RequestMethod.POST)
public String search(@ModelAttribute User user, BindingResult result){     //요청 파라미터가 바인딩된다.
    if(result.hasError()){
        return "/user/add";     // /user/add 뷰에 user 객체와 result 객체가 전달된다.
    }

    userService.add(user);
}
```

- `BindingResult`를 사용할 때에는 반드시 `@ModelAttribute` 바로 뒤에 나와야 한다.
- 타입 변환에 실패한 경우 `BindingResult` 오브젝트에 담겨서 컨트롤러에 전달된다.
- `BindingResult`는 모델에 추가될 때의 키 값이 `org.springframework.validation.BindingResult.모델이름`이 된다.
- 클라이언트가 폼 데이터를 이용하는 경우 `bindingResult`를 이용해 에러 메시지를 보여주고 클라이언트에게 폼을 수정할 기회를 제공한다.

`@ModelAttribute`는 일반 메서드에서도 사용이 가능하다. `@ModelAttribute` 메서드를 만들어두면, 같은 클래스 내에 다른 메서드에 해당 모델이 자동으로 삽입된다. 따라서 클래스 내에 공통적으로 사용하는 정보라면 `@ModelAttribute` 메서드를 사용하는 것도 바람직하다.

**예시 코드**

```java
@ModelAttribute("codes")
public User user(){
    User user = new User();
    user.setCommonInfo("common");   //기본으로 설정되는 값.
    return user;
}
```

#### SessionStatus

저장된 세션을 `SessionStatus.setComplete()`을 이용하여 제거할 때 주로 사용된다.

#### @RequestBody

이 애노테이션이 붙은 파라미터에는 HTTP request body 부분이 그대로 전달된다. 일반적인 GET/POST 요청 파라미터라면 사용할 일이 거의 없지만, XML이나 JSON 기반의 메시지를 사용하는 요청의 경우에는 이 방법이 매우 유용하다.

XML, JSON으로 변환할 때에는 `MarshalingHttpMessageConverter`, `MappingJacksonHttpMessageConverter`를 사용할 수 있다.

<br><hr>

### 리턴 타입의 종류

#### ModelAndView

사용할 수는 있으나 추천되지 않는다.

#### String

메소드의 리턴 타입이 스트링이면 이 리턴 값은 뷰 이름으로 사용된다. `@ResponseBody`와 함께 사용하면 `HttpMessageConverter`에 의해 변환되어 `HttpServletResponse`의 출력스트림에 삽입된다. XML이나 JSON 통신을 할 때 주로 사용된다.

#### void

메소드의 리턴 값이 void라면 `RequestToViewNameResolver`에 의해 메소드의 이름과 똑같은 뷰의 이름을 생성한다.

<br><hr>

### @SessionAttribute와 SessionStatus

게시물이나 회원 정보를 수정하는 컨트롤러를 작성한다고 생각해보자. 여러 구현 방법이 있겠지만 각 방법의 장단점을 생각해보록 한다.

#### 히든 필드를 이용한 구현

```java
@RequestMapping(value="/user/edit", method=RequestMethod.POST)
public String submit(@ModelAttribute User user){
    userService.updateUser(user);
    return "user/editsuccess";
}
```

위와 같이 컨트롤러 메서드를 구현하고, 화면에서 히든 필드를 이용해 변경할 수 없는 값들을 채워 넣는다. 변경할 수 없는 값들을 채워넣지 않으면, 비즈니스 로직에서 잘못된 값을 사용하거나, DAO에서 잘못된 값을 삽입할 수 있다.

하지만 이런 히든필드의 방식은 데이터가 HTML 소스에 그대로 나타나기 때문에 데이터 보안에 치명적이다. 따라서 가급적 사용하지 말아야할 방식이다.

#### DB 재조회를 이용한 구현

DB를 재조회하고, 클라이언트가 넘겨준 값으로 변경해주는 방법이다.

```java
@RequestMapping(value="/user/edit" method="RequestMethod.POST")
public String submit(@ModelAttribute User formUser, @RequestParam int id){
    User user = userService.getUser(id);
    user.setName(formUser.getName());
    user.setEmail(formUser.getEmail());
    ...

    userService.updateUser(user);
    return "user/editsuccess";
}
```

이 방식은 완벽해보이지만 몇 가지 단점이 있다. 첫 번째는 매번 DB에서 다시 조회한다는 점이다. 성능에는 큰 영향을 끼치진 않겠지만 부담이 되는 것은 사실이다. 두 번째는 폼에서 전달받은 정보를 일일이 복사하는 일은 매우 번거롭고 실수하기 쉽다는 점이다.

#### @SessionAttribute를 이용한 구현

```java
@Controller
@SessionAttributes("user")
public class UserController{
    ...

    @RequestMapping(value="/user/edit" method="RequestMethod.GET")
    public String form(@RequestParam int id, Model model){
        model.addAttribute("user", userService.getUser(id));
        return "user/edit";
    }

    @RequestMapping(value="/user/edit" method="RequestMethod.POST")
    public String submit(@ModelAttribute User user){
        userService.updateUser(user);
        return "user/editsuccess";
    }
}
```

SessionAttributes가 해주는 역할은 크게 2가지다. 첫째, 컨트롤러 메소드가 생성하는 모델정보 중에서 `@SessionAttributes`에 지정한 이름과 동일한 것이 있다면 이를 세션에 저장해준다. 두 번째, `@ModelAttribute`가 지정된 파라미터가 있을 때 이 파라미터에 전달해줄 오브젝트를 세션에서 가져오는 것이다. 이 중에서 폼에서 전달된 필드정보만 변경된다.

이 방식으로 구현하면 불필요하게 수정하지도 않을 필드를 폼에 노출한다거나 DB에서 User를 반복해서 읽거나, 서비스 계층 오브젝트와 DAO에게 폼에서 수정되는 필드가 무엇인지 알고 있도록 강요할 필요가 없다.

더 이상 필요없는 세션 어트리뷰트는 `SessionStatus`의 `setComplete()` 메소드를 통해 제거해주는 것을 잊지말자. 세션은 서버의 메모리를 사용하기 때문이다.

<br><hr>

## Validation

`Validation`은 프레젠테이션 계층의 컨트롤러에서 처리해야 한다고 보는 개발자가 있는 반면, 비즈니스 로직과 관련이 있으니 서비스 계층에서 처리해야 한다는 개발자가 있다. 아직까지 정답이 없는 논쟁이므로 본인에게 편한대로 하자.

Validation을 하는 3가지 방법을 살펴보자.

### 컨트롤러 코드에서 검증기를 사용하는 방법

```java
@Controller
public class UserController{
    @Autowired
    UserValidator userValidator;

    @Autowired
    UserService userService;

    @PostMapping("/add")
    public String add(@ModelAttribute User user, BindingResult result){
        userValidator.validate(user, result);
        if(result.hasErrors()){
            return "addForm";
        }
        else{
            userService.add(user);
        }
    }
}
```

- Validator를 구현한 UserValidator를 DI받아 컨트롤러 내에서 사용하는 방법이다.

### @Valid를 이용한 자동검증

```java
@Controller
public class UserController{
    @Autowired
    UserValidator userValidator;

    @Autowired
    UserService userService;

    @InitBinder
    public void initBinder(webDataBinder dataBinder){
        dataBinder.setValidator(userValidator);
    }

    @PostMapping("/add")
    public String add(@ModelAttribute @Valid User user, BindingResult result){
        if(result.hasErrors()){
            return "addForm";
        }
        else{
            userService.add(user);
        }
    }
}
```

- `@Valid` 애노테이션이 붙어있으면 등록된 모든 Validation을 수행한다.

### 서비스 계층을 활용하는 Validator

```java
@Controller
public class UserController{

    @Autowired
    UserService userService;


    @PostMapping("/add")
    public String add(@ModelAttribute @Valid User user, BindingResult result){
        userService.checkValid(user, result);
        if(result.hasErrors()){
            return "addForm";
        }
        else{
            userService.add(user);
        }
    }
}
```

- 프레젠테이션 계층에서 서비스 계층으로 `Validation`을 넘긴 모습이다.
- 이 방식의 단점은 컨트롤러에서 두 번 이상 서비스 계층을 호출한다는 점이다. 이 때문에 서비스 계층에 트랜잭션 경계를 설정해뒀다면 트랜잭션이 두 번 생성될 수도 있다. 그럼에도 전체적인 코드가 깔끔해지고, 각 오브젝트는 자기 책임에 충실하게 독립적으로 작성할 수 있다는 장점이 있다.
