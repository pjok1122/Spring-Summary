# 데이터 엑세스 기술(토비 Spring 3.1 Vol2. chapter3)

## 공통 개념

- 데이터 액세스 계층은 `DAO 패턴`이라 불리는 방식으로 분리하는 것이 원칙이다. DAO 패턴이란, DTO 또는 도메인 오브젝트만을 사용하는 인터페이스를 통해 데이터 액세스 기술을 외부에 노출하지 않도록 만드는 것이다.

- **DAO 내부에서 발생하는 예외는 모두 런타임 예외로 전환**해야 한다. 서비스 계층 코드는 DAO가 던지는 대부분의 예외는 직접 다뤄야 할 이유가 없다.

- 데이터 액세스 기술을 사용하는 코드는 대부분 `try/catch/finally`와 판에 박힌 반복된 코드로 작성되기 쉽다. 스프링은 판에 박힌 코드를 피하고 꼭 필요한 내용만을 담을 수 있도록 `데이터 액세스를 위한 템플릿을 제공`한다.

## DataSource

JDBC를 통해 DB를 사용하려면 Connection 타입의 DB 연결 오브젝트가 필요하다. 보통 미리 정해진 개수만큼 DB 커넥션을 `DB 커넥션풀`에 준비해두고, 애플리케이션이 요청할 때마다 풀에서 꺼내 하나씩 할당해주고 사용이 끝나면 풀에 반납하는 풀링 기법을 사용한다. `DataSource`는 DB 커넥션 풀을 관리하는 기능을 제공한다.

스프링에서는 `DataSource`를 빈으로 등록하기를 강력히 권장한다. 단, 개발 중에 사용하던 테스트용 `DataSource`가 운영서버에는 적용이 되지 않을 수 있으므로 개발용 테스트용 따로 만들어 사용하는 것이 좋다.

### SimpleDriverDataSource

스프링이 제공하는 가장 단순한 DataSource 구현 클래스. 커넥션 풀을 관리하지 않으므로 테스트용으로만 사용한다.

### SingleConnectionDataSource

하나의 물리적인 Connection만 만들어놓고 재사용하는 DataSource. 쓰레드 환경에서 사용하면 안된다.

### 아파치 Commons DBCP

가장 유명한 오픈소스 DB 커넥션 풀 라이브러리다.

### c3p0 JDBC/DataSource Resource Pool

c3p0는 JDBC 3.0 스펙을 준수하는 Connection과 Statement 풀을 제공하는 라이브러리다.

### JNDI/WAS DB 풀

대부분의 자바 서버는 자체적으로 DB 풀 서비스를 제공한다. 서버가 제공하는 DB 풀을 사용해야 하는 경우에는 `JNDI를 통해 서버의 DataSource에 접근`해야 한다.

서버에 `jdbc/DefaultDS`라는 이름으로 등록된 서버의 데이터 소스가 있다고 하면, `<jee:jndi-lookup id="dataSource" jndi-name="jdbc/DefaultDS" />` 태그를 이용해 빈으로 등록이 가능하다.

JNDI에서 검색해서 가져오는 빈 오브젝트는 서버 밖에서는 제대로 동작하지 않기 때문에 테스트환경에서 활용하기 어렵다. 스프링 테스트 프레임워크에서 제공하는 JNDI 목 오브젝트를 사용해서 이를 해소하자.

## JDBC

JDBC는 자바의 데이터 액세스 기술의 기본이 되는 로우레벨의 API다. DB 벤더와 개발팀은 JDBC(인터페이스)를 구현한 드라이버를 제공하기 때문에 사용 방법은 **DB가 변경되어도 그대로 사용할 수 있다.**

최신 ORM 기술도 내부적으로는 DB와의 연동을 위해 JDBC를 이용한다.

JDBC는 간단한 SQL을 하나 실행하는 데도 매우 번잡한 코드가 필요하고, DB에 따라 일관성 없는 정보를 가진 채로 던져지는 체크 예외를 처리해야 하며, SQL은 코드 내에서 직접 문자로 제공해야 하는 등의 불편을 감수해야 한다. `스프링 JDBC`는 이러한 JDBC 개발의 단점을 `템플릿/콜백 패턴`을 이용해 극복할 수 있게 해주고, 가장 간결한 형태의 API 사용법을 제공한다.

