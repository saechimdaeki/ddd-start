# Chap10 이벤트.
## 시스템 간 강결합 문제
쇼핑몰에서 구매를 취소하면 환불을 처리해야한다. 이때 환불 기능을 실행하는 주체는 주문 도메인 엔티티가 될 수 있다.

도메인 객체에서 환불 기능을 실행하려면 다음 코드처럼 환불 기능을 제공하는 도메인 서비스를 파라미터로 전달받고 취소 도메인 기능에서 도메인 서비스를 실행하게 된다.

```java
public class Order{
    ...
    // 외부 서비스를 실행하기 위해 도메인 서비스를 파라미터로 전달받음
    public void cancel(RefundService refundService){
        verifyNotYetShipped();
        this.state = OrderState.CANCELED;

        this.refundStatus = State.REFUND_STARTED;
        try{
            refundService.refund(getPaymentId());
            this.refundStatus = State.REFUND_COMPLETED;
        }catch(Exception ex){
            ???
        }
    }
    ...
```
응용 서비스에서 환불 기능을 실행할 수도 있다. 

```java
public class CancelOrderService{
    private RefundService refundService;

    @Transactional
    public void cancel(ORderNo orderNo){
        Order order = findOrder(orderNo);
        order.cancel();

        order.refundStarted();
        try{
            refundService.refund(order.getPaymentId());
            order.refundCompleted();
        }catch(Exception ex){
            ???
        }
    }
    ...
```

보통 결제 시스템은 외부에 존재하므로 RefundService는 외부에 있는 결제 시스템이 제공하는 환불 서비스를 호출한다. 이때 두가지 문제가 발생할 수 있다.

첫 번째 문제는 외부 서비스가 정상이 아닐 경우 트랜잭션 처리를 어떻게 해야할지 애매하다는 것이다. 환불 기능을 실행하는 과정에서 익셉션이

발생하면 트랜잭션을 롤백해야 할까? 아니면 일단 커밋해야 할까?

두번째 문제는 성능에 대한 것이다. 환불을 처리하는 외부 시스템의 응답 시간이 길어지면 그 만큼 대기 시간도 길어진다. 즉 외부서비스 성능에 직접적인 영향을 받게된다.

또한 Order는 주문을 표현하는 도메인 객체인데 결제 도메인의 환불 관련 로직이 뒤섞이게 된다. 지금까지 언급한 문제가 발생하는 이유는

주문 바운디드 컨텍스트와 결제 바운디드 컨텍스트간의 `강결합` 때문이다. 주문이 결제와 강하게 결합되어 있어서 주문 바운디드 컨텍스트가

결제 바운디드 컨텍스트에 영향을 받게되는 것이다.

이런 강한 결합을 없앨 수 있는 방법이 있는데 바로 `이벤트`를 사용하는 것이다. 특히 비동기 이벤트를 사용하면 두 시스템 간의 결합을 크게

낮출 수 있다.

## 이벤트 개요
`이벤트` 라는 용어는 '과거에 벌어진 어떤 것'을 의미한다. 예를 들어 사용자가 암호를 변경한 것을 '암호가 변경했음 이벤트'가 벌어졌다고 할 수 있다.

### 이벤트 관련 구성요소

도메인 모델에 이벤트를 도입하려면 다음과 같은 네 개의 구성요소인 이벤트, 이벤트 생성 주체, 이벤트 디스패처(퍼블리셔), 이벤트 핸들러(구독자)를 구현해야 한다.


<img width="612" alt="image" src="https://user-images.githubusercontent.com/40031858/170066573-fdf3a849-9ee0-44aa-b863-dd9831e68b56.png">

도메인 모델에서 이벤트 생성 주체는 엔티티, 밸류, 도메인 서비스와 같은 도메인 객체이다. 이들 도메인 객체는 도메인 로직을 실행해서

상태가 바뀌면 관련 이벤트를 발생시킨다. `이벤트 핸들러`는 이벤트 생성 주체가 발생한 이벤트에 반응한다. 이벤트 핸들러는 생성주체가

발생한 이벤트를 전달받아 이벤트에 담긴 데이터를 이용해서 원하는 기능을 실행한다. 이벤트 생성 주체와 이벤트 핸들러를 연결해 주는 것이

`이벤트 디스패처`다. 이벤트 생성 주체는 이벤트를 생성해서 디스패처에 이벤트를 전달한다. 이벤트를 전달받은 디스패처는 해당 이벤트를

처리할 수 있는 핸들러에 이벤트를 전파한다. 이벤트 디스패처의 구현 방식에 따라 이벤트 생성과 처리를 동기나 비동기로 실행하게 된다.

