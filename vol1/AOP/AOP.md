# AOP

AOP 는 IoC/DI, 서비스 추상화와 더불어 스프링의 3대 기반기술의 하나다. AOP 를 바르게 이용하려면, 등장배경과 스프링이 그것을 도입한 이유, 그 적용을 통해 얻을 수 있는 장점이 무엇인지에 대한 충분한
이해가 필요하다.

스프링에 적용된 가장 인기 있는 AOP 의 적용 대상은 바로 선언적 트랜잭션 기능이다. 

## 메서드 분리

- 트랜잭션 경계 설정과 비지니스 로직이 공존하는 메서드

이 코드의 특징은, 트랜잭션 경계설정의 코드와 비지니스 로직 코드간에 서로 주고받는 정보가 없고 완벽하게 독립적인 코드이다. 단, 비지니스 로직 코드가 트랜잭션의 시작과 종료 작업 사이에 수행돼야 한다는
사항만 지키면 된다.

```java
public void upgradeLevels() throws Exception {
  // 트랜잭션 경계 설정 : s
  TransactionStatus stauts = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
  try {
  
  // 비지니스 로직 : s
  List<User> users = userDao.getAll();
  for(User user : users) {
    if(canUpgradeLevel(user)) {
      upgradeLevel(user);
    }
  }
  // 비지니스 로직 : e
  
  this.transactionManager.commit(status);
 } catch(Exception e) {
  this.transactionManager.rollback(status);
  throw e;
 }
 // 트랜잭션 경계 설정 : e
}
```

위 코드에서 비지니스 로직 코드를 upgradeLevelsInternal() 이라는 private 메서드를 만들어서 분리시키면 훨씬 깔끔해진 코드가 된다.

## DI 를 이용한 클래스 분리

어차피 서로 직접적으로 정보를 주고받는 것이 없다면, 따로 클래스로 빼서 관리하는 것이 좋다.

만약, UserService 는 현재 클래스로 되어있는데 다른 클래스에서 이를 직접 참조하는 경우가 생길 것이다. 그 경우에 UserService 에 있는 트랜잭션 코드를 빼버리면 참조하는 다른 클래스에서
트랜잭션 기능이 없는 UserService 를 쓰게 된다. 따라서 결합도를 낮추기 위해 UserService 를 인터페이스로 만들고 해당 구현체를 만들어 사용하는 것이 좋다.

`DI 의 기본 아이디어는 실제 사용할 오브젝트의 클래스 정체는 감춘 채 인터페이스를 통해 간접적으로 접근하는 것이다.` 그 덕분에 구현 클래스는 얼마든지 외부에서 변경할 수 있다.
보통 이렇게 인터페이스를 이용해 구현 클래스를 클라이언트에 노출하지 않고 런타임 시에 DI 를 통해 적용하는 방법을 사용하는 이유는, 일반적으로 구현 클래스를 바꿔가면서 사용하기 위해서이다.

보통 웹 개발할 때(정식 운영 중에는) 한 개의 Service 는 한 개의 구현체와 매핑이 된다.(정규 구현 클래스를 DI 해주는 것처럼 한 번에 한 가지 클래스를 선택해서 적용하도록 되어있다.)
하지만 꼭 그래야 하는 법은 없다.

UserService 를 구현한 또 다른 구현체를 만들 수 있다. 첫 번째는 비지니스 로직을 담고있는 UserService 구현체와 하나는 트랜잭션 코드를 담당하는 구현체를 만들어서, 트랜잭션 코드를 담당하는
구현체에서 비지니스 로직을 담고있는 구현체를 DI 받아서 실질적인 처리를 위임하는 것이다.

```java
public interface UserService {
  void add(User user);
  void upgradeLevels();
}
```

```java
public class UserServiceImpl implements UserService {
   UserRepository userRepository;
   MailSender mailSender;
   
   public void upgradeLevels() {
    List<User> users = userDao.getAll();
    for(User user : users) {
      if(canUpgradeLevel(user)) {
        upgradeLevel(user);
      }
    }
   }
}
```

```java
public class UserServiceTx implements UserService {
  UserService userService;
  PlatformTransactionManager transactionManager;
  
  public void setTransactionManager(PlatformTransactionManager transactionManager) {
    this.transactionManager = transactionManager;
  }
  
  public void setUserService(UserService userService) {
    this.userService = userService;
  }
  
  // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
  public void add(User user) {
    userService.add(user);
  }
  
  // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.
  public void upgradeLevels() {
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
      userService.upgradeLevels();
      this.transactionManager.commit(status);
    } catch(RuntimeException e) {
      this.transactionManager.rollback(status);
      throw e;
    }
  } 

}
```

UserServiceTx 는 비지니스 로직을 전혀 가지고 있지않고, 다른 구현 오브젝트에 기능을 위임한다.

따라서 의존관계는 아래와 같이 구성된다.

> Client -> UserServiceTx(Proxy) -> UserServiceImpl(Target)

이렇게 하더라도 문제가 하나 있는데, `클라이언트가 핵심기능을 가진 클래스를 직접 사용해버리면 부가기능이 적용될 기회가 없어진다는 것`이다. 그래서 부가기능은 마치 자신이 핵심 기능을 가진
클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심기능을 사용하도록 만들어야 한다.

