## 7.4 인터페이스 상속을 통한 안전한 기능 확장

일반적으로 운영중인 서비스에서 SQL 을 변경하기 위해서는 새로 배포해야하지만 무중단 상황에서 SQL 을 수정하도록 만들어보자.

지금까지 공부한 DI 개념을 통해서 이를 구현해보자. 

DI 는 가능한 한 인터페이스를 통해서 의존성을 가진 오브젝트끼리 느슨한 결합이 되어야 한다. 또한, 인터페이스 분리 원칙을 준수하여 관심사에 따라 적절히 분리되어야 한다.

```         
class BaseSqlService : interface SqlService
-> field
interface SqlReader
interface SqlRegistry (sql 등록, 탐색) -> MySqlRegistry
```

기존에 만들어뒀던 SQL 등록과 탐색을 위한 부분과 업데이트를 위한 부분은 인터페이스 분리 원칙을 준수해서 분리해보자.

```java
public interface UpdatableSqlRegistry extends SqlRegistry {
    public void updateSql(String key, String sql) throws SqlUpdateFailureException;

    public void updateSql(Map<String, String> sqlMap) throws SqlUpdateFailureException;
}
```

기존 BaseSqlService 는 등록, 탐색용으로 두고 SQL 업데이트를 위한 별도의 클래스를 만들고 업데이트를 위한 인터페이스를 필드로 두고 DI 를 통해서 주입받아 사용하도록 설계했다.

```java
public class SqlAdminService implements AdminEventListener {
    private UpdatableSqlRegistry updatableSqlRegistry;

    public void setUpdatableSqlRegistry(UpdatableSqlRegistry updatableSqlRegistry) {
        this.updatableSqlRegistry = updatableSqlRegistry;
    }

    public void updateEventListener(UpdateEvent event) {
        this.updatableSqlRegistry.updateSql(event.get(KEY_ID), event.get(SQL_ID));
    }
}
```

## 7.5 DI 를 이용해 다양한 구현 방법 적용하기

운영중인 시스템에서 사용중인 정보를 실시간으로 변경할 때 우선적으로 고려해야하는 것은 동시성이다.

기존에 사용하던 HashMapRegistry 는 JDK 의 HashMap 을 사용하고 있는데 이는 멀티 스레드 환경에서 데이터를 조작하다보면 예기치 못한 오류가 발생할 가능성이 있다. 

때문에 ConcurrentHashMap 을 사용하여 멀티 스레드 환경에서 변경 시에 문제가 없도록 한다. 

### ConCurrentHashMap

**CAS란?**
```
CPU 명령어 수준의 원자적 연산
"현재 값이 내가 예상한 값과 같으면 새 값으로 교체해줘, 아니면 실패 반환"

기대값(null) == 실제값(null) → 교체 성공 ✅
기대값(null) != 실제값(Node) → 교체 실패, 재시도 🔄
```

**전체 동작 흐름 요약**
```
put(key, value) 호출
       │
       ▼
해당 버킷이 비어있나?
  YES → CAS로 원자적 삽입 (lock 없음) ✅
  NO  → synchronized(버킷 첫 노드) 로 삽입 🔒
                    │
                    ▼
         같은 버킷: 대기
         다른 버킷: 동시 실행 ✅

get(key) 호출
  → volatile 읽기만 → lock 없음 ✅
```

ConcurrentHashMap 을 사용하면 멀티 스레드 환경에서 동시성도 보장해주고 성능도 나쁘지 않지만 데이터가 많아지고 조회와 변경이 잦다면 문제가 발생할 수 있다.

또 다른 방법으로 내장 DB 를 사용해서 구현해보도록 하자.

내장형 DB 는 어플리케이션 시작 시에 세팅하고 종료 시에 사라지는 데이터베이스로 대표적으로 H2 가 많이 사용된다.

```java
public class EmbeddedDbSqlRegistry implements UpdatableSqlRegistry {
    SimpleJdbcTemplate jdbc;

    public void setDataSource(DataSource datasource) {
        this.jdbc = new SimpleJdbcTemplate(datasource);
    }

    public void registerSql(String key, String sql) {
        jdbc.update("insert into sqlmap(key_, sql_) values(?,?)", key, sql);
    }

    public String findSql(String key) throws SqlNotFoundException {
        try {
            return jdbc.queryForObject("select sql_ from sqlmap where key_ = ?", String.class, key);
        } catch (EmptyResultDataAccessException e) {
            throw new SqlNotFoundException(key + "에 해당하는 SQL 을 찾을 수 없습니다.");
        }
    }

    public void updateSql(String key, String sql) throws UpdateSqlFailureException {
        int affected = jdbc.update("update sqlmap set sql_ ? where key_ = ?", sql, key);

        if (affected == 0) {
            throw new UpdateSqlFailureException(key + "에 해당하는 SQL 을 찾을 수 없습니다.")
        }
    }

    public void updateSql(Map<String, String> sqlmap) throws UpdateSqlFailureException {
        for (Map.Entry<String, String> entry : sqlmap.entrySet()) {
            updateSql(entry.getKey(), entry.getValue());
        }
    }
}
```

위 구현 코드를 보면 하나의 sql 을 변경하는 코드와 여러 개의 sql 을 동시에 변경하는 코드가 있다. 

여러 개의 sql 을 변경할 때 각 sql 이 같이 변경되어야 하는 것들이 있을텐데 현재는 트랜잭션이 적용되지 않아 어떤 것은 변경에 성공하고 어떤 것은 실패할 수 있다.

이렇게 되면 운영환경에서 심각한 오류를 야기할 수 있기 때문에 반드시 트랜잭션이 적용되어야 한다.

HashMap 으로 관리하는 경우 이런 트랜잭션을 적용하기가 굉장히 까다롭기 때문에 내장형 DB 를 사용하면 트랜잭션을 적용하기가 굉장히 편하다.