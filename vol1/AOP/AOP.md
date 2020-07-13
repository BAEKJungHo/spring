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
