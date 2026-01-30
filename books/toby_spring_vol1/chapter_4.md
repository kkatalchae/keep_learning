# 4.2 예외 전환

JDBC 는 JDK 에서도 자주 사용되는 기술 중 하나이다.

JDBC 에서 DB 접근에 대한 표준 인터페이스를 제공하기 때문에 표준화된 JDBC 의 API 에만 익숙해진다면 DB 의 종류에 상관없이 일관된 방법으로 프로그램을 개발할 수 있다. 

하지만 DB 가 바뀌는 상황을 가정하면 두 가지 걸림돌이 존재한다. 

### 첫번째는 JDBC 에서 사용하는 SQL 이다. 

SQL 은 어느정도 표준화된 언어이고 몇 가지 표준규약이 있긴 하지만, 대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능도 제공한다. 

개발을 하다보면 DB 가 바뀌는 일이 많지 않기 때문에 성능 최적화 등의 이유로 DB 의 특정 기능을 이용하는 경우가 많다. 만약에 DB 가 바뀌어야 하는 상황이 찾아온다면 특정 DB 에 종속된 문법이나 기능들을 전부 교체해야하기 때문에 SQL 은 큰 걸림돌이 된다. 

### 두번째는 바로 SQLException 이다.

DB 에서 오류가 발생하는 케이스는 굉장히 다양하다. SQL 문법상의 오류가 있을 수도 있고, 제약조건을 위반했을 수도 있고, DB 커넥션의 문제일 수도 있고 데드락이 걸렸을 수도 있다. 이처럼 오류가 발생할 수 있는 케이스는 굉장히 다양한 반면에 JDBC는 SQLException 이라는 하나의 예외만을 반환한다. 

때문에 앞서 작성했던 코드처럼 SQLException.getErrorCode() 를 통해서 상세한 오류 원인을 파악해야한다. 여기서 가져오는 에러 코드는 DB 마다 다르기 때문에 DB 를 교체하는 상황에서 SQLException 은 큰 문제가 된다. 

### JDBCTempalte

스프링에서 제공하는 JDBCTemplate 에서는 SQLException 을 unchecked Exception 인 DataAceessException 으로 래핑해서 처리한다. 

또한, DataAccessException 의 서브클래스로 예외 상황에 대한 예외 클래스를 정의하고 있기 때문에 어떤 원인으로 에러가 발생했는지 파악하기 용이하다. 

### 현대의 DB 예외처리 

Mybatis, JPA 등 데이터 엑세스 기술은 진화했지만 데이터 엑세스 과정에서 예외를 던지는 부분은 JDBC 를 JDBCTemplate 에서 이용하는 것과 크게 변화하지 않았다. 

JDBCTempalte 는 SQLExceptionTranslator 라는 클래스로 에러 코드에 대한 해석을 위임한다. 

내부적으로 sql-error-codes.xml 라는 파일에 맵핑에 대한 정보들을 두고 DB 별로 핵심케이스에 대해서 정의하고 있다. 

```xml
<bean id="MySQL" class="org.springframework.jdbc.support.SQLErrorCodes">
    <property name="duplicateKeyCodes">
        <value>1062</value>
    </property>
</bean>
```

벤더별로 정의된 에러코드에 따라서 추상화된 예외를 리턴하기도 하지만 이렇게 되면 모든 DB 에 대한 정보를 다 알아야한다. 

따라서 맵핑이 되지 않을 경우 SQLState 라는 값을 통해서 어떤 예외를 리턴할지 결정한다. 

```
SQLState class code

- 23 → integrity violation

- 40 → transaction rollback
```

이러한 케이스를 벗어난 것들은 UncategorizedSQLException 로 처리한다. 

우리가 흔히 사용하는 Mybatis, JPA 의 경우는 어떤 흐름으로 이뤄지는지 살펴보자.

```
JDBC ->SQLException
 → PersistenceException (MyBatis)
 → SQLExceptionTranslator
 → DataAccessException

 JDBC -> SQLException
 → HibernateException
 → PersistenceException (JPA)
 → Spring 예외 변환
 → DataAccessException
```

데이터 엑세스 기술은 변화했지만 JDBC 에서는 SQLException 을 던지고 데이터 엑세스 기술은 반환된 SQLExcption 을 unchecked exceptioin 으로 전환하여 던지고 이를 스프링이 해석해서 표준적이고 구체적인 예외로 리턴한다는 흐름은 변화하지 않았다. 


