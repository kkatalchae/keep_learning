# 애노테이션 기반 설정으로의 전환

## XML vs 자바 Config

토비의 스프링은 XML 설정의 단점을 세 가지로 정리한다.

첫째, 작성량이 많다. 빈 하나를 등록하는 데도 반복적인 태그 구조를 매번 써야 한다. 설정이 늘어날수록 실제 의미있는 정보보다 구조를 위한 코드가 더 많아진다.

```xml
<bean id="userDao" class="com.example.UserDao">
    <property name="dataSource" ref="dataSource"/>
    <property name="sqlService" ref="sqlService"/>
</bean>
```

둘째, 오타에 취약하다. XML은 문자열 기반이라 컴파일러가 검증해주지 않기 때문에 오타가 나도 런타임에서야 발견된다.

```xml
<!-- 오타가 나도 컴파일 에러 없음 → 런타임에서야 발견 -->
<property name="datasorce" ref="dataSource"/>
```

셋째, 리팩토링이 번거롭고 안전하지 못하다. 클래스명이나 메서드명이 바뀌면 XML 전체를 수동으로 찾아서 고쳐야 한다. IDE의 리팩토링 도구가 XML 내부 문자열까지 안전하게 추적해주지 않는다.

Java Config를 사용하면 이 문제들이 해결된다. 빈 등록을 메서드 호출로 표현하기 때문에 코드가 간결해지고, 타입으로 연결되기 때문에 오타가 나면 즉시 컴파일 에러가 발생한다. IDE의 리팩토링 도구도 코드와 동일하게 동작한다.

```java
@Bean
public UserDao userDao() {
    return new UserDao(dataSource(), sqlService()); // 타입으로 연결 → 오타 즉시 컴파일 에러
}
```

Java Config가 이런 단점들을 해결해주지만, XML 같은 선언적 포맷이 사라진 건 아니다. 선언적 포맷은 구조가 단순하고 코드를 모르는 사람도 읽고 수정하기 쉽다는 장점이 있기 때문에 여전히 유효하다. 결국 둘 중 하나가 옳은 게 아니라 상황에 따라 판단해야 하는 문제다.

선언적 포맷과 코드 기반 설정 중 무엇을 선택할지 판단할 때 고려할 포인트는 네 가지다.

**설정이 정적인가, 동적인가.** 환경변수 몇 개 바꾸는 수준이면 선언적 포맷으로 충분하지만, 조건에 따라 다르게 조합하거나 동적 생성이 필요하면 코드가 유리하다.

**설정을 언제든 바꿀 수 있어야 하는가.** 코드 기반 설정은 변경 시 재배포가 필요하다. 자주 바뀌는 값이라면 환경변수나 외부 프로퍼티 파일로 분리하는 것이 낫다.

**설정 오류를 언제 발견하고 싶은가.** 런타임보다 빌드 타임에 오류를 잡고 싶다면 코드 기반이 유리하다. 선언적 포맷은 boilerplate가 많아질수록 오타나 실수가 숨을 공간도 많아진다.

**설정을 누가 관리하는가.** 개발자가 아닌 인프라팀이나 운영팀이 직접 수정해야 한다면 선언적 포맷의 진입장벽이 낮다.

이런 흐름은 애플리케이션 설정에만 국한되지 않는다. 인프라 영역에서도 서버, 네트워크 같은 자원을 수동으로 구성하던 방식이 Terraform, Pulumi 같은 도구를 통해 코드로 관리하는 **IaC(Infrastructure as Code)** 흐름으로 이어지고 있다. 기존 방식의 단점인 수동 작업의 실수, 환경 간 불일치, 변경 이력 추적 어려움을 극복하기 위한 움직임이라는 점에서 스프링이 XML에서 Java Config로 전환한 맥락과 맞닿아 있다.

## @Autowired 의 동작 원리와 생성자 주입

스프링은 `@Autowired`를 만나면 주입할 빈을 두 단계로 찾는다. 먼저 타입으로 찾고, 같은 타입이 여러 개면 이름으로 찾는다.

