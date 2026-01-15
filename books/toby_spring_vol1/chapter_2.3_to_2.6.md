## 2.3 개발자를 위한 테스팅 프레임워크 JUnit

단위 테스트는 항상 일관된 결과를 보장해야 한다.

하지만 UserDaoTest 는 항상 같은 결과를 보장하지 않는다. add() 를 통해서 유저를 추가하고 다시 add() 를 테스트할 때 이전에 실행된 유저가 DB 에 생겼기 때문에 동일한 ID 로 인해 테스트는 실패할 것이다. 

위의 경우 테스트가 동일한 결과를 반환하지 않는 이유는 기존 결과에 영향을 받기 때문이다. 이를 해결하기 위해서 deleteAll(), getCount() 메소드를 추가해보자. 

```java
@Test
public void addAndGet() {

    dao.deleteAll();
    assertThat(dao.getCount(), is(0));

    // 기존 로직

}
```

이제 유저를 추가하기 전에 유저 정보를 모두 지우기 때문에 항상 동일한 결과를 반환할 것이다. 

getCount() 라는 함수에 대해서 assertThat 으로 검증을 했지만 하나의 결과만으로 다른 오류가 없다는 것을 확신하는 것은 굉장히 위험한 일이다. 그러므로 좀 더 꼼꼼하게 검증해보도록 하자.

테스트 시나리오는 다음과 같다.
1. deleteAll() 을 통해서 DB 를 비운다.
2. getCount() 를 통해서 사용자가 0명임을 확인한다.
3. 반복해서 유저를 추가한다.
4. 추가될 때마다 getCount() 를 통해서 유저가 1명씩 늘어나는지 확인한다.

```java
@Test
public void count() {

    dao.deleteAll();
    assertThat(dao.getCount(), is(0));

    dao.add(new User("kim", "kim", "1234"));
    assertThat(dao.getCount(), is(1));

    dao.add(new User("lee", "lee", "1234"));
    assertThat(dao.getCount(), is(2));

    dao.add(new User("park", "park", "1234"));
    assertThat(dao.getCount(), is(3));

}
```

다음으로는 get() 메소드에 대한 검증을 추가적으로 진행해보자. 
만약 get() 메소드를 통해서 가져온 결과가 없다면 어떻게 처리하는 것이 좋을까? 

1. null 을 반환한다.
2. 예외를 던진다. 

이렇게 두 가지 처리 방법이 있을 수 있는데 우리는 2번 케이스에 대해 다뤄보자.

```java
@Test(expected = EmptyResultDataAccessException)
public void getUserFailure() {
    ApplicationContext context = new GenericXmlApplicationContext("ApplicationContext.xml");

    UserDato dao = context.getBean("userDao", UserDao.class);
    dao.deleteAll();
    assertThat(dao.getCount(), is(0));
    
    dao.get("unknown");
    
}
```

테스트를 작성할 때 성공적인 케이스만 고려하여 작성하고 마는 경우가 많다. 이런 정상적인 부분만 고려한다면 사용자들은 정말 기상천외한 다양한 방법들로 서비스를 이용하기 때문에 기능이 정상적으로 동작한다고 보장하기 어렵다. 

때문에 부정적인 케이스에 대해서도 고려하여 테스트를 작성하는 것이 좋다. 조금만 더 생각하면 다양한 상황과 입력값에 대한 꼼꼼한 테스트를 작성할 수 있다. 

### 2.3.5 테스트 코드 개선

UserDaoTest 를 보면 반복되는 부분을 쉽게 찾을 수 있다.

```java
 ApplicationContext context = new GenericXmlApplicationContext("ApplicationContext.xml");

    UserDato dao = context.getBean("userDao", UserDao.class);
```

메소드 추출로 반복되는 코드를 하나로 만들 수도 있지만 JUnit 에서 제공하는 기능을 활용해보자.

```java
public class UserDaoTest {
    private UserDao dao;

    @Before
    public void setUp() {
         ApplicationContext context = new GenericXmlApplicationContext("ApplicationContext.xml");
         UserDato dao = context.getBean("userDao", UserDao.class);
    }
}

```

