## 템플릿과 콜백

템플릿과 콜백 패턴은 스프링에서 자주 사용되는 개념이다. 책에서 다루고 있는 `JdbcTemplate` 뿐만 아니라 `RedisTemplate, RestTemplate, TransactionTemplate` 등 다양한 곳에서 해당 패턴을 사용해서 구현하고 있다. 

`TransactionTemplate` 이라는 예시를 통해서 템플릿, 콜백 패턴에 대해서 알아보자. 

데이터베이스에 하나의 작업을 할 때 트랜잭션을 열고 비즈니스 로직을 실행하고 작업이 완료되면 `commit` 명령어를 통해서 작업을 반영하고 작업 중 예외가 발생하면 작업을 롤백처리하고 자원을 반환하는 흐름이 있다. 

여기서 가변적인 부분은 비즈니스 로직을 실행하는 것이고 나머지는 고정적인 부분이다. 

고정적인 부분은 템플릿으로 가변적인 부분은 콜백으로 만들자. 

**TransactionTemplate 의 책임**
- 트랜잭션 시작
- 콜백 호출
- 예외 여부에 따라 커밋/롤백 결정
- 트랜잭션 종료 처리

**TransactionCallback 의 책임**


콜백은 비즈니스 로직의 실행이라는 단 하나의 책임을 가진다.

트랜잭션의 시작, 커밋/롤백 결정, 트랜잭션 종료 처리에 대해서는 관심사가 아니다.

위 관심사에 대해서 변경 사항이 있다고 해도 콜백에는 변화가 없다. 

---

코드 예시를 통해서 어떤 형태로 구현이 되어있는지 살펴보자.

우선 가변적인 로직을 실행하는 콜백을 인터페이스 형태로 만들고 리턴 타입을 제네릭으로 선언하여 가변적으로 리턴타입을 구성할 수 있도록 한다. 

```java
public interface TransactionCallback<T> {
    T doInTransaction();
}

```

고정적인 부분을 template 로 만든다. 콜백을 파라미터로 받아서 고정적인 로직 속에서 비즈니스 로직이 실행될 수 있도록 한다. 

```java
public class TransactionTemplate {

    private final TransactionManager transactionManager;

    public TransactionTemplate(TransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public <T> T execute(TransactionCallback<T> callback) {
        transactionManager.begin();
        try {
            T result = callback.doInTransaction();
            transactionManager.commit();
            return result;
        } catch (RuntimeException e) {
            transactionManager.rollback();
            throw e;
        }
    }
}

```

이제 transactionTemplate 를 사용해서 비즈니스 로직을 넘겨주면 개발자는 Transaction 에 대해서는 고민하지 않고 비즈니스 로직에 대해서만 집중해서 개발할 수 있다. 

책에서는 인터페이스를 구현한 내부 익명 클래스를 통해서 예시를 보여주고 있지만 조금 더 가독성 좋고 현대에서 많이 쓰이는 람다 패턴을 사용했다. 람다는 객체가 아닌 것처럼 보이지만 컴파일 타임에 인터페이스에 대한 구현 객체로 변환된다. 

```java
@Service
public class OrderService {

    private final TransactionTemplate transactionTemplate;

    public OrderService(TransactionTemplate transactionTemplate) {
        this.transactionTemplate = transactionTemplate;
    }

    public void placeOrder() {
        transactionTemplate.execute(() -> {
            System.out.println("주문 생성");
            System.out.println("결제 처리");
            return null;
        });
    }
}
```

좀 더 현대적으로 코드를 바꿔보면 단순히 메서드에 `@Transactional` 어노테이션을 붙이는 것 만으로 위의 코드들과 같은 효과를 낼 수 있다. 

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder() {
        System.out.println("주문 생성");
        System.out.println("결제 처리");
        return null;
    }
}
```

어노테이션을 붙이면 빈 생성시에 빈을 감싼 프록시 객체를 생성한다. 

어노테이션이 붙은 메서드 자체가 콜백이 된다.

이후 메소드 실행시의 흐름은 다음과 같다.

```
클라이언트
  ↓
프록시.placeOrder()
  ↓
TransactionInterceptor.invoke()
  ↓
실제 OrderService.placeOrder()
```


