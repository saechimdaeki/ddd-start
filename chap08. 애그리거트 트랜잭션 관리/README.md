# Chap08. 애그리거트 트랜잭션 관리 

## 애그리거트와 트랜잭션
한 주문 애그리거트에 대해 운영자는 배송 상태로 변경할 때 사용자는 배송지 주소를 변경하면 어떻게 될까?

<img width="475" alt="image" src="https://user-images.githubusercontent.com/40031858/169721321-058b0ed6-316f-479f-96e7-f900e6b64d88.png">


위의 그림은 웅영자와 고객이 동시에 한 주문 애그리거트를 수정하는 과정을 보여준다. 트랜잭션마다 리포지토리는 새로운 애그리거트 객체를 생성하므로

운영자 스레드와 고객 스레드는 같은 주문 애그리거트를 나타내는 다른 객체를 구하게 된다. 운영자 스레드와 고객 스레드는 개념적으로 동일한

애그리거트지만 물리적으로 서로 다른 애그리거트 객체를 사용한다. 때문에 운영자 스레드가 주문 애그리거트 객체를 배송 상태로 변경하더라도 고객

스레드가 사용하는 주문 애그리거트 객체에는 영향을 주지 않는다. 고객 스레드 입장에서 주문 애그리거트 객체는 아직 배송 상태 전이므로 배송지 정보를 변경할 수 있다.

이 상황에서 두 스레드는 각각 트랜잭션을 커밋할 때 수정한 내용을 DB에 반영한다. 이 시점에 배송상태로 바뀌고 배송지 정보도 바뀌게 된다.

이 순서의 문제점은 운영자는 기존 배송지 정보를 이용해서 배송 상태로 변경했는데 그 사이 고객은 배송지 정보를 변경했다는 점이다.

즉 애그리거트의 일관성이 깨지는 것이다. 일관성이 깨지는 문제가 발생하지 않도록 하려면 다음 두 가지 중 하나를 해야 한다
- 운영자가 배송지 정보를 조회하고 상태를 변경하는 동안, 고객이 애그리거트를 수정하지 못하게 막는다
- 운영자가 배송지 정보를 조회한 이후에 고객이 정보를 변경하면, 운영자가 애그리거트를 다시 조회한 뒤 수정하도록 한다.

이 두가지는 애그리거트 자체의 트랜잭션과 관련이 있다. DBMS가 지원하는 트랜잭션과 함께 애그리거트를 위한 추가적인 트랜잭션 처리 기법이 필요하다.

애그리거트에 대해 사용할 수 있는 대표적인 트랜잭션 처리방식에는 `선점 잠금`과 `비선점 잠금`의 두가지 방식이있다.

## 선점잠금

`선점 잠금`은 먼저 애그리거트를 구한 스레드가 애그리거트 사용이 끝날 때까지 다른 스레드가 해당 애그리거트를 수정하지 못하게 막는 방식이다.

다음은 선점잠금의 동작 방식을 보여준다

<img width="458" alt="image" src="https://user-images.githubusercontent.com/40031858/169721561-a93f28ac-31a5-4ed2-961b-f0657e3e87bb.png">

위 그림에서 스레드1이 선점 잠금 방식으로 애그리거트를 구한 뒤 이어서 스레드2가 같은 애그리거트를 구하고 있다. 이때 스레드 2는 스레드 1이 애그리거트에

대한 잠금을 해제할 때까지 `블로킹`된다. 스레드1이 애그리거트를 수정하고 트랜잭션을 커밋하면 잠금을 해제한다. 이 순간 대기하고 있던

스레드2가 애그리거트에 접근하게 된다. 스레드1이 트랜잭션을 커밋한 뒤에 스레드2가 애그리거트를 구하게 되므로 스레드2는 스레드1이 수정한

애그리거트의 내용을 보게 된다. 한 스레드가 애그리거트를 구하고 수정하는 동안 다른 스레드가 수정할 수 없으므로 동시에 애그리거트를 수정할 때 

발생하는 데이터 충돌 문제를 해소할 수 있다. 앞서 배송지 정보수정과 배송 상태 변경을 동시에 하는 문제에 선점 잠금을 적용하면 다음과 같이 동작한다.

<img width="462" alt="image" src="https://user-images.githubusercontent.com/40031858/169721649-114f20d0-e4cf-47ba-ae9d-0befd60548df.png">