> Client -> 핵심기능 인터페이스 / 부가기능 -> 핵심기능 인터페이스 / 핵심 인터페이스

이렇게 마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는것을 대리자, 대리인 같은 역할을 한다고 해서 `프록시(proxy)` 라고 부른다.
그리고 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 `타깃(target)`, 또는 `실체(real subject)` 라고 부른다.

> Client -> Proxy -> target(real subject)

`프록시의 특징은 타깃과 같은 인터페이스를 구현했다는 것과 프록시가 타깃을 제어할 수 있는 위치에 있다는 것이다.`

프록시는 사용 목적에 따라 두 가지로 구분할 수 있다. 

첫 번째는 클라이언트가 타깃에 __접근하는 방법을 제어__ 하기 위해서다. 

두 번째는 타깃에 __부가적인 기능을 부여__ 해주기 위해서다.

두 가지 모두 대리 오브젝트라는 개념의 프록시를 두고 사용한다는 점은 동일하지만, 목적에 따라서 디자인 패턴에서는 다른 패턴으로 구분한다.

## 데코레이터 패턴

자바 IO 패키지의 InputStream 과 OutputStream 구현 클래스는 데코레이터 패턴이 사용된 대표적인 예다. 

```java
// InputStream 이라는 인터페이스를 구현한 타킷인 FileInputStream 에 버퍼 읽기 기능을 제공해 주는 BufferedInputStream 이라는 데코레이터를 적용한 예다.
InputStream is = new BufferedInputStream(new FileInputStream("a.txt"));
```

데코레이터 패턴은 타킷의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경 하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

## 다이나믹 프록시

프록시는 기존 코드에 영향을 주지 않으면서 타깃의 기능을 확장하거나 접근 방법을 제어할 수 있는 유용한 방법이다. 그럼에도 불구하고 많은 개발자들은 타깃 코드를 직접 고치고 말지 
번거롭게 프록시를 만들지는 않겠다고 생각하는데, 그 이유가 프록시를 만드는 일이 상당히 번거롭기 때문이다.

java.lang.reflect 패키지 안에 프록시를 손쉽게 만들 수 있도록 지원해주는 클래스들이 있다.

- 프록시 역할
  - 위임
  - 부가작업
  
```java
public class UserServiceTx implements UserService {
  UserService userService; // target 
  
  public void add(User user) { // 메서드 구현과 위임
    this.userService.add(user);
  }
  
  public void upgradeLevels() { // 메서드 구현
    TransactionStatus = this.transactionManager.getTransaction(new DefaultTransactionDefinition()); // 부가기능 수행
    try {
      userServic.upgradeLevels(); // 위임
      this.transactionManager.commit(status); // 부가기능 수행
    } catch (RuntimeException e) { // 부가기능 수행
    this.transactionManager.rollback(status);// 부가기능 수행
    throw e; // 부가기능 수행
   }
  }
}
```

- 프록시를 만들기 번거로운 이유
  - 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거롭다.
  - 부가기능 코드가 중복될 가능성이 많다는 점이다.
  
첫 번째 문제를 해결하는 데 유용한 것이 바로 JDK 의 다이나믹 프록시이다.

## 리플렉션

다이나믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다. 리플렉션은 자바의 코드 자체를 추상화해서 접근하도록 만든 것이다.

자바의 모든 클래스는 그 클래스 자체의 구성정보를 담은 Class 타입의 오브젝트를 하나씩 가지고 있다. `클래스이름.class` or `인스턴스.getClass()` 로 Class 타입의 오브젝트를 가지고 올 수 있다.
클래스 오브젝트를 이용하면 클래스 코드에 대한 메타정보를 가져오거나 오브젝트를 조작할 수 있다.

> https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/index.html

## JoinPoint 와 Signature

- AOP 관련해서 JoinPoint를 파라미터로 전달받을 경우 반드시 첫번째 파라미터로 지정해야 함(그 외는 예외 발생)
- JoinPoint 인터페이스는 호출되는 대상 객체, 메서드 그리고 전달되는 파라미터 목록에 접근할 수 있는 메서드를 제공

  ■ Signature getSignature( ) - 호출되는 메서드에 대한 정보를 구함

  ■ Object getTarget( ) - 대상 객체를 구함

  ■ Object[ ] getArgs( ) - 파라미터 목록을 구함


- org.aspectj.lang.Signature 인터페이스는 호출되는 메서드와 관련된 정보를 제공하기 위해 다음과 같은 메서드를 정의

  ■ String getName( ) - 메서드의 이름을 구함

  ■ String toLongName( ) - 메서드를 완전하게 표현한 문장을 구함(메서드의 리턴 타입, 파라미터 타입 모두 표시)

  ■ String toShortName( ) - 메서드를 축약해서 표현한 문장을 구함(메서드의 이름만 구함)


- Around Advice의 경우 org.aspectj.lang.ProceedingJoinPoint를 첫 번째 파라미터로 전달받는데 해당 인터페이스는 프록시 대상 객체를 호출할 수있는 proceed() 메서드를 제공
- ProceedingJoinPoint는 JoinPoint 인터페이스를 상속받았기 때문에 Signature를 이용하여 대상 객체, 메서드 및 전달되는 파라미터에 대한 정보를 구할 수 있음