## Spring JDBC

`SimpleJdbcTemplate`, `SimpleJdbcInsert`, `SimpleJdbcCall` 을 이용하여 최소한의 코드만으로 JDBC의 모든 기능을 활용할 수 있다.

_cf) `SimpleJdbcTemplate`은 현재 Deprecated 되었지만, JdbcTemplate 또는 `NamedParameterJdbcTemplate`을 이용하면 모든 기능을 그대로 사용할 수 있다._

### Spring JDBC가 해주는 일

- Connection 열고 닫기
- Statement 준비와 닫기
- Statement 실행
- ResultSet 루프 : ResultSet에 담긴 쿼리 실행 결과가 한 건이상이라면 루프를 돌며 각각 처리해준다.
- 예외처리와 변환 : JDBC 작업 중 발생하는 모든 예외는 스프링 JDBC의 예외 변환기가 처리해준다.
- 트랜잭션 처리

## SimpleJdbcTemplate

스프링 Jdbc를 사용한다면 가장 많이 이용하게 될 JDBC용 템플릿이다. SimpleJdbcTemplate이 제공하는 주요한 메서드를 예제 코드와 함께 살펴보는 것이 주 내용이다.

### SimpleJdbcTemplate 생성

`SimpleJdbcTemplate`는 `Thread-safe`하기 떄문에 싱글톤 빈으로 만들어두고 DI받아 사용할 수 있다. 하지만 스프링 개발자는 관례적으로 DAO의 코드에서 DataSource를 제공받아서 `SimpleJdbcTemplate`을 생성하는 방식을 선호한다.

**예시 코드**

```java
public class MemberDao{
    SimpleJdbcTemplate simpleJdbcTemplate;

    @Autowired
    public void init(DataSource dataSource){
        this.simpleJdbcTemplate = new SimpleDriverDataSource(dataSource);
    }
}
```

### SQL 파라미터

SQL에 매번 달라지는 값이 있는 경우에는 스트링 조합으로 SQL을 만들기보다는 `?`와 같은 치환자를 넣어두고 파라미터 바인딩 방법을 사용하는 것이 편리하다. 스프링 JDBC에서는 `?`와 같은 치환자 말고도, `이름 치환자`를 제공한다.

- `INSERT INTO MEMBER(ID,NAME,POINT) VALUES(?,?,?);`
- `INSERT INTO MEMBER(ID,NAME,POINT) VALUES(:id, :name, :point);`

이름 치환자는 맵이나 오브젝트에 담긴 내용을 키 값이나 프로퍼티 이름을 이용해 바인딩할 수 있다. 즉, Map 오브젝트는 이름 치환자를 가진 SQL과 함께 SimpleJdbcTemplate에 전달되어 바인딩 파라미터로 사용된다.

Map 오브젝트말고도, `BeanPropertySqlParameterSource`를 사용하면 도메인 오브젝트나 DTO를 사용하게 해준다.

**예시 코드**

```java
simpleJdbcTemplate.update("INSERT INTO MEMBER(ID,NAME,POINT, args) VALUES(:id, :name, :point)", new MapSqlParameterSource()
.addValue("id",1).addValue("name", "Spring").addValue("point", 3.5));
```

```java
Member member = new Member(1,"Spring", 3.5);
simpleJdbcTemplate.update("INSERT INTO MEMBER(ID,NAME,POINT, args) VALUES(:id, :name, :point)", new BeanPropertySqlParameterSource(member));
```

### SQL 조회 메서드

#### 리턴 값이 단일 컬럼, 단일 로우 경우

```java
int count = simpleJdbcTemplate.queryForInt("select count(*) from member
            where point> :min", new MapSqlParameterSource("min", min));
```

```java
String name = simpleJdbcTemplate.queryForObject("select name from member where id=?", String.class, id);
```

#### 리턴 값이 다중 컬럼, 단일 로우 경우

- `BeanPropertyRowMapper`는 주어진 클래스의 프로퍼티 이름과 SQL 결과를 자동 매핑해주는 `RowMapper`의 구현체이다.

```java
Member member = simpleJdbcTemplate.queryForOjbect("select * from member where id=?", new BeanPropertyRowMapper<Member>(Member.class), id);
```

```java
Map<String,Object> map = simpleJdbcTemplate.queryForMap("select * from member where id=?", id);
```

