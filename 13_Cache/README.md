# Spring Cache

## 캐시 추상화에 대한 이해

- 버퍼와 캐시는 유사하지만 다르다. 버퍼는 입출력 속도가 다른 엔티티를 조율하기 위해 사용된다. 출력버퍼에 모아서 전송하고, 입력버퍼에서 필요할 때 읽어들인다. 반면에 캐시는 한 번 조회한 결과를 재사용하는 것을 말한다.

- 스프링프레임워크에서 제공하는 캐싱 서비스는 캐시 구현체가 아니라 `캐시 추상화`이며 캐시 데이터를 적용하기 위한 실제 스토리지가 필요하다.

- 스프링이 제공하는 `캐시 추상화`는 `org.springframework.cache.Cache`, `org.springframework.cache.CacheManager`이다.

- 스프링은 `캐시 추상화`의 구현체인 `concurrentMap` 기반의 캐시 Ehcache 2.x, Gemfire cache, Caffeine, EHcache 3.x를 제공한다.

- 캐시 추상화에는 멀티 쓰레드, 멀티 프로세스에 대한 처리가 없지만, 캐시 구현체에서 보통 처리된다.

- 서버 여러대를 운영할 경우, 데이터가 변경되면 캐시를 전파해야할 수도 있다.

- `캐시추상화`를 사용하고 싶다면 **캐시정책을 선언**과 **어떤 스토리지를 사용할지 설정**하기만 하면 된다.

## 애노테이션 기반의 캐시 선언

- `@Cacheable` : 캐싱할 메서드에 사용한다.
- `@CacheEvict` : 캐싱을 제거할 때 사용한다.
- `@CachePut` : 캐시를 수정할 때 사용한다.
- `@Caching` : 여러개의 `@CacheEvict` `@CachePut`을 하나의 메서드에 적용해야 할 때 사용한다.
- `@CacheConfig` : 클래스에 공통적으로 적용할 속성이 있을 때 사용한다.

> 주의사항 : @CachePut과 @Cacheable은 함께 사용될 수 없다.


### @Cacheable


#### 기본키 생성 전략

- 메서드에 동일한 인자가 호출됐을 때, 캐시를 적용하기 위해서는 동일한 인자가 들어왔는지 구분할 수 있어야 한다. 캐시 저장소는 key, value 타입의 데이터를 저장하는 스토리지를 사용하므로 메서드의 인자가 key로 할당되어야 한다. 캐시 추상화는 기본전략으로 `KeyGenerator`를 제공한다.

- 파라미터가 없다면 `SimpleKey.EMPTY`를 key로 사용하고, 파라미터가 1개라면 해당 파라미터를 key로 사용하고, 파라미터가 여러개라면 모든 파라미터를 포함하는 `SimpleKey`를 key로 사용한다.

- key는 각 파라미터의 `hashCode()`를 기반으로 생성되기 때문에 충돌의 가능성이 있다. `SimpleKeyGenerator`는 충돌이 발생하는 상황에는 복합키를 사용한다.

#### 메서드 일부만을 key로 쓰기

- 메서드의 일부 파라미터는 캐싱과 무관한 로직일 수 있다. 이런 경우에는 인자의 일부만을 key로 사용한다.

- 속성 key에는 SPEL 문법을 사용할 수 있다.


**예시 코드**

```java
@Cacheable(cacheNames="books", key="#isbn")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

@Cacheable(cacheNames="books", key="#isbn.rawNumber")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

//인자값이 null일 수 있을 때
@Cacheable(key = "#kim?.name:'Unknown'")        
public List<String getList(Person kim){ ... }

//인자값이 여러개일 때 (단일키)
@Cacheable(key = "#kim + #page + #list")
public List<String getList(Person kim, int page, List<String> list){ ... }

//인자값이 여러개일 때 (복합키)
@Cacheable(key = "{#kim, #page, #list}")
public List<String getList(Person kim, int page, List<String> list){ ... }
```

#### 캐시 동기화

- 멀티 쓰레드 환경에서는 두 쓰레드가 동일 메서드(동일 인자)를 동시에 호출할 수 있다. 이때, 캐시 추상화는 기본적으로 `lock`을 제공하지 않기 때문에 동일한 값이 여러번 계산될 수 있다.

- `sync=true` 옵션을 주면 하나의 쓰레드만 lock을 걸고 작업을 수행한다. lock은 캐시 추상화 구현체에서 지원하지 않을 수도 있다. (대부분은 지원함)

```java
@Cacheable(cacheNames="foos", sync=true) 
public Foo executeExpensiveOperation(String id) {...}
```

#### 캐시 조건

- 조건을 만족하는 경우에만 캐싱하고 싶을 때 조건을 줄 수 있다.

