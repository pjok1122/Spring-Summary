## 인메모리 데이터 베이스 H2

인메모리 데이터베이스는 `H2`, `HSQL`, `Derby` 세 가지가 있다. 콘솔의 기능이 있는 H2가 그나마 사용하기 좋다.

Dependency로 `web`, `JDBC`, `H2`를 추가하자.

pom.xml에는 다음과 같은 의존성이 추가된다.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

### DataSource, JdbcTemplate

스프링 부트는 `Spring-JDBC`가 클래스 패스에 존재하면 `DB`와 관련된 빈들을 자동설정 해준다. 자동등록되는 빈 중에서 가장 대표적인 빈 두 개가 `DataSource`와 `JdbcTemplate` 이다.

- `DataSource`는 커넥션 풀을 관리하는 객체인데, 자동으로 `HikariDataSource`를 등록해주는 것으로 보인다.
- `JdbcTemplate`은 지저분한 `JDBC API`를 좀 더 사용하기 쉽게 템플릿을 제공한다. 따라서 개발자는 Connection 생성,소멸 등을 관리할 필요가 없어진다.

### h2 콘솔 허용

h2 콘솔을 사용하기 위해서는 `spring-dev-tools` 의존성을 추가하거나, `spring.h2.console.enabled=true`로 설정해주면 된다. 설정이 끝나면, `/h2-console`로 데이터베이스를 확인할 수 있다.

- 커넥션에 대한 정보를 얻고 싶은 경우

```java
connection.getMetaData().getURL();
connection.getMetaData().getUserName();
```

### 사용 예시

`JdbcTemplate`이 제공하는 기능을 한 눈에 살펴보기 위해, 테이블을 생성할 때에는 `JDBC API`를 이용하여 작성했다. `JDBC API`를 이용하는 경우 반드시 커넥션을 잘 닫아줘야 한다. 코드가 길어지므로 여기서는 생략했다.

**예제 코드**

```java
@Component
public class H2Runner implements ApplicationRunner {

	@Autowired
	DataSource dataSource;

	@Autowired
	JdbcTemplate jdbcTemplate;
	SimpleJdbcInsert simpleJdbcInsert;

	@Override
	public void run(ApplicationArguments args) throws Exception {

        // 기존의 JDBC API. 추가적으로 try/catch를 이용해 예외 발생에 대한 처리를 해야 한다.
		Connection connection = dataSource.getConnection();
		System.out.println(connection.getMetaData().getURL());
		System.out.println(connection.getMetaData().getUserName());
		Statement statement = connection.createStatement();
		String sql = "CREATE TABLE USER(ID INTEGER NOT NULL, NAME VARCHAR(255), PRIMARY KEY(id))";
		statement.executeUpdate(sql);
		statement.close();
		connection.close();

        // Spring jdbc에서 제공하는 JdbcTemplate을 이용하면 반복되는 작업을 작성하지 않을 수 있다.
		jdbcTemplate.update("INSERT INTO USER VALUES(?,?)", 1, "name");

        // SimpleJdbcInsert는 insert문을 작성하지 않도록 도와준다. 생성시 단순히 테이블을 지정하고, 값을 넣어줄 컬럼 또는 자동으로 넣어줄 컬럼을 선택하면 된다.
		simpleJdbcInsert = new SimpleJdbcInsert(dataSource).withTableName("USER");
		simpleJdbcInsert.execute(new MapSqlParameterSource().addValue("id", 5).addValue("name", "Youngjae"));
	}
}
```

<br><hr>

## DBCP

`DBCP(Database Connection Pool)`를 지원하는 제품은 `HikariCP`, `Tomcat CP`, `Commons DBCP2` 정도가 있다. 스프링 부트는 `HikariCP`를 default로 설정하고 있다.

DBCP에 대한 설정을 추가하고싶다면, `application.properties`에 `spring.datasource.hikari.*` 값을 수정하면 된다. 각 프로퍼티들이 갖는 정보는 공식문서를 참고하는 편이 좋다.

[hikari](https://github.com/brettwooldridge/HikariCP#frequently-used)

조금만 더 첨언하자면, 커넥션은 CPU 코어의 개수만큼 동시 실행이 가능하다. 이를 고려하여 maximum-pool-size를 결정하자.

## Mysql 사용하기

Mysql을 사용하기 위해서는 먼저 Mysql에 접속할 수 있도록 도와주는 커넥터를 설치해야한다.

- 의존성 추가

```xml
<dependency>
   <groupId>mysql</groupId>
   <artifactId>mysql-connector-java</artifactId>
</dependency>
```

### 계정 생성

의존성 추가가 끝났으면, 사용하고자 하는 데이터베이스와 데이터베이스에 접근이 가능한 계정을 만들어 주어야 한다.

- 계정 생성 방법. '%'는 외부에서의 접속을 허용하는 설정입니다.

```sql
create user id@localhost identified by password;
create user id@'%' identified by password;
```

- 권한 부여 방법

```sql
grant all privileges db이름.테이블이름 to id;
```

### DB property 설정하기

DB 로그인 관련 프로퍼티 application.properties에 설정한다.

```properties
spring.datasource.hikari.maximum-pool-size=8
spring.datasource.url=jdbc:mysql://localhost:3306/springdb?serverTimezone=UTC
spring.datasource.username=yjhn0715
spring.datasource.password=1234
```

**예제 코드**

```java
@Component
public class MysqlRunner implements ApplicationRunner {

	@Autowired
	DataSource dataSource;

	@Autowired
	JdbcTemplate jdbcTemplate;

	@Override
	public void run(ApplicationArguments args) throws Exception {
		System.out.println(dataSource.getClass());	//HikariDataSource

//		jdbcTemplate.execute("CREATE TABLE `USER`(\r\n" +
//				"	seq BIGINT(10) NOT NULL AUTO_INCREMENT,\r\n" +
//				"	id VARCHAR(20) NOT NULL UNIQUE,\r\n" +
//				"	name VARCHAR(10),\r\n" +
//				"	PRIMARY KEY (`seq`),\r\n" +
//				"	UNIQUE KEY uniq_key(`id`)\r\n" +
//				");");

		jdbcTemplate.update("DELETE FROM USER");
		jdbcTemplate.update("INSERT INTO USER(id,name) VALUES(?,?)",
				 "pjok1122", "yj");
		jdbcTemplate.update("INSERT INTO USER(id,name) VALUES(?,?)",
				"yjhn0715", "yjhn");

		User user = jdbcTemplate.queryForObject("SELECT * FROM USER WHERE id=?",
				new BeanPropertyRowMapper<User>(User.class), "pjok1122");
		System.out.println(user);	//User [seq=10, id=pjok1122, name=yj]
	}
}
```

**docker 코드**

Mysql이 없는 경우 도커를 이용하면 편리하다.

```docker
docker run -p 3306:3306 --name mysql_boot -e MYSQL_ROOT_PASSWORD=1 -e MYSQL_DATABASE=springdb -e MYSQL_USER=yjhn0715 -e MYSQL_PASSWORD=1234 -d mysql
```

Mysql 라이센스(GPL) 는 소스 코드 공개의 의무가 있다. 따라서 Mysql보단 MariaDB를 사용하자. 사용 방법은 동일하다. docker 생성 시, `-d mysql`을 `-d mariadb`로만 변경한다.

- 사실 `mariadb`보다도 `PostgreSQL`을 사용하는게 낫다.

<br><hr>