```java
// DataSource 타입의 빈이 두 개 등록된 경우
@Bean
public DataSource mainDataSource() { ... }

@Bean
public DataSource batchDataSource() { ... }

// 필드명이 'mainDataSource'와 일치하는 빈으로 결정
@Autowired
private DataSource mainDataSource;
```

타입으로도 결정이 안 되고 이름으로도 매칭되는 빈이 없으면 `NoUniqueBeanDefinitionException`이 발생한다. 이때는 `@Qualifier`로 명시적으로 지정할 수 있다.

```java
@Autowired
@Qualifier("mainDataSource")
private DataSource dataSource;
```

의존성을 주입하는 방법은 필드 주입, 수정자 주입, 생성자 주입 세 가지가 있다.

```java
// 1. 필드 주입
@Autowired
private UserDao userDao;

// 2. 수정자 주입
@Autowired
public void setUserDao(UserDao userDao) {
    this.userDao = userDao;
}

// 3. 생성자 주입
private final UserDao userDao;

public UserService(UserDao userDao) {
    this.userDao = userDao;
}
```

수정자 주입은 선택적 의존성이나 변경 가능한 의존성에 사용할 수 있지만, 객체가 완전히 초기화되기 전에 사용될 위험이 있다. 최근에는 생성자 주입이 권장되는데 이유는 네 가지다.

`final`로 선언할 수 있어서 객체 생성 이후 의존성이 바뀌지 않음을 보장한다. 필드 주입과 수정자 주입은 `final`을 쓸 수 없다.

필요한 의존성이 없으면 객체 자체를 만들 수 없기 때문에 누락된 의존성을 컴파일 타임에 발견할 수 있다. 필드 주입은 런타임에서야 `NullPointerException`으로 발견된다.

스프링 컨테이너 없이 순수 자바로 테스트가 가능하다. 필드 주입은 스프링 컨테이너 없이는 의존성을 주입할 방법이 없다.

```java
// 필드 주입 — 스프링 컨테이너 없이는 의존성 주입 불가
UserService service = new UserService(); // userDao가 null

// 생성자 주입 — 순수 자바로 테스트 가능
UserService service = new UserService(new MockUserDao());
```

A가 B를, B가 A를 주입받는 순환 참조가 있을 때 애플리케이션 시작 시점에 바로 감지한다. 필드 주입은 실제 호출 시점까지 모른다.

Lombok의 `@RequiredArgsConstructor`를 사용하면 `final` 필드에 대한 생성자를 자동 생성해줘서 생성자 주입의 boilerplate도 줄일 수 있다.

## @Configuration 과 @Bean 의 동작 원리

`@Configuration`은 스프링 빈 설정을 담당하는 **설정 클래스**를 정의하기 위한 애노테이션이다. 애플리케이션에 필요한 빈들을 한 곳에서 구성하고 조립하는 역할을 한다.

`@Component`는 스프링이 **클래스 자신을 빈으로 등록**하도록 하는 애노테이션이다. 주로 비즈니스 로직을 담은 클래스에 붙여 컴포넌트 스캔의 대상이 되게 한다.

```java
// @Configuration — 빈들을 조립하는 설정 클래스
@Configuration
public class AppConfig {
    @Bean
    public UserDao userDao() { return new UserDao(dataSource()); }

    @Bean
    public DataSource dataSource() { return new SimpleDriverDataSource(); }
}

// @Component — 자기 자신이 빈으로 등록되는 클래스
@Component
public class UserService {
    private final UserDao userDao;

    public UserService(UserDao userDao) { this.userDao = userDao; }
}
```

두 애노테이션은 스프링이 내부적으로 처리하는 방식이 다르다. `@Configuration`이 붙은 클래스는 스프링이 **CGLIB으로 프록시 서브클래스를 동적으로 생성**해서 컨테이너에 등록한다. 이 프록시가 `@Bean` 메서드를 오버라이드해서, 호출 시마다 컨테이너를 먼저 확인하는 로직을 끼워 넣는다.

