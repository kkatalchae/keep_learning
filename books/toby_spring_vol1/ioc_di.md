### 제어의 역전 (Inversion of Control, IoC)

디자인 패턴 중 팩토리 패턴을 통해서 제어의 역전이라는 개념을 이해해보자.
```
public interface Logger {
    public void log(String message);
}

public class ConsoleLogger implements Logger {
    public void log(String message) {
        System.out.println(message);
    }
}

public class FileLogger implements Logger {
    public void log(String message) {
        // 파일에 로그 기록
    }
}

public class OrderService {
    private Logger logger;

    public OrderService() {
        this.logger = new ConsoleLogger();
    }
}
```

로깅이 필요한 서비스가 만 개가 있다고 했을 때 로깅의 구현 방법이 바뀌었다고 가정하면 모든 서비스에서 구현체를 변경하여 생성해야한다. 

이런 유지보수의 문제점을 해결하기 위해 로깅 구현체를 생성한다는 관심사를 별도의 클래스로 분리하자. 

```
public LoggerFactory {
    public static Logger getLogger(String className) {
        return new ConsoleLogger();
    }
}

public class OrderService {
    private Logger logger;

    public OrderService() {
        this.logger = LoggerFactory.getLogger();
    }
}
```

객체 생성의 책임을 별도의 클래스로 만들었다. 이런 패턴을 디자인 패턴 중 팩토리 패턴이라고 한다. 

기존에는 `Logger` 를 사용하는 측에서 어떤 구현체가 있는지, 어떤 구현체를 사용해야하는지 생각하고 직접 객체를 생성해줘야했다. 

하지만 객체 생성에 대한 책임을 별도의 클래스로 분리함으로서 어떤 구현체가 있는지 어떤 구현체를 사용해야하는지 사용하는 측에서는 몰라도 되도록 되었다. 이렇게 사용하는 측에서 제어하지 않고 제 3자에서 제어를 하는 것을 제어의 역전이라고 한다. 


### 스프링의 제어의 역전

스프링에서 구현하고 있는 제어의 역전 개념을 살펴보기 전에 스프링 빈이라는 개념에 대해서 알아보자.

코드를 개발하다가 보면 `@Bean`, `@Configuration` 어노테이션을 흔하게 찾아볼 수 있다. 

> 스프링 빈은 스프링 프레임워크에서 관리하는 오브젝트를 의미한다. 

스프링에는 빈 오브젝트의 생성과 관계 설정들을 관리하는 `BeanFactory` 라는 인터페이스가 존재한다. 빈을 사용하는 측에서 빈을 생성하고 관계를 설정하는 것이 아니라 `BeanFactory` 라는 오브젝트를 통해서 제어하기 때문에 `BeanFactory` 를 스프링의 IoC 컨테이너라고 한다.

제어의 역전이라는 개념을 명확하게 표현하고 있는 것은 `BeanFactory` 라는 인터페이스이지만 우리는 `ApplicationContext` 라는 인터페이스를 자주 사용한다. 

### 싱글톤 패턴

스프링은 기본적으로 싱글톤이라는 디자인 패턴으로 빈 오브젝트를 관리한다. 

싱글톤은 프로그램 내에서 하나의 오브젝트만 존재하도록 하는 디자인 패턴이다. 

만약에 전역적으로 쓰이는 오브젝트가 하나의 HTTP 요청마다 생성된다고 가정해보자.

하나의 요청에서 5개의 오브젝트가 만들어진다고 가정했을 때 초당 500개의 요청이 온다면 초당 2500개의 오브젝트가 만들어진다. 자바가 아무리 메모리 관리 기능이 뛰어나다고 하더라도 이렇게 많은 오브젝트를 관리하는 것은 부하가 많이 갈 것이다. 

때문에 많은 엔터프라이즈 분야에서는 싱글톤 패턴을 통해서 오브젝트를 관리한다. 
스프링에서 `Servlet` 이라는 오브젝트를 싱글톤으로 하나만 만들어서 멀티 스레드 환경에서 공유하여 사용하도록 한다. 