@Before 를 사용하면 각각의 테스트가 실행되기 전에 setUp() 메서드를 먼저 실행한 후에 각각의 테스트 메서드를 실행한다. @After 도 존재하는데 이는 각 테스트 메서드 실행 후에 실행된다.

JUnit 은 각각의 테스트를 진행할 때마다 새로운 오브젝트를 만들고 테스트 종료 후 오브젝트를 삭제한다. 이는 각 테스트가 서로 영향을 주지 않고 독립적이라는 것을 보장하기 위함이다.

> **픽스처**
> 
> 테스트를 수행하는데 필요한 정보나 오브젝트.
>
> 일반적으로 테스트에 반복적으로 사용되기 때문에 @Before 에 생성하면 좋다.

## 2.4 스프링 테스트 적용

JUnit 이 각 테스트마다 오브젝트를 새로 만들면서 문제점이 하나 발생한다. 바로 ApplicationContext 라는 오브젝트를 매번 새로 만든다는 점이다. 뭐가 문제냐고 할 수 있지만 ApplicationContext 를 새로 만들게되면 모든 싱글톤 빈들을 초기화한다. 그만큼 리소스를 많이 쓴다. 

이제 스프링에서 제공하는 방법을 이용해서 applicationContext 를 관리해보자.

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/applicationContext.xml")
public class UserDaoTest {
    @Autowired
    private ApplicationContext context;

    @Before
    public void setUp() {
         dao = context.getBean("userDao", UserDao.class);
    }
}
```

이렇게 코드를 작성하고 각각의 테스트 메소드에서 ApplicationContext 의 주소값을 출력해보면 동일한 것을 볼 수 있다. 

또한, 각각의 테스트 코드 실행 시간이 1.34, 0.15, 0.16초가 걸리는 것을 볼 수가 있는데 ApplicationContext 를 생성하는 첫번째 테스트가 가장 오래 걸린다는 점을 볼 수 있다. 

`@Autowired` 를 사용하면 ApplicationContext 에서 DL 방식으로 UserDao 빈을 찾지 않아도 손쉽게 주입받을 수 있다. 

지금까지 스프링을 통해서 테스트 코드를 사용해봤는데 스프링을 사용하면 기본적으로 많은 것들이 필요하기 때문에 스프링의 기능을 직접 사용하는 부분이 아니라면 제외하고 순수하게 자바만으로 테스트 코드를 구성하는 것이 좋다. 

## 2.5 학습 테스트로 배우는 스프링

### 학습 테스트의 장점
- 다양한 조건에 따른 기능을 손쉽게 확인해볼 수 있다. 
- 학습 테스트 코드를 개발 중에 참고할 수 있다. 
- 프레임워크나 제품을 업그레이드할 때 호환성 검증을 도와준다. 
- 테스트 작성에 대한 좋은 훈련이 된다.
- 새로운 기술을 공부하는 과정이 즐거워 진다.


>**버그 테스트**
>
>코드에 오류가 있을 때 그 오류를 잘 드러내줄 수 있는 테스트 


**장점** 
- 테스트의 완성도를 높여준다.
- 버그의 내용을 명확하게 분석하게 해준다.
- 기술적인 문제를 해결하는 데 도움이 된다.

> **동등 분할**
>
> 같은 결과를 내는 값의 범위를 구분해서 각 대표값으로 테스트하는 방법을 말한다. 
> 어떤 작업의 결과의 종류가 true, false, 예외 발생 세 가지라면 각 결과를 내는 입력값이나 상황의 조합을 만들어 모든 경우에 대한 테스트를 해보는 것이 좋다 .
> 
> **경계값 분석**
> 
> 에러는 동등분할 범위의 경계에서 주로 발생한다는 특징을 이용해서 경계의 근처에 있는 값을 이용해 테스트하는 방법이다.
> 보통 숫자의 입력인 경우 0이나 그 주변 값 또는 정수의 최대값, 최소값으로 테스트해보면 좋다.