```java
// 의사코드로 표현한 프록시의 동작
@Override
public DataSource dataSource() {
    if (컨테이너에 이미 존재) return 컨테이너에서 반환;
    else { 생성 → 컨테이너 등록 → 반환 }
}
```

`@Component`는 프록시 없이 클래스 자신이 그대로 등록된다. `@Component` 안에 `@Bean` 메서드를 정의해도 컨테이너에 빈으로는 등록되지만, 클래스 내부에서 해당 메서드를 직접 호출하면 프록시를 거치지 않아 매번 `new`로 새 인스턴스가 생성된다. 실무에서는 빈을 등록만 하고 외부에서 주입받는 패턴을 주로 사용하기 때문에 자주 마주치는 상황은 아니지만, 빈 설정 클래스를 작성할 때는 `@Configuration`을 사용하는 것이 안전하다.

## @Profile 과 @PropertySource — 환경별 설정 분리

배포 환경에 따라 설정값을 다르게 주어야 할 때 `@Profile`을 사용하면 활성화된 프로필에 따라 등록되는 빈을 다르게 할 수 있다.

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("local")
    public DataSource localDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }

    @Bean
    @Profile("prod")
    public DataSource productionDataSource() {
        HikariDataSource ds = new HikariDataSource();
        // ...
        return ds;
    }
}
```

`@PropertySource`를 사용하면 외부 프로퍼티 파일을 로드해서 `Environment`를 통해 값을 주입받을 수 있다.

```java
@Configuration
@PropertySource("classpath:database.properties")
public class DataSourceConfig {

    @Autowired
    Environment env;

    @Bean
    public DataSource dataSource() {
        SimpleDriverDataSource ds = new SimpleDriverDataSource();
        ds.setUrl(env.getProperty("db.url"));
        ds.setUsername(env.getProperty("db.username"));
        ds.setPassword(env.getProperty("db.password"));
        return ds;
    }
}
```

`Environment`는 프로퍼티의 출처를 추상화한다. System Properties, 환경변수, 프로퍼티 파일 등 어디서 값을 읽어오는지 코드가 신경 쓰지 않아도 된다. 동일한 코드에서 로컬엔 `.properties` 파일, 운영엔 환경변수로 주입받을 수 있고, 설정의 출처가 바뀌어도 애플리케이션 코드는 변경할 필요가 없다.

## 설정 클래스 분리와 조합

`@Import`를 사용하면 다른 설정 클래스를 명시적으로 가져와 조합할 수 있다.

```java
@Configuration
@Import({ DatabaseConfig.class, SecurityConfig.class })
public class AppConfig { }
```

`@ComponentScan`을 사용하면 지정한 패키지 하위를 스캔해서 `@Component`, `@Configuration` 등이 붙은 클래스를 자동으로 빈으로 등록한다.

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig { }
```

`@Import`는 명시적으로 콕 집어서 가져오는 방식, `@ComponentScan`은 범위를 지정해서 자동으로 긁어오는 방식이다.

Spring Boot에서는 이 두 가지를 포함한 여러 애노테이션이 `@SpringBootApplication` 하나로 합쳐져 있다.

```java
// @SpringBootApplication의 실제 구성
@SpringBootConfiguration        // = @Configuration
@EnableAutoConfiguration        // 자동 설정 활성화 (@Import 기반)
@ComponentScan                  // 현재 패키지 하위 자동 스캔
public @interface SpringBootApplication { }
```

`@SpringBootConfiguration`은 해당 클래스를 설정 클래스로 등록하고, `@ComponentScan`은 하위 패키지를 자동으로 스캔하며, `@EnableAutoConfiguration`은 내부적으로 `@Import`를 사용해 수백 개의 자동 설정 클래스를 조건부로 등록한다. `@SpringBootApplication` 하나로 합쳐져 있어서 직접 `@Import`를 쓸 일이 줄었지만, 내부적으로는 책에서 배운 메커니즘이 그대로 동작하고 있다.