### 이벤트의 구성
이벤트는 발생한 이벤트에 대한 정보를 담는다. 이 정보는 다음을 표현한다.
- 이벤트 종류: 클래스 이름으로 이벤트 종류를 표현
- 이벤트 발생시간
- 추가 데이터: 주문번호, 신규 배송지 정보 등 이벤트와 관련된 정보

배송지를 변경할 때 발생하는 이벤트를 생각해보자. 이 이벤트를 위한 클래스는 다음과 같이 작성할 수 있다.
```java
@AllArgsConstructor
@Getter
public class ShippingInfoChangedEvent{
    private String orderNumber;
    private long timestamp;
    private ShippingInfo newShippingInfo;
}
```
이 이벤트를 발생하는 주체는 Order 애그리거트다. Order 애그리거트의 배송지 변경 기능을 구현한 메서드는 다음 코드처럼 배송지 정보를

변경한 뒤에 이 이벤트를 발생시킬 것이다. 이 코드에서 Events.raise()는 디스패처를 통해 이벤트를 전파하는 기능을 제공한다.

```java
public class Order{
    public void changeShippingInfo(ShippingInfo newShippingInfo){
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo())
        Events.raise(new ShippingInfoChangedEvent(number,newShippingInfo));
    }
    ...
```
ShippingInfoChangedEvent를 처리하는 핸들러는 디스패처로부터 이벤트를 전달받아 필요한 작업을 수행한다. 예를들어 변경된 배송지 정보는

물류 서비스에 전송하는 핸들러는 다음과 같이 구현할 수 있다.
```java
public class ShippingInfoChangedHandler{
    @EventListener(ShippingInfoChangedEvent.class)
    public void handle(ShippingInfoChangedEvent evt){
        shippingInfoSynchronizer.sync(
            evt.getOrderNumber(),
            evt.getNewShippingInfo());
    }
}
```
이벤트는 이벤트 핸들러가 작업을 수행하는 데 필요한 데이터를 담아야 한다. 이 데이터가 부족하면 핸들러는 필요한 데이터를 읽기 위해 관련 API를 

호출하거나 DB에서 데이터를 직접 읽어야한다. 예를 들어 ShippingInfoChangedEvent가 바뀐 배송지 정보를 포함하고 있지 않다고 가정해 보자.

이 핸들러가 같은 VM에서 동작하고 있다면 다음과 같이 주문 데이터를 로딩해서 배송지 정보를 추출해야 한다.

```java
public class ShippingInfoChangedHandler{
    @EventListener(ShippingInfoChangedEvent.class)
    public void handle(ShippingInfoChangedEvent evt){
        Order oder = orderRepository.findById(evt.getOrderNo());
        shippingInfoSynchronizer.sync(
            order.getNumber().getValue(),
            order.getShippingInfo());
    }
    ...
```
이벤트는 데이터를 담아야 하지만 그렇다고 이벤트 자체에 관련 없는 데이터를 포함할 필요는 없다. 배송지 정보를 변경해서 발생시킨 ShippingInfoChangedEvent가

이벤트 발생과 직접 관련된 배송지 정보를 포함하는 것은 맞지만 배송지 정보 변경과 전혀 관련 없는 주문 상품번호와 개수를 담을 필요는 없다.


### 이벤트 용도
이벤트는 크게 두가지 용도로 쓰인다. 첫번째 용도는 `트리거`다. 도메인의 상태가 바뀔 때 다른 후처리가 필요하면 후처리를 실행하기

위한 트리거로 이벤트를 사용할 수 있다. 주문에서는 주문 취소 이벤트를 트리거로 사용할 수 있다. 주문을 취소하면 환불을 처리해야 하는데

이때 환불 처리를 위한 트리거로 주문 취소 이벤트를 사용할 수 있다.

<img width="624" alt="image" src="https://user-images.githubusercontent.com/40031858/170069187-cfcea0a4-c286-4a3a-b28c-992b5a4a4773.png">

이벤트의 두 번째 용도는 `서로 다른 시스템간의 데이터 동기화`이다. 배송지를 변경하면 외부 배송 서비스에 바뀐 배송지 정보를 전송해야 한다.

주문 도메인은 배송지 변경 이벤트를 발생시키고 이벤트 핸들러는 외부 배송 서비스와 배송지 정보를 동기화 할수 있다.

이벤트를 사용하면 서로 다른 도메인 로직이 섞이는 것을 방지할 수 있다.

## 이벤트, 핸들러, 디스패처 구현