운영자 스레드가 먼저 선점 잠금 방식으로 주문 애그리거를 구하면 운영자 스레드가 잠금을 해제할 때까지 곡개 스레드는 대기 상태가 된다.

운영자 스레드가 배송 상태로 변경한 뒤 트랜잭션을 커밋하면 잠금을 해제한다. 잠금이 해제된 시점에 고객 스레드가 구하는 주문 애그리거트는

운영자 스레드가 수정한 배송 상태의 주문 애그리거트다. 배송 상태이므로 주문 애그리거트는 배송지 변경 시 에러를 발생하고 트랜잭션은 실패하게 된다.

선점 잠금은 보통 DBMS가 제공하는 행단위 잠금을 사용해서 구현한다. `for update`와 같은 쿼리를 사용해서 특정 레코드에 한 커넥션만

접근할 수 있는 잠금장치를 제공한다. JPA EntityManager는 LockModeType을 인자로 받는 find() 메서드를 제공한다.

`LockModeType.PESSIMISTIC_WRITE`를 값으로 전달하면 해당 엔티티와 매핑된 테이블을 이용해서 선점 잠금 방식을 적용할 수 있다.

```java
Order order = entityManger.find(
    Order.class , orderNo, LockModeType.PESSIMISTIC_WRITE);
```

JPA 프로바이더와 DBMS에 따라 잠금 모드 구현이 다른다. 하이버네이트의 경우 `PESSIMISTIC_WRITE`를 잠금 모드로 사용하면 `for update` 쿼리를 이용해 선점 잠금을 구현한다.

스프링 데이터 JPA는 `@Lock` 애노테이션을 사용해 잠금 모드를 지정한다.

```java
public interface MemberRepository extends JpaRepository<Member,MemberId>{
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdForUpdate(@Param("id") MemberId memberId);
}
```

### 선점 잠금과 교착 상태

선점 잠금 기능을 사용할 때는 잠금 순서에 따른 교착 상태가 발생하지 않도록 주의해야한다. 예를 들어, 다음 순서로 두 스레드가 잠금 시도를 한다고 해보자.
1. 스레드1: A 애그리거트에 대한 선점 잠금 구함
2. 스레드2: B 애그리거트에 대한 선점 잠금 구함
3. 스레드1: B 애그리거트에 대한 선점 잠금 시도
4. 스레드2: A 애그리거트에 대한 선점 잠금 시도

이 순서에 따르면 스레드1은 영원히 B애그리거트에 대한 선점 잠금을 구할 수 없다. 스레드2가 B애그리거트에 대한 잠금을 이미 선점하고 있기 때문이다. 동일한 이유로

스레드2는 A애그리거트에 대한 잠금을 구할 수 없다. 두 스레드는 상대방 스레드가 먼저 선점한 잠금을 구할 수 없어 더이상 다음 단계를

진행하지 못하게 된다. 즉, 스레드1과 스레드2는 교착 상태에 빠진다.

선점 잠금에 따른 교착 상태는 상대적으로 사용자 수가 많을 때 발생할 가능성이 높고, 사용자 수가 많아지면 교착 상태에 빠지는 스레드는 더 빠르게 증가한다.

이런 문제가 발생하지 않도록 하려면 잠금을 구할 때 최대 대기시간을 지정해야한다. JPA서 선점 잠금을 시도할 때 최대 대기 시간을 지정하려면 다음과 같이 힌트를 사용한다
```java
Map<String,Object> hints = new HashMap<>();
hints.put("javax.persistence.lock.timeout",2000);
Order order = entityManager.find(
    Order.calss , orderNo, LockModeType.PESSIMISTIC_WRITE, hints);
```

스프링 데이터 JPA는 `@QueryHints` 애노테이션을 사용해 쿼리 힌트를 지정할 수 있다.

```java
public interface MemberRepository extends JpaRepository<Member,MemberId>{
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({
        @QueryHint(name="javax.persistence.lock.timeout",value = "2000")
    })
    @Query("select m from Member m where m.id =:id")
    Optional<Member> findByIdForUpdate(@Param("id") MemberId memberId);
}
```
## 비선점 잠금
선점 잠금이 강력해 보이긴 하지만 선점 잠금으로 모든 트랜잭션 충돌 문제가 해결되는 것은 아니다

