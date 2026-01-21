## 3.1 다시 보는 초난감 DAO

```java
public void deleteAll() throws SQLException {
    Connection c = dataSource.getConnection();

    PreparedStatement ps = c.prepareStatement("delete from users");
    ps.executeUpdate();

    ps.close();
    c.close();
}
```

이 메소드에서는 Connection, PreparedStatement 두 개의 공유 리소스를 사용한다. 

정상적인 흐름대로 메소드가 실행된다면 close() 메소드를 통해 공유 리소스를 반환한다. 

하지만 이전 로직에서 예외가 발생하면 공유 리소스 반환 전에 메소드를 빠져나가면서 공유 리소스가 정상적으로 반환되지 않을 수 있다.

일반적으로 서버는 일정 개수의 커넥션을 미리 만들어두고 커넥션 풀로 관리한다. 만약 공유된 리소스 자원이 반환되지 않는 오류가 지속적으로 발생한다면 서버에서 관리하는 리소스가 부족하게 되고 심각한 서버 오류가 발생하여 서버가 중단될 수도 있다. 

따라서 예외 상황에서도 공유된 리소스가 반드시 반환될 수 있도록 try/catch/finally 를 적용해보자.

```java
public void deleteAll() throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();
        ps = c.prepareStatement("delete from users");
        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
        if (ps != null) {
            ps.close();
        } catch (SQLException e) {
        }

        if (c != null) {
            try {
                c.close();
            } catch (SQLException e) {
            }
        }
    }

}
```

## 3.2 변하는 것과 변하지 않는 것

앞서 살펴본 deleteAll() 메소드를 변하는 것과 변하지 않는 것으로 구분해보자. 

여기서 변하는 것은 쿼리를 만드는 부분이다. DB 커넥션을 가져와서 공유 자원을 반납하는 부분은 변하지 않는다. 

메소드 추출과 템플릿 메소드 패턴을 적용해보았지만 해당 방향은 맞지 않는다는 것을 알게되었다. 

개방 폐쇄 원칙을 잘 지키면서도 템플릿 메소드 패턴보다 유연하고 확장성이 뛰어난 것이, 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스에만 의존하도록 만드는 **전략 패턴**이다.

deleteAll() 메소드의 컨텍스트는 다음과 같다.
- DB 커넥션 가져오기
- PreparedStatement 를 만들어줄 외부 기능 호출하기
- 전달받은 PreparedStatement 실행하기
- 예외가 발생하면 이를 다시 메소드 밖으로 던지기
- 모든 경우에 만들어진 PreparedStatement, Connection 을 닫아주기

여기서 외부 기능을 호출하는 것이 전략 패턴에서 말하는 전략이다. 

해당 기능을 인터페이스로 만들고 인터페이스의 메소드 호출을 통해 PreparedStatement 생성 전략을 호출해주면 된다.

```java
public interface StatementStrategy {
    PreparedStatement makeStatement(Connection connection) throws SQLException;
}

public class DeleteAllStatement implements StatementStrategy {
    public PreparedStatement makeStatement(Connection connection) throws SQLException {
        PreparedStatement ps = c.prepareStatement("delete from users");
        return ps;
    }
}

public void deleteAll() throws SQLException {
    ...

    try {
        c = dataSource.getConnection();

        StatementStrategy strategy = new DeleteAllStatement();
        ps = strategy.makeStatement(c);

        ps.executeUpdate();
    } catch (SQLExcetption) {
    ...
}
```

위 코드를 보면 이전에 발생했던 문제가 동일하게 발생한 것을 볼 수 있다. 

이전처럼 deleteAll() 에서 이미 구체적인 전략에 대해서 알고 있다는 것은 전략 패턴에도 개방 폐쇄 원칙에도 맞지 않다. 

전략 패턴에 따르면 앞단의 클라이언트에서 컨텍스트에 전략을 넘겨주도록 해야한다. 

이를 앞선 부분에서는 객체 생성에 대한 책임을 팩토리 패턴으로 분리하고 이를 일반화한 DI 라는 개념에 대해서 살펴봤다. 

이제 다시 DI 를 전략 패턴에 적용을 시켜보자. 중요한 점은 고정적인 부분을 가변적인 부분으로부터 독립시켜야 한다는 것이다. 

```java
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
    Connection c = null;
    PreparedStatement ps = null;

    try {
        c = dataSource.getConnection();

        ps = stmt.makeStatement(c);

        ps.executeUpdate();
    } catch (SQLException e) {
        throw e;
    } finally {
        if (ps != null) {
            ps.close();
        } catch (SQLException e) {
        }

        if (c != null) {
            try {
                c.close();
            } catch (SQLException e) {
            }
        }
    }
}

public void deleteAll() throws SQLException {

    StatementStrategy stmt = new DeleteAllStatement();
    jdbcContextWithStatementStrategy(stmt);
}
```

이제 의존관계와 책임의 관점에서 볼 때, 이상적인 클라이언트-컨텍스트 관계를 가지고 있다고 볼 수 있다. 

클라이언트가 컨텍스트가 사용할 전략을 정해서 전달한다는 면에서 DI 구조라고 이해할 수 있다.

> **마이크로 DI**
> 
> 의존 관계 주입은 다양한 형태로 적용할 수 있다. 
> 
> DI 의 가장 중요한 개념은 제3자의 도움을 통해 두 오브젝트 사이의 유연한 관계가 설정되도록 만드는 것이다. 
> 
> IoC 컨테이너의 도움없이 코드 내에서 적용한 경우를 마이크로 DI 라고 한다. 

## JDBC 전략 패턴의 최적화

지금까지 변하지 않는 것과 변하는 부분을 전략 패턴을 통해서 깔끔하게 분리해냈다. 

하지만 그럼에도 개선할 부분이 남아있다. 
- DAO 메소드마다 새로운 StatementStrategy 구현 클래스를 만들어야 한다는 점
- DAO 메소드에 User 와 같은 부가적인 정보가 필요한 경우 이를 담아둘 인스턴스 변수와 이를 전달받는 생성자를 만들어야 한다는 점

구현 클래스를 만들면서 클래스 개수가 많아지는 문제는 간단하게 해결할 수 있다. 메소드 내부에 클래스를 선언하는 것이다. 

> **중첩 클래스의 종류**
> 
> 다른 클래스 내부에 정의되는 클래스를 중첩 클래스라고 한다. 
>
> 중첩 클래스는 독립적으로 오브젝트로 만들어질 수 있는 스태틱 클래스와 자신이 정의된 클래스의 오브젝트 안에서만 만들어질 수 있는 내부 클래스로 구분된다. 
>
> 내부 클래스는 다시 범위에 따라 세 가지로 구분된다.
> 
> - 멤버 필드처럼 오브젝트 레벨에 정의되는 멤버 내부 클래스
>
> - 메소드 레벨에 정의되는 로컬 클래스
> 
> - 이름을 갖지 않는 익명 클래스