이벤트와 관련된 코드는 다음과 같다.
- `이벤트 클래스` : 이벤트를 표현한다
- `디스패처` : 스프링이 제공하는 ApplicationEventPublisher를 이용한다
- `Events` : 이벤트를 발행한다. 이벤트 발행을 위해 ApplicationEventPublisher를 사용한다
- `이벤트 핸들러` : 이벤트를 수신해서 처리한다. 스프링이 제공하는 기능을 사용한다.

### 이벤트 클래스
이벤트 자체를 위한 상위 타입을 존재하지 않는다. 원하는 클래스를 이벤트로 사용하면 된다.
```java

@AllArgsConstructor
@Getter
public class OrderCanceledEvent{
    //이벤트는 핸들러에서 이벤트를 처리하는 데 필요한 데이터를 포함한다.
    private String orderNumber;
}
```

모든 이벤트가 공통으로 갖는 프로퍼티가 존재한다면 관련 상위 클래스를 만들 수도 있다. 예를 들어 모든 이벤트가 발생 시간을 갖도록 하려면

다음과 같은 상위 클래스를 만들고 각 이벤트 클래스가 상속받도록 할 수 있다.

```java
@Getter
public abstract class Event{
    private long timeStamp;

    public Event(){
        this.timestamp = System.currentTimeMillis();
    }
}
```
이제 발생 시간이 필요한 이벤트 클래스는 Event 클래스를 상속받아 구현하면 된다

```java
//발생 시간이 필요한 각 이벤트 클래스는 Event를 상속받아 구현한다
public class OrderCanceledEvent extends Event{
    private String orderNumber;

    public OrderCanceledEvent(String number){
        super();
        this.orderNumber = number;
    }
    ...
```

### Events 클래스와 ApplicationEventPublisher
이벤트 발생과 출판을 위해 스프링이 제공하는 `ApplicationEventPublisher`를 사용한다. 스프링 컨테이너는 `ApplicationEventPublisher`도 된다.

Events클래스는 ApplicationEventPublisher를 사용해서 이벤트를 발생시키도록 구현하자.

```java
public class Events{
    private static ApplicationEventPublisher publisher;

    static void setPublisher(ApplicationEventPublisher publisher){
        Events.publisher = publisher;
    }

    public static void raise(Object event){
        if(publisher != null){
            publisher.publishEvent(event);
        }
    }
}
```

Events 클래스의 raise() 메서드는 ApplicationEventPublisher가 제공하는 publishEvent()메서드를 이용해 이벤트를 발생시킨다. 

Events#setPublisher() 메서드에 이벤트 퍼블리셔를 전달하기 위해 스프링 설정 클래스를 작성하자

```java
@Configuration
public class EventsConfiguration{
    @Autowired
    private ApplicationContext applicationContext;

    @Bean
    public InitializingBean eventsinitializer(){
        return () -> Events.setPublisher(applicationContext);
    }
}
```
eventsinitializer() 메서드는 InitializingBean 타입 객체를 빈으로 설정한다. 이 타입은 스프링 빈 객체를 초기화할 때 사용하는 인터페이스로, 

이 기능을 사용해서 Events 클래스를 초기화했다. 참고로 ApplicationContext는 ApplicationEventPublisher를 상속하고 있으므로 Events클래스를 초기화 할때 가능하다.

### 이벤트 발생과 이벤트 핸들러
이벤트를 발생시킬 코드는 Events.raise() 메서드를 사용한다. 예를 들어 Order#cancel() 메서드는 다음과 같이 구매 취소 로직을 수행한 뒤 Events.raise()를 이용해 관련 이벤트를 발생시킨다.

```java
public class Order{
    public void cancel(){
        verifyNotYetShipped();
        this.state = OrderState.CANCELED;
        Events.raise(new OrderCanceledEvent(number.getNumber()));
    }
    ...
```
이벤트를 처리할 핸들러는 스프링이 제공하는 `@EventListener` 애노테이션을 사용해서 구현한다.

```java
@Service
@AllArgsConstructor
public class OrderCanceledEventHandler{
    private RefundService refundService;

    @EventListener(OrderCanceledEvent.class)
    public void handle(OrderCanceledEvent event){
        refundService.refund(event.getOrderNumber());
    }
}
```

ApplicationEventPublisher#publishEvent() 메서드를 실행할 때 OrderCanceledEvent 타입 객체를 전달하면, OrderCanceledEvent.class 값을

갖는 @EventListener 애노테이션을 붙인 메서드를 찾아 실행한다. 위 코드는 OrderCanceledEventHandler의 handle()메서드를 실행한다.