<img width="523" alt="image" src="https://user-images.githubusercontent.com/40031858/169722100-4aab0279-a45e-4067-8c0a-e5268f141156.png">

여기에서 문제는 운영자가 배송지 정보를 조회하고 배송 상태로 변경하는 사이에 고객이 배송지를 변경한다는 것이다. 운영자는 고객이 변경하기 전 배송지 정보를

이용하여 배송 준비를 한 뒤에 배송 상태로 변경하게 된다. 즉, 배송 상태 변경 전에 배송지를 한 번 더 확인하지 않으면 운영자는 다른 배송지로

물건을 발송하게 되고, 고객은 배송지를 변경했음에도 불구하고 엉뚱한 곳으로 주문한 물건을 받는 상황이 발생한다.

이 문제는 선점 잠금 방식으로는 해결할 수 없다. 이때 필요한 것이 `비선점 잠금`이다. 비선점 잠금은 동시에 접근하는 것을 막는 대신 변경한 데이터를

실제 DBMS에 반영하는 시점에 변경 가능 여부를 확인하는 방식이다. 비선점 잠금을 구현하려면 애그리거트에 버전으로 사용할 숫자 타입

프로퍼티를 추가해야 한다. 애그리거트를 수정할 때마다 버전으로 사용할 프로퍼티 값이 1씩 증가하는데 이때 다음과 같은 쿼리를 사용한다.

```sql
update aggtable set version = version+1, colx =?, coly = ?
where aggid = ? and version = 현재버전
```

이 쿼리는 수정할 애그리거트와 매핑되는 테이블의 버전 값이 현재 애그리거트의 버전과 동일한 경우에만 데이터를 수정한다. 그리고 수정에 성공하면

버전 값을 1 증가시킨다. 다른 트랜잭션이 먼저 데이터를 수정해서 버전 값이 바뀌면 데이터 수정에 실패하게 된다.
<img width="694" alt="image" src="https://user-images.githubusercontent.com/40031858/169727418-27170a02-47fe-4ee2-acce-c1ec43b7a4fe.png">

위 그림에서 스레드1과 스레드2는 같은 버전을 갖는 애그리거트를 읽어와 수정한다. 두 스레드 중 스레드1이 먼저 커밋을 시도하는데 이 시점에 애그리거트 버전은 여전히

5이므로 애그리거트 수정에 성공하고 버전은 6이된다. 스레드1이 트랜잭션을 커밋한 후에 스레드2가 커밋을 시도하면 이미 애그리거트 버전이 6이므로

스레드2는 데이터 수정에 실패한다.

JPA는 버전을 이용한 비선점 잠금을 지원한다. 다음과 같이 버전으로 사용할 필드에 @Version 애노테이션을 붙이고 매핑되는 테이블에 버전을 저장할 칼럼을 추가하면 된다.

```java
@Entity
@Table(name="purchase_order")
@Access(AccessType.FIELD)
public class Order{
    @EmbeddedId
    private OrderNo number;

    @Version
    private long version;
}
```

응용 서비스는 버전에 대해 알 필요가 없다. 리포지토리에서 필요한 애그리거트를 구하고 알맞은 기능만 실행하면 된다. 기능 실행 과정에서

애그리거트 데이터가 변경되면 JPA는 트랜잭션 종료 시점에 비선점 잠금을 위한 쿼리를 실행한다
```java
public class ChangeShippingService{
    @Transactional
    public void changeShipping(ChnageShippingRequest changeReq){
        Order order = orderRepository.findById(new OrderNo(changeReq.getNumber()));
        checkNoOrder(order);
        order.changeShippingInfo(changeReq.getShippingInfo());
    }
}
```
비선점 잠금을 위한 쿼리를 실행할 때 쿼리 실행 결과로 수정된 행의 개수가 0이면 이미 누군가 앞서 데이터를 수정한 것이다. 이는 트랜잭션이

충돌한 것이므로 트랜잭션 종료 시점에 익셉션이 발생한다. 표현 영역 코드는 이 익셉션이 발생했는지에 따라 트랜잭션 충돌이 일어났는지 확인할 수 있다.

