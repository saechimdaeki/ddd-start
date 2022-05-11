# Chap01. 도메인 모델 시작하기

## `도메인`이란?
책을 구매할 때 온라인 서점을 자주 이요한다고 가정하자. 어떤 책이 나왔는지 검색하고, 목차와 서평을 보기도 한다고 하자.

개발자 입장에서 바라보면 온라인 서점은 구현해야 할 소프트웨어의 대상이다. 상품조회, 구매, 결제 등의 기능을 제공해야한다.

이때 온라인 서점은 소프트웨어로 해결하고자 하는 문제영역, 즉 도메인에 해당한다.

한 도메인은 다시 하위 도메인으로 나눌 수 있고 온라인 서점 도메인은 다음과 같은 예시를 들 수 있다.

![image](https://user-images.githubusercontent.com/40031858/167854802-6ec9eb8c-3105-4e5c-a179-29a9f4982e27.png)

물론 특정 도메인을 위한 소프트웨어라고 해서 도메인이 제공해야 할 모든 기능을 직접구현하는 것은 아니다. 예를 들어 배송의 경우

자체적으로 시스템을 구축하기 보다는 외부 배송업체의 시스템을 사용하고 배송 추적 정보를 제공하는 데 필요한 기능만 일부 연동하는 식으로 하곤한다.

## 도메인 모델 패턴
일반적인 애플리케이션의 아키텍쳐는 다음과 같이 네 개의 영역으로 구성된다.

![image](https://user-images.githubusercontent.com/40031858/167856090-0818c4eb-9082-435d-a410-69ff72d1f016.png)

각 영역의 역할은 다음과 같습니다.

|||
|:--|:--|
|영역|설명|
|사용자 인터페이스 또는 표현|사용자의 요청을 처리하고 사용자에게 정보를 보여준다. 여기서 사용자는<br/>소프트웨어를 사용하는 사람뿐만 아니라 외부 시스템일 수도 있다|
|응용|사용자가 요청한 기능을 실행한다. 업무 로직을 직접 구현하지 않으며 <br/>도메인 계층을 조합해서 기능을 실행한다 |
|도메인|시스템이 제공할 도메인 규칙을 구현한다|
|인프라스트럭처|데이터베이스나 메시징 시스템과 같은 외부 시스템과의 연동을 처리한다|

앞서 살펴본 도메인 모델이 도메인 자체를 이해하는데 필요한 개념 모델을 의미한다면, 지금 살펴볼 도메인 모델은 마틴 파울러가 쓴

[엔터프라이즈 애플리케이션 아키텍쳐 패턴] 책의 도메인 모델 패턴을 의미한다. 도메인 모델은 아키텍처 상의 도메인 계층을

객체 지향 기법으로 구현하는 패턴을 말한다. 도메인 계층은 도메인의 핵심 규칙을 구현한다. 주문 도메인의 경우 '출고 전에 배송지를 변경 할수 있다'

규칙과 '주문 취소는 배송 전에만 할 수 있다' 라는 규칙을 구현한 코드게 도메인 계층에 위치하게 된다. 

이런 도메인 규칙을 객체 지향 기법으로 구현하는 패턴이 `도메인 모델 패턴`이다. 예시 코드를 보자.

```java
public class Order{
    private OrderState state;
    private ShippingInfo shippingInfo;

    public void changeShippingInfo(ShippingInfo newShippingInfo){
        if(!state.isShippingChangeable()){
            throw new IllegalStateException("can't change shipping in " +state);
        }
        this.shippingInfo = newShippingInfo;
    }
    ...
}

public enum OrderState{
    PAYMENT_WAITING{
        public boolean isShippingChangeable(){
            return true;
        }
    },
    PREPARING{
        public boolean isShippingChangeable(){
            return true;
        }
    },
    SHIPPED, DELIVERING, DELIVERY_COMPLETED;

    public boolean isShippingChangeable(){
        return false;
    }
}
```

위의 코드는 주문 도메인의 일부 기능을 도메인 모델 패턴으로 구현한 것이다. 주문 상태를 표현하는 OrderState는

배송지를 변경할 수 있는지를 검사할 수 있는 isShippingChangeable()메소드를 제공하고 있다. 즉 OrderState는

주문 대기중이거나 상품 준비 중에는 배송지를 변경할 수 있다는 도메인 규칙을 구현하고 있다.

이제 이를 큰틀에서 보면 OrderState는 Order에 속한 데이터이므로 배송지 정보 변경 가능 여부를 판단하는 코드를 Order로

이동할 수도 있다. 다음은 Order클래스에서 판단하도록 수정한 코드이다.
```java
public class Order{
    private OrderState state;
    private ShippingInfo shippingInfo;

    public void changeShipingInfo(ShippingInfo newShippingInfo){
        if(!isShippingChangeable()){
            throw new IllegalStateException("can't change shipping in " + state);
        }
        this.shippingInfo = newShippingInfo;
    }

    private boolean isShippingChangeable(){
        return state == OrderState.PAYMENT_WAITING || 
            state == OrderState.PREPARING;
    }
    ...
}

public enum OrderState{
    PAYMENT_WAITING , PREPARING , SHIPPED , DELIVERING, DELIVERY_COMPLETED;
}
```

배송지 변경 가능 여부를 판단하는 기능이 Order에 있든 OrderState에 있든 중요한 점은 주문과 관련된 중요 업무 규칙을

주문 도메인 모델인 Order나 OrderState에서 구현한다는 점이다. 핵심 규칙을 구현한 코드는 도메인 모델에만 위치하기 

때문에 규칙이 바뀌거나 규칙을 확장해야 할 때 다른 코드에 영향을 덜주고 변경 내역을 모델에 반영할 수 있게 된다.

## 도메인 모델 추출

도메인을 모델링할 때 기본이 되는 작업은 모델을 구성하는 핵심 구성요소, 규칙, 기능을 찾는 것이다. 이과정은 요구사항에서 출발한다.

주문 도메인과 관련된 몇 가지 요구사항을 보자.
- 최소 한 종류 이상의 상품을 주문해야 한다
- 한 상품을 한 개 이상 주문할 수 있다
- 총 주문 금액은 각 상품의 구매 가격 합을 모두 더한 금액이다
- 각 상품의 구매 가격 합은 상품 가격에 구매 개수를 곱한 값이다
- 주문할 때 배송지 정보를 반드시 지정해야 한다
- 배송지 정보는 받는 사람 이름, 전화번호, 주소로 구성된다
- 출고를 하면 배송지를 변경할 수 없다
- 출고 전에 주문을 취소할 수 있다
- 고객이 결제를 완료하기 전에는 상품을 준비하지 않는다.

이 요구사항에서 알 수 있는 것은 주문은 '출고상태로 변경하기','배송지 정보 변경하기', '주문 취소하기', '결제 완료하기' 기능을 제공한다는 것이다.

아직 상세구현은 아니지만 메소드로 추가할 수 있다.

```java
public class Order{
    public void changeShipped(){...}
    public void changeShippingInfo(ShippingInfo newShipping){...}
    public void cancel(){...}
    public void completePayment(){...}
}
```

다음 요구사항은 주문 항목이 어떤 데이터로 구성되는지 알려준다
- 한 상품을 한 개 이상 주문할 수 있다
- 각 상품의 구매 가격 합은 상품 가격에 구매 개수를 곱한 값이다.

두 요구사항에 따른 OrderLine을 구현해보자.
```java
public class OrderLine{
    private Product product;
    private int price;
    private int quantity;
    private int amounts;

    public OrderLine(Product product, int price, int quantity){
        this.product = product;
        this.price = price;
        this.quantity = quantity;
        this.amounts = calculateAmounts();
    }

    private int calculateAmounts(){
        return price * quantity;
    }
    public int getAmounts(){...}
    ...
}
```

OrderLine은 한 상품을 얼마에 몇개 살지를 담고 있고 calculateAmounts()메소드로 가격을 구한다

다음 요구사항은 Order와 OrderLine과의 관계를 알려준다.
- 최소 한 종류 이상의 상품을 주문해야 한다
- 총 주문 금액은 각 상품의 구매 가격합을 모두 더한 금액이다

한 종류 이상의 상품을 주문할 수 있으므로 Order는 최소 한 개 이상의 OrderLine을 포함해야한다.

```java
public class Order{
    private List<OrderLine> orderLines;
    private Monty totalAmounts;

    public Order(List<OrderLine> orderLines){
        setOrderLines(orderLines);
    }

    private void setOrderLines(List<OrderLine> orderLines){
        verifyAtLeastOneOrMoreOrderLines(orderLines);
        this.orderLines = orderLines;
        calculateTotalAmounts();
    }

    private void verifyAtLeastOneOrMoreOrderLines(List<OrderLine> orderLines){
        if(orderLines == null || orderLines.isEmpty()){
            throw new IllegalArgumentException("no OrderLine");
        }
    }

    private void calculateTotalAmounts(){
        int sum = orderLiens.stream().mapToInt(x -> x.getAmounts()).sum();
        this.totalAmounts = new Money(sum);
    }
}
```

배송지 정보는 이름, 전화번호, 주소데이터를 가지므로 ShippingInfo 클래스를 다음과 같이 정의할 수 있다.
```java
public class ShippingInfo{
    private String receiverName;
    private String receiverPhoneNumber;
    private String shippingAddress1;
    private String shippingAddress2;
    private String shippingZipcode;

    // 생성자 , getter
}
```

앞서 요구사항에 '주문할 때 배송지 정보를 반드시 지정해야 한다'라는 내용이 있다 이는 Order를 생성할 때 

OrderLine의 목록뿐만 아니라 ShippingInfo도 함께 전달해야 함을 의미한다. 이를 생성자에 반영한다.

```java
public class Order{
    private List<OrderLine> orderLines;
    private ShippingInfo shippingInfo;

    ...

    public Order(List<OrderLine> orderLines, SHippingInfo shippingInfo){
        setOrderLines(orderLines);
        setShippingInfo(shippingInfo);
    }

    private void setShippingInfo(ShippingInfo shippingInfo){
        if(shippingInfo == null)
            throw new IllegalArgumentException("no SHippingInfo");
        this.shippingInfo = shippingInfo;
    }
    ...
}
```

생성자에서 호출하는 setShppingInfo() 메소드는 ShippingInfo가 null이면 익셉션이 발생하는데 이렇게 함으로써

'배송지 정보 필수' 라는 도메인 규칙을 구현한다.

도메인을 구현하다 보면 특정 조건이나 상태에 따라 제약이나 규칙이 달리 적용되는 경우가 많은데 다음요구사항이 그에 해당된다.

- 출고를 하면 배송지 정볼르 변경할 수 없다
- 출고 전에 주문을 취소할 수 있다

이 요구사항은 출고 상태가 되기 전과 후의 제약사항을 기술하고 있다. 출고 상태에 따라 배송지 정보 변경 기능과 

주문 취소 기능은 다른 제약을 갖는다. 이 요구사항을 충족하려면 주문은 최소한 출고 상태를 표현할 수 있어야한다.
- 고객이 결제를 완료하기 전에는 상품을 준비하지 않는다.

이 요구사항은 결제 완료 전을 의미하는 상태와 결제 완료 내지 상품 준비중이라는 상태가 필요함을 알려준다.

다른 요구사항을 확인해서 추가로 존재할 수 있는 상태를 분석한 뒤, 다음과 같이 열거 타입을 이용해서 상태정보를 표현할 수 있다.

```java
public enum OrderState{
    PAYMENT_WAITING, PREPARING, SHIPPED, DELIVERING, DELIVERY_COMPLETED, CANCELED
}
```
배송지 변경이나 주문 취소 기능은 출고 전에만 가능하다는 제약 규칙이 있으므로 이 규칙을 적용하기 위해 changeShippingInfo()

와 cancel() 은 verifyNotYetShipped() 메소드를 먼저 실행한다

```java
public class Order{
    private OrderState staet;

    public void changeShippingInfo(ShippingInfo newShippingInfo){
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
    }

    public void cancel(){
        verifyNotYetShipped();
        this.state = OrderState.CANCELED;
    }

    private void verifyNotYetShipped(){
        if(state != OrderState.PAYMENT_WAITING && state != OrderState.PREPARING)
            throw new IllegalStateException("already shipped")
    }
}
```

이런식으로 주문과 관련된 요구사항에서 도메인 모델을 점진적으로 만들어 나갔다. 보통 이렇게 만든 모델은 요구사항 정련을 위해

도메인 전문가나 다른 개발자와 논의하는 과정에서 공유하기도 한다.

## 엔티티와 밸류
도출한 모델은 크게 엔티티(Entity)와 밸류(Value)로 구분할 수 있다. 앞서 요구사항 분석 과정에서 만든 모델은

다음과 같은데 이그림에는 엔티티도 존재하고 밸류도 존재한다.

![image](https://user-images.githubusercontent.com/40031858/167866282-c78b63af-3155-40ae-88b7-c69a624abf8d.png)

### 엔티티
엔티티의 가장 큰 특징은 식별자를 가진다는 것이다. 식별자는 엔티티 객체마다 고유해서 각 엔티티는 서로 다른 식별자를 갖는다.

예를 들어 주문 도메인에서 각 주문은 주문번호를 가지고 있는데 이 주문번호는 각 주문마다 서로 다르다. 따라서 주문 번호가 주문의 식별자가 된다.

앞서 주문 도메인 모델에서 주문에 해당하는 클래스가 Order이므로 Order가 엔티티가 되며 주문 번호를 속성으로 갖는다.

![image](https://user-images.githubusercontent.com/40031858/167866667-30775409-29e6-4c5c-8b7e-1d5b29bb4548.png)

주문에서 배송지 주소가 바뀌거나 상태가 바뀌더라도 주문번호가 바뀌지 않는 것처럼 엔티티의 식별자는 바뀌지 않는다.

엔티티를 생성하고 속성을 바꾸고 삭제할 때까지 식별자는 유지된다. 엔티티의 식별자는 바뀌지 않고 고유하기 때문에

두 엔티티 객체의 식별자가 같으면 두 엔티티는 같다고 판단할 수 있다. 엔티티를 구현한 클래스는 다음과 같이 식별자를 이용해서

`equals()` 메소드와 `hashCode()` 메소드를 구현할 수 있다.

```java
public class Order{
    private String orderNumber;

    @Override
    public boolean equals(Object obj){
        if(this == obj) return true;
        if(obj == null) return false;
        if (obj.getClass() != Order.class) return false;
        Order other = (Order) obj;
        if(this.orderNumber ==null) return false;
        return this.orderNumber.equals(other.orderNumber);
    }

    @Override
    public int hashCode(){
        final int prime = 31;
        int result = 1;
        result = prime * result +((orderNumber == null) ? 0: orderNumber.hashCode());
        return result;
    }
}

```

### 엔티티의 식별자 생성

엔티티의 식별자를 생성하는 시점은 도메인의 특징과 사용하는 기술에 따라 달라진다. 흔히 식별자는 다음 중 한가지 방식으로 생성한다

- 특정 규칙에 따라 생성
- UUID나 Nano ID와 같은 고유 식별자 생성기 사용
- 값을 직접 입력
- 일련번호 사용

### 밸류 타입
ShippingInfo 클래스는 다음과 같이 받는 사람과 주소에 대한 데이터를 갖고있다.

```java
public class ShippingInfo{
    private String receiverName;        //받는사람
    private String receiverPhoneNumber;   // 받는사람

    private String shippingAddress1;    //주소
    private String shippingAddress2;    //주소
    private String shippingZipcode;     //주소

    ... 생성자, getter
}

```

ShippingInfo클래스의 reciverName필드와 receiverPhoneNumber 필드는 서로 다른 두 데이터를 담고 있지만 두 필드는 개념적으로

받는 사람을 의미한다. 즉 두 필드는 실제로 하나의 개념을 표현하고 있다. 비슷하게 shippingAddress1 필드, shippingAddress2필드, 

shippingZipcode 필드는 주소라는 하나의 개념을 표현한다. 밸류 타입은 개념적으로 완전히 하나를 표현할 때 사용한다.

예를 들어 받는 사람을 위한 밸류 타입인 Receiver를 다음과 같이 작성할 수 있다.

```java
public class Receiver{
    private String name;
    private String phoneNumber;

    public Receiver(String name, String phoneNumber){
        this.name = name;
        this.phoneNumber = phoneNumber;
    }

    public String getName(){
        return name;
    }

    public String getPhoneNumber(){
        return phoneNumber;
    }
}
```

Receiver는 `받는 사람`이라는 도메인 개념을 표현한다. 앞서 ShippingInfo의 receiverName 필드와 receiverPhoneNumber

필드가 이름을 갖고 받는 사람과 관련된 데이터라는 것을 유추한다면 Receiver는 그 자체로 받는 사람을 뜻한다. 밸류 타입을 사용함으로써

개념적으로 완전한 하나를 잘 표현할 수 있는 것이다. ShippingInfo의 주소 관련 데이터도 다음의 Address 밸류 타입을 사용해서 보다 명확하게 표현할 수 있다.

```java
public class Address{
    private String address1;
    private String address2;
    private String zipcode;

    public Address(String address1, String address2, String zipcode){
        this.address1 = address1;
        this.address2 = address2;
        this.zipcode = zipcode;
    }
}

```

이제 밸류 타입을 이요해서 ShippingInfo클래스를 다시 구현하면 배송정보가 받는 사람과 주소로 구성된다는 것을 쉽게 알 수 있다.

```java
public class ShippingInfo{
    private Receiver receiver;
    private Address address;
}

```

밸류 타입이 꼭 두 개 이상의 데이터를 가져야 하는것은 아니다. 의미를 명확히 표현하기 위해 밸류타입을 사용하는 경우도 있는데 다음과 같다.

```java
public class OrderLine{
    private Product product;
    private int price;
    private int quantity;
    private int amounts;
    ...
}
```
OrderLine의 price와 amounts는 int 타입의 숫자를 사용하고 있지만 이들은 '돈'을 의미하는 값이다.

따라서 '돈'을 의미하는 Money 타입을 만들어 사용하면 코드를 이해하는데 도움이 된다.

```java
public class Money{
    private int value;

    public Money(int value){
        this.money = money;
    }

    public int getValue(){
        return this.value
    }
}
```

다음은 Money를 사용하도록 OrderLine을 변경한 코드다

```java
public class OrderLine{
    private Product produdct;
    private Money price;
    private int quantity;
    private Money amounts;
    ...
}
```

밸류 타입의 또 다른 장점은 밸류 타입을 위한 기능을 추가할 수 있다는 것이다. 예를 들어 Money 타입은 다음과 같이 돈계산을 위한 기능을 추가할 수 있다.

```java
public class Money{
    private int value;

    ...생성자, getter

    public Money add(Money money){
        return new Money(this.value + money.value);
    }

    public Money multiply(int multiplier){
        return new Money(value * multiplier);
    }
}
```

밸류 객체의 데이터를 변경할때는 기존데이터를 변경하기보다는 위의 코드처럼 변경한 데이터를 갖는 새로운 밸류 객체를 생성하는 방식을 선호한다.

### 엔티티 식별자와 밸류 타입

엔티티 식별자의 실제 데이터는 String과 같은 문자열로 구성된 경우가 많다. Money가 단순 숫자가 아닌 도메인의 '돈'을 의미하는 것처럼

이런 식별자는 단순히 문자열이 아니라 도메인에서 특별한 의미를 지니는 경우가 많기에 식별자를 위한 밸류 타입을 사용해서 의미를 잘 드러나도록 할 수 있다.

예를 들어 주문번호를 표현하기 위해 Order의 식별자 타입으로 String대신 OrderNo 밸류 타입을 사용하면 타입을 통해 해당 필드가 주문번호라는 것을 알 수 있다.

```java
public class Order{
    //OrderNo 타입 자체로 id가 주문번호임을 알 수 있다.
    private OrderNo id;
    ...
    public OrderNo getId(){
        return id;
    }
}
```
OrderNo 대신에 Stringa 타입을 사용하면 'id'라는 이름만으로는 해당 필드가 주문번호인지를 알 수 없다.

필드의 의미가 드러나도록 하려면 'id'라는 필드 이름 대신 'orderNo'라는 필드 이름을 사용해야 한다.

반면에 식별자를 위해 OrderNo 타입을 만들면 타입 자체로 주문번호라는 것을 알 수 있으므로 필드 이름이 'id'여도

실제 의미를 찾는 것은 어렵지 않다.

### 도메인 모델에 set메소드 넣지 않기

get/set 메소드를 습관적으로 추가할 때가 있다. 사용자 정보를 담는 UserInfo 클래스를 작성할 때 다음과 같이 데이터 필드에

대한 get/set 메소드를 습관처럼 작성할 수 있다.

```java
@Getter @Setter
public class UserInfo{
    private String id;
    private String name;

    public UserInfo(){}
}

```

도메인 모델에 get/set 메소드를 무조건 추가하는 것은 좋지 않은 버릇이다. 특히 set 메소드는 도메인의 핵심 개념이나 의도를 코드에서 사라지게한다.

Order의 메소드를 다음과 같이 set메소드로 변경해보자.
```java
public class Order{
    ...
    public void setShippingInfo(SHippingInfo newShipping){..}
    public void setOrderState(OrderState state){...}
}
```

앞서 changeShippingInfo()가 배송지 정보를 새로 변경한다는 의미를 가졌다면 setShippingInfo() 메소드는 단순히 배송지 값을 설정한다는 것을 의미한다.

completePayment()는 결제를 완료했다는 반면에 setOrderState()는 단순히 주문상태값을 설정한다는 것을 의미한다.

즉 set메소드는 필드값만 변경하고 끝나기에 상태변경과 관련된 도메인 지식이 코드에서 사라지게 된다.

set메소드의 또 다른 문제는 도메인 객체를 생성할 때 온전하지 않은 상태가 될 수 있다. 
```java
Order order = new Order();

//set 메소드로 필요한 모든 값을 전달해야 함
order.setOrderLine(lines);
order.setShippingInfo(shippingInfo);

//주문자(Order)를 설정하지 않은 상태에서 주문 완료 처리
order.setState(OrderState.PREPARING);
```

위 코드는 주문자 설정을 누락하고 있다. 주문자 정보를 담고 있는 필드인 order가 null인 상황에서 order.setState()

메소드를 호출해서 상품 준비 중 상태로 바꾼것이다. orderer가 정상인지 확인하기 위해 orderer가 null인지

검사하는 코드를 setState() 메소드에 위치하는 것도 맞지 않다.

도메인 객체가 불완전한 상태로 사용되는 것을 막으려면 생성 시점에 필요한 것을 전달해 주어야 한다.

즉 생성자를 통해 필요한 데이터를 모두 받아야 한다.
```java
Order order = new Order(orderer, lines, shippingInfo, OrderState.PREPARING);
```

생성자로 필요한 것을 모두 받으므로 다음처럼 생성자를 호출하는 시점에 필요한 데이터가 올바른지 검사할 수 있다.

```java
public class Order{
    public Order(Orderer orderer, List<OrderLine> orderLines,
                ShippingInfo shippingInfo, OrderState state){
            setOrderer(orderer);
            setOrderLines(orderLines);
            // 다른 값 설정
        }
    private void setOrderer(Orderer orderer){
        if(orderer == null) throw new IllegalArgumentException("no orderer");
        this.orderer = orderer;
    }
    
    private void setOrderLines(List<OrderLine> orderLines){
        verifyAtLeastOneOrMoreOrderLines(orderLines);
        this.orderLines = orderLines;
        calculateTotalAmounts();
    }

    private void verifyAtLeastOneOrMoreOrderLines(List<OrderLine> orderLines){
        if(orderLines == null || orderLines.isEmpty()){
            throw new IllegalArgumentException("no OrderLine");
        }
    }

    private void calculateTotalAmounts(){
        this.totalAmounts = orderLines.stream().mapToInt(x -> x.getAmounts()).sum();
    }
}
```

이 코드의 set메소드는 앞서 언급한 set 메소드와 중요한 차이점이 있는데 그것은 바로 접근 범위가 private라는 점이다.

이 코드에서 set메소드는 클래스 내부에서 데이터를 변경할 목적으로 사용된다. private이기 때문에 외부엑서 데이터를 변경할

목적으로 set메소드를 사용할 수 없다.

불변 밸류 타입을 사용하면 자연스럽게 밸류 타입에는 set 메소드르 구현하지 않는다. set메소드를 구현해야 할 특별한

이유가 없다면 불변 타입의 장점을 살릴 수 있도록 밸류 타입은 불변으로 구현한다.