<img width="885" alt="image" src="https://user-images.githubusercontent.com/40031858/170074609-3fb149b7-1363-425b-a873-2ceb2c339662.png">

1. 도메인 기능을 실행한다
2. 도메인 기능은 Events.raise()를 이용해서 이벤트를 발생시킨다.
3. Events.raise()는 스프링이 제공하는 ApplicationEventPublisher를 이용해서 이벤트를 출판한다.
4. ApplicationEventPublisher는 @EventListener(이벤트타입.class) 애노테이션이 붙은 메서드를 찾아 실행한다.

으용서비스와 동일한 트랜잭션 범위에서 이벤트 핸들러를 실행한다. 즉 도메인 상태변경과 이벤트 핸들러는 같은 트랜잭션 범위에서 실행된다.

## 동기 이벤트 처리 문제
이벤트를 사용해서 강결합 문제는 해소했지만 남아있는 문제가 있는데 바로 외부 서비스에 영향을 받는 문제이다.

```java
// 1. 응용 서비스 코드
@Transactional // 외부 연동 과정에서 익셉션이 발생하면 트랜잭션 처리는?
public void cancel(OrderNo orderNo){
    Order order = findOrder(orderNo);
    order.cancel(); // order.cancel()에서 OrderCanceledEvent 발생
}

// 2. 이벤트를 처리하는 코드
@Service
public class OrderCancelEventHandler{
    ..생략
    @EventListener(OrderCanceledEvent.class)
    public void handler(OrderCanceledEvent event){
        // 느려지거나 익셉션이 발생하면?
        refundService.refund(event.getOrderNumber());
    }
}

```

이 코드에서 refundService.refund()가 외부 환불 서비스와 연동한다고 가정해보자. 외부 환불 기능이 느려지면 cancel() 메서드도 느려진다. 

생각해 볼만한 것은 외부 환불 서비스 실행에 실패했다고 해서 반드시 트랜잭션을 롤백 해야하는지에 대한 문제다. 일단 구매 취소 자체는 처리하고 환불만 재처리하거나 수동으로 처리할 수도 있다.

외부 시스템과의 연동을 동기로 처리할 때 발생하는 성능과 트랜잭션 범위 문제를 해소하는 방법은 이벤트를 비동기로 처리하거나 이벤트와 트랜잭션을 연계하는 것이다.

## 비동기 이벤트 처리

이벤트를 비동기로 구현할 수 있는 방법은 다양한데, 다음 네가지 방식으로 비동기 이벤트 처리를 구현하는 방법에 대해 살펴보자
- 로컬 핸들러를 비동기로 실행하기
- 메시지 큐를 사용하기
- 이벤트 저장소와 이벤트 포워더 사용하기
- 이벤트 저장소와 이벤트 제공 API 사용하기


### 로컬 핸들러 비동기 실행
이벤트 핸들러를 비동기로 실행하는 방법은 이벤트 핸들러를 별도 스레드로 실행하는 것이다. 스프링이 제공하는 `@Async` 애노테이션을 사용하면

비동기로 이벤트 핸들러를 실행할 수 있다. 이를 위해 두가지만 하면 된다.
- @EnableAsync 애노테이션을사용해서 비동기 기능을 활성화한다.
- 이벤트 핸들러 메서드에 @Async 애노테이션을 붙인다

```java
@SpringBootApplication
@EnableAsync
public class Application{
    public static void main(String[] args){
        SpringApplication.run(Application.class,args);
    }
}
```

이제 비동기로 실행할 이벤트 핸들러 메서드에 `@Async` 애노테이션만 붙이면 된다.
```java
@Service
public class OrderCanceledEventHandler{
    @Async
    @EventListener(OrderCanceledEvent.class)
    public void handle(OrderCanceledEvent event){
        refundService.refund(event.getOrderNumber());
    }
}
```
스프링은 OrderCanceledEvent가 발생하면 handle() 메서드를 별도 스레드를 이용해서 비동기로 실행한다.

### 메시징 시스템을 이용한 비동기 구현.
비동기로 이벤트를 처리해야 할 때 사용하는 또다른 방법은 `카프카`나 `래빗`과 같은 메시징 시스템을 사용하는 것이다. 이벤트가 발생하면

이벤트 디스패처는 아래 그림과 같이 이벤트를 메시지 큐에 보낸다. 메시지 큐는 이벤트를 메시지 리스너에 전달하고, 메시지 리스너는 알맞은