```java
@Controller
public class OrderController{
    private ChangeShippingService changeShippingService;

    @PostMapping("/changeShipping")
    public String changeShipping(ChangeShippingRequest changeReq){
        try{
            changeShippingService.changeShipping(changeReq);
            return "changeShippingSuccess";
        }catch(OptimisticLockingFailureException ex){
            //누군가 먼저 같은 주문 애그리거트를 수정했으므로
            // 트랜잭션이 충돌했다는 메시지를 보여ㄷ준다
            return "changeShippingTxConflict";
        }
    }
}
```

<img width="601" alt="image" src="https://user-images.githubusercontent.com/40031858/169728496-0b736dda-d4d7-446f-a57f-d9578c498cd7.png">

위 그림의 과정2에서 운영자는 배송 상태 변경을 요청할 때 앞서 과정 1을 통해 받은 애그리거트 버전값을 함께 전송한다. 시스템은 애그리거트를 조회할 때 버전 값도 함께 읽어온다.

만약 과정1에서 받은 버전 A와 과정 2.1을 통해 읽은 애그리거트의 버전 B가 다르면 과정1과 과정2사이에 다른 사용자가 해당 애그리거트를 수정한 것이다.

이 경우 시스템은 운영자가 이전 데이터를 기준으로 작업을 요청한 것으로 간주하여 과정2.1.2와 같이 수정할 수 없다는 에러를 응답한다.

만약 버전 A와 버전 B가 같다면 과정1과 과정2사이에 애그리거트를 수정하지 않은 것이다. 이 경우 시스템은 과정 2.1.3과 같이 애그리거트를 수정하고,

과정 2.1.4를 이용해서 변경 내용을 DBMS에 반영한다. 과정 2.1.1과 과정 2.1.4 사이에 아무도 애그리거트를 수정하지 않았다면 커밋에 성공하므로 성공 결과를 응답한다.

만약 과정 2.1.1과 과정 2.1.4 사이에 누군가 애그리거트를 수정해서 커밋했다면 버전 값이 증가한 상태가 되므로 트랜잭션 커밋에 실패하고 결과로 에러를 응답한다.

응용 서비스에 전달할 요청 데이터는 사용자가 전송한 버전 값을 포함한다. 예를 들어 배송 상태 변경을 처리하는 응용 서비스가 전달받는 데이터는

다음과 같이 주문번호와 함께 해당 주문을 조회한 시점의 버전 값을 포함해야 한다.

```java
public class StartShippingRequest{
    private String orderNumber;
    private long version;

    ...생성자,getter
}
```

응용 서비스는 전달받은 버전 값을 이용해서 애그리거트 버전과 일치하는지 확인하고, 일치하는 경우에만 기능을 수행한다

```java
public class StartShippingService{
    @PreAuthorize("hasRole('ADMIN')")
    @Transactional
    public void startShipping(StartShippingRequest req){
        Order order = orderRepository.findById(new OrderNo(req.getOrderNumber()));
        checkOrder(order);
        if(!order.matchVersion(req.getVersion())){
            throw new VersionConflictException();
        }
        order.startShipping();     
    }
}
```

Order#matchVersion(long version) 메서드는 현재 애그리거트의 버전과 인자로 전달받은 버전이 일치하면 true를 리턴하고 그렇지 않으면 false를 리턴한다.

matchVersion() 결과가 true가 아니면 버전이 일치하지 않는 것이므로 사용자가 이전 버전의 애그리거트 정보를 바탕으로 상태 변경을 요청한 것이다.

따라서 응용 서비스는 버전이 충돌했다는 익셉션을 발생시켜 표현 계층에 이를 알린다.

표현 계층은 버전 충돌 익셉션이 발생하면 버전 충돌을 사용자에게 알려 사용자가 알맞은 후속 처리를 할 수 있도록 한다.

```java
@Controller
public class OrderAdminController{
    private StartShippingService startShippingService;

    @PostMapping("/startShipping")
    public String startShipping(StartShippingRequest startReq){
        try{
            startShippingService.startShipping(startReq);
            return "shippingStarted";
        }catch(OptimisticLockingFailureException | VersionConflictException ex){
            //트랜잭션 충돌
            return "startShippingTxConflict";
        }
    }
}
```

이 코드는 비선점 잠금과 관련해서 발생하는 두개의 익셉션을 처리하고 있다. VersionConflictException은 이미 누군가가 애그리거트를 수정했다는 것을 의미하고,

OptimisticLockingFailureException은 누군가가 거의 동시에 애그리거트를 수정했다는 것을 의미한다.

