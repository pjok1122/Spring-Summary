# DB ConnectionPool 이란?

데이터베이스와 통신을 하기 위해서는 반드시 커넥션이 필요하다. 이 커넥션을 통해 쿼리를 요청하고, 그 결과를 받을 수 있게 된다. 하지만 커넥션을 만드는 비용은 생각보다 크기 때문에 application에서 커넥션을 여러 개 만들어놓고 커넥션을 재사용하는 방식을 주로 사용한다.

커넥션 풀은 대부분 잘 이해하고 있지만, 그 내부 설정들을 잘못 설정하고 쓰는 경우가 있으므로 한 번 알아보자.

## CommonDBCP Properties

- initialSize: 커넥션을 처음 생성할 때 몇 개를 만들지를 결정한다. (default: 0)
- minIdle: 커넥션 풀에 유지하고자 하는 최소 커넥션 수를 의미한다. (default: 0)
- maxIdle: 커넥션 풀에 유지하고자 하는 최대 커넥션 수를 의미한다. (default: 8)
- maxActive: 동시에 사용할 수 있는 최대 커넥션 수를 의미한다. (default: 8)
- maxWait: 커넥션 풀에서 커넥션을 얻어올 때까지 대기하는 최대 시간을 의미한다. (default: inf)
- testOnBorrow: 커넥션을 풀에서 얻어올 때 테스트 쿼리를 수행하여 커넥션이 유효한 상태인지를 확인한다. validationQuery가 설정되어있어야 한다. (default: true이나 false 권장)
- validationQuery: 커넥션의 유효성을 검증하기 위해 호출되는 SQL문으로 보통 SELECT 1 같은 쿼리를 사용한다.
- testOnWhileIdle: 커넥션 풀 안의 커넥션들을 대상으로 유효성 검사를 수행한다. 얼마나 자주 실행할 지는 아래의 설정과 함께 사용한다. (default: false)
- timeBetweenEvictionRunsMillis: 커넥션 풀의 유효상태를 검사하는 Evictior Thread를 얼마나 자주 실행할 지를 설정한다. (default: -1)
- minEvictableTimeMillis: Evictor Thread 동작 시에 커넥션이 설정된 시간보다 오랫동안 사용되고 있지 않은 경우 제거된다.

이렇게 주요한 설정들을 알아봤긴 하지만 어떻게 설정해야 하는지가 또 중요하다.

### 커넥션 풀 사이즈

먼저 커넥션 풀 사이즈는 어떤 어플리케이션을 서비스하냐에 따라 천차만별이다. 사용하고 있는 DB에 따라서 최대로 만들 수 있는 커넥션 수가 있고, 이 수치를 모든 어플리케이션에서 나눠서 사용할 수 있다. 예를 들어 MySQL이 최대 5만개의 커넥션을 지원하고 하나의 장비에서 100개의 커넥션 풀을 생성해둔다면, 최대 500대의 장비까지 이용할 수 있다. 하지만 이렇게 MySQL의 모든 커넥션을 사용하는건 바람직하지 않다. 항상 해당 DB를 사용하는 서비스가 늘어날 수 있음을 염두해두고, 최악의 경우 50%까지의 커넥션만 사용한다고 생각하는 것이 좋다.

적절한 수치는 application에 따라 다르지만 이용자 수가 많지 않고, 무거운 연산이 없다면 커넥션 풀은 10개로도 충분하다. 10개로 설정하기로 결정했다면 다음과 같이 설정하는 것이 좋다.

```
initialSize: 10
minIdle: 10
maxIdle: 10
maxActive: 10
```

이렇게 설정하면 커넥션 풀에는 항상 10개의 커넥션이 고정적으로 존재하므로 별도의 커넥션이 만들어지고 사라지는 부하가 발생하지 않기 때문이다.

만약, minIdle 값을 5로 설정했다면 Evictor Thread가 커넥션이 놀고있는 커넥션을 제거하여 5개로 유지하게 된다. 그런데 갑자기 5개의 이상이 요청이 들어올 경우 커넥션을 새로 만들어야 하므로 커넥션을 생성하는 부하가 발생한다.

만약, maxActive:10, maxIdle:5, minIdle:5라고 설정했다면 5개이상의 요청이 들어왔을 때 새로운 커넥션을 만들게 된다. 새로 만들게 되는 이유는 maxActive가 10개이므로 5개까지는 더 동시에 사용할 수 있기 때문인데, 이때 생성된 커넥션은 커넥션 풀이 가득 차 있기 때문에 계속 생성하고 사용 후에 버려지기 때문에 리소스 낭비가 심하게 된다.

