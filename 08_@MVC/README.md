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
