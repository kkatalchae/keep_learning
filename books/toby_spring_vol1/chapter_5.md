# 트랜잭션

## 트랜잭션이란?

트랜잭션은 데이터베이스에서 하나의 논리적인 작업의 단위를 말합니다.

예를 들면 계좌이체를 했을 때 보내는 쪽의 계좌에서는 돈의 액수가 줄어들고 받는 쪽의 계좌에서는 돈이 늘어나야합니다. 이 때 이 두 가지는 하나의 단위지 각각이 아닙니다. 

한 쪽에서는 돈이 줄어들었는데 에러가 발생해서 한 쪽의 돈이 늘어나지 않았다면 큰 문제가 될 것입니다. 이렇듯 분리된 각각의 작업이 아닌 하나로 묶여야 하는 데이터 작업을 트랜잭션이라고 합니다.

우리는 그동안 userDao 를 만들 때 DB pool 에서 커넥션 객체를 가져와 작업이 끝나면 사용했던 커넥션은 DB pool 로 반환하는 작업을 했습니다.

그렇다면 커넥션과 트랜잭션은 어떤 관계를 가지고 있을까요??

트랜잭션은 커넥션 내부에서 일어나는 일이라고 볼 수 있습니다. 

커넥션은 데이터베이스와 어플리케이션 사이에 통신을 하는 통로를 열어두는 느낌이라면 트랜잭션은 데이터베이스에게 어떤 요청을 하기 위해서 작업들을 어플리케이션에 보관하고 있다가 한번에 반영을 하는 느낌입니다. 

트랜잭션의 기본적인 흐름은 다음과 같습니다.
```
커넥션 시작 -> 트랜잭션 시작 -> 데이터베이스 작업
-> 성공 ->commit()
-> 실패 ->rollback ()
-> 트랜잭션 종료 -> 커넥션 종료 
```

### 트랜잭션의 핵심 특징

- 원자성 (Atomicity): 트랜잭션 내의 모든 작업은 모두 반영되거나, 모두 롤백(취소)되어야 합니다.
- 일관성 (Consistency): 성공적으로 완료된 트랜잭션은 언제나 일관성 있는 데이터베이스 상태를 유지합니다.
- 고립성 (Isolation): 동시에 실행되는 트랜잭션은 서로 간섭할 수 없으며, 독립적으로 실행되어야 합니다.
- 지속성 (Durability): 성공적으로 완료된 결과는 시스템이 고장 나더라도 영구적으로 반영되어야 합니다. 

```java
@Service
@AllArgsConstructor
public class UserService() {
    private DataSource datasource;

    public void upgradeLevels() throws SQLException{
        // 트랜잭션 동기화 관리자를 이용해 동기화 작업을 초기화
        TransactionSynchronizationManager.initSynchronization();
        // DB 커넥션을 생성하고 트랜잭션을 시작한다.
        // 이후의 DAO 작업은 모두 여기서 시작한 트랜잭션 안에서 진행된다.
        // 아래 두 줄이 DB 커넥션 생성과 동기화를 함께 해준다.
        Connection c = DataSourceUtils.getConnection(dataSource);
        c.setAutoCommit(false);

        try {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (canUpgradeLevel(user)) {
                    upgradeLevel(user);
                }
            }

            c.commit();
        }catch(Exception e) {
            c.rollback();
            throw e;
        } finally {
            // 스프링 DataSourceUtils 유틸리티 메소드를 통해 커넥션을 안전하게 닫는다.
            DataSourceUtils.releaseConnection(c, dataSource);
            // 동기화 작업 종료 및 정리
            TransactionSynchronizationManager.unbindResource(this.dataSource);
            TransactionSynchronizationManager.clearSynchronization();
        }
    }
}
```

위 코드는 책에서 설명하고 있는 트랜잭션의 흐름을 잘 보여준다. 커넥션을 가져오고 트랜잭션을 열고 예외가 발생하면 작업내용을 롤백하고 성공하면 커밋하고 트랜잭션을 닫고 커넥션을 닫는다.

이런 복잡한 것들이 최근에는 하나의 어노테이션으로 해결할 수 있다. 

```java
@Transactional
public void upgradeLevels() throw Excetpion {
    List<User> users = userDao.getAll();
    for (User user : users) {
        if (canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}
```

스프링은 간단한 어노테이션을 붙이는 것으로 위에서 봤던 복잡한 작업들이 이뤄지도록 지원하고 있습니다. 

`@Transactional` 어노테이션은 `Spring AOP(Aspect Oriented Programming)` 을 기반으로 프록시 패턴을 통해서 동작합니다.

### @Transactional 사용시 유의점

- 프록시는 상속이나 인터페이스 구현을 통해서 만들어지기 때문에 `prviate`, `protected` 메소드는 트랜잭션이 적용되지 않는다는 특징을 가지고 있습니다.
- `RuntimeException` 만을 롤백하기 때문에 `RuntimeException` 을 상속하지 않는 예외가 발생했을 때는 롤백되지 않는 점을 유의해야 합니다.
- 예외를 `try ~ catch` 로 잡는다면 롤백되지 않는다 반드시 잡은 에러를 다시 던져줘야 롤백이 동작한다.
- @Async 를 사용한 비동기는 또 다른 스레드를 사용하기 때문에 `ThreadLocal` 의 커넥션에 접근하지 못해 트랜잭션이 정상적으로 동작하지 않습니다.