버전 충돌 상황에 대한 구분이 명시적으로 필요 없다면 응용 서비스에서 프레임워크용 익셉션을 발생시키는것도 고려할 수 있다.

```java
public void startShipping(StartShippingRequest req){
    Order order = orderRepository.findById(new OrderNo(req.getOrderNumber()));
    checkOrder(order);
    if(!order.matchVersion(req.getVersion())){
        //프레임워크가 제공하는 비선점 트랜잭션 충돌 관련 익셉션 사용
        throw new OptimisticLockingFailureException("version conflict");
    }
    order.startShipping();
}
```

### 강제 버전 증가

애그리거트에 애그리거트 루트 외에 다른 엔티티가 존재하는데 기능 실행 도중 루트가 아닌 다른 엔티티의 값만 변경된다고 하자. 이경우 JPA는 루트 엔티티의 버전 값을

증가시키지 않는다. 연관된 엔티티의 값이 변경된다고 해도 루트 엔티티 자체의 값은 바뀌는 것이 없으므로 루트 엔티티의 버전 값은 갱신하지 않는 것이다.

그런데 이런 JPA 특징은 애그리거트 관점에서 보면 문제가 된다. JPA는 이런 문제를 처리할 수 있도록 EntityManager#find()메서드로

엔티티를 구할 때 강제로 버전 값을 증가시키는 잠금 모드를 지원한다. 다음은 비선점 강제 버전 증가 잠금 모드를 사용해서 엔티티를 구하는 코드다.

```java
@Repository
public class JpaOrderRepository implements OrderRepository{
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public Order findByIdOptimisticLockMode(OrderNo id){
        return entityMager.find(
            Order.class , id , LockModeType.OPTIMISTIC_FORCE_INCREMENT);
    }
}
```

LockModeType.OPTIMISTIC_FORCE_INCREMENT를 사용하면 해당 엔티티의 상태가 변경되었는지에 상관없이 트랜잭션 종료 시점에 

버전 값 증가 처리를 한다. 이 잠금 모드를 사용하면 애그리거트 루트 엔티티가 아닌 다른 엔티티나 밸류가 변경되더라도 버전 값을 증가시킬 수 있으므로

비선점 잠금 기능을 안전하게 적용할 수 있다. 스프링 데이터 JPA를 사용하면 @Lock 애노테이션을 이용해 지정하면 된다.
## 오프라인 선점 잠금

문서를 편집할 때 누군가 먼저 편집중이면 다른사용자가 문서를 수정하고 있다고 보여지는 경우가 있다. 이런 안내를 통해 여러 사용자가

동시에 한 문서를 수정할 때 발생하는 충돌을 사전에 방지할 수 있게 해준다.

한 트랜잭션 범위에서만 적용되는 선점 잠금 방식이나 나중에 버전 충돌을 확인하는 비선점 잠금 방식으로는 이를 구현할 수 없다.

이때 필요한 것이 `오프라인 선점 잠금 방식`이다.

단일 트랜잭션에서 동시 변경을 막는 선점 잠금 방식과 달리 오프라인 선점 잠금은 여러 트랜잭션에 걸쳐 동시 변경을 막는다. 첫 번째 트랜잭션을 시작할 때 

오프라인 잠금을 선점하고, 마지막 트랜잭션에서 잠금을 해제한다. 잠금을 해제하기 전까지 다른 사용자는 잠금을 구할 수 없다.

예를 들어 수정 기능을 생각해보면 보통 수정 기능은 두개의 트랜잭션으로 구성된다. 첫 번째 트랜잭션은 폼을 보여주고, 두번째 트랜잭션은 데이터를 수정한다.

오프라인 선점 잠금을 사용하면 아래 그림의 과정 1처럼 폼 요청 과정에서 잠금을 선점하고, 과정 3처럼 수정 과정에서 잠금을 해제한다.

이미 잠금을 선점한 상태에서 다른 사용자가 폼을 요청하면 과정2처럼 잠금을 구할 수 없어 에러화면을 보게 된다.

<img width="521" alt="image" src="https://user-images.githubusercontent.com/40031858/169729431-1ebda665-75c1-477e-8d7b-1f343cc4ac4f.png">

사용자 A가 과정 3의 수정 요청을 수행하지 않고 프로그램을 종료하면 어떻게 될까? 이 경우 잠금을 해제하지 않으므로 다른 사용자는 영원히 잠금을 구할 수 없는 상황이 발생한다.

