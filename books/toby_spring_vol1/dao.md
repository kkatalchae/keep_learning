# 토비의 스프링 1-1 ~ 1-3 정리


## 흐름 정리

```
public class UserDao {
    public void add(User user) {
        // DB 생성 로직 ...

        // SQL 문 생성 로직 ...

        // SQL 실행 로직 ...

        // 결과 반환 로직 ...

        // DB 연결 해제 로직 ...
    }

    public User get(String id) {

        // DB 생성 로직 ...

        // SQL 문 생성 로직 ...

        // SQL 실행 로직 ...

        // 결과 반환 로직 ...

        // DB 연결 해제 로직 ...
    }
}
```

가장 초기의 DAO 상태이다. 이를 보면 공통적인 코드가 반복적인 것을 볼 수 있다. 이를 개선하기 위해 하나의 공통적인 로직을 하나의 메서드로 분리한다. (리팩토링 - 메소드 추출)

> **DAO(Data Access Object)**
> 
> DAO(Data Access Object)는 DB를 사용해 데이터를 조회하거나 조작하는 기능을 전담하도록 만든 오브젝트를 말한다.

```
public class UserDao {

    public Connection getConnection() {}

    public void add(User user) {

        getConnection();

        ... 이후 로직
    }

    public User get(String id) {
        
        getConnection();

        ... 이후 로직
    }
}
```

이렇게 중복되는 로직의 코드를 한 번만 작성하여 코드의 양을 줄이고 커넥션을 가져오는 로직이 바뀌었을 때 하나의 메서드만 고쳐도 되어 유지보수가 용이하게 되었다. 

유저라는 정보를 제공해야하는데 A, B사에 다른 DB 를 통해서 제공해야하는 요구사항이 생겼다고 가정해보자. 
객체지향 프로그래밍 중의 특징 중인 다형성을 통해서 해당 요구사항을 해결해보자. 

```
public class UserDao {
    public abstract Connection getConnection() {

    }
}

public class AUserDao extends UserDao {
    public Connection getConnection() {
        // A 사의 커넥션 반환 로직
    }
}

public class BUserDao extends UserDao {
    public Connection getConnection() {
        // B 사의 커넥션 반환 로직
    }
}
```

UserDao 에서 getConnection() 메서드를 추상 메서드를 선언해서 상속받는 클래스에서 반드시 해당 메서드를 구현하도록 했다. 이렇게 하나의 클래스에서 파생되지만 다르게 동작하도록 기능을 확장했다. 

> **개방 폐쇄 원칙(Open/Closed Principle)**
> 
> 클래스나 모듈은 확장에 대해서는 열려있어야하고 변경에는 닫혀있어야 한다는 원칙.

하지만 여전히 UserDao 입장에서 DB 커넥션을 얻는 부분은 메인 관심사가 아니기 때문에 아예 클래스를 분리해보자. 

```
public class SimpleConnectionMaker {
    public Connection makeConnection() {
        // 커넥션 만드는 로직
    }
}

public class UserDao {

    private SimpleConnectionMaker simpleConnectionMaker;

    UserDao () {
        this.simpleConnectionMaker = new SimpleConnectionMaker();
    }

    public void add(User user) {
        Connection conn = simpleConnectionMaker.makeConnection();
        
        // 이후 로직
    }

    public User get(String id) {
        Connection conn = simpleConnectionMaker.makeConnection();

        // 이후 로직
    }
}
```

공통의 관심사는 별도의 클래스로 분리했지만 다시 여러 고객사마다 다른 커넥션은 제공할 수 없는 확장에 닫힌 구조로 회귀했다.

관심사를 분리하면서도 분리된 관심사에 수정이 있을 때 UserDao 는 수정에 영향을 적게 받게 만들 수 있을까?

> **높은 응집도와 낮은 결합도**
> 
> 높은 응집도와 낮은 결합도는 소프트웨어 엔지니어링의 핵심 원칙이다. 
> 응집도가 높다는 것은 하나의 클래스가 하나의 관심사에만 집중되어 있다는 것을 의미한다.
> 결합도가 낮다는 것은 다른 클래스나 모듈과의 관계를 맺을 때 최소한의 정보만 주고 받는다는 것을 의미한다.

이를 위해서 자바에서 제공하는 인터페이스를 사용할 것이다. 인터페이스는 객체지향 프로그래밍의 특징인 추상화를 구현하는 강력한 도구이다.

