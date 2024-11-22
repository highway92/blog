# DML 사용시 ID 혹은 다른 컬럼들을 Return 받기

### 들어가기 앞서
oracle 사용도 처음, mybatis사용도 처음이라 꽤나 시간을 허비했다.
oracle에는 auto increment기능이 없기 때문에(적어도 내가 사용중인 버전엔) Trigger와 Sequence로 auto Increment를 구성했다.

아래로는 
insert 후 id를 return 받는 과정에 대한 설명이며 <selectKey>와 sequence의 nextval나 currval를 사용하는 방법은 아니다.
개인적으로는 seq를 직접호출하는 것을 좋아하지 않기 때문에 아래와 같은 방법으로 진행했다.


### Mapper interface

결론부터 말하자면 아래와 같이 작성하면 id를 return 받을 수 있다.

```java
@Mapper
public interface AuthLoginDao {
    // 로그인 시도를 insert 합니다.
    void createLoginAttempt(AuthLoginEntity.CreateLoginAttempt entity);
}
```

```xml
<mapper namespace="AuthLoginDao">
    <insert id="createLoginAttempt"
            parameterType="AuthLoginEntity$CreateLoginAttempt"
            keyProperty="id"
            keyColumn="ID"
            useGeneratedKeys="true"
    >
        INSERT INTO TB_LOGIN_ATTEMPT ( USER_ID, ACCESS_IP )
        VALUES (#{userId}, #{accessIp})
    </insert>
```

```java
public class AuthLoginEntity {
    @Data
    public static class CreateLoginAttempt {
        private Long id;
        private final String userId;
        private final String accessIp;
    }
}
```

구조는 아주 간단하다 로그인 시도가 있을 경우 입력한 userId와 ip를 저장한다.
Entity를 살펴보면 userId와 accessIp를 final로 선언되어 있지만 id는 INSERT작업이후 return 받아야 하기 때문에 final이 아니다.
** 또한 유의해야 할것이 return 받고자 하는 property의 setter를 호출하는 형태기에 setter 구현이 되어있어야 한다. (annotation 혹은 수작업)

### 자세히 살펴보기

```xml
<mapper namespace="AuthLoginDao">
    <insert id="createLoginAttempt"
            parameterType="AuthLoginEntity$CreateLoginAttempt"
            keyProperty="id,newUserId,newAccessIp"
            keyColumn="ID,USER_ID,ACCESS_IP" 
            useGeneratedKeys="true"
    >
        INSERT INTO TB_LOGIN_ATTEMPT ( USER_ID, ACCESS_IP )
        VALUES (#{userId}, #{accessIp})
    </insert>
```
useGeneratedKeys : 이 옵션을 true로 설정하면 mybatis에서 "RETURNING INTO "를 사용한다. 공식문서에도 oracle과 관련된 사용법은 없었는데 시행착오 끝에 알게 되었다.
keyProperty : Entity의 주입받고자 하는 Property명을 적으면 된다. 여러개도 가능하다. ex) id,newUserId,newAccessIp;
keyColumn : Return 받을 컬럼명을 적는다. 이 역시 여러개도 가능하다. ex) ID,USER_ID,ACCESS_IP

```java
public class AuthLoginEntity {
    @Data
    public static class CreateLoginAttempt {
        private Long id;
        private final String userId;
        private final String accessIp;

        private String newUserId;
        private String newAccessIp;
    }
}
```

이렇게 작성될 경우의 Entity property 값을 살펴보자

1. AuthLoginDao.createLoginAttempt(AuthLoginEntity.CreateLoginAttempt entity) 가 실행되기 전
```java
entity {
        private Long id = null;
        private final String userId = "유우저아이디";
        private final String accessIp; = "127.0.0.1";

        private String newUserId = null;
        private String newAccessIp = null;
    }
```


2. AuthLoginDao.createLoginAttempt(AuthLoginEntity.CreateLoginAttempt entity) 가 실행된 후
```java
entity {
        private Long id = 1; // 새롭게 생성된 Record의 ID Column 값
        private final String userId = "유우저아이디";
        private final String accessIp; = "127.0.0.1";

        private String newUserId = "유우저아이디"; //keyColumn와 keyProperty를 통해 지정된 값
        private String newAccessIp = "127.0.0.1"; //keyColumn와 keyProperty를 통해 지정된 값
    }
```

