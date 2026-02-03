# 5. 서비스 추상화

## 5.1 사용자 레벨 관리 기능 추가 

**기능 요구사항**
- 사용자의 레벨은 BASIC, SILVER, GOLD 세 가지 중 하나다. 
- 사용자가 처음 가입하면 BASIC 레벨이 되며, 이후 활동에 따라서 한 단계씩 업그레이드될 수 있다. 
- 가입 후 50회 로그인을 하면 BASIC 에서 SILVER 레벨이 된다. 
- SLIVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨이 된다. 
- 사용자의 레벨의 변경 작업은 일정한 주기를 가지고 일괄적으로 진행된다. 변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않는다. 


우선 레벨이라는 정보를 저장하는 것에 대해서 생각을 해보자. 

DB 레벨에서 생각할 때 레벨이라는 정보는 숫자로 넣으면 적은 메모리를 차지하면서 정보를 저장할 수 있다.

프로그램 레벨에서 보면 단순히 숫자만 보면 이게 어떤 레벨이라는 것을 알기가 어렵기 때문에 상수로 선언할 수 있다. 

```java
class User {
    private static final int BASIC = 1;
    private static final int SILVER = 2;
    private static final int GOLD = 3;

    int level;
}
```

이렇게 상수로 선언하고 유저에 필드로 level 을 선언하고 사용하면 어떤 문제점이 발생할까?

레벨에는 1,2,3 만 값이 들어가야하는데 level 에는 0, 4, 100 ... 그냥 int 자료형에 맞는 값이면 모두 데이터베이스에 저장될 수 있다. 

자바 5 부터는 enum 이라는 타입을 제공하는데 이를 이용하면 같은 성격의 값들을 관리할 때 안정적이고 편리하게 관리할 수 있다. 

```java
public enum Level {
    BASIC(1),
    SILVER(2),
    GOLD(3);

    private final int value;

    Level(int value) {
        this.value = value;
    }

    public int intValue() {
        return value;
    }

    public static Level valueOf(int value) {
        switch(value) {
            case 1: return BASIC;
            case 2: return SILVER;
            case 3: return GOLD;
            default: throw new AssertionError("Unknown value: " + value);
        }
    }
}
```

이제 로직을 구현해보자. 

```java
public void upgradeLevels() {
    List<User> users = userDao.getAll();
    for(User user : users) {

        Boolean changed = null;

        if(user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
            user.setLevel(Level.SILVER);
            changed = true;
        } else if (user.getLevel() == Level.SILVER && user.getRecommend() >= 30) {
            user.setLevel(Level.GOLD);
            changed = true;
        } else if (user.getLevel() == Level.GOLD) {
            changed = false;
        }

        if (changed) {
            userDao.update(user);
        }
    }
}
```

우선 요구사항에 맞게 작성된 로직을 테스트해보자. 

요구사항에 따르면 로그인한 횟수(Login) 이 BASIC, SILVER 일 때 각각 50, 30일 때 레벨이 업그레이드된다. 

테스트할 때 경계값인 50, 30을 고려해서 49, 31 이러한 값을 사용해서 테스트를 하면 좋다. 

코드를 개선할 때 다음과 같은 질문을 해보면 좋다.
- 코드에 중복된 부분은 없는가?
- 코드가 무엇을 하는 것인지 이해하기 불편하지 않은가?
- 코드가 있어야 할 자리에 있는가?
- 앞으로 변경이 일어난다면 어떤 것이 있을 수 있고, 그 변화에 쉽게 대응할 수 있게 작성되어 있는가?

우선 if 블록이 가독성이 떨어진다는 생각이 우선 든다.

if 문을 구성하고 있는 로직을 뜯어보면 현재 레벨이 무엇인지 파악하는 로직, 업그레이드 조건을 담은 로직, 다음 단계의 레벨이 무엇이며, 업그레이드를 위한 세팅 로직, 업그레이드 여부를 판단하기 위한 플래그를 조작하는 로직, 변경 여부에 따른 업그레이드 실행 로직 으로 구성되어 있다. 

우선 조건문의 로직을 분리하면 아래처럼 바꿀 수 있을 것 같다. 

```java
public void upgradeLevels() {
    List<User> users = userDao.getAll();
    for(User user : users) {
        if(canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}

private boolean canUpgradeLevel(User user) {
    Level currentLevel = user.getLevel();
    switch(currentLevel) {
        case BASIC: return user.getLogin() >= 50;
        case SILVER: return user.getRecommend() >= 30;
        case GOLD: return false;
        default: throw new IllegalArgumentException("Unknown Level: " + currentLevel);
    }
}

private void upgradeLevel(User user) {
    if (user.getLevel == Level.BASIC) {
        user.setLevel(Level.SILVER);
    } else if (user.getLevel() == Level.SILVER) {
        user.setLevel(Level.GOLD);
    }

    userDao.update(user);
}
```

업그레이드 조건에 대한 판단 로직을 분리하니 if 문의 분기가 훨씬 가독성 좋게 바뀌었다. 

upgradeLevel() 메소드에서 아쉬운 부분을 찾아보자. 

해당 코드에는 다음 단계가 무엇인지 하는 로직과 그때 사용자 오브젝트의 level 필드를 변경해준다는 로직이 함께 있고, 너무 드러나있다. 또한, 예외상황에 대한 처리가 없다. 

GOLD 레벨은 최상위 레벨이기 때문에 업그레이드하려고 하면 update 할 필요가 없는데도 DB 에 업데이트 로직이 동작할 것이다. 

User 에서 Level 이라는 개념을 조금 분리시켜보자. 

```java
public enum Level {
    GOLD(3, null),
    SILVER(2, GOLD),
    BASIC(1, SILVER);

    private final int value;
    private final Level next;

    Level(int value, Level next) {
        this.value = value;
        this.next = next;
    }

    public int intValue() {
        return value;
    }

    public Level nextLevel() {
        return this.next;
    }

    public static Level valueOf(int value) {
        switch(value) {
            case 1: return BASIC;
            case 2: return SILVER;
            case 3: return GOLD;
            default: throw new AssertionError("Unknown value: " + value);
        }
    }
}

public void upgradeLevel() {
    Level nextLevel = this.level.nextLevel();

    if (nextLevel == null) {
        throw new IllegalStateException(this.level + "은 업그레이드가 불가능합니다");
    } else {
        this.level = nextLevel;
    }
}
```

위처럼 바꾸면 userService 에서는 업그레이드가 불가능한 예외상황에 대한 처리도 할 수 있고 다음 레벨이 무엇인지 판단하지 않아도 된다. 그저 Level 에 다음 레벨이 뭔지만 알려달라고 요청하면 된다. 

앞서 살펴본 canUpgradeLevel() 메소드에서 다음 레벨로 넘어가기 위한 조건 50회, 30회를 정수로 두고 있는데 만약 이 조건이 여러 군데에서 활용이 된다면 정수를 반복해서 사용해야하는데 이런 부분은 상수로 선언해두고 사용하면 조건이 변경되었을 때 상수에 정의된 정수값만 수정해주면 되기 때문에 상수로 정의해두고 사용하는 것이 좋다.

```java
public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
public static final int MIN_RECOMMEND_FOR_GOLD = 30;
```