이런 사태를 방지하기 위해 오프라인 선점 방식은 잠금 유효 시간을 가져야 한다. 유효 시간이 지나면 자동으로 잠금을 해제해서 다른 사용자가 

잠금을 일정 시간 후에 다시 구할 수 있도록 해야한다.

### 오프라인 선점 잠금을 위한 LockManager 인터페이스와 관련 클래스

오프라인 선점 잠금은 크게 잠금 선점 시도, 잠금 확인, 잠금 해제, 잠금 유효시간 연장의 네가지 기능이 필요하다. 이 기능을 위한 LockManager인터페이스는 다음과 같다

```java
public interface LockManager{
    LockId tryLock(String type, String id) throws LockException;
    void checkLock(LockId lockId) throws LockException;
    void releaseLock(LockId lockId) throws LockException;
    void extendLockExpiration(LockId lockId, long inc) throws LockException;
}
```
tryLock() 메서드는 type과 id를 파라미터로 갖는다. 이 두 파라미터에는 각각 잠글 대상 타입과 식별자를 값으로 전달하면 된다. 예를 들어 식별자가 10인 

Article에 대한 잠긍르 구하고 싶다면 tryLock()을 실행할 때 'domain.Article'을 type값으로 주고 '10'을 id값으로 주면 된다.

tryLock()은 잠금을 식별할 때 사용할 LockId를 리턴한다.

일단 잠금을 구하면 잠금을 해제하거나 잠금이 유효한지 검사하거나 잠금 유효시간을 늘릴 때 LockId를 사용한다.

```java
public class LockId{
    private String value;

    public LockId(String value){
        this.value = value;
    }

    public String getValue(){
        return value;
    }
}
```

오프라인 선점 잠금이 필요한 코드는 LockManager#tryLock()을 이용해서 잠금을 시도한다. 잠금에 성공하면 tryLock()은 LockId를 리턴한다. 

이 LockId는 다음에 잠금을 해제할 때 사용한다. LockId가 없으면 잠금을 해제할수 없으므로 어딘가에 보관해야 한다.

다음은 컨트롤러가 오프라인 선점 잠금 기능을 이용해서 데이터 수정 폼에 동시에 접근하는 것을 제어하는 코드다.

수정 폼에서 데이터를 전송할 때 LockId를 전송할 수 있도록 LockId를 모델에 추가했다.
```java
public DataAndLockId getDataWithLock(Long id){
    LockId lockId = lockManager.tryLock("data",id);
    Data data = someDao.select(id);
    return new DataAndLockId(data,lockId);
}

@RequstMapping("/some/edit/{id}")
public String editForm(@PathVariable("id") Long id, ModelMap model){
    DataAndLockId dl = dataService.getDataWithLock(id);
    model.addAttribute("data",dl.getData());
    model.addAttribute("lockId",dl.getLockId());
    return "editForm";
}
```

잠금을 해제하는 코드는 다음과 같이 전달받은 LockId를 이용한다

```java
public void edit(EditRequest editReq, LockId lockId){
    lockManaager.checkLock(lockId);
    ...
    lockManager.releaseLock(lockId);
}

@RequestMapping(value="/some/edit/{id}" , method = RequestMethod.POST)
public String edit(@PathVariable("id") Long id,
                @ModelAttribute("editReq") EditRequest editReq,
                @RequestParam("id") String lockIdValue){
                    editReq.setId(id);
                    someEditService.edit(editReq,new LockId(lockIdValue));
                    model.addAttribute("data",data);
                    return "editSuccess";
                }
```

코드를 보면 LockManager#checkLock() 메서드를 가장 먼저 실행하는데, 잠금을 선점한 이후에 실행하는 기능은 다음과 같은 상황을

고려하여 반드시 주어진 LockId를 갖는 잠금이 유효한지 확인해야 한다
- 잠금 유효 시간이 지났으면 이미 다른 사용자가 잠금을 선점한다.
- 잠금을 선점하지 않은 사용자가 기능을 실행했다면 기능 실행을 막아야 한다.

### DB를 이용한 LockManager 구현

잠금 정보를 저장할 테이블과 인덱스를 다음과 같이 생성한다
```sql
create table locks(
    `type` varchar(255),
    id varchar(255),
    lockid varchar(255),
    expiration_time datetime,
    primary key (`type`,id)
) character set utf8;

create unique index locks_idx ON locks (lockid);
```