따라서 Connection pool을 사용할 거라면 위의 세 가지 설정은 반드시 같은 값으로 설정하고 사용하는 것이 좋다. DB에 연결되고 있는 커넥션이 너무 많아서 극한으로 튜닝해야 하는 상황이 아니라면 말이다. 하지만 그 정도의 상황이라면 DB가 이미 정상적인 상황이 아닐 것 같다.

### 유효성 검사

커넥션 풀에 커넥션을 유지하는 것은 매우 좋은 전략이다. 하지만 커넥션 풀에 있는 모든 커넥션이 사용가능한 상태라고 보장할 수는 없다. 네트워크 이슈로 커넥션이 망가질 수도 있고, DB의 재시작으로 인해 커넥션이 전부 만료될 수도 있다. 또 MySQL의 경우 커넥션이 유지될 수 있는 최대 시간을 의미하는 wait_timeout이라는 값이 존재한다. 이 시간이 다 되면 커넥션은 유효하지 않은 상태가 된다. 하지만 이 모든 상황을 application에 있는 connection pool은 알 수가 없다. 따라서 커넥션 풀 안에 들어있는 커넥션이 망가져있을 수 있다는 이야기가 된다.

따라서 커넥션 풀에서 커넥션을 꺼낼 때 이 커넥션이 망가져있을 수 있으므로 '검증'하는 로직이 필요하다. 우리 서비스가 매우 중요한 서비스라서 100% 요청이 성공해야 한다면 `testOnBorrow` 설정을 true로 하고 `validaitonQuery`를 `SELECT 1`과 같은 값으로 설정하면 좋다. 하지만 이 설정은 쿼리를 하기 전에 반드시 validationQuery를 수행하므로 DB의 네트워크 부하는 2배로 늘어나게 된다. 따라서 가급적이면 사용성이 낮은 Admin과 같은 환경에서 설정하는 것이 유리하다.

반대로 우리 서비스가 트래픽이 어느정도 있는 상황이고, 실패하더라도 재처리가 가능한 구조라면 `testOnBorrow` 설정보다는 `testOnWhileIdle` 설정이 유리할 수 있다. `testOnWhileIdle`은 `timeBetweenEvictionRunsMillis`와 같이 설정해야 하며 특정 주기마다 validationQuery를 수행해서 유효하지 않은 커넥션은 제거하고 새로 만들게 된다. 이때 주기가 너무 짧으면 부하가 많이 발생할 수 있다. 권장 시간은 1분, 5분, 10분정도이다.

이와 별개로 mySQL에는 autoReconnect라는 JDBC Driver 설정이 있다. 이 설정을 주면 커넥션 유효성 검사 없이도 커넥션에 문제가 있는 경우, 재연결을 시도할 수 있다. 언뜻보기에는 좋아보이지만, 치명적인 문제가 있다. 커넥션에 문제가 있는 경우 SQLException을 발생시키고 재접속 처리를 수행하는데, SQLException은 RuntimeException이 아니므로 `@Transactional` 애노테이션을 붙였더라도 롤백처리가 제대로 되지 않는다. 따라서 권장되지 않는 방법이다.

### Common DBCP2 설정 코드

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-dbcp2</artifactId>
    <version>2.2.0</version>
</dependency>
```

```yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    poolName: test
    validationQuery: SELECT 1
    testOnBorrow: false
    testWhileIdle: true
    initialSize: 20
    minIdle: 20
    maxIdle: 20
    maxActive: 20
    maxWait: 2000
    timeBetweenEvictionRunsMillis: 80000