#### 리턴 값이 다중 컬럼, 다중 로우인 경우

```java
List<Member> members = simpleJdbcTemplate.query("select * from member where point> ?", new BeanPropertyRowMapper<Member>(Member.class), point);
```

### SQL 배치 메소드

SQL 배치 메소드는 update()로 실행하는 SQL들을 배치 모드로 실행하게 해준다. 내부적으로 JDBC statement의 addBatch()와 executeBatch() 메소드를 이용해 여러 개의 SQL을 한 번에 처리한다. DB 호출을 최소화할 수 있기 때문에 성능이 향상될 수 있다.

```java
dao.simpleJdbcTemplate.batchUpdate("update member set name = :name where id= :id", new SqlParameterSource[] {
    new MapSqlParameterSource().addValue("id",1).addValue("name","Spring3"),
    new BeanPropertySqlParameterSource(new Member(2, "Book3"))
});
```

## SimpleJdbcInsert

SQL을 이용하는 DB 프로그래밍의 가장 귀찮은 일은 비슷한 구조의 SQL을 반복적으로 만들어야 한다는 점이다. 게다가 개발자가 직접 타이핑하기 때문에 `Type Safe`하지 못하다는 단점도 있다. `SimpleJdbcInsert`는 귀찮은 Insert문을 작성하지 않도록 도와주는 객체이다.

SimpleJdbcInsert는 `Thread-safe`하지만, 테이블마다 객체를 생성해야 하므로 빈으로 등록해서 사용하기에는 적합하지 않다.

### SimpleJdbcInsert 생성

- 테이블은 메소드 체인형태로 `.withTableName()`으로 지정할 수 있다.

```java
SimpleJdbcInsert jdbcInsert = new SimpleJdbcInsert(dataSource).withTableName("member");
jdbcInsert.execute(new BeanPropertySqlParameterSource(new Member(1,"Spring", 3.5)));
```

- 자동으로 설정되는 키를 지정(`usingGeneratedKeyColumns`)하고, 그 키를 반환받을 때에는 `executeAndReturnKey`를 사용한다.

```java
SimpleJdbcInsert registerInsert = new SimpleJdbcInsert(dataSource)
                .withTableName("register")
                .usingGeneratedKeyColumns("id");
int key = registerInsert.executeAndReturnKey(
        new MapSqlParameterSource("name", "Spring")).intValue();
```

<br><hr>

## MyBatis

`MyBatis`는 자바 객체와 SQL 문 사이의 자동매핑 기능을 지원하는 ORM 프레임워크다. `MyBatis`의 가장 큰 특징은 SQL을 자바 코드에서 분리해서 별도의 XML 파일 안에 작성하고 관리할 수 있다는 점이다. (XML 말고 `@Select`와 같은 애노테이션을 이용해서 매핑할 수도 있다.)

MyBatis도 내부적으로는 JDBC API를 사용하기 때문에 JDBC 커넥션에 관한 메타정보들을 properties에 저장해야 한다.

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/springdb
spring.datasource.username=yjhn0715
spring.datasource.password=pass
```

### 객체 생성

```java
package youngjae.study.model;

@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
@ToString
@Alias("car")
public class Car {
	private int id;
	private String name;
	private String description;
}
```

롬복을 이용해서 `Car`라는 객체를 만들었다. 파일의 경로는 `youngjae.study.model`이다.

### 객체 매퍼 생성

```java
package youngjae.study.mapper;

@Mapper
public interface CarMapper {
	Car findById(int id);
	List<Car> selectAllCars();
	void insertCar(Car car);
}
```

`@Mapper` 애노테이션이 붙은 인터페이스는 `MapperScan`에 의해 읽어져, CarMapper를 구현한 프록시가 생성된다. 현재 파일의 경로는 `youngjae.study.mapper.*` 이므로 이 경로를 `MapperScan` 대상으로 지정해야 한다.

### MapperScan

```java
package youngjae.study.config;