이벤트 핸들러를 이용해서 이벤트를 처리한다. 이때 이벤트를 메시지 큐에 저장하는 과정과 메시지 큐에서 이벤트를 읽어와 처리하는 과정은 별도

스레드나 프로세스로 처리된다.

<img width="839" alt="image" src="https://user-images.githubusercontent.com/40031858/170154720-df882854-119a-411f-9923-a6867393e8eb.png">


필요하다면 이벤트를 발생시키는 도메인 기능과 메시지 큐에 이벤트를 저장하는 절차를 한 트랜잭션으로 묶어야 한다. 도메인 기능을 실행한 결과를

DB에 반영하고 이 과정에서 발생한 이벤트를 메시지 큐에 저장하는 것을 같은 트랜잭션 범위에서 실행하려면 글로벌 트랜잭션이 필요하다.

글로벌 트랜잭션을 사용하면 안전하게 이벤트를 메시지 큐에 전달할 수 있는 장점이 있지만 반대로 글로벌 트랜잭션으로 인해 전체 성능이 떨어지는 단점도 있다.

메시지 큐를 사용하면 보통 이벤트를 발생시키는 주체와 이벤트 핸들러가 별도 프로세스에서 동작한다. 이것은 이벤트 발생 JVM과 이벤트 처리 JVM이

다르다는 것을 의미한다. 물론 한 JVM에서 이벤트 발생 주체와 이벤트 핸들러가 메시지 큐를 이용해서 이벤트를 주고받을 수 있지만, 동일 JVM에서

비동기 처리를 위해 메시지 큐를 사용한느 것은 시스템을 복잡하게 만들 뿐이다. 래빗MQ처럼 많이 사용되는 메시징 시스템은 글로벌 트랜잭션 지원과

함께 클러스터와 고가용성을 지원하기 때문에 안정적으로 메시지를 전달할 수 있는 장점이 있다. 카프카는 글로벌 트랜잭션을 지원하지 않지만 다른

메시징 시스템에 비해 높은 성능을 보여준다.

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/40031858/170155278-304ae875-cc61-4390-ad9c-422fe1afd41c.png">

이벤트가 발생하면 핸들러는 스토리지에 이벤트를 저장한다. 포워더는 주기적으로 이벤트 저장소에서 이벤트를 가져와 이벤트 핸들러를 실행한다.

포워더는 별도 스레드를 이용하기 때문에 이벤트 발행과 처리가 비동기로 처리된다. 이 방식은 도메인의 상태와 이벤트 저장소로 동일한 DB를 사용한다.

즉, 도메인의 상태 변화와 이벤트 저장이 로컬 트랜잭션으로 처리된다. 이벤트를 물리적 저장소에 보관하기 때문에 핸들러가 이벤트 처리에 실패할 경우

포워더는 다시 이벤트 저장소에서 이벤트를 읽어와 핸들러를 실행하면 된다. 이벤트 저장소를 이용한 두 번째 방법은 아래그림과 같이 이벤트를 외부에 제공하는 API를 사용하는것이다.

<img width="1032" alt="image" src="https://user-images.githubusercontent.com/40031858/170155472-0acce6d5-e9e0-4459-99ce-8730175c129a.png">

API 방식과 포워더 방식의 차이점은 이벤트를 전달하는 방식에 있다. 포워더 방식이 포워더를 이용해서 이벤트를 외부에 전달한다면, API 방식은

외부 핸들러가 API 서벙를 통해 이벤트 목록을 가져간다. 포워더 방식은 이벤트를 어디까지 처리했는지 추적하는 역할이 포워더에 있다면 API 방식에서는

이벤트 목록을 요구하는 외부 핸들러가 자신이 어디까지 이벤트를 처리했는지 기억해야 한다

### 이벤트 저장소 구현

포워더 방식과 API 방식 모두 이벤트 저장소를 사용하므로 이벤트를 저장할 저장소가 필요하다. 구조는 다음과 같다.

<img width="1096" alt="image" src="https://user-images.githubusercontent.com/40031858/170156399-593056ef-1257-4da6-a2c2-4b4ff835b0ea.png">

- EventEntity : 이벤트 저장소에 보관할 데이터이다. EventEntity는 이벤트를 식별하기 위한 id, 이벤트 타입인 type, 직렬화한 데이터 형식인 contentType,

    이벤트 데이터 자체인 payload, 이벤트 시간인 timestamp를 갖는다
- EventStore : 이벤트를 저장하고 조회하는 인터페이스를 제공한다
- JdbcEventStore : JDBC를 이용한 EventStore 구현 클래스다
- EventApi : REST API를 이용해서 이벤트 목록을 제공하는 컨트롤러다.

