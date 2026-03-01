# MyBatis 완전 정복

## 목차
1. [개요](#1-개요)
2. [등장 배경](#2-등장-배경)
3. [최근 사용 방식](#3-최근-사용-방식)
4. [기본 세팅 및 CRUD](#4-기본-세팅-및-crud)
5. [동작 흐름](#5-동작-흐름)
6. [파라미터 바인딩 #{} vs ${}](#6-파라미터-바인딩--vs-)
7. [유의사항](#7-유의사항)

---

## 1. 개요

MyBatis는 Java 진영의 **SQL Mapper 프레임워크**다.

> "SQL과 Java 코드를 분리하자"

개발자가 SQL을 직접 작성하되, 그 SQL을 Java 코드 안에 뒤섞지 않고 XML이나 어노테이션으로 따로 관리할 수 있게 해준다. SQL 실행 결과는 Java 객체로 자동 매핑해준다.

한 마디로 정리하면,

```
MyBatis = JDBC 래퍼 + SQL 관리(XML) + 자동 결과 매핑
```

### ORM(JPA)과의 차이

| | JPA | MyBatis |
|---|---|---|
| SQL | 자동 생성 | 개발자가 직접 작성 |
| 철학 | 객체 중심 | SQL 중심 |
| 제어권 | 프레임워크에 위임 | 개발자가 직접 제어 |
| 복잡한 쿼리 | 어려움 | 유리함 |

---

## 2. 등장 배경

MyBatis 이전에는 순수 **JDBC**로 DB를 다뤘는데, 코드가 매우 번거로웠다.

```java
Connection conn = null;
PreparedStatement pstmt = null;
ResultSet rs = null;

try {
    conn = DriverManager.getConnection(url, user, password);
    pstmt = conn.prepareStatement("SELECT id, name, email FROM users WHERE id = ?");
    pstmt.setLong(1, userId);
    rs = pstmt.executeQuery();

    if (rs.next()) {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setName(rs.getString("name"));
        user.setEmail(rs.getString("email"));
        return user;
    }
} catch (SQLException e) {
    e.printStackTrace();
} finally {
    if (rs != null) try { rs.close(); } catch (SQLException e) {}
    if (pstmt != null) try { pstmt.close(); } catch (SQLException e) {}
    if (conn != null) try { conn.close(); } catch (SQLException e) {}
}
```

JDBC 방식의 문제점은 다음과 같다.

- SQL 한 줄 실행에 커넥션 획득 → 쿼리 준비 → 파라미터 세팅 → 실행 → 결과 매핑 → 자원 반납이라는 **반복 코드(boilerplate)** 가 매번 따라붙는다.
- SQL이 Java 코드 안에 문자열로 박혀 있어 수정하기 불편하다.
- 자원 반납을 하나라도 빠뜨리면 커넥션 누수가 발생한다.

MyBatis는 이 고통을 해결하기 위해 2010년 iBatis에서 이름을 바꾸고 발전했다. 반복 코드는 프레임워크가 처리하고, 개발자는 SQL 작성과 결과 매핑 정의에만 집중할 수 있게 됐다.

---

## 3. 최근 사용 방식

현재는 거의 **Spring Boot + MyBatis** 조합으로 사용한다.

- **XML 방식이 주류** : SQL이 복잡해질수록 어노테이션 방식보다 XML이 관리하기 편하다. 국내 금융권, 공공기관 프로젝트에서 특히 많이 보인다.
- **JPA와 혼용** : 단순 CRUD는 JPA로, 복잡한 통계/조회 쿼리는 MyBatis로 처리하는 전략이 실무에서 자주 쓰인다.
- **MyBatis-Plus** : JPA처럼 기본 CRUD를 자동화해주면서 MyBatis의 SQL 제어권도 함께 갖는 확장 라이브러리. 아시아권에서 많이 쓴다.

---

## 4. 기본 세팅

### 주요 설정 옵션

| 옵션 | 설명 | 기본값 |
|---|---|---|
| `mapper-locations` | XML Mapper 파일 경로 | 없음 (필수) |
| `type-aliases-package` | XML에서 클래스명만으로 타입 참조 가능 | 없음 |
| `map-underscore-to-camel-case` | snake_case → camelCase 자동 변환 | `false` |
| `log-impl` | SQL 로그 출력 방식 | 없음 |
| `default-statement-timeout` | 쿼리 타임아웃 (초) | 없음 (무제한) |
| `cache-enabled` | 2차 캐시 사용 여부 | `true` |
| `lazy-loading-enabled` | 지연 로딩 사용 여부 | `false` |
| `call-setters-on-nulls` | NULL 값도 setter 호출 여부 | `false` |

실무에서 거의 항상 설정하는 것은 `mapper-locations`, `type-aliases-package`, `map-underscore-to-camel-case`, `log-impl` 이 네 가지다.

### 주요 XML 속성

| 속성 | 설명 |
|---|---|
| `namespace` | Mapper 인터페이스 전체 경로와 반드시 일치 |
| `id` | Mapper 인터페이스 메서드명과 반드시 일치 |
| `parameterType` | 입력 파라미터 타입 (생략 가능) |
| `resultType` | 반환 타입 |
| `useGeneratedKeys` | DB에서 생성된 PK를 받아올지 여부 |
| `keyProperty` | 생성된 PK를 세팅할 객체의 필드명 |

---

## 5. 동작 흐름

### 초기화 단계 (애플리케이션 시작 시 1회)

```
Spring Boot 시작
    │
    ├── DataSource 생성 (DB 커넥션 풀 초기화)
    │
    ├── SqlSessionFactory 생성
    │       │
    │       ├── application.yml 설정값 로드
    │       ├── mapper-locations의 XML 파일 전체 파싱
    │       │       └── namespace + id를 키로 SQL 구문을 메모리에 등록
    │       └── type-aliases 등록
    │
    └── Mapper 인터페이스 스캔 (@MapperScan or @Mapper)
            └── 각 인터페이스에 대한 프록시 객체 생성 → Spring Bean 등록
```

### 실행 단계 (요청마다)

`userMapper.findById(1L)` 을 호출하면 내부에서 아래 흐름으로 처리된다.

```
userMapper.findById(1L) 호출
    │
    ├── [1] 프록시 가로채기
    │       MapperProxy.invoke() 실행
    │       → 메서드명으로 실행할 MappedStatement 결정
    │
    ├── [2] SqlSession 획득
    │       @Transactional 있음 → 기존 SqlSession 재사용
    │       @Transactional 없음 → 새 SqlSession 생성
    │               └── 커넥션 풀에서 Connection 하나 가져옴
    │
    ├── [3] Executor 실행
    │       │
    │       ├── 로컬 캐시 확인
    │       │       캐시 히트 → 바로 반환
    │       │       캐시 미스 → DB로 진행
    │       │
    │       ├── StatementHandler : PreparedStatement 생성
    │       ├── ParameterHandler : #{} 자리에 파라미터 바인딩
    │       └── SQL 실행 (JDBC)
    │
    ├── [4] ResultSetHandler
    │       ResultSet → Java 객체 변환
    │       resultType → 컬럼명 기반 자동 매핑
    │       resultMap  → 정의된 매핑 규칙 적용
    │
    ├── [5] 로컬 캐시에 결과 저장
    │
    ├── [6] SqlSession 처리
    │       @Transactional 있음 → 트랜잭션 끝날 때까지 유지
    │       @Transactional 없음 → 즉시 반납
    │
    └── [7] 결과 반환
```

### 핵심 컴포넌트 역할

| 컴포넌트 | 역할 |
|---|---|
| `SqlSessionFactory` | SqlSession을 찍어내는 공장. 애플리케이션에 1개 존재 |
| `SqlSession` | 실제 DB 작업 창구. 트랜잭션 단위로 생성/반납 |
| `Executor` | SQL 실행과 캐시 관리 담당 |
| `StatementHandler` | JDBC PreparedStatement 생성 담당 |
| `ParameterHandler` | `#{}` 자리에 파라미터 바인딩 담당 |
| `ResultSetHandler` | ResultSet → Java 객체 변환 담당 |

한 줄 요약 :

> **프록시가 메서드 호출을 가로채서 → XML에서 SQL을 꺼내고 → JDBC로 실행한 뒤 → ResultSet을 Java 객체로 변환해서 반환**

---

## 6. 파라미터 바인딩 #{} vs ${}

### #{} — PreparedStatement 바인딩

```xml
<select id="findById" resultType="User">
    SELECT * FROM users WHERE id = #{id}
</select>
```

JDBC 레벨에서 이렇게 동작한다.

```java
PreparedStatement pstmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
pstmt.setLong(1, id);  // 값을 따로 세팅
```

SQL 구조와 값이 완전히 분리되기 때문에 **SQL Injection이 원천 차단**된다.

### ${} — 문자열 직접 치환

```xml
<select id="findByName" resultType="User">
    SELECT * FROM users WHERE name = '${name}'
</select>
```

사용자가 `name`에 `' OR '1'='1`을 입력하면,

```sql
-- 의도한 쿼리
SELECT * FROM users WHERE name = 'john'

-- 실제 실행되는 쿼리 (SQL Injection 발생)
SELECT * FROM users WHERE name = '' OR '1'='1'
```

### ${} 는 왜 존재하는가

`#{}`는 **값(value)** 자리에만 쓸 수 있다. 테이블명이나 컬럼명은 PreparedStatement로 바인딩이 안 되기 때문에 동적 정렬 같은 경우에 `${}`를 사용한다.

```xml
<select id="findAll" resultType="User">
    SELECT * FROM users
    ORDER BY ${sortColumn} ${sortDirection}
</select>
```

### ${} 를 안전하게 쓰는 법

`${}`를 써야 하는 상황이라면 **반드시 화이트리스트로 검증**해야 한다.

```java
public List<User> findAll(String sortColumn, String sortDirection) {
    List<String> allowedColumns = List.of("name", "email", "created_at");
    List<String> allowedDirections = List.of("ASC", "DESC");

    if (!allowedColumns.contains(sortColumn) || !allowedDirections.contains(sortDirection)) {
        throw new IllegalArgumentException("허용되지 않는 정렬 조건입니다.");
    }

    return userMapper.findAll(sortColumn, sortDirection);
}
```

| | `#{}` | `${}` |
|---|---|---|
| 동작 방식 | PreparedStatement 파라미터 바인딩 | 문자열 직접 치환 |
| SQL Injection | 안전 | 위험 |
| 사용 용도 | 값 (WHERE 조건, INSERT 값 등) | 컬럼명, 테이블명, 정렬 방향 |
| 원칙 | **기본적으로 이것을 사용** | 꼭 필요한 경우에만, 반드시 검증 후 사용 |

---

## 7. 유의사항

### resultType 자동 매핑 — 조용히 null이 들어온다

컬럼명과 필드명이 정확히 일치하지 않으면 에러 없이 null이 들어온다.

```java
user.getName();  // 에러 없이 조용히 null
```

`map-underscore-to-camel-case: true` 설정으로도 해결이 안 된다면 `<resultMap>`으로 명시적으로 정의한다.

### namespace와 id는 반드시 일치

```xml
<mapper namespace="com.example.mapper.UserMapper">  <!-- 인터페이스 전체 경로 -->
    <select id="findById" ...>                       <!-- 메서드명 -->
```

패키지 구조가 바뀌면 namespace도 같이 바꿔줘야 한다. 불일치 시 `Invalid bound statement` 에러가 발생한다.

### 다중 파라미터는 @Param 명시

```java
// @Param 없이 여러 파라미터 → 에러
List<User> findByNameAndEmail(String name, String email);

// @Param 명시 → 정상 동작
List<User> findByNameAndEmail(@Param("name") String name,
                              @Param("email") String email);
```

### 동적 SQL에서 null과 빈 문자열 함께 체크

```xml
<!-- null만 체크하면 빈 문자열이 들어왔을 때 조건이 추가됨 -->
<if test="name != null and name != ''">
    AND name = #{name}
</if>
```

### foreach의 collection 속성명

```java
List<User> findByIds(@Param("ids") List<Long> ids);
```

```xml
<!-- @Param으로 지정한 이름을 collection에 사용 -->
<foreach collection="ids" item="id" open="(" separator="," close=")">
    #{id}
</foreach>
```

`@Param` 없이 List를 넘기면 `collection`에 `list`, 배열이면 `array`라고 써야 하는 혼란이 생긴다. **`@Param`을 항상 명시**하는 것이 일관성 있다.

### XML 특수문자 이스케이프

```xml
<!-- 에러 발생 -->
<if test="age > 20">

<!-- 방법 1: 이스케이프 문자 -->
<if test="age &gt; 20">

<!-- 방법 2: CDATA 섹션 (SQL이 복잡할 때 편함) -->
<select id="findAdults" resultType="User">
    <![CDATA[
        SELECT * FROM users WHERE age > 20 AND age < 60
    ]]>
</select>
```

### 유의사항 요약

| 유의사항 | 핵심 |
|---|---|
| resultType 자동 매핑 | 컬럼명 불일치 시 에러 없이 null |
| namespace / id 불일치 | `Invalid bound statement` 에러 |
| 다중 파라미터 | `@Param` 항상 명시 |
| 동적 SQL 조건 | null과 빈 문자열 함께 체크 |
| foreach collection명 | `@Param` 명시로 혼란 방지 |
| XML 특수문자 | 이스케이프 또는 CDATA 사용 |