```

```java
public class DatabaseConfig {
  @Bean
  @ConfigurationProperties(prefix = "spring.datasource")
  public DataSource dataSource() {
    return DataSourceBuilder.create()
                            .url(url)
                            .username(username)
                            .password(password)
                            .build();
  }
}
```

<hr><br>

## HikariCP Properties

https://github.com/brettwooldridge/HikariCP#frequently-used

- minimumIdle: 커넥션 풀에 존재할 수 있는 최소 커넥션 수를 의미한다. (default: maximumPoolSize와 동일)
- maximumPoolSize: 커넥션 풀에 존재하는 커넥션 수 + 커넥션 풀에 존재하지 않는 커넥션 수를 의미하므로 최대로 가질 수 있는 커넥션 수를 의미한다.
- connectionTimeout : 커넥션을 얻을 때까지 대기하는 시간을 의미한다.
- idleTimeout: 커넥션 풀에서 커넥션이 사용되지 않은 상태로 얼마나 오랫동안 머무를 수 있는지를 의미한다. 오래된 커넥션은 `minimumIdle` 개수까지만 줄어들 수 있다. 이 동작은 `minimumIdle`의 개수가 `maximumPoolSize`보다 작은 경우에만 동작한다.
- maxLifeTime: 커넥션 풀에 있는 커넥션의 생명주기를 의미한다. 해당 시간이 지나면 커넥션 풀에서 커넥션은 사라지게 된다. 단, 사용 중인 커넥션은 사용이 끝날 때까지 폐기되지 않는다.
- keepAliveTime: 커넥션 풀의 상태를 어떤 주기로 체크할 지를 의미한다. 이때 커넥션이 유효한지 확인하기 위해서 JDBC의 `ping`이나 `connectionTestQuery`가 수행된다. (default: 0 비활성화)

### 커넥션 풀 사이즈

Common DBCP와 마찬가지로 minimumIdle 수치와 maximumPoolSize를 같게 설정하는 것이 좋다. 그 이유는 minimumIdle만큼의 커넥션이 이미 사용 중이라면, 계속 새로운 커넥션이 만들어지고 사라지는 과정이 필요하기 때문이다. 만약  사용 중인 커넥션의 개수가 maximumPoolSize에 도달한 경우, 다음 요청은 커넥션을 얻기 위해 connectionTimeout의 시간만큼 대기하게 된다. 만약 connectionTimeout을 30초로 사용하고 있다면 해당 요청은 최대 30초까지 대기하게 된다.

### 유효성 검사

HikariCP의 커넥션 관리 방법과 유사하다. `maxLifeTime`이라는 값을 지정하여 너무 오랫동안 살아있는 커넥션은 의도적으로 끊어준다. 일반적으로 연결 시간에 제한이 있는 데이터베이스와 통신할 때 커넥션이 갑자기 끊어지는 것을 막기 위해 사용된다. DB에서 하나의 커넥션을 사용할 수 있는 시간이 30분이라면 `maxLifeTime`은 해당 수치보다 무조건 작게 설정해야 한다. `maxLifeTime`이 작을수록 커넥션을 자주 없애고 새로 만들지만, 이때 부하가 한 번에 몰리는 것을 방지하기 위해 커넥션마다 생명주기에 약간의 오차를 주는 방식을 사용한다.

이상적인 상황에서는 위의 설정만으로도 충분하다. 하지만 의도치않은 DB의 재시작이나 네트워크 오류로 인해서 커넥션이 지정된 시간보다 빨리 망가져버린 경우에는 특정 쓰레드가 유효하지 않은 커넥션을 얻게될 수 있고, 새로운 커넥션을 받는만큼의 오버헤드가 발생할 수 있다. 만약 이런 상황 조차도 용납하기 어려운 서비스라면, `keepAliveTime` 설정도 고려해볼만하다.

`keepAliveTime`은 특정 주기마다 해당 커넥션이 유효한 지를 확인해서 connection pool에 유효하지 않은 커넥션이 존재할 확률이 대폭 낮춰준다. 하지만 DBCP에서 살펴봤던 것처럼 너무 잦은 `keepAliveTime`은 DB에 많은 testQuery를 발생시켜 DB 부하를 높일 수도 있다. 따라서 본인의 상황을 고려해서 설정하는 것이 좋다.

### HikariCP 설정 코드

application.yml 설정 파일에 다음과 같이 설정한다.

```yml
spring:
  datasource:
    hikari:
      jdbc-url: ${POSTGRE_URL}
      username: ${POSTGRE_USER}
      password: ${POSTGRE_PASSWORD}
      maximum-pool-size: 50
      connection-timeout: 5000
      max-lifetime: 50000
```

그 외 설정할 수 있는 프로퍼티 및 default 값을 보려면 `com.zaxxer.hikari.HikariConfig`를 살펴보면 된다.

```java
public class DatabaseConfig {
  @Bean
  @ConfigurationProperties(prefix = "spring.datasource.hikari")
  public DataSource budsDataSource() {
    return DataSourceBuilder.create()
                            .build();
  }
}
```

이렇게 설정하면 `spring.datasource.hikari` 밑에 있는 프로퍼티들이 `HikaraDataSource`를 만들 때 바인딩된다.

