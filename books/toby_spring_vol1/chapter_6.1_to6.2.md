# AOP

## 트랜잭션 코드의 분리

AOP(Aspect Oriented Programming) 은 스프링에서 가장 중요한 개념 중 하나이다. 우리는 앞선 내용에서 계속해서 로직을 분리해왔다. 하나의 기능을 구현할 때 DB 접근, 트랜잭션 처리, 비즈니스 로직, 메일 후 처리 많은 것들이 필요하다. AOP 는 기능에 가장 핵심적인 부분인 비즈니스 로직에 집중할 수 있도록 이외의 기능들을 분리하는 것이다.

이전 코드를 보면 레벨을 업그레이드하는 기능에 비즈니스 로직과 트랜잭션에 관련된 코드가 섞여있다. 

```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    
    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(users)) {
                upgradeLevel(user);
            }
        }

        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}
```

트랜잭션 코드를 분리하기 위해 이전에 해왔던 방법처럼 UserService 를 인터페이스로 선언하고 구현체인 UserServiceImpl 를 통해 구현해보자.

```java
public interface UserService {
    void add(User user);
    void upgradeLevels();
}

public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    List<User> users = userDao.getAll();
    for (User user : users) {
        if (canUpgradeLevel(users)) {
            upgradeLevel(user);
        }
    }
}
```

이제 트랜잭션 기능을 담은 구현체를 하나 더 만들 것이다.

```java 
public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager;

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void add(User user) {
        userService.add(user);
    }

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            userService.upgradeLevels();
            this.transactionManager.commit(status);
        } catch (Exception e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
}
```

코드를 살펴보면 `UserService` 라는 인터페이스를 구현한 구현체 클래스가 두 개다. 만약 `@Autowired` 를 통해 의존성을 주입한다면 어떤 구현체를 사용하게 될까?

인터페이스를 구현하고 있는 구현체가 하나라면 해당 클래스가 주입된다. 여러 개일 경우 `@Primary` 어노테이션을 붙이면 해당 빈이 기본값으로 세팅되어 기본적으로 해당 빈을 주입한다.

`@Qualifier` 어노테이션으로 어떤 빈을 주입할지 결정할 수도 있다. 

변수명과 빈의 이름을 일치시키면 그것만으로도 어떤 빈을 사용할지 결정하여 주입할 수도 있다. 

`@Profile` 을 통해서 환경에 따라서 어떤 빈을 주입할지 결정할 수도 있다. 

이런 여러 방법이 있지만 `@Qualifier > @Primary > 변수명 맵핑` 순으로 우선순위를 가진다.

아무리 변수명과 빈을 맵핑시켜놓았다고 하더라도 `@Primary` 어노테이션이 붙은 구현체가 있다면 해당 빈을 사용한다. `@Primary` 가 있더라도 `@Qualifier` 가 붙은 곳에서는 명시된 빈이 `@Primary` 빈보다 우선하여 주입된다. 

예시에서는 트랜잭션을 담당하는 구현체에서는 다른 구현체에 비즈니스로직을 위임하고 다른 구현체를 주입받아 사용한다. 


## 고립된 단위 테스트 

가장 편하고 좋은 테스트 방법은 가능한 한 작은 단위로 쪼개서 테스트하는 것이다. 

테스트 단위가 작으면 오류가 발생했을 때 원인을 찾기 쉽고 테스트의 의도나 내용이 분명해지고, 만들기도 쉽다. 

단위 테스트라는 단위라는 것이 굉장이 모호하다. 어떤 것을 단위로 잡아야 하는걸까? 기능 하나를 단위로 볼 수도 있고 하나의 클래스, 메소드를 단위로 볼 수도 있다. 어쩄든 본질은 하나의 단위에 초점을 맞춘 것이 단위 테스트라는 것이다.

책에서는 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것을 단위 테스트라고 부른다고 한다. 

반면, 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부의 DB 나 파일, 서비스 등의 리소스가 참여하는 테스트를 통합 테스트라고 부른다고 한다. 

**단위 테스트와 통합 테스트 중 어떤 것을 선택할 지에 대한 가이드 라인**
- 항상 단위 테스트를 먼저 고려한다.
- 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다. 
- 하나의 클래스나 성격과 목적이 같은 긴밀한 클래스 몇 개를 모아서 외부와의 의존관계를 모두 차단하고 필요에 따라 스텁이나 목 오브젝트 등의 테스트 대역을 이용하도록 테스트를 만든다.
- 스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트다.

단위 테스트를 할 때는 외부 의존성에 대한 부분을 배제하고 테스트 대상을 고립시켜 테스트해야한다.

이를 위해서 외부 의존성을 목 오브젝트로 대체하는데 활용하는 것이 Mockito 프레임워크이다.

사용법은 아래와 같다.

```java
@ExtendWith(MockitoExtension.class)
public class UserServiceTest {
    
    // 외부 의존성에 대한 목 오브젝트 생성
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private EmailService emailService;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void 회원가입_성공() {
        // Given
        User newUser = new User("test@test.com", "홍길동");

        // 외부 의존성의 동작 정의 
        when(userRepository.existsByEmail("test@test.com")).thenReturn(false);
        when(userRepository.save(any(User.class))).thenReturn(newUser);
        
        // When
        User result = userService.registerUser(newUser);
        
        // Then
        assertNotNull(result);
        assertEquals("홍길동", result.getName());

        // 외부 의존성에 대한 호출 검증
        verify(userRepository).save(any(User.class));
        verify(emailService).sendWelcomeEmail("test@test.com");
    }
    
    @Test
    void 중복_이메일_예외발생() {
        // Given
        User newUser = new User("test@test.com", "홍길동");
        when(userRepository.existsByEmail("test@test.com")).thenReturn(true);
        
        // When & Then
        // 외부 의존성에 대한 동작 정의(예외 처리)
        assertThrows(DuplicateEmailException.class, 
            () -> userService.registerUser(newUser));
        
        verify(userRepository, never()).save(any());
        verify(emailService, never()).sendWelcomeEmail(any());
    }
}
```