이렇게 권장되는 패턴이기도 하지만 사용하기가 까다롭고 문제점들도 존재하기 때문에 조심해서 사용해야 한다. 

**싱글톤 패턴의 한계**
- private 생성자를 가지고 있기 때문에 상속할 수 없다. 
- 싱글톤 오브젝트는 테스트하기 어렵다.
- 서버 환경에서는 싱글톤이 하나만 만들어진다는 것을 보장하지 못한다.
- 싱글톤의 사용은 전역 상태를 만들 수 있기 때문에 바람직하지 못하다.

### 싱글톤 레지스트리

싱글톤 레지스트리는 스프링 IoC 컨테이너가 빈 객체를 싱글톤으로 관리하는 저장소입니다. 일반적인 싱글톤 패턴처럼 클래스가 스스로 단일 인스턴스를 보장하는 것이 아니라, 컨테이너가 평범한 자바 객체(POJO)를 생성해서 저장하고 재사용하는 방식입니다.

**구체적인 구현 방식**
```
1. 핵심 자료구조
java// DefaultSingletonBeanRegistry 클래스 내부
public class DefaultSingletonBeanRegistry {
    // 싱글톤 객체를 저장하는 맵 (캐시)
    private final Map<String, Object> singletonObjects = 
        new ConcurrentHashMap<>(256);
    
    // 빈 이름으로 싱글톤 객체 조회
    public Object getSingleton(String beanName) {
        return this.singletonObjects.get(beanName);
    }
    
    // 싱글톤 객체 등록
    protected void addSingleton(String beanName, Object singletonObject) {
        this.singletonObjects.put(beanName, singletonObject);
    }
}
```

**동작 흐름**
```
[애플리케이션 시작]
1. 컴포넌트 스캔 (@Component, @Service, @Repository 등)
   └─ BeanDefinition 생성 (빈의 메타정보)

2. 빈 생성 및 등록
   ├─ 빈 이름으로 레지스트리 조회
   ├─ 없으면 → 리플렉션으로 객체 생성
   ├─ 의존성 주입
   └─ singletonObjects 맵에 저장
        예: ("userService", UserService 인스턴스)

[런타임 - 빈 요청시]
1. getBean("userService") 호출
2. singletonObjects.get("userService")로 조회
3. 이미 생성된 객체 반환 (재사용)
```

**핵심 포인트**

- `ConcurrentHashMap` 사용: Thread-safe한 싱글톤 관리,
빈 이름(key) - 객체(value) 구조로 관리

- `Lazy vs Eager`: 기본은 애플리케이션 시작시 미리 생성(Eager), `@Lazy` 사용시 최초 요청시 생성

- `리플렉션 사용`: public 생성자를 통해 객체 생성 가능

결국 싱글톤 레지스트리는 Map에 빈을 한 번만 저장하고 계속 재사용하는 단순한 캐시 메커니즘이지만, 이를 통해 전통 싱글톤 패턴의 모든 한계를 극복한 것이다.

### 의존성 주입

의존성 주입이라는 개념에 대해서 살펴보기 전에 의존성이라는 개념부터 알아보자.

```
public class OrderService {
    private UserService userService;
    
    public void createOrder() {
        // OrderService가 UserService를 "의존"한다
        userService.getUser();
    }
}
```

위와 같은 코드가 있을 때 OrderService 는 UserService 의 getUser 라는 기능을 사용하고 있다. 만약 UserService 의 getUser 라는 기능에 문제가 생겼다고 가정해보자. 그렇다면 OrderService 의 createOrder 기능도 문제가 생길 것이다. 

이렇게 자신이 아닌 다른 객체의 변경 혹은 문제에 영향을 받는 다면 의존 관계에 있다고 할 수 있다. 

실제 코드를 사용할 때 보이는 의존성 주입은 이렇다.

```
public class OrderService {
    private final UserService userService; 

    public OrderService(UserService userService) {
        this.userService = userService; <- 의존성 주입
    }
    
    public void createOrder() {
        userService.getUser();
    }
}
```

메모리의 관점에서 의존성 주입을 보면 다음과 같다.