```
public interface ConnectionMaker {
    public Connection makeConnection();
}

public class AConnectioinMaker implement ConnectionMaker {
    public Connection makeConnection() {
        // A사 커넥션 만드는 로직
    }
}

public class BConnectioinMaker implement ConnectionMaker {
    public Connection makeConnection() {
        // B사 커넥션 만드는 로직
    }
}

public class UserDao {
    private ConnectionMaker connectionMaker;

    public void setConnectionMaker(ConnectionMaker connectionMaker) {
        this.connectionMaker = connectionMaker;
    }
}

public class UserDaoTest {
    public static void main(String[] args) {

        ConnectionMaker connectionMaker = new AConnectioinMaker();

        UserDao userDao = new UserDao(connectionMaker);
    }
}
```

이제 DB 커넥션을 구하는 관심사는 ConnectionMaker 인터페이스와 인터페이스를 구현하는 클래스들로 분리되었다. 뿐만 아니라 또 다른 C, D, E 고객사가 늘어나도 구현 클래스들만 추가하면 되고 UserDao 는 수정할 필요가 없어졌다. 




## 디자인 패턴

## 싱글톤 패턴 (Singleton)

### 개념
싱글톤은 클래스의 인스턴스가 프로그램 전체에서 단 하나만 존재하도록 보장하는 패턴입니다.

### 사용 이유
데이터베이스 연결, 설정 관리자, 로거 같이 애플리케이션 전체에서 공유해야 하는 자원을 관리할 때 사용합니다. 여러 객체를 만들면 메모리 낭비나 상태 불일치 문제가 생길 수 있는데, 싱글톤으로 이를 방지할 수 있습니다.

### 자바 예제

```java
public class DatabaseConnection {
    private static DatabaseConnection instance;
    
    // private 생성자로 외부에서 인스턴스 생성 방지
    private DatabaseConnection() {
        System.out.println("데이터베이스 연결 생성");
    }
    
    // 전역 접근점 제공
    public static synchronized DatabaseConnection getInstance() {
        if (instance == null) {
            instance = new DatabaseConnection();
        }
        return instance;
    }
    
    public void query(String sql) {
        System.out.println("쿼리 실행: " + sql);
    }
}

// 사용 예시
DatabaseConnection db1 = DatabaseConnection.getInstance();
DatabaseConnection db2 = DatabaseConnection.getInstance();
// db1과 db2는 같은 인스턴스
```

---

## 2. 팩토리 패턴 (Factory)

### 개념
팩토리 패턴은 객체 생성 로직을 별도의 클래스나 메서드로 분리하는 패턴입니다.

### 사용 이유
어떤 객체를 생성할지 결정하는 로직이 복잡하거나, 생성할 객체의 타입이 실행 시점에 결정될 때 유용합니다. 클라이언트 코드는 구체적인 클래스를 몰라도 되고, 새로운 타입을 추가할 때도 기존 코드를 수정하지 않아도 됩니다.

### 자바 예제

```java
// 공통 인터페이스
interface Animal {
    void makeSound();
}

class Dog implements Animal {
    @Override
    public void makeSound() {
        System.out.println("멍멍!");
    }
}

class Cat implements Animal {
    @Override
    public void makeSound() {
        System.out.println("야옹!");
    }
}

// 팩토리 클래스
class AnimalFactory {
    public static Animal createAnimal(String type) {
        switch (type.toLowerCase()) {
            case "dog":
                return new Dog();
            case "cat":
                return new Cat();
            default:
                throw new IllegalArgumentException("알 수 없는 동물 타입");
        }
    }
}

// 사용 예시
Animal animal = AnimalFactory.createAnimal("dog");
animal.makeSound();  // 멍멍!
```

---

## 3. 빌더 패턴 (Builder)

### 개념
빌더 패턴은 복잡한 객체를 단계적으로 생성하는 패턴입니다.

### 사용 이유
생성자에 매개변수가 많거나 선택적 매개변수가 많을 때, 가독성 있고 안전하게 객체를 만들 수 있습니다. 특히 불변 객체를 만들 때 유용하며, 객체 생성 과정이 명확해집니다.

### 자바 예제

```java
public class User {
    private final String name;        // 필수
    private final String email;       // 필수
    private final int age;            // 선택
    private final String phone;       // 선택
    private final String address;     // 선택
    
    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
        this.phone = builder.phone;
        this.address = builder.address;
    }
    
    public static class Builder {
        private final String name;
        private final String email;
        private int age = 0;
        private String phone = "";
        private String address = "";
        
        public Builder(String name, String email) {
            this.name = name;
            this.email = email;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public Builder address(String address) {
            this.address = address;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
    
    @Override
    public String toString() {
        return "User{name='" + name + "', email='" + email + 
               "', age=" + age + ", phone='" + phone + "'}";
    }
}

// 사용 예시
User user = new User.Builder("김철수", "kim@example.com")
    .age(30)
    .phone("010-1234-5678")
    .build();
```