Order 타입의 1번 식별자를 갖는 애그리거트에 대한 잠금을 구하고 싶다면 다음의 insert 쿼리를 이용해 locks 테이블에 데이터를 삽입하면 된다.

```sql
insert into locks values('Order','1','생성한lockid','2022-05-22 20:10:00');
```
type과 id 칼럼을 주요키로 지정해서 동시에 두 사용자가 특정 타입 데이터에 대한 잠금을 구하는 것을 방지했다. 각 잠금마다 새로운 LockId를

사용하므로 lockid 필드를 유니크 인덱스로 설정했고 잠금 유효시간을 보관하기 위해 expiration_time 칼럼을 사용했다.

locks 테이블의 데이터를 담을 LockData 클래스를 다음과 같이 작성하자.

```java
@AllArgsConstructor
@Getter
public class LockData{
    private String type;
    private String id;
    private String lockId;
    private long expirationTime;

    public boolean isExpired(){
        return expirationTime < System.currentTimeMillis();
    }
}
```

locks 테이블을 이용해서 LockManager를 구현한 코드는 다음과 같다

```java
@Component
public class SpringLockManager implements LockManger{
    private int lockTimeout = 5 * 60 * 1000;
    private JdbcTemplate jdbcTemplate;

    private RowMapper<LockData> lockDataRowMapper = (rs,rowNum) ->
            new LockData(rs.getString(1),rs.getString(2),
                    rs.getString(3),rs.getTimestamp(4).getTime());
    
    @Transactional(propagation = Propagation.REQUIRES_NEW
    @Override
    public LockId tryLock(String type, String id) throws LockException{
        checkAlreadyLocked(type,id);
        LockId lockId = new LockId(UUID.randomUUID().toString());
        locking(type,id,lockId);
        return lockId;
    }

    private void checkAlreadyLocked(String type, String id){
        List<LockData> locks = jdbcTemplate.query(
            "select * from locks where type = ? and id =?",
            lockDataRowMapper,type,id);
        Optional<LockData> lockData = handleExpiration(locks);
        if(lockData.isPresent()) throw new AlreadyLockedException();
    }
    private Optional<LockData> handleExpiration(List<LockData> locks){
        if(locks.isEmpty()) return Optional.empty();
        LockData lockData = locks.get(0);
        if(lockData.isExpired()){
            jdbcTemplate.update(
                "delete from locks where type = ? and id =?",
                lockData.getType(),lockData.getId());
            return Optional.empty();
        }else{
            return Optional.of(lockData);
        }
    }
    private void locking(String type, String id, LockId lockId){
        try{
            int updatedCount = jdbcTemplate.update(
                "insert into locks values(?,?,?,?)",
                type, id, lockId.getValue(), new Timestamp(getExpirationTime()));
            if(updatedCount == 0) throw new LockingFailException();
        }catch(DuplicateKeyException e){
            throw new LockingFailException(e);
        }
    }
    private long getExpirationTime(){
        return System.currentTimeMillis() + lockTimeout;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @Override
    public void checkLock(LockId lockId) throws LockException{
        Optional<LockData> lockData = getLockData(lockId);
        if(!lockData.isPresent()) throw new NoLockException();
    } 

    private Optional<LockData> getLockData(LockId lockId){
        List<LockData> locks = jdbcTemplate.query(
            "select * from locks where lockid = ?",
            lockDataRowMapper, lockId.getValue());
        return handleExpiration(locks);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @Override
    public void extendLockExpiration(LockId lockId, long inc){
        Optional<LockData> lockDataOpt = getLockData(lockId);
        LockData lockData = 
                    lockDataOpt.orElseThrow(()->new NoLockException());
        jdbcTemplate.update(
            "update locks set expiration_time = ? where type = ? AND id = ?",
            new Timestamp(lockData.getTimestamp()+inc),
            lockData.getType(),lockData.getId());
    }
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @Override
    public void releaseLock(LockId lockId) throws LockException{
        jdbcTemplate.update(
            "delete from locks where lockid = ?", lockId.getValue());
    }

    @Autowired
    public void setJdbcTemplate(JdbcTemplate jdbcTemplate){
        this.jdbcTemplate = jdbcTemplate;
    }
}
```