```
[스프링 싱글톤 레지스트리 - Heap 메모리]
┌─────────────────────────────────────┐
│ singletonObjects (Map)              │
│                                     │
│ "userService" → [UserService@123]  │  ← 실제 객체 (단 하나)
│ "orderService" → [OrderService@456]│
└─────────────────────────────────────┘
         ↑
         │ 참조
         │
[OrderService@456 객체]
┌─────────────────────────────────┐
│ userService = UserService@123   │  ← 참조값만 저장
└─────────────────────────────────┘
```

의존관계에 있는 객체는 의존하고 있는 객체에 대한 참조값만 전달받아서(주입) 참조값만 저장하고 있고 이 참조값을 사용하여 의존하고 있는 객체의 기능을 사용한다. 이것이 의존성 주입이라는 개념이다. 

```
// UserRepository 인터페이스
public interface UserRepository {
    User findById(Long id);
}

@Service
public class UserService {
    private final UserRepository userRepository;
    
    // 컴파일 타임: UserRepository 인터페이스에만 의존
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User getUser(Long id) {
        return userRepository.findById(id);
    }
}
```
컴파일 타임에 UserService가 아는 것:

UserRepository라는 인터페이스만 알고 있음
어떤 메소드가 있는지만 알면 됨
구체적인 구현체는 전혀 모름 (JpaUserRepository인지, MemoryUserRepository인지 모름)
코드가 컴파일되고 .class 파일이 생성될 때까지 구현체 정보 불필요


```
실제 실행 시점에 주입되는 구체적인 구현체
java// 구현체 1
@Repository
public class JpaUserRepository implements UserRepository {
    @Override
    public User findById(Long id) {
        // JPA로 DB 조회
        return entityManager.find(User.class, id);
    }
}

// 구현체 2
@Repository
public class MemoryUserRepository implements UserRepository {
    @Override
    public User findById(Long id) {
        // 메모리에서 조회
        return memoryMap.get(id);
    }
}
```

**런타임에 스프링이 하는 일**
```
[애플리케이션 시작 - 런타임]
1. 스프링 컨테이너가 빈 스캔
   - UserService 발견
   - JpaUserRepository 발견 (@Repository)
   
2. 의존성 해결
   - UserService는 UserRepository 타입 필요
   - JpaUserRepository가 UserRepository 구현체
   
3. 실제 주입 (런타임에 결정)
   UserService userService = new UserService(jpaUserRepository);
   
→ 이 순간 UserService는 실제로 JpaUserRepository 인스턴스를 받음
```

### 의존성 주입의 방법

1. 필드 주입 (Field Injection)
```
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private EmailService emailService;
    
    public void createUser() {
        userRepository.save();
        emailService.send();
    }
}
```
**특징**

- 코드가 간결함
- 과거에 많이 사용됨
- 스프링에서만 동작 (순수 자바로는 테스트 불가)

2. 수정자(Setter) 주입
```
@Service
public class UserService {
    private UserRepository userRepository;
    private EmailService emailService;
    
    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```
**특징**

- 선택적 의존성에 사용
- 의존성을 나중에 변경 가능 (가변)
- 의존성 없이도 객체 생성 가능

3. 생성자 주입 (Constructor Injection) ⭐ 권장
```
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    // @Autowired 생략 가능 (생성자가 하나일 때)
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

| 측면 | 필드 주입 | Setter 주입 | 생성자 주입 |
| --- | --- | --- | --- |
| 불변성 | ❌ final 불가 | ❌ final 불가 | ✅ final 가능 |
| 필수 의존성 표현 | ❌ 불명확 | ❌ 선택적 | ✅ 명확 |
| 순환 참조 감지 | ❌ 런타임 | ❌ 런타임 | ✅ 시작 시점 |
| 테스트 용이성 | ❌ 스프링 필요 | △ 가능하나 복잡 | ✅ 순수 자바 |
| NPE 방지 | ❌ 위험 | ❌ 위험 | ✅ 안전 |
| SRP 위반 감지 | ❌ 어려움 | ❌ 어려움 | ✅ 쉬움 |