---

## 4. 옵저버 패턴 (Observer)

### 개념
옵저버 패턴은 한 객체의 상태가 변경되면 그것을 구독하고 있는 다른 객체들에게 자동으로 알려주는 패턴입니다.

### 사용 이유
이벤트 기반 시스템, GUI 프로그래밍, 실시간 알림 시스템 등에서 사용됩니다. 객체 간의 결합도를 낮추면서도 상태 변화를 효과적으로 전파할 수 있습니다.

### 자바 예제

```java
import java.util.ArrayList;
import java.util.List;

// 관찰자 인터페이스
interface Observer {
    void update(String message);
}

// 주제 (Subject)
class NewsAgency {
    private List<Observer> observers = new ArrayList<>();
    private String news;
    
    public void addObserver(Observer observer) {
        observers.add(observer);
    }
    
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }
    
    public void setNews(String news) {
        this.news = news;
        notifyObservers();
    }
    
    private void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(news);
        }
    }
}

// 구체적인 관찰자
class NewsSubscriber implements Observer {
    private String name;
    
    public NewsSubscriber(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String message) {
        System.out.println(name + "이(가) 뉴스를 받았습니다: " + message);
    }
}

// 사용 예시
NewsAgency agency = new NewsAgency();
NewsSubscriber sub1 = new NewsSubscriber("구독자1");
NewsSubscriber sub2 = new NewsSubscriber("구독자2");

agency.addObserver(sub1);
agency.addObserver(sub2);

agency.setNews("속보: 새로운 디자인 패턴 발견!");
// 출력:
// 구독자1이(가) 뉴스를 받았습니다: 속보: 새로운 디자인 패턴 발견!
// 구독자2이(가) 뉴스를 받았습니다: 속보: 새로운 디자인 패턴 발견!
```

---

## 5. 전략 패턴 (Strategy)

### 개념
전략 패턴은 알고리즘을 캡슐화하고 실행 시점에 선택할 수 있게 하는 패턴입니다.

### 사용 이유
같은 목적을 달성하는 여러 방법이 있을 때, 조건문 없이 깔끔하게 처리할 수 있습니다. 결제 방식, 정렬 알고리즘, 압축 방식 등 다양한 전략을 유연하게 교체해야 할 때 사용합니다.

### 자바 예제

```java
// 전략 인터페이스
interface PaymentStrategy {
    void pay(int amount);
}

// 구체적인 전략들
class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    
    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println(cardNumber + " 카드로 " + amount + "원 결제");
    }
}

class KakaoPayPayment implements PaymentStrategy {
    private String email;
    
    public KakaoPayPayment(String email) {
        this.email = email;
    }
    
    @Override
    public void pay(int amount) {
        System.out.println(email + " 카카오페이로 " + amount + "원 결제");
    }
}

// 컨텍스트
class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }
    
    public void checkout(int amount) {
        paymentStrategy.pay(amount);
    }
}

// 사용 예시
ShoppingCart cart = new ShoppingCart();

cart.setPaymentStrategy(new CreditCardPayment("1234-5678-9012"));
cart.checkout(50000);  // 카드로 결제

cart.setPaymentStrategy(new KakaoPayPayment("user@example.com"));
cart.checkout(30000);  // 카카오페이로 결제
```

---

## 6. 데코레이터 패턴 (Decorator)

### 개념
데코레이터 패턴은 객체에 동적으로 새로운 기능을 추가하는 패턴입니다.

### 사용 이유
상속을 사용하지 않고도 객체의 기능을 확장할 수 있어서 유연합니다. 커피에 옵션을 추가하거나, 텍스트에 서식을 추가하거나, 입출력 스트림에 기능을 추가하는 등의 상황에서 사용됩니다.

### 자바 예제

```java
// 기본 인터페이스
interface Coffee {
    String getDescription();
    double getCost();
}

// 기본 커피
class SimpleCoffee implements Coffee {
    @Override
    public String getDescription() {
        return "기본 커피";
    }
    
    @Override
    public double getCost() {
        return 2000;
    }
}

// 데코레이터 추상 클래스
abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
}

// 구체적인 데코레이터들
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", 우유 추가";
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 500;
    }
}

class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", 설탕 추가";
    }
    
    @Override
    public double getCost() {
        return coffee.getCost() + 200;
    }
}

// 사용 예시
Coffee coffee = new SimpleCoffee();
System.out.println(coffee.getDescription() + " = " + coffee.getCost() + "원");
// 출력: 기본 커피 = 2000원

coffee = new MilkDecorator(coffee);
System.out.println(coffee.getDescription() + " = " + coffee.getCost() + "원");
// 출력: 기본 커피, 우유 추가 = 2500원

coffee = new SugarDecorator(coffee);
System.out.println(coffee.getDescription() + " = " + coffee.getCost() + "원");
// 출력: 기본 커피, 우유 추가, 설탕 추가 = 2700원
```