EventEntity 클래스는 다음과 같다. 이벤트 데이터를 정의한다.
```java
@Getter
public class EventEntry{
    private Long id;
    private String type;
    private String contentType;
    private String payload;
    private long timestamp;

    public EventEntry(String type, String contentType, String payload){
        this.type = type;
        this.contentType = contentType;
        this.payload = payload;
        this.timestamp = System.currentTimeMillis();
    }
    public EventEntry(Long id, String type, String contentType, String payload, 
                long timestamp){
            this.id = id;
            this.type = type;
            this.contentType = contentType;
            this.payload = payload;
            this.timestamp = timestamp;            
        }
}
```

EventStore는 이벤트 객체를 직렬화해서 payload에 저장한다. EventStore인터페이스는 다음과 같다.
```java
public interface EventStore{
    void save(Object event);
    List<EventEntry> get(long offset, long limit);
}
```

이벤트는 과거에 벌어진 사건이므로 데이터가 변경되지 않는다. 이런 이유로 EventStore 인터페이스는 새로운 이벤트를 추가하는 기능과 조회하는 기능만

제공하고 기존 이벤트 데이터를 수정하는 기능은 제공하지 않는다. EventStore 인터페이스를 구현한 JdbcEventStore클래스는 다음과 같다.

```java
@Component
@AllArgsConstructor
public class JdbcEventStore implements EventStore{
    private ObjectMapper objectMapper;
    private JdbcTemplate jdbcTemplate;

    @Override
    public void save(Object event){
        EventEntry entry = new EventEntry(event.getClass().getName(),
                "application/json",toJson(event));
        jdbcTemplate.update(
            "insert into evententry " +
                "(type, content_type, payload, timestamp) " +
                "values (?,?,?,?)",
            ps -> {
                ps.setString(1, entry.getType());
                ps.setString(2, entry.getContentType());
                ps.setString(3, entry.getPayload());
                ps.setTimestamp(4, new Timestamp(entry.getTimestamp()));
            });
    }

    private String toJson(object event){
        try{
            return objectMapper.writeValueAsString(event);
        }catch(JsonProcessingException e){
            throw new PayloadConvertException(e);
        }
    }

    @Override
    public List<EventEntry> get(long offset, long limit){
        return jdbcTemplate.query(
                "select * from evententry order by id asc limit ?,?",
                ps ->{
                    ps.setLong(1,offset);
                    ps.setLong(2,limit);
                },
                (rs, rowNum) ->{
                    return new EventEntry(
                        rs.getLong("id"),
                        rs.getString("type"),
                        rs.getString("content_type"),
                        rs.getString("payload"),
                        rs.getTimestamp("timestamp").getTime());
                });
    }
}

```

### 이벤트 저장을 위한 이벤트 핸들러 구현
이벤트 저장소를 위한 기반이 되는 클래스를 모두 구현했으므로 이제 발생한 이벤트를 이벤트 저장소에 추가하는 이벤트 핸들러를 구현하자.

```java
@Component
@AllArgsConstructor
public class EventStoreHandler{
    private EventStore eventStore;

    @EventListener(Event.class)
    public void handle(Event event){
        eventStore.save(event);
    }
}
```

EventStoreHandler의 handle() 메서드는 eventStore.save() 메서드를 이용해서 이벤트 객체를 저장한다.

### REST API 구현
REST API는 단순하다. offset과 limit의 웹 요청 파라미터를 이용해서 EventStore#get을 실행하고 그 결과를 JSON으로 리턴하면 된다.

```java
@RestController
@AllArgsConstructor
public class EventApi{
    private EventStore eventStore;

    @GetMapping("/api/events")
    public List<EventEntry> list(
            @RequestParam("offset") Long offset,
            @RequestParam("limit") Long limit){
        return eventStore.get(offset,limit);
    }
}

```

API를 사용하는 클라이언트는 일정 간격으로 다음 과정을 실행한다.

1. 가장 마지막에 처리한 데이터의 offset인 lastOffset을 구한다. 저장한 lastOffset이 없으면 0을 사용한다
2. 마지막에 처리한 lastOffset을 offset으로 사용해서 API를 실핸한다
3. API 결과로 받은 데이터를 처리한다
4. offset + 데이터 개수를 lastOffset으로 저장한다

마지막에 처리한 lastOffset을 저장하는 이유는 같은 이벤트를 중복해서 처리하지 않기 위해서이다. API를 사용하는 과정을 그림으로 정리하면 다음과 같다.