- `condition=#name.length() < 32`와 같은 조건을 걸면, 이름의 길이가 32보다 크다면 어떤 경우에도 캐시를 탐색하지 않는다.

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32") 
public Book findBook(String name)
```

- `unless`에 조건을 사용하면, 연산 결과가 false일 때만 캐시에 추가한다. `condition`과 달리 메서드가 호출된 후에 체크하기 때문에 메서드의 리턴값을 `#result`으로 받는다. 반환값이 `Optional<>`이더라도 `#result`는 객체가 된다.

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result?.isHardback")
public Optional<Book> findBook(String name)
```


### @CachePut

- 이 애노테이션이 붙어있으면 메서드는 강제로 실행되고 캐시를 업데이트한다. `@Cacheable`과 함께 쓰이지 않는다.

- `@CachePut` 옵션은 `@Cacheable`과 거의 동일하다.

- `@CachePut`은 비즈니스 로직의 일부로 쓰이기보다는, 캐시 초기화에 사용된다.

### @CacheEvict

- 이 애노테이션은 캐시를 지울 때 사용한다.

- `allEntries`를 true로 주면, 캐시 엔트리를 전부 제거한다. 각각의 엔트리를 찾아서 지우는 것보다 속도가 빠르다.

- `beforeInvocation` 속성을 이용해서 메서드 호출 전에 캐시를 비울지, 호출 후에 캐시를 비울지 선택할 수 있다.

```java
@CacheEvict(cacheNames="books", allEntries=true) 
public void loadBooks(InputStream batch)
```

### @Caching

- @Caching 애노테이션은 앞서 살펴본 `@Cachexxx` 애노테이션들을 중첩해서 사용하고 싶을 때 사용한다.

```java
@Caching(evict = { @CacheEvict("primary"), @CacheEvict(cacheNames="secondary", key="#p0") })
public Book importBooks(String deposit, Date date)
```


### @CacheConfig

- 클래스 단위로 공통되는 속성을 줄 때 사용된다.

- `cacheName`, `KeyGenerator`, `CacheManager`, `CacheResolver` 같은 속성을 줄 수 있다. 

```java
@CacheConfig("books") 
public class BookRepositoryImpl implements BookRepository {

    @Cacheable
    public Book findBook(ISBN isbn) {...}
}
```

### @EnableCaching

- 캐시를 사용하겠음을 선언할 때 사용한다. 이 애노테이션을 사용함으로써 많은 것들(?)이 자동으로 동작하게 된다.

- `@Cacheable`과 같은 애노테이션이 붙은 메서드는 AOP를 통해 캐싱 작업이 추가된다. 그 과정은 `@Transactional` 애노테이션과 매우 유사하다. 캐시도 결국 스토리지에 데이터를 저장하고 삭제하는 일이기 때문에 커밋과 롤백을 수행한다고 보면 된다.

> 주의사항 : 프록시 객체 내부에서 호출하는 메서드는 @Cacheable과 같은 애노테이션이 동작하지 않는다.

## 캐시 스토리지 설정

- 스프링이 자체적으로 지원하는 캐시들은 `CacheManager`의 구현체를 Bean으로 등록해주기만 하면 된다.

### JDK ConcurrentMap

- `org.springframework.cache.support.SimpleCacheManager`를 빈으로 등록하면 된다.

- 테스트, 또는 간단한 애플리케이션에 적합하다.

### Ehcache-based Cache

- `org.springframework.cache.ehcache.EhCacheCacheManager`를 빈으로 등록한다.

- `org.springframework.cache.ehcache.EhCacheManagerFactoryBean`를 빈으로 등록한다.

### JSR-107 Cache

- `org.springframework.cache.jcache.JCacheCacheManager`를 빈으로 등록한다.


# Springboot Cache

- `@EnableCaching` 애노테이션을 사용하면 캐시 인프라를 자동으로 설정해준다. (캐시가 있다면!)

- 아무런 캐시 라이브러리를 사용하지 않으면 default로 `concurrentMap(memory)`을 사용한다. 메모리를 사용하기 때문에 실 프로젝트에서는 사용하지 않고, 위에서 살펴본 캐시 추상화를 구현한 객체를 빈으로 등록하여 사용한다.

- `spring-boot-starter-cache`를 의존성에 추가하면 `JCacheCacheManager`가 추가된다. 의존성에 추가된다고 바로 DI가 되지는 않는다. default로 제공되는 ConcurrentMap의 우선순위가 더 높기 때문에, 명시적으로 선언해야 한다.

- 각 구현체에 따라서 제공하는 properties들이 다양하기 때문에 사용하고자 하는 캐시의 공식문서를 보는 것을 추천한다.

**예시**

의존성에 `spring-boot-starter-cache`와 `spring-boot-starter-data-redis`를 추가한다.

프로퍼티는 다음과 같이 `spring.cache.redis.*` 밑에 설정할 수 있는 요소들이 존재한다.

```properties
spring.cache.cache-names=cache1,cache2
spring.cache.redis.time-to-live=600000
```

`@EnableCaching` 애노테이션만 추가하면 자동으로 localhost에 있는 redis와 연결하는 것 같다. 