@Configuration
@MapperScan(basePackages = "youngjae.study.mapper")
public class MybatisConfig {}
```

MapperScan은 `basePackages`를 지정하여 어느 패키지를 `@Mapper` 스캔 대상으로 지정할지 결정할 수 있다. 우리의 `CarMapper`는 `youngjae.study.mapper.*`에 존재하므로 위의 예시처럼 경로를 전달한다.

### 매핑파일(\*.xml)

이제 SQL문을 작성하고 객체와 연결시켜주는 xml파일을 생성해야 한다. 위에서 CarMapper 인터페이스가 `findById`, `selectAllCars`, `insertCar` 3개의 메서드를 가지고 있으므로 3개의 메소드에 대한 SQL문을 작성해야 한다.

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="youngjae.study.mapper.CarMapper">

	<select id="findById" parameterType="int" resultType="youngjae.study.model.Car">
		SELECT ID, NAME, DESCRIPTION
		FROM CAR
		WHERE ID = #{id}
	</select>

	<select id="selectAllCars" resultType="car">
		SELECT *
		FROM CAR
	</select>

	<insert id="insertCar">
		INSERT INTO CAR(NAME, DESCRIPTION)
		VALUES(#{name}, #{description})
	</insert>

</mapper>
```

- DOCTYPE 부분은 매핑 파일이라는 정보를 의미한다.
- `namespace`는 해당 xml에서 사용되는 경로는 `youngjae.study.mapper.CarMapper`로 시작한다는 의미이다. 이렇게 namespace를 두는 이유는 객체와 xml 파일에 있는 `id`를 쉽게 매핑시키기 위함이다. 우리가 앞서 정의한 `Mapper`에서 `findById`에 대한 경로를 나열해보면, `youngjae.study.mapper.CarMapper.findById`가 된다. 따라서 우리는 `<select id="youngjae.study.mapper.CarMapper.findById"> SQL Query </select>`로 매핑을 시킬 수 있다. 하지만 이렇게 작성하면 반복되는 코드가 많아지고 길이가 길어진다. 따라서 우리는 `youngjae.study.mapper.CarMapper`를 namespace로 두고 사용한다. 사실상 baseURL과 비슷한 개념이라고도 볼 수 있겠다.

- `parameterType`은 SQL문의 매개변수로 전달되는 객체의 타입을 의미한다.

- `resultType`은 SQL 쿼리의 실행결과로 전달되는 객체의 타입을 의미하는데, 결과가 `List` 타입이어도 요소 한개의 타입만을 작성한다. 따라서 selectAllCars()의 타입도 컬렉션이 아니라 `car`라는 단일 객체 타입이다.

- `resultType`은 `findById`에 명시한 것처럼 `youngjae.study.model.Car`라는 풀네임으로 작성해도 되지만, `Alias`를 이용해서 줄여쓸 수 있다. `youngjae.study.model.Car`에 대한 Alias를 `car`로 지정해뒀다고 보면 된다.

- `Alias`를 지정하는 방법은 해당 클래스를 정의할 때, `@Alias("car")`와 같이 별칭을 지정하면 끝이다.

### properties

위의 설정대로 진행해도 어플리케이션은 동작하지 않는다. 가장 큰 이유는 매핑파일(`*.xml`)을 어느 패키지에서 읽어야 할지 결정하지 않았다는 사실과 @Alias 애노테이션을 어느 패키지에서 읽어야 할지 결정하지 않았기 때문이다.

```properties
# 어느 파일이 mapper.xml의 대상인지 결정.
mybatis.mapper-locations=classpath:/mapper/*.xml

# alias를 읽어들일 패키지가 어디인지 결정.
mybatis.type-aliases-package=youngjae.study.model
```

### 테스트 코드

```java
@Component
public class MybatisRunner implements ApplicationRunner {

	@Autowired
	CarMapper carMapper;

	@Override
	public void run(ApplicationArguments args) throws Exception {
		carMapper.insertCar(Car.builder().name("carA").description("cheaper").build());
		carMapper.insertCar(Car.builder().name("carB").build());
		List<Car> selectAllCars = carMapper.selectAllCars();
		selectAllCars.forEach(System.out::println);
	}
}
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class CarMapperTest {

	@Autowired
	CarMapper carMapper;

	@Test
	public void select_car() {
		carMapper.insertCar(Car.builder().name("carC").description("car C is faster than others").build());
		List<Car> selectAllCars = carMapper.selectAllCars();
		selectAllCars.forEach(System.out::println);
	}
}
```

assertThat()으로 검증하는 것이 일반적이지만, 마땅한 테스트 코드를 만들기가 애매해서 가시적으로 출력문만 뽑아봤다.

<br><hr>