<img width="732" alt="image" src="https://user-images.githubusercontent.com/40031858/170158945-064cbd5a-4220-42b3-81e3-116d635d0ee9.png">

위 그림은 클라이언트가 1분 주기로 최대 5개의 이벤트를 조회하는 상황을 정리한 것이다. 클라이언트 API를 이용해서 언제든지 원하는 이벤트를 

가져올 수 있기 때문에 이벤트 처리에 실패하면 다시 실패한 이벤트부터 읽어와 이벤트를 재처리할 수 있다. API서버에 장애가 발생한 경우에도

주기적으로 재시도를 해서 API서버가 살아나면 이벤트를 처리할 수 있다.

### 포워더 구현
포워더는 API 방식의 클라이언트 구현과 유사하다. 포워더는 일정 주기로 EventStore에서 이벤트를 읽어와 이벤트 핸들러에 전달하면 된다.

API방식 클라이언트와 마찬가지로 전달한 이벤트의 offset을 기억해 두었다고 다음 조회 시점에 마지막으로 처리한 offset부터 이벤트를 가져오면 된다.

```java
@Component
public class EventForwarder{
    private static final int DEFAULT_LIMIT_SIZE = 100;
    
    private EventStore eventStore;
    private OffsetStore offsetStore;
    private EventSender eventSender;
    private int limitSize = DEFAULT_LIMIT_SIZE;

    public EventForwarder(EventStore eventStore,
                    OffsetStore offsetStore,
                    EventSender eventSender){
        this.eventStore = eventStore;
        this.offsetStore = offsetStore;
        this.eventSender = eventSender;                    
    }

    @Scheduled(initialDelay = 1000L, fixedDelay = 1000L)
    public void getAndSend(){
        long nextOffset = getNextOffset();
        List<EventEntry> events = eventStore.get(nextOffset,limitSize);
        if(!events.isEmpty()){
            int processedCount = sendEvent(events);
            if(processedCount > 0){
                savenextOffset(nextOffset + processedCount);
            }
        }
    }
    private long getNextOffset(){
        return offsetStore.get();
    }

    private int sendEvent(List<EventEntry> events){
        int processedCount = 0;
        try{
            for(EventEntry entry : events){
                eventSender.send(entry);
                processedCount++;
            }
        }catch(Exception ex){
            //로깅 처리
        }
        return processedCount;
    }
    private void saveNextOffset(long nextOffset){
        offsetStore.update(nextOffset);
    }
}
```

getAndSend() 메서드를 주기적으로 실행하기 위해 스프링의 `@Scheduled` 애노테이션을 사용했다. 

```java
public interface OffsetStore{
    long get();
    void update(long nextOffset);
}
```

OffsetStore를 구현한 클래스는 offset 값을 DB테이블에 저장하거나 로컬 파일에 보관해서 마지막 offset 값을 물리적 저장소에 보관하면 된다.

```java
public interface EventSender{
    void send(EventEntry event);
}
```

이 인터페이스를 구현한 클래스는 send() 메서드에서 외부 메시징 시스템에 이벤트를 전송하거나 원하는 핸들러에 이벤트를 전달하면 된다.

## 이벤트 적용 시 추가 고려 사항

이벤트를 구현할 때 추가로 고려해야할 점이 있다.

첫 번째는 이벤트 소스를 EventEntry에 추가할지 여부이다. 이 기능을 구현하려면 이벤트에 발생 주체 정보를 추가해야한다.

두 번째로 고려해야 할점은 포워더에서 전송 실패를 얼마나 허용할 것이냐에 대한 것이다. 포워더는 이벤트 전송에 실패하면 실패한 이벤트로부터

다시 읽어와 전송을 시도한다. 포워더를 구현할 때는 실패한 이벤트의 재전송 횟수 제한을 두어야 한다. 예를 들어 동일 이벤트를 전송하는데

3회 실패했다면 해당 이벤트는 생략하고 다음 이벤트로 넘어간다는 등의 정책이 필요하다.

세 번째 고려할 점은 이벤트 손실에 대한 것이다. 이벤트 저장소를 사용하는 방식은 이벤트 발생과 이벤트 저장을 한 트랜잭션으로 처리하기 때문에

트랜잭션에 성공하면 이벤트가 저장소에 보관된다는 것을 보장할 수 있다. 반면에 로컬핸들러를 이용해서 이벤트를 비동기로 처리할 경우 이벤트 처리에 실패하면 이벤트를 유실하게 된다.

