# Chap02. 아키텍처 개요

## 네 개의 영역
'표현', '응용' , '도메인' , '인프라스트럭처'는 아키텍쳐를 설계할 때 출현하는 전형적인 네가지 영역이다. 네 영역 중 표현 영역은 사용자의 요청을 받아

응용 영역에 전달하고 응용 영역의 처리 결과를 다시 사용자에게 보여주는 역할을 한다. 웹 애플리케이션을 개발할 대 많이 사용하는 스프링 MVC 프레임워크가

표현 영역을 위한 기술에 해당한다.

![image](https://user-images.githubusercontent.com/40031858/168454442-a9ff69d1-4c1b-4a39-b9b7-1d3f9a0ab8f3.png)

표현 영역을 통해 사용자의 요청을 전달받는 응용 영역은 시스템이 사용자에게 제공해야 할 기능을 구현하는데 '주문 등록','주문 취소','상품 상세 조회'와 같은

기능 구현을 예로 들 수 있다. 응용 영역은 기능을 구현하기 위해 도메인 영역의 도메인 모델을 사용한다. 주문 취소 기능을 제공하는 응용 서비스를

예로 살펴보면 다음과 같이 주문 도메인 모델을 사용해 기능을 구현한다
```java
public class CancelOrderService{

    @Transactional
    public void cancelOrder(String orderId){
        Order order = findOrderById(orderId);
        if(order == null) throw new OrderNotFoundException(orderId);
        order.cancel();
    }
}
```

응용 서비스는 로직을 직접 수행하기보다는 도메인 모델에 로직 수행을 위임한다. 위 코드도 주문 취소 로직을 직접 구현하지 않고 Order객체에 위임하고 있다.

![image](https://user-images.githubusercontent.com/40031858/168454545-371b4c73-e234-4c39-a9ae-0414b6dd45ac.png)


도메인 영역은 도메인 모델을 구현한다. 도메인 모델은 도메인의 핵심 로직을 구현한다. 예를 들어 주문 도메인은 '배송지 변경', '결제 완료', '주문 총액 계산' 과 같은 핵심 롲기을 도메인 모델에서 구현한다.

인프라스트럭처 영역은 구현 기술에 대한것을 다룬다. 이 영역은 RDBMS 연동을 처리하고, 메시징 큐에 메시지를 전송하거나 수신하는 기능을 구현하고, 몽고나 레디스의 데이터 연동을 처리한다.

인프라스트럭처 영역은 논리적인 개념을 표현하기보다는 실제 구현을 다룬다.

## DIP
가격 할인 계산을 하려면 아래 그림과 같이 고객 정보를 구해야 하고, 구한 고객 정보와 주문 정보를 이용해서 룰을 실행해야 한다.
![image](https://user-images.githubusercontent.com/40031858/168454641-8c7b23a4-1a86-4192-99c7-292758798d7b.png)

여기서 CalculateDiscountService는 고수준 모듈이다. 고수준 모듈은 의미 있는 단일 기능을 제공하는 모듈로 '가격 할인 계산'이라는 기능을 구현한다.

고수준 모듈의 기능을 구현하려면 여러 하위 기능이 필요하다. 가격 할인 계산 기능을 구현하려면 고객 정보를 구해야 하고 룰을 실행해야 하는데 이 두기능이 하위 기능이다.

저수준 모듈은 하위 기능을 실제로 구현한것이다. `고수준 모듈이 제대로 동작하려면 저수준 모듈을 사용해야한다.` 그런데 고수준 모듈이 저수준

모듈을 사용하면 두 가지 문제, 즉 구현 변경과 테스트가 어렵다는 문제가 발생한다.

`DIP`는 이 문제를 해결하기 위해 저수준 모듈이 고수준 모듈에 의존하도록 바꾼다. 고수준 모듈을 구현하려면 저수준 모듈을 사용해야 하는데.

반대로 저수준 모듈이 고수준 모듈에 의존하도록 하려면 어떻게 해야할까? `비밀을 추상화한 인터페이스에 있다.`

CalculateDiscountService입장에서 봤을때 룰 적용을 Drools로 구현했는지 자바로 직접구현했는지 중요하지 않다. 

고객 정보와 구매 정보에 룰을 적용해서 할인 금액을 구한다는 것만 중요할 뿐이다. 이를 추상화한 인터페이스는 다음과 같다.

```java
public interface RuleDiscounter{
    Money applyRules(Customer customer, List<OrderLine> orderLines);
}
```

이 CalculateDiscountService가 RuleDiscounter를 이용하도록 바꿔보자.

```java
public class CalculateDiscountService{
    private RuleDiscounter ruleDiscounter;

    public CalculateDiscountService(RuleDiscounter ruleDiscounter){
        this.ruleDiscounter = ruleDiscounter;
    }

    public Money calculateDiscount(List<OrderLine> orderLines, String customerId){
        Customer customer = findCustomer(customerId);
        return ruleDiscounter.applyRules(customer, orderLines);
    }
}
```

CalculateDiscountService에는 Drools에 의존하는 코드가 없다. 단지 RuleDiscounter가 룰을 적용한다는 사실만 알 뿐이다.

실제 RuleDiscounter의 구현 객체는 생성자를 통해서 전달받는다.

`룰 적용을 구현한 클래스는 RuleDiscounter 인터페이스를 상속받아 구현한다. 다시 말하지만 Drools 관련 코드를 이해할 필요는 없다`

```java
public class DroolsRuleDiscounter implements RuleDiscounter{
    private KieContainer kContainer;

    public DroolsRuleEngine(){
        KieServices ks = KieServes.Factory.get();
        kContainer = ks.getKieClasspathContainer();
    }

    @Override
    public Money applyRules(Customer customer , List<OrderLine> orderLines){
        KieSession kSession = kContainer.newKieSession("discountSession");
        try{
            ...코드 생략
            kSession.fireAllRules();
        }finally{
            kSession.dispose();
        }
        return money.toImmutableMoney();
    }
}
```

DIP를 적용해서 다음과 같이 구조가 변경되었다.

![image](https://user-images.githubusercontent.com/40031858/168454880-f1f1b230-5876-4d8f-9f03-8b1ca3aef922.png)

구조를 보면 CalculateDiscountService는 더 이상 구현 기술인 Drools에 의존하지 않는다. '룰을 이용한 할인 금애 계산'은 추상화한

RuleDiscounter 인터페이스에 의존할 뿐이다. '룰을 이용한 할인 금액 계산' 은 고수준 모듈의 개념이므로 RuleDiscounter 인터페이슨는 고수준 모듈에 속한다.

![image](https://user-images.githubusercontent.com/40031858/168454896-69a024b0-21f9-4e8a-bc82-fa037af48dd6.png)

`DIP`를 적용하면 위와 같이 저수준 모듈이 고수준 모듈에 의존하게 된다. 고수준 모듈이 저수준 모듈을 사용하려면 고수준 모듈이 저수준 모듈에

의존해야 하는데, 반대로 `저수준 모듈이 고수준 모듈에 의존한다고 해서 DIP 즉, 의존 역전 원칙 `이라고 부른다.

DIP를 적용하면 앞의 다른 영역이 인프라스트럭처 영역에 의존할 때 발생했던 두 가지 문제인 구현 교체가 어렵다는 것과 테스트가 어려운 문제를 해소할 수 있다.

먼저 구현 기술 교체 문제를 보자. 고수준 모듈은 더이상 저수준 모듈에 의존하지 않고 구현을 추상화한 인터페이스에 의존한다.

실제 사용할 저수준 구현 객체는 다음 코드처럼 의존 주입을 이용해서 전달받을 수 있다.

```java
//사용할 저수준 객체 생성
RuleDiscounter ruleDiscounter = new DroolsRuleDiscounter();

//생성자 방식으로 주입
CalculateDiscountService disService = new CalculateDiscountService(ruleDiscounter);
```

구현 기술을 변경하더라도 CalculateDiscountService를 수정할 필요가 없다. 다음처럼 사용 할 저수준 구현 객체를 생성하는 코드만 변경하면 된다.
```java
//사용자 저수준 구현 객체 변경
RuleDiscounter ruleDiscounter = new SimpleRuleDiscounter();

//사용자 저수준 모듈을 변경해도 고수준 모듈을 수정할 필요가 없다
CalculateDiscountService disService = new CalculateDiscountService(ruleDiscounter);
```

스프링과 같은 의존 주입을 지원하는 프레임워클르 사용하면 설정 코드를 수정해서 쉽게 구현체를 변경할 수 있다.

이제 CalculateDiscountService가 제대로 동작하려면 Customer를 찾는 기능도 구현해야 한다. 이를 위한 고수준 인터페이스를

CustomerRepository라고 하자. CalculateDiscountService는 다음과 같이 두 인터페이스인 CustomerRepository와 RuleDiscounter를 사용해서 기능을 구현한다.

```java
public class CalculateDiscountService{
    private CustomerRepository customerRepository;
    private RuleDiscounter ruleDiscounter;

    public CalculateDiscounterService(CustomerRepository customerRepository , RuleDiscounter ruleDiscounter){
        this.customerRepository = customerRepository;
        this.ruleDiscounter = ruleDiscounter;
    }

    public Money calculateDiscount(List<OrderLine> orderLines, String customerId){
        Customer customer = findCoustomer(customerId);
        return ruleDiscounter.applyRules(customer,orderLines);
    }

    private Customer findCustomer(String customerId){
        Customer customer = customerRepository.findById(customerId);
        if(customer == null) throw new NoCustomerException();
        return customer;
    }
}

```

CalculateDiscountService가 제대로 동작하는지 테스트하려면 CustomerRepository와 RuleDiscounter를 구현한 객체가 필요하다.

만약 CalculateDiscountService가 저수준 모듈에 직접 의존했다면 저수준 모듈이 만들어지기 전까지 테스트를 할 수 없었겠지만 CustomerRepository와

RuleDiscounter는 인터페이스이므로 대역 객체를 사용해서 테스트를 진행할 수 있다. 다음은 대역 객체를 사용해서 Customer가 존재하지 않는 경우

익셉션이 발생하는지 검증하는 테스트 코드이다.
```java
public class ClaculateDiscountServiceTest{
    @Test
    public void noCustomer_thenExceptionShouldBeThrown(){
        //테스트 목적의 대역 객체
        CustomerRepository stubRepo = mock(CustomerRepository.class);
        when(stubRepo.findById("noCustId")).thenReturn(null);

        RuleDiscounter stubRule = (cust, lines) -> null;

        //대용 객체를 주입 받아 테스트 진행

        CalcualteDiscountService calDisSvc = new CalculateDiscountService(stubRepo, stubRule);

        assertThrows(NoCustomerException.class, () -> calDisSvc.calculateDiscount(someLines, "noCustId"));
    }
}
``` 

### DIP 주의 사항
DIP를 잘못 생각하면 단순히 인터페이스와 구현 클래스를 분리하는 정도로 받아들일 수 있다.

DIP의 핵심은 고소준 모듈이 저수준 모듈에 의존하지 않도록 하기 위함인데 DIP를 적용한 결과 구조만 보고 다음과 같이 저수준 모듈에서 인터페이스를 추출하는 경우가 있다

### `DIP를 잘못 적용한 예`
![image](https://user-images.githubusercontent.com/40031858/168457235-9df4c5d7-7535-4b1a-bb14-c08b8fffab72.png)


위의 그림은 잘못된 구조이다. 이 구조에서 도메인 영역은 구현 기술을 다루는 인프라스트럭처 영역에 의존하고 있다. 여전히 고수준 모듈이 

저수준 모듈에 의존하고 있는 것이다. RuleEngine 인터페이스는 고수준 모듈인 도메인 관점이 아니라 룰 엔진이라는 저수준 모듈관점에서 도출한 것이다.

DIP를 적용할 때 하위 기능을 추상화한 인터페이스는 고수준 모듈 관점에서 도출한다. CalculateDiscountService 입장에서 봤을 때

할인 금액을 구하기 위해 룰 엔진을 사용하는지 직접 연산하는지는 중요하지 않다. 단지 규칙에 따라 할인 금액을 계산한다는 것이 중요할 뿐.

즉 '할인 금액 계산'을 추상화한 인터페이스는 저수준 모듈이 아닌 고수준 모듈에 위치한다.
![image](https://user-images.githubusercontent.com/40031858/168457319-14225ecd-fab1-4be7-aecd-f1dc236e2310.png)

### DIP와 아키텍쳐
인프라스트럭처 영역은 구현 기술을 다루는 저수준 모듈이고 응용 영역과 도메인 영역은 고수준 모듈이다. 인프라스트럭처 계층이 가장

하단에 위치하는 계층형 구조와 달리 아키텍쳐에 DIP를 적용하면 다음과 같이 인프라스터럭처 영역이 응용 영역과 도메인 영역에 의존(상속) 하는 구조가 된다.

![image](https://user-images.githubusercontent.com/40031858/168457373-d1d32217-6a7f-4c81-93b3-01b4102bd4ae.png)


이런식으로 DIP를 적용하면 응용 영역과 도메인 영역에 영향을 최소화하면서 구현체를 변경하거나 추가할 수 있다.
![image](https://user-images.githubusercontent.com/40031858/168457507-9870cddd-1553-4ddf-981d-d13ec4bf56ff.png)

## 도메인 영역의 주요 구성요소

|||
|:--|:--|
|`요소`|`설명`|
|엔티티<br/>ENTITY|고유의 식별자를 갖는 객체로 자신의 라이프 사이클을 갖는다. 주문, 회원, 상품과 같이<br/>도메인의 고유한 개념을 포함한다. 도메인 모델의 데이터를 포함하며<br/> 해당 데이터와 관련된 기능을 함께 제공한다.|
|밸류<br/>VALUE|고유의 식별자를 갖지 않는 객체로 주로 개념적으로 하나의 값을 표현할 때 사용된다. <br/> 배송지 주소를 표현하기 위한 주소나 구매 금액와 같은 타입이 밸류 타입이다<br/> 엔티티의 속성으로 사용할 뿐만 아니라 다른 밸류 타입의 속성으로도 사용할 수 있다.|
|애그리거트<br/>AGGREGATE|애그리거트는 연관된 엔티티와 밸류 객체를 개념적으로 하나로 묶은 것이다. 예를 들어<br/>주문과 관련된 Order엔티티, OrderLine 밸류, Orderer 밸류 객체를 '주문'애그리거트로 묶을 수 있다.|
|레포지토리<br/>REPOSITORY|도메인 모델의 영속성을 처리한다. 예를 들어 DBMS 테이블에서 엔티티 객체를 로딩하거나 <br/> 저장하는 기능을 제공한다|
|도메인 서비스<br/>DOMAIN SERVICE|특정 엔티티에 속하지 않은 도메인 로직을 제공한다. '할인 금애 계산'은 상품, 쿠폰, 회원 등급<br/> 구매 금액 등 다양한 조건을 이용해서 구현하게 되는데, 이렇게 도메인 로직이<br/> 여러 엔티티와 밸류를 필요로 하면 도메인 서비스에서 로직을 구현한다.|

### 엔티티와 밸류
이 두모델의 가장 큰 차이점은 도메인 모델의 `엔티티는 데이터와 함께 도메인 기능을 함께 제공한다는 점`이다. 예를 들어 주문을 표현하는

엔티티는 주문과 관련된 데이터뿐만 아니라 배송지 주소 변경을 위한 기능을 함께 제공한다.

```java
public class Order{
    //주문 도메인 모델의 데이터
    private OrderNo number;
    private Orderer orderer;
    private ShippingInfo shippingInfo;
    ...

    //도메인 모델 엔티티는 도메인 기능도 함게 제공
    public void changeShippingInfo(ShippingInfo newShippingInfo){
        ...
    }
}
```

도메인 모델의 엔티티는 단순히 데이터를 담고 있는 데이터 구조라기보다는 데이터와 함께 기능을 제공하는 객체이다. 도메인 관점에서 기능을 구현하고

기능 구현을 캡슐화해서 데이터가 임의로 변경되는것을 막는다.

또 다른 차이점은 도메인 모델의 `엔티티는 두 개 이상의 데이터가 개념적으로 하나인 경우 밸류 타입을 이용해서 표현할 수 있다는 것이다`.

위 코드에서 주문자를 표현하는 Orderer는 밸류타입으로 다음과 같이 주문자 이름과 이메일 데이터를 포함할 수 있다.
```java
public class Orderer{
    private String name;
    private String email;
    ...
}
```

RDBMS와 같은 관계형 데이터베이스는 밸류 타입을 제대로 표현하기 힘들다. Order객체의 데이터를 저장하기위한 테이블은 별도 테이블로 분리해서 저장해야 한다.

밸류는 불변으로 구현할 것을 권장하며 이는 엔티티의 밸류 타입 데이터를 변경할 때는 객체 자체를 완전히 교체한다는 것을 의미한다.

예를 들어 배송지 정보를 변경하는 코드는 기존 객체의 값을 변경하지 않고 다음과 같이 새로운 객체를 필드에 할당한다.

```java
public class Order{
    private ShippingInfo shippingInfo;
    ...
    
    //도메인 모델 엔티티는 도메인 기능도 함께 제공
    public void changeShippingInfo(ShippingInfo newShippingInfo){
        checkShippingInfoChagneable();
        setShippingInfo(newShippingInfo);
    }

    private void setShippingInfo(ShippingInfo newSHippingInfo){
        if(newShippingInfo == null) throw new IlleagalArgumentException();
        //밸류 타입의 데이터를 변경할 때는 새로운 객체로 교체한다.

        this.shippingInfo = newShippingInfo;
    }
}
```

### 애그리거트
도메인이 커질수록 개발할 도메인 모델도 커지면서 많은 엔티티와 밸류가 출현한다. 엔티티와 밸류 개수가 많아질수록 모델은 점점 더 복잡해진다.

도메인 모델이 복잡해지면 개발자가 전체 구조가 아닌 한 개 엔티티와 밸류에만 집중하는 상황이 발생한다. 이때 상위 수준에서 모델을 관리하지 않고

개별 요소에만 초점을 맞추다 보면, 큰 수준에서 모델을 이해하지 못해 큰 틀에서 모델을 관리할 수 없는 상황에 빠질 수 있다.

지도를 볼 때 매우 상세하게 나온 대축척 지도를 보면 큰 수준에서 어디에 위치하고 있는지 이해하기 어려우므로 큰 수준에서 보여주는 소축척

지도를 함께 봐야 현재 위치를 보다 정확하게 이해할 수 있다. 이와 비슷하게 도메인 모델도 개별 객체뿐만 아니라 상위 수준에서 모델을 볼 수 있어야

전체 모델의 관계와 개별 모델을 이해하는데 도움이 된다. 도메인 모델에서 전체 구조를 이해하는데 도임이 되는 것이 바로 `애그리거트`이다.

`애그리거트`는 관련 객체를 하나로 묶은 군집이다. 애그리거트의 대표적인 예가 주문이다. 주문 이라는 도메인 개념은 '주문','배송지 정보','주문자' 등의 

하위 모델로 구성된다. 이 하위 개념을 표현한 모델을 하나로 묶어서 '주문' 이라는 상위 개념으로 표현할 수 있다.

애그리거트를 사용하면 개별 객체가 아닌 관련 객체를 묶어서 객체 군집 단위로 모델을 바라볼 수 있게 된다. 개별 객체 간의 관계가 아닌

애그리거트 간의 관계로 도메인 모델을 이해하고 구현하게 되며, 이를 통해 큰틀에서 도메인 모델을 관리할 수 있다.

애그리거트는 군집에 속한 객체를 관리하는 루트 엔티티를 갖는다. 루트 엔티티는 애그리거트에 속해 있는 엔티티와 밸류 객체를 이용해서

애그리거트가 구현해야 할 기능을 제공한다. 애그리거트를 사용하는 코드는 애그리거트 루트가 제공하는 기능을 실행하고 애그리거트 루트를

통해서 간접적으로 애그리거트 내의 다른 엔티티나 밸류 객체에 접근한다. 이것은 애그리거트의 내부 구현을 숨겨서 애그리거트 단위로

구현을 캡슐화할 수 있도록 보인다.

예를 들어 Order의 배송지 정보 변경 기능은 배송지를 변경할 수 있는지 확인한 뒤에 배송지 정보를 변경한다.
```java
public class Order{
    ...
    public void changeShippingInfo(ShippingInfo newInfo){
        checkShippingInfoChangeable(); //배송지 변경 가능 여부 확인
        this.shippingInfo = newInfo;
    }

    private void checkShippingInfoChangeable(){
        ... 배송지 정보를 변경할 수 있는지 여부를 확인하는 도메인 규칙 구현
    }
}
```

checkShippingInfoChangeable() 메소드는 도메인 규칙에 따라 배송지를 변경할 수 있는지 확인한다. 예를 들어 이미 배송이 시작된 경우

익셉션을 발생하는 식으로 도메인 규칙을 구현할 것이다.

주문 애그리거트는 Order를 통하지 않고 ShippingInfo를 변경할 수 있는 방법을 제공하지 않는다. 즉 배송지를 변경하려면 루트 엔티티인

Order를 사요해야 하므로 배송지 정보를 변경할때에는 Order가 구현한 도메인 로직을 항상 따르게 된다.

### 리포지토리
도메인 객체를 지속적으로 사용하려면 RDBMS, NoSQL, 로컬 파일과 같은 물리적인 저장소에 도메인 객체를 보관해야 한다. 이를 위한 도메인 모델이

리포지토리(Repository)이다. 엔티티나 밸류가 요구사항에서 도출되는 도메인 모델이라면 레포지토리는 구현을 위한 도메인 모델이다.

레포지토리는 애그리거트 단위로 도메인 객체를 저장하고 조회하는 기능을 정의한다. 예를 들어 주문 애그리거트를 위한 리포지토리는 다음과 같이 정의할 수 있다.

```java
public interface OrderRepository{
    Order findByNumber(OrderNumber number);
    void save(Order order);
    void delete(Order order);
}
```

OrderRepository의 메소드를 보면 대상을 찾고 저장하는 단위가 애그리거트 루트인 Order인 것을 알 수 있다. Order는 애그리거트에 속한

모든 객체를 포함하고 있으므로 결과적으로 애그리거트 단위로 젖아하고 조회한다. 도메인 모델을 사용해야 하는 코드는 리포지토리를 통해서 도메인 객체를

구한 뒤에 도메인 객체의 기능을 실행한다. 예를 들어 주문 취소 기능을 제공하는 응용 서비스는 다음 코드처럼 OrderRepository를 이용해서 Order

객체를 구하고 해당 기능을 실행한다.

```java
public class CalcelOrderService{
    private OrderRepository orderRepository;

    public void cancel(OrderNumber number){
        Order order = orderRepository.findByNumber(number);
        if(order == null) throw new NoOrderException(number);
        order.cancel();
    }
}
```

도메인 모델 관점에서 OrderRepository는 도메인 객체를 영속화하는 데 필요한 기능을 추상화 한것으로 고수준 모듈에 속한다.

기반 기술을 이용해서 OrderRepository를 구현한 클래스는 저수준 모듈로 인프라스트럭쳐 영역에 속한다.

![image](https://user-images.githubusercontent.com/40031858/168458143-014a8386-5ab0-4d62-9f64-a229b4db0cb0.png)

응용 서비스는 의존 주입과 같은 방식을 사용해서 실제 리포지토리 구현 객체에 접근한다

응용 서비스와 리포지토리는 밀접한 연관이 있다. 그 이유는 다음과 같다.
- 응용 서비스는 필요한 도메인 객체를 구하거나 저장할 때 리포지토리를 사용한다
- 응용 서비스는 트랜잭션을 관리하는데, 트랜잭션 처리는 리포지토리 구현 기술의 영향을 받는다.

리포지토리를 사용하는 주체가 응용서비스이기 때문에 리포지토리는 응용 서비스가 필요로하는 메소드를 제공한다. 다음 두 메소드가 기본이 된다.
- 애그리거트를 저장하는 메소드
- 애그리거트 루트 식별자로 애그리거트를 조회하는 메소드

이 두 메소드는 다음 형태를 갖는다
```java
public interface SomeRepository{
    void save(Some some);
    Some findById(SomeId id);
}
```

## 요청 처리 흐름

사용자 입장에서 봤을 때 웹 애플리케이션이나 데스크톱 애플리케이션과 같은 소프트웨어는 기능을 제공한다. 사용자가 애플리케이션에 기능 실행을 요청하면

그 요청을 처음 받는 영역은 표현 영역이다. 스프링 MVC를 사용해서 웹 애플리케이션을 구현했다면 컨트롤러가 사용자의 요청을 받아 처리하게 된다.

표현 영역은 사용자가 전송한 데이터 형식이 올바른지 검사하고 문제가 없다면 데이터를 이용해서 응용 서비스에 기능 실행을 위임한다.

이때 표현 영역은 사용자가 전송한 데이터를 응용 서비스가 요구하는 형식으로 변환해서 전달한다. 웹 브라우저를 이용해서 기능 실행을 요청하면

아래 그림처럼 표현 영역에 해당하는 컨트롤러는 과정 1.1에서 HTTP요청 파라미터를 응용 서비스가 필요로 하는 데이터로 변환해서 응용 서비스를 실행할 때 인자로 전달한다.

![image](https://user-images.githubusercontent.com/40031858/168458397-cb98cddc-bbec-4cbc-9f86-7161ed6d65fa.png)

응용 서비스는 도메인 모델을 이용해서 기능을 구현한다. 기능 구현에 필요한 도메인 객체를 리포지토리에서 가져와 실행하거나 신규 도메인 객체를

생성해서 리포지토리에 저장한다. 두개 이상의 도메인 객체를 사용해서 구현하기도 한다.

'예매하기'나 '예매 취소'와 같은 기능을 제공하는 응용 서비스는 도메인의 상태를 변경하므로 변경상태나 물리 저장소에 올바르게 반영 되도록

트랜잭션을 관리해야 한다. 예를 들어 스프링 프레임워크를 사용하면 다음과 같이 스프링에서 제공하는 @Transactional 애노테이션을 이용해

트랜잭션을 처리할 수 있을 것이다.

```java
public class CancelOrderService{
    private OrderRepository orderRepository;

    @Transactional //응용 서비스는 트랜잭션을 관리한다.
    public void cancel(OrderNumber number){
        Order order = orderRepository.findByNumber(number);
        if(order == null) throw new NoOrderException(number);
        order.cancel();
    }
    ...
}
```
응용 서비스 구현은 6장에서 살펴보자..!
