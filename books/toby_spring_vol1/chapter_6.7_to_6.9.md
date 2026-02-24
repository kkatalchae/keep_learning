### 트랜잭션 전파 속성

| 전파 속성              | 기존 트랜잭션 존재 시               | 기존 트랜잭션 없을 시 | 특징 / 사용 상황                        |
| ------------------ | -------------------------- | ------------ | --------------------------------- |
| **REQUIRED** (기본값) | 기존 트랜잭션에 참여                | 새 트랜잭션 시작    | 가장 일반적. 상위 트랜잭션과 함께 커밋/롤백         |
| **REQUIRES_NEW**   | 기존 트랜잭션 일시 중단 → 새 트랜잭션 시작  | 새 트랜잭션 시작    | 완전히 독립. 로깅, 감사, 알림 처리에 적합         |
| **SUPPORTS**       | 기존 트랜잭션에 참여                | 트랜잭션 없이 실행   | 읽기 전용 로직에 자주 사용                   |
| **NOT_SUPPORTED**  | 기존 트랜잭션 일시 중단 → 트랜잭션 없이 실행 | 트랜잭션 없이 실행   | 트랜잭션 영향 피하고 싶을 때                  |
| **MANDATORY**      | 기존 트랜잭션에 참여                | 예외 발생        | 반드시 트랜잭션 내부에서만 실행                 |
| **NEVER**          | 예외 발생                      | 트랜잭션 없이 실행   | 트랜잭션 존재 자체를 금지                    |
| **NESTED**         | 기존 트랜잭션 내 중첩 트랜잭션 생성       | 새 트랜잭션 시작    | Savepoint 기반. 부분 롤백 가능 (DB 지원 필요) |

### 트랜잭션의 속성

**readOnly**
| 항목             | 영향               |
| -------------- | ---------------- |
| Flush 동작       | 자동 Flush 억제 가능   |
| Dirty Checking | 변경 감지 최소화        |
| 성능             | 조회 성능 개선 가능      |
| DB             | 일부 DB는 읽기 전용 최적화 |


**isolation**
| 수준                   | 설명             | 방지 문제                  |
| -------------------- | -------------- | ---------------------- |
| **DEFAULT**          | DB 기본값 사용      | 권장 기본값                 |
| **READ_UNCOMMITTED** | 커밋 전 데이터 읽기 허용 | Dirty Read ❌           |
| **READ_COMMITTED**   | 커밋된 데이터만 읽음    | Dirty Read 방지          |
| **REPEATABLE_READ**  | 같은 조회 결과 보장    | Non-repeatable Read 방지 |
| **SERIALIZABLE**     | 완전 직렬화         | Phantom Read 방지 (성능↓)  |

**timeout**
- 지정 시간 초과 시 롤백 처리
- 장시간 락, 대기 방지
- 외부 API 호출 시 사용

**rollbackFor**
- 체크 예외도 롤백하려고 할 시에 사용
- 비즈니스 예외 롤백 제어 가능

**noRollbackFor**
- 특정 예외 발생해도 커밋

**transactionManager**
- 다중 트랜잭션 매니저 환경에서 사용
- 여러 DB 사용 시 필수 옵션

### @Transactional

```java
package org.springframework.transcation.annotation;

@Traget({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    String value() default "";
    Propagation propagation() default Propagation.REQUIRED;
    Isolation isolation default Isolation.DEFAULT;
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
    boolean readOnly() default false;
    Class<? extends Throwable>[] rollbackFor() default {};
    String[] rollbackForClassName() default {};
    Class<? extends Throwable>[] noRollbackFor() default {};
    String[] noRollbackForClassName() default {};
}
```

@Transactional 어노테이션을 살펴보면 해당 어노테이션을 메소드와 타입(클래스, 인터페이스)에 사용할 수 있다. 

어노테이션은 런타임까지 유지될 수 있도록 해서 런타임에도 어노테이션 정보를 리플랙션을 통해서 얻을 수 있도록 되어있다. 또한, 상속을 통해서도 어노테이션 정보를 얻을 수 있도록 되어있다.