네 번째 고려할 점은 이벤트 순서에 대한것이다. 이벤트 발생 순서대로 외부 시스템에 전달해야 할 경우 이벤트 저장소를 사용하는 것이 좋다. 

이벤트 저장소는 이벤트를 발생순서대로 저장하고 그 순서대로 이벤트 목록을 제공하기 때문이다. 

다섯 번째 고려할점은 이벤트 재처리에 대한것이다. 동일한 이벤트를 다시 처리해야 할 때 이벤트를 어떻게 할지 결정해야 한다. 가장 쉬운 방법은 마지막으로 처리한 이벤트의

순번을 기억해 두었다가 이미 처리한 순번의 이벤트가 도착하면 해당 이벤트를 처리하지 않고 무시하는 것이다.

### 이벤트 처리와 DB 트랜잭션 고려

이벤트를 처리할 때는 DB 트랜잭션을 함께 고려해야 한다. 예를 들어 주문 취소와 환불 기능을 다음과 같이 이벤트를 이용해서 구현했다고 하자
- 주문 취소 기능은 주문 취소 이벤트를 발생시킨다.
- 주문 취소 이벤트 핸들러는 환불 서비스에 환불 처리를 요청한다
- 환불 서비스는 외부 API를 호출해서 결제를 취소한다.

이벤트 발생과 처리를 모두 동기로 처리하면 실행흐름은 다음과 같을 것이다.

<img width="1199" alt="image" src="https://user-images.githubusercontent.com/40031858/170164814-f4598184-cccb-4bce-9a9d-ea182c4a90e9.png">

이와 같은 실행흐름에는 고민할 상황이 있다. 12번 과정까지 다 성공하고 13번 과정에서 DB를 업데이트하는데 실패하는 상황이 바로 그것이다

다 성공하고 13번 과정에서 실패하면 결제는 취소됐는데 DB에는 주문이 취소 되지 않은 상태로 남게 된다.

이벤트를 비동기로 처리할 때도 DB트랜잭션을 고려해야 한다. 

<img width="1190" alt="image" src="https://user-images.githubusercontent.com/40031858/170165570-8faa9205-3f6e-440c-99a7-c853e145f553.png">

주문 취소 이벤트를 비동기로 처리했을 때의 실행흐름이며 이벤트 핸들러를 호출하는 5번 과정은 비동기로 실행한다.

DB업데이트와 트랜잭션을 다 커밋한 뒤에 환불로직인 11~13번 과정을 실행했다고 하자. 만약 12번 과정에서 외부 API 호출에 실패하면

DB에는 주문이 취소된 상태로 데이터가 바뀌었는데 결제는 취소되지 않은 상태로 남게 된다.

이벤트 처리를 동기로 하든 비동기로 하든 이벤트 처리 실패와 트랜잭션 실패를 함께 고려해야 한다. 트랜잭션 실패와 이벤트 처리 실패를

모두 구려하면 복잡해지므로 경우의 수를 줄이면 도움이 된다. 경우의 수를 줄이는 방법은 트랜잭션이 성공할 때만 이벤트 핸들러를 실행하는 것이다.

스프링은 `@TransactionalEventListener` 애노테이션을 지원한다. 이 애노테이션은 스프링 트랜잭션 상태에 따라 이벤트 핸들러를 실행할 수 있게 한다.

```java
@TransactionalEventListener(
    classes = OrderCanceledEvent.class,
    phase = TransactionPhanse.AFTER_COMMIT
)
public void handle(OrderCanceledEvent event){
    refundService.refund(event.getOrderNumber());
}
```
위 코드에서 phase 속성 값으로 TransactionPhase.AFTER_COMMIT을 지정하면 트랜잭션 커밋 성공한 뒤에 핸들러 메서드를 실행한다.

중간에 에러가 발생해서 트랜잭션이 롤백 되면 핸들러 메서드를 실행하지 않는다. 이 기능을 사용하면 이벤트 핸들러를 실행했는데 트랜잭션이 롤백 되는 상황으 납ㄹ생하지 않는다.

이벤트 저장소로 DB를 사용해도 동일한 효과를 볼 수 있다. 이벤트 발생 코드와 이벤트 저장 처리를 한 트랜잭션으로 처리하면 된다. 이렇게 하면 트랜잭션이

성공할 때만 이벤트가 DB에 저장되므로 트랜잭션은 실패했는데 이벤트 핸들러가 실행되는 상황은 발생하지 않는다.

트랜잭션이 성공할 때만 이벤트 핸들러를 실행하게 되면 트랜잭션 실패에 대한 경우의 수가 줄어 이제 이벤트 처리 실패만 고민하면 된다.