---

## 객체지향 설계 원칙 (SOLID)

### 단일 책임 원칙 (Single Responsibility Principle, SRP)

클래스는 단 하나의 책임만 가져야 하며, 변경할 이유도 하나만 있어야 한다는 원칙입니다. 

쉽게 말하면 한 클래스는 한 가지 일만 해야 한다는 것이죠.
예를 들어 사용자 정보를 관리하는 클래스가 데이터베이스 저장, 이메일 발송, 로깅까지 모두 처리한다면 SRP를 위반하는 것입니다. 각 기능을 별도 클래스로 분리해야 합니다. 이렇게 하면 한 부분을 수정할 때 다른 부분에 영향을 주지 않고, 코드를 이해하기도 쉬워집니다.

### 개방-폐쇄 원칙 (Open-Closed Principle, OCP)

소프트웨어 개체는 확장에는 열려 있어야 하지만 수정에는 닫혀 있어야 한다는 원칙입니다. 즉, 기존 코드를 변경하지 않으면서 새로운 기능을 추가할 수 있어야 합니다.

인터페이스나 추상 클래스를 사용해서 구현하면, 새로운 기능이 필요할 때 기존 코드를 수정하지 않고 새로운 클래스를 추가하는 방식으로 확장할 수 있습니다. 예를 들어 결제 시스템에 새로운 결제 수단을 추가할 때, 기존 결제 코드를 수정하지 않고 새로운 결제 클래스만 추가하면 되는 구조가 이상적입니다.

### 리스코프 치환 원칙 (Liskov Substitution Principle, LSP)

자식 클래스는 부모 클래스를 완전히 대체할 수 있어야 한다는 원칙입니다. 즉, 부모 클래스가 들어갈 자리에 자식 클래스를 넣어도 프로그램이 정상적으로 동작해야 합니다.

예를 들어 '새(Bird)' 클래스에 '날다(fly)' 메서드가 있을 때, '펭귄(Penguin)' 클래스가 새를 상속받으면 문제가 생깁니다. 펭귄은 날 수 없으니까요. 이런 경우 상속 구조를 재설계하거나 인터페이스를 분리해야 합니다.

### 인터페이스 분리 원칙 (Interface Segregation Principle, ISP)

클라이언트는 자신이 사용하지 않는 메서드에 의존하지 않아야 한다는 원칙입니다. 하나의 큰 인터페이스보다는 여러 개의 작은 인터페이스로 분리하는 것이 좋습니다.

예를 들어 '다기능 프린터' 인터페이스에 출력, 스캔, 팩스 기능이 모두 있다면, 출력만 하는 단순 프린터는 사용하지 않는 스캔과 팩스 메서드도 구현해야 합니다. 이럴 때는 각 기능을 별도 인터페이스로 분리해서 필요한 것만 구현하도록 하는 게 좋습니다.

### 의존성 역전 원칙 (Dependency Inversion Principle, DIP)

고수준 모듈은 저수준 모듈에 의존해서는 안 되며, 둘 다 추상화에 의존해야 한다는 원칙입니다. 또한 추상화는 세부 사항에 의존해서는 안 되고, 세부 사항이 추상화에 의존해야 합니다.

쉽게 말하면 구체적인 구현 클래스에 직접 의존하지 말고, 인터페이스나 추상 클래스에 의존하라는 것입니다. 예를 들어 '주문 처리' 클래스가 특정 '데이터베이스' 클래스에 직접 의존하면, 데이터베이스를 바꿀 때 주문 처리 코드도 수정해야 합니다. 대신 '저장소' 인터페이스에 의존하게 하면, 데이터베이스 구현체만 교체하면 됩니다.

> 💡 **결론**
>
> 디자인 패턴, 객체 지향 설계 원칙은 결국 코드의 결합도를 낮추고 응집도를 높이는 구체적인 방법론 중의 하나이다. 방법론들을 숙지하고 원하는 목표를 이루기 위해 이러한 수단을 적절하게 활용하는 것이 중요하다.
> 
> 결합도를 낮추고 응집도를 높이면 다음과 같은 효과들을 얻을 수 있다. 
> - 코드의 유지보수성이 향상된다.
> - 테스트가 쉬워진다.
> - 코드의 재사용성이 높아진다.
> - 코드를 파악하고 이해하기 쉬워진다.
> - 코드의 변경하기 쉽다. 
