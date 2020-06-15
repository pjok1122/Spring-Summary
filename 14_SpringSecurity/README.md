## Feature

- `Authentication` : 자원에 접근하고자 하는 사람의 신원을 확인하는 방법 (ID / PW)

### Password Stroage History

- 패스워드 저장의 권장사항은 **암호학적 해시함수**와 **Salt**를 사용하는 것입니다.

- 여전히 해시함수로 SHA-256을 많이 사용하지만, 더이상 SHA-256은 안전한 해시 함수가 아닙니다.

- `SHA-256`의 대체제로 `adaptive one way function`을 사용합니다. `적응형 단방향 함수`라고 번역되는 이 함수는 의도적으로 `CPU`의 사용량을 늘려 공격자의 공격에 필요한 시간을 억지로 늘리는 기법입니다. (ex. SHA-256을 100번 적용하여 패스워드 저장.)

- `적응형 단방향 함수`의 예시로는 `bcrpyt`, `PBKDF2`, `scrypt`, `argon2` 정도가 있습니다.

- `적응형 단방향 함수`를 사용하는 것은 공격에 안전해지지만, 서비스의 성능이 떨어지는 것을 알 수 있습니다. 언제나 안전과 성능은 trade off 관계임을 인지하고, 서비스의 특징에 맞춰 생각하십시오.

- 서비스에서는 가급적 `session`, `OAuth Token`, `Json Web Token` 등을 사용하여 성능을 개선하여야 합니다.

### PasswordEncoder

#### DelegatingPasswordEncoder

- `xxxPasswordEncoder`에게 인코딩을 `위임`하는 PasswordEncoder 입니다.
- Spring Security의 default PasswordEncoder 입니다.

**예시**

```java
String idForEncode = "bcrypt";
Map encoders = new HashMap<>();
encoders.put(idForEncode, new BCryptPasswordEncoder());
encoders.put("noop", NoOpPasswordEncoder.getInstance());
encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
encoders.put("scrypt", new SCryptPasswordEncoder());
encoders.put("sha256", new StandardPasswordEncoder());

PasswordEncoder passwordEncoder =
    new DelegatingPasswordEncoder(idForEncode, encoders);
```

```
[DelegatingPasswordEncoder 결과]

{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG 
{noop}password 
{pbkdf2}5d923b44a6d129f3ddf3e3c8d29412723dcbde72445e8ef6bf3b508fbf17fa4ed4d6b99ca763d8dc 
{scrypt}$e0801$8bWJaSu2IKSn9Z9kM+TPXfOc/9bdYSrN1oD9qfVThWEwdRTnO7re7Ei+fUZRJ68k9lTyuTeUp4of4g24hHnazw==$OAOec05+bXxvuu/1qZ6NUR+xQYvYv7BeL1QxwRpY5Pc=  
{sha256}97cde38028ad898ebc02e690819fa220e88c62e0699403e94fff291cfffaf8410849f27605abcbc0
```

#### BCrpytPasswordEncoder

- 강도를 조절할 수 있고, 강도가 높을수록 해시화 시간이 오래 걸립니다.
- 인증을 수행하는 데에 의도적으로 1초 정도 소요되도록 설정하는 것이 좋습니다.

**예시**

```java
// Create an encoder with strength 16
BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(16);
String result = encoder.encode("myPassword");
assertTrue(encoder.matches("myPassword", result));
```

#### Argon2PasswordEncoder

- 크래킹 방지를 위해 메모리를 많이 사용한다는 점이 특징입니다.
- 인증을 수행하는 데에 의도적으로 1초 정도 소요되도록 설계되었습니다.

#### Pbkdf2PasswordEncoder

- `FIPS(Federal Information Processing Standard)`에 가장 적합한 해시 함수입니다.
- 인증을 수행하는 데에 의도적으로 1초 정도 소요되도록 설계되었습니다.

#### SCryptPasswordEncoder

- 크래킹 방지를 위해 메모리를 많이 사용한다는 점이 특징입니다.
- 인증을 수행하는 데에 의도적으로 1초 정도 소요되도록 설계되었습니다.


### Protecting Against Exploits

#### CSRF Attack

- Spring은 `Synchronizer Token Pattern`과 `SameSite Attribute`를 이용하여 CSRF 공격을 방어할 수 있습니다.

##### Synchronizer Token Pattern

- 흔히 CSRF Token이라고 불리는 토큰을 이용한 해결책입니다.

- 모든 `POST`, `PUT`, `DELETE` 요청은 Session Cookie 외에도 CSRF Token 이라는 난수가 포함되어있어야 하고, 포함되어있지 않을 경우 요청을 거절합니다.

- CSRF 토큰은 매 요청마다 다른 값을 지니므로 `SOP(Single Origin Policy)`에 의해 안전을 보장받습니다.

**예시**

```html
<form method="post"
    action="/transfer">
<input type="hidden"
    name="_csrf"
    value="4bfd1575-3ad1-4d21-96c7-4ef2d9f86721"/>
<input type="text"
    name="amount"/>
<input type="text"
    name="routingNumber"/>
<input type="hidden"
    name="account"/>
<input type="submit"
    value="Transfer"/>
</form>
```

##### SameSite Attribute

- `Spring Security`에서 사용하는 방법이 아닌, `Spring Session`에서 제공하는 방법입니다.

- 쿠키에 SameSite라는 속성을 추가하여 CSRF 공격을 막는 방법입니다.

- 쿠키에 `SameSite`라는 속성을 추가하면, 동일한 사이트에서 HTTP 요청을 할 때에만 해당 쿠키를 포함시켜 요청을 전송합니다.


>이 외에도 다양한 공격을 방어할 수 있지만, Spring Security의 핵심에서 벗어나므로 생략합니다.

