## 4.1 사라진 SQLException

예외가 발생한다면 어떻게 해야할까?

우선 절대 하지 말아햐 하는 행동들이 있다.

- 발생한 예외를 try - catch 로 잡고 catch 문에서 아무것도 하지 않는다.
- catch 문에서 단순히 로그를 출력한다.
- 발생한 예외를 적절히 처리하지 않고 그냥 `throws Exception` 처리한다.

그렇다면 예외가 발생했을 때 어떻게 처리하는게 좋을까?

예외를 처리할 때 반드시 지켜야 할 핵심 원칙은 한 가지다. 모든 예외는 적절하게 복구되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 한다.

### 예외의 종류와 특징

- Error : 시스템에 비정상적인 상황이 발생했을 경우 사용되며 대응할 수 없는 경우.
- Exception
  - checked Exception : `RuntimeException` 을 상속하지 않은 것, 해당 예외는 catch 문으로 잡든지 아니면 메소드 시그니처에 throws 처리를 해야한다. 처리하지 않을 경우 컴파일 에러 발생
  - unchecked Exception : `RuntimeException` 을 상속한 것

### 예외처리 방법

- 예외 복구  

예를 들면 DB 에 락이 걸려서 SQLException 이 발생했다면 잠시 뒤 다시 시도했을 때 정상적으로 기능이 동작할 가능성이 있다. 이럴 때 예외 상황에서 시간을 두고 다시 시도하는 코드를 짜서 예외에 대해서 처리할 수 있다.  

```java
int maxRetry = 5;
while (maxRetry -- > 0) {
    try {
        // 예외가 발생할 가능성이 있는 시도
        return;
    } catch(SQLException e) {
        // 재시도 로직
    } finally {
        // 리소스 반납 및 정리 작업
    }
    throw new RetryFailedException(); // 최대 재시도 횟수를 넘기면 직접 예외 발생
}
```

- 예외처리 회피

해당 방법은 예외 처리를 예외가 발생한 메소드에서 처리하는 것이 아니라 메소드를 호출한 클라이언트에게 던지는 것이다.

메소드 시그니처에 throws 처리하거나 catch 로 잡은 후 로그를 남기거나 처리르 한 후에 잡은 예외를 다시 던지는 것이다.

```java
public void add() throws SQLException {

}

public void add() throws SQLException {
    try {

    } catch (SQLException) {
        throw e;
    }
}
```

발생한 예외를 클라이언트에게 처리하도록 던지게 되면 클라이언트는 제대로 처리하기 어렵다. 아마 클라이언트도 해당 예외를 처리하지 못하고 던지고 결국 서버로 전달되는 발생해서는 안되는 상황이 발생할 가능성이 높다. 

따라서 예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분명해야 한다. 다른 오브젝트에서 예외처리 책임을 분명히 지게 하거나, 자신을 사용하는 쪽에서 예외를 다루는 게 최선의 방법이라는 분명한 확신이 있어야 한다.

- 예외 전환

예외 전환은 예외를 메소드 밖으로 던지는 점에서 예외 회피와 유사하지만 발생한 예외를 그대로 넘기는 게 아니라 적절한 예외로 전환해서 던진다는 점에서 차이가 있다. 

예를 들면 `SQLException` 이 발생하면 SQL 을 잘못 작성해서 예외가 발생한 것인지, 제약사항을 위반해서 발생한 것인지 DB 연결에 실패한 것인지 예외를 처리하는 쪽에서 분명하게 알기 어렵다. 

그래서 예외가 발생했을 때 catch 로 잡아서 좀 더 의미가 분명한 `DuplicateUserIdException` 으로 던지는 것이다. 

```java
public void add(User user) throws DuplicateUserIdException, SQLException {
    try {

    } catch (SQLException e) {
        if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY) {
            throw DuplicateUserIdException();
        } else {
            throw e;
        }
    }
}
```

### 예외처리 전략

앞선 add() 에서 체크 예외를 사용해서 예외를 메소드 바깥으로 던졌다. `DuplicateUserIdException` 은 충분히 복구가 가능할 것으로 보이니 클라이언트에서 잡아서 대응할 수 있을 것이다. 하지만 `SQLException` 은 대부분 대응이 불가능한 것이기 때문에 어플리케이션 밖으로 던져질 가능성이 높다. 그럴바에 `SQLException` 이 발생했을 때 `RuntimeException` 으로 던져서 클라이언트에서 `SQLException` 에 대한 처리를 하지 않도록 하는 편이 낫다.

```java
public class DuplicateUserIdException extends RuntimeException {
    public DuplicateUserIdException(Throwable cause) {
        super(cause);
    }
}

public void add() throws DuplicateUserIdException {
    try {

    } catch (SQLException e) {
        if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY) {
            throw new DuplicateUserIdException(e);
        } else {
            throw new RuntimeException(e);
        }
    } 
}
```

일반적으로 checked Exception 보다 unchecked Exception 을 사용하는 편이 더 장점이 많다. 하지만 컴파일러가 예외처리를 강제하지 않기 때문에 신경쓰지 않으면 예외 상황이 적절하게 처리되지 않을 가능성이 있다. 때문에 런타임 예외를 사용하는 경우, 문서화를 통해 메소드를 사용할 때 발생할 수 있는 예외의 종류와 원인, 활용 방법을 자세히 설명해두어야 한다.



