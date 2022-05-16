# Chap03. 애그리거트

## 애그리거트
온라인 쇼핑몰 시스템을 개발할때 다음과 같이 상위 수준 개념을 이용해서 전체 모델을 정리하면 전반적인 관곌르 이해하는데 도움이 된다.

![image](https://user-images.githubusercontent.com/40031858/168499076-ad5155e2-eeb1-46f4-81f1-8e7b0c002512.png)


상위 수준 모델을 개별 객체 단위로 그려보면 아래 그림과 같다. 

위 그림과 같이 상위 모델에 대한 이해 없이 아래 그림만 보고 상위 수준에서 개념을 파악하려면 더 오랜시간이 걸린다.

![image](https://user-images.githubusercontent.com/40031858/168499306-7899b9ef-81cc-444c-9e42-62e15e636b1f.png)

복잡한 도메인을 이해하고 관리하기 쉬운 단위로 만들려면 상위 수준에서 모델을 조망할 수 있는 방법이 필요한데, 그 방법이 `애그리거트`다.

`애그리거트`는 관련된 객체를 하나의 군으로 묶어준다. 수많은 객체를 애그리거트로 묶어서 바라보면 상위수준에서 도메인 모델간의 관계를 파악할 수 있다.

![image](https://user-images.githubusercontent.com/40031858/168499560-c0ac1d7b-02a3-42a2-b130-e97184d35376.png)

애그리거트는 모델을 이해하는 데 도움을 줄 뿐만 아니라 일관성을 관리하는 기준도 된다. 모델을 보다 잘 이해할 수 있고 애그리거트 단위로

일관성을 관리하기 때문에, 애그리거트는 복잡한 도메인을 단순한 구조로 만들어준다. 복잡도가 낮아지는 만큼 도메인 기능을 확장하고

변경하는데 필요한 노력(개발 시간)도 줄어든다. 위의 그림처럼 `애그리거트`는 경계를 갖는다. 한 `애그리거트`에 속한 객체는 다른 `애그리거트`에 속하지 않는다.

`애그리거트`는 독립된 객체 군이며 각 `애그리거트`는 자기 자신을 관리할 뿐 다른 `애그리거트`를 관리하지 않는다. 예를들어 주문 애그리거트는 배송지를 변경하거나

주문 상품 개수를 변경하는 등 자기 자신은 관리하지만, 주문 애그리거트에서 회원의 비밀번호를 변경하거나 상품의 가격을 변경하지는 않는다.

경계를 설정할 때 기본이 되는 것은 도메인 규칙과 요구사항이다. 도메인 규칙에 따라 함게 생성되는 구성요소는 한 애그리거트에 속할 가능성이 높다.

예를 들어 주문할 상품 개수, 배송지 정보, 주문자 정보는 주문 시점에 함께 생성되므로 이들은 한 애그리거트에 속한다. 또한 OrderLine의 

주문 상품 개수를 변경하면 도메인 규칙에 따라 Order의 총 주문 금액을 새로 계산해야 한다. 사용자 요구사항에 따라 주문 상품 개수와 배송지를 

함께 변경하기도 한다. 이렇게 함께 변경되는 빈도가 높은 객체는 한 애그리거트에 속할 가능성이 높다.

좋은 예가 상품과 리뷰다. 상품 상세 페이지에 들어가면 상품 상세 정보와 함께 리뷰 내용을 보여줘야 한다는 요구사항이 있을 때 Product엔티티와

Review 엔티티가 한 애그리거트에 속한다고 생각할 수 있다. 하지만 Product와 Review는 함께 생성되지 않고, 함께 변경되지도 않는다.

게다가 Product를 변경하는 주체가 상품 담당자라면 Review를 생성하고 변경하는 객체는 고객이다.

![image](https://user-images.githubusercontent.com/40031858/168499985-d683d935-bff8-4a00-b8be-409686ee65df.png)

Review의 변경이 Product에 영향을 주지 않고 반대로 Product의 변경이 Review에 영향을 주지 않기 때문에 이 둘은 한 애그리거트에 속하기 보다는

서로 다른 애그리거트에 속한다.

## 애그리거트 루트
주문 애그리거트는 다음을 포함한다.
- 총 금액인 totalAmounts를 갖고 있는 Order 엔티티
- 개별 구매 상품의 개수인 quantity와 금액인 price를 갖고 있는 OrderLine 밸류

구매할 상품의 개수를 변경하면 한 OrderLine의 quantity를 변경하고 더불어 Order의 totalAmounts도 변경해야한다.

그렇지 않으면 다음 도메인 규칙을 어기고 데이터 일관성이 깨진다
- 주문 총 금액은 개별 상품의 주문 개수 X 가격의 합이다.

애그리거트는 여러 객체로 구성되기 때문에 한 객체만 상태가 정상이면 안된다. 도메인 규칙을 지키려면 애그리거트에 속한 모든 객체가 

정상상태를 가져야한다. 애그리거트에 속한 모든 객체가 일관성 상태를 유지하려면 애그리거트 전체를 관리할 주체가 필요한데, 이 책임을

지는 것이 바로 애그리거트의 루트 엔티티이다. 애그리거트 루트 엔티티는 애그리거트의 대표 엔티티다.

애그리거트에 속한 객체는 애그리거트 루트 엔티티에 직접 또는 간접적으로 속하게된다.

### 도메인 규칙과 일관성

애그리거트 루트가 단순히 애그리거트에 속한 객체를 포함하는 것으로 끝나는 것은 아니다. 애그리거트 루트의 핵심 역할은 애그리거트의 일관성이

깨지지 않도록 하는 것이다. 이를 위해 애그리거트 루트는 애그리거트가 제공해야 할 도메인 기능을 구현한다. 예를 들면 주문 애그리거트는

배송지 변경, 상품 변경과 같은 기능을 제공하고, 애그리거트 루트인 Order가 이 기능을 구현한 메소드를 제공한다.

```java
public class Order{
    //애그리거트 루트는 도메인 규칙을 구현한 기능을 제공한다.
    public void changeShippingInfo(ShippingInfo newShippingInfo){
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
    }

    private void verifyNotYetShipped(){
        if(state!= OrderState.PAYMENT_WAITING && state != OrderState.PREPARING)
            throw new IllegalArgumentException("alread shipped");
    }
}

```

애그리거트 외부에서 애그리거트에 속한 객체를 직접 변경하면 안된다. 이것은 애그리거트 루트가 강제하는 규칙을 적용할 수 없어 

모델의 일관성을 깨는 원인이 된다.
```java
ShippingInfo si = order.getShippingInfo();
si.setAddress(newAddress);
```

이 코드는 애그리거트 루트인 Order에서 ShippingInfo를 가져와 직접 정보를 변경하고 있다. 주문 상태에 상관없이 배송지 주소를 변경하는데,

이는 업무 규칙을 무시하고 직접 DB테이블의 데이터를 수정하는 것과 같은 결과를 만든다. 즉 논리적인 데이터 일관성이 깨지게 되는 것이다.

일관성을 지키기 위해 다음과 같이 상태 확인 로직을 응용 서비스에 구현할 수도 있다. 하지만 이렇게 되면 동일한 검사 로직을 여러 응용 서비스에서

중복으로 구현할 가능성이 높아져 유지보수에 도움이 되지 않는다.
```java
ShippingInfo si = order.getShippingInfo();

//주요 도메인 로직이 중복되는 문제
if(state!= OrderState.PAYMENT_WAITING && state != OrderState.PREPARING){
    throw new IllegalArgumentException();
}
si.setAddress(newAddress);
```

불필요한 중복을 피하고 애그리거트 루트를 통해서만 도메인 로직을 구현하게 만들려면 도메인 모델에 대해 다음의 두 가지를 습관적으로 적용해야 한다.
- 단순히 필드를 변경하는 set메서드를 공개(public) 범위로 만들지 않는다
- 밸류 타입은 불변으로 구현한다

먼저 습관적으로 공개set메서드를 피해야한다. 보통 공개 set메서드는 다음과 같이 필드에 값을 할당하는 것으로 끝나는 경우가 많다.

```java
//도메인 모델에서 공개 set 메서드는 가급적 피해야 한다
public void setName(String name){
    this.name = name;
}
```

공개 set 메서드는 도메인의 의미나 의도를 표현하지 못하고 도메인 로직을 도메인 객체가 아닌 응용 영역이나 표현 영역으로 분산시킨다. 도메인 로직이

한 곳에 응집되지 않으므로 코드를 유지보수할 때에도 분석하고 수정하는데 더 많은 시간이 필요하다

도메인 모델의 엔티티나 밸류에 공개 set 메서드만 넣지 않아도 일관성이 깨질 가능성이 줄어든다.

공개 set 메소드를 사용하지 않으면 의미가 드러나는 메서드를 사용해서 구현할 가능성이 높아진다. 공개 set 메서드를 만들지 않는 것의 연장으로 

밸류는 불변타입으로 구현한다. 밸류 객체의 값을 변경할 수 없으면 애그리거트 루트에서 밸류 객체를 구해도 애그리거트 외부에서 밸류 객체의 상태를 변경할 수 없다.

```java
ShippingInfo si = order.getShippingInfo();
si.setAddress(newAddress); // ShippingInfo가 불변이면, 이 코드는 컴파일 에러!
```

애그리거트 외부에서 내부 상태를 함부로 바꾸지 못하므로 애그리거트의 일관성이 깨질 가능성이 줄어든다. 밸류 객체가 불변이면 밸류 객체의 값을 변경하는 방법은

새로운 밸류 객체를 할당하는 것뿐이다. 즉 다음과 같이 애그리거트 루트가 제공하는 메소드에 새로운 밸류 객체를 전달해서 값을 변경하는 방법밖에 없다.

```java
public class Order{
    private ShippingInfo shippingInfo;

    public void changeShippingInfo(ShippingInfo newShippingInfo){
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
    }

    //set 메서드의 접근 허용 범위는 private이다.
    private void setShippingInfo(ShippingInfo newShippingInfo){
        //밸류가 불변이면 새로운 객체를 할당해서 값을 변경해야 한다
        //불변이므로 this.shippingInfo.setAddress(newShippingInfo.getAddress())와 같은
        //코드를 사용할 수 없다.
        this.shippingInfo = newShippingInfo;
    }
}
```

밸류 타입의 내부 상태를 변경하려면 애그리거트 루트를 통해서만 가능하다. 그러므로 애그리거트 루트가 도메인 규칙을 올바르게만 구현하면 애그리거트 저체의 일관성을 유지할수있다.

### 애그리거트 루트의 기능 구현
애그리거트 루트는 애그리거트 내부의 다른 객체를 조합해서 기능을 완성한다. 예를 들어 Order는 총 주문 금액을 구하기 위해 OrderLine목록을 사용한다.
```java
public class Order{
    private Money totalAmounts;
    private List<OrderLine> orderLines;

    private void calculateTotalAmounts(){
        int sum = orderLines.stream()
                    .mapToInt(ol -> ol.getPrice() * ol.getQuantity()).sum();
        this.totalAmounts = new Money(sum);
    }
}

```

또 다른 예로 회원을 표현하는 Member 애그리거트 루트는 암호를 변경하기 위해 Password객체에 암호가 일치하는지를 확인할 것이다.
```java
public class Member{
    private Password password;

    public void changePassword(String currentPassword, String newPassword){
        if(!password.match(currentPassword)){
            throw new PasswordNotMatchException();
        }
        this.password = new Password(newPassword);
    }
}

```

애그리거트 루트가 구성요소의 상태만 참조하는 것은 아니다. 기능 실행을 위임하기도 한다. 예를 들어 구현 기술의 제약이나 내부 모델링 규칙

때문에 OrderLine 목록을 별도 클래스로 분리했다고 해보자.

```java
public class OrderLines{
    private List<OrderLine> lines;

    public Money getTotalAmounts(){...구현}

    public void changeOrderLines(List<OrderLine> newLines){
        this.lines = newLines;
    }
}

```
이 경우 Order의 changeOrderLines() 메소드는 다음과 같이 내부의 orderLines필드에 상태 변경을 위임하는 방식으로 기능을 구현한다.

```java
public class Order{
    private OrderLines orderLines;

    public void changeOrderLines(List<OrderLine> newLines){
        orderLines.changeOrderLines(newLines);
        this.totalAmounts = orderLines.getTotalAmounts();
    }
}
```

OrderLines는 changeOrderLines()와 getTotalAmounts() 같은 기능을 제공하고 있다. 만약 Order가 getOrderLines()와 같이 OrderLines를

구할 수 있는 메서드를 제공하면 애그리거트 외부에서 OrderLines의 기능을 실행할 수 있게 된다.

```java
OrderLines lines = order.getOrderLines();

// 외부에서 애그리거트 내부 상태 변경!
// order의 totalAmounts가 값이 OrderLines가 일치하지 않게 됨
lines.changeOrderLines(newOrderLines);
```

이 코드는 주문의 OrderLine 목록이 바꾸니ㅡㄴ데 총합은 계산하지 않는 버그를 만든다. 이런 버그가 생기지 않도록 하려면 애초에 애그리거트 외부에서

OrderLine목록을 변경할 수 없도록 OrderLines를 불변으로 구현하면 된다.

### 트랜잭션 범위

트랜잭션 범위는 작을수록 좋다. 한 트랜잭션이 한 개 테이블을 수정하는 것과 세개의 테이블을 수정하는 것을 비교하면 성능에서 차이가 발생한다.

한 개 테이블을 수정하면 트랜잭션 충돌을 막기위해 잠그는 대상이 한 개 테이블의 한 행으로 한정되지만, 세 개의 테이블을 수정하면 잠금 대상이 더 많아진다.

`잠금 대상이 많아진다는 것은 그만큼 동시에 처리할 수 있는 트랜잭션 개수가 줄어든다는 것을 의미하고 이것은 전체적인 성능을 떨어트린다.`

동일하게 한 트랜잭션에서는 한 개의 애그리거트만 수정해야 한다. 한 트랜잭션에서 두 개 이상의 애그리거트를 수정하면 트랜잭션 충돌이 발생할 가능성이

더 높아지기 때문에 한번에 수정하는 애그리거트 개수가 많아질수록 전체 처리량이 떨어지게 된다.

한 트랜잭션에서 한 애그리거트만 수정한다는 것은 애그리거트에서 다른 애그리거트를 변경하지 않는다는 것을 의미한다. 한 애그리거트에서 다른 애그리거트를

수정하면 결과적으로 두 개의 애그리거트를 한 트랜잭션에서 수정하게 되므로, 애그리거트 내부에서 다른 애그리거트의 상태를 변경하는 기능을 실행하면 안된다.

예를 들어 배송지 정보를 변경하면서 동시에 배송지 정보를 회원의 주소로 설정하는 기능이 있다고 해보자. 이때 주문 애그리거트는 다음과 같이

회원 애그리거트의 정보를 변경하면 안 된다.

```java
public class Order{
    private Orderer orderer;

    public void ShipTo(ShippingInfo newShippingInfo, boolean useNewShippingAddrAsMemberAddr){
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);
        if(useNewShippingAddrAsMemberAddr){
            //다른 애그리거트의 상태를 변경하면 안 됨!
            orderer.getMember().changeAddress(newShippingInfo.getAddress());
        }
    }
    ...
}

```

이것은 애그리거트가 자신의 책임 범위를 넘어 다른 애그리거트의 상태까지 관리하는 꼴이 된다. 애그리거트는 최대한 서로 독립적이어야 하는데 한 애그리거트가

다른 애그리거트의 기능에 의존하기 시작하면 애그리거트 간 결합도가 높아진다. 결합도가 높아지면 높아질수록 향후 수정 비용이 증가하므로 

애그리거트에서 다른 애그리거트의 상태를 변경하지 말아야한다. 만약 부득이하게 한 트랜잭션으로 두 개이상의 애그리거트를 수정해야 한다면 애그리거트에서

다른 애그리거트를 직접 수정하지 말고 응용 서비스에서 두 애그리거트를 수정하도록 구현한다.

```java
public class ChangeOrderService{
    // 두 개 이상의 애그리거트를 변경해야 하면,
    // 응용 서비스에서 각 애그리거트의 상태를 변경한다.
    @Transactional
    public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo,
            boolean useNewShippingAddrAsMemberAddr){
        
        Order order = orderRepository.findbyId(id);
        if(order == null) throw new OrderNotFoundException();
        order.shipTo(newShippingInfo);
        if(useNewshippingAddrAsMemberAddr){
            Member member = findMember(order.getOrderer());
            member.changeAddress(newShippingInfo.getAddress());
        }
    }
}

```

도메인 이벤트를 사용하면 한 트랜잭션에서 한 개의 애그리거트를 수정하면서도 동기나 비동기로 다른 애그리거트의 상태를 변경하는 코드를 작성할 수 있다.

이내용은 10장에서 살펴보자.

한 트랜잭션에서 한개의 애그리거트를 변경하는 것을 권장하지만, 다음 경우에는 한 트랜잭션에서 두 개 이상의 애그리거트를 변경하는 것을 고려할 수 있다.
- 팀표준: 팀이나 조직의 표준에 따라 사용자 유스케이스와 관련된 응용 서비스의 기능을 한 트랜잭션으로 실행해야 하는 경우가 있다.
- 기술 제약: 기술적으로 이벤트 방식을 도입할 수 없는 경우 한 트랜잭션에서 다수의 애그리거트를 수정해서 일관성을 처리해야한다
- UI 구현의 편리: 운영자의 편리함을 위해 주문 목록화면에서 여러 주문의 상태를 한번에 변경하고 싶을것이다. 이경우 한트랜잭션에서 여러 주문 애그리거트의 상태를 변경해야한다.

## 리포지토리와 애그리거트

`애그리거트는 개념상 완전한 한개의 도메인 모델을 표현하므로 객체의 영속성을 처리하는 리포지토리는 애그리거트 단위로 존재한다.`

Order와 OrderLine을 물리적으로 각각 별도의 DB테이블에 저장한다고 해서 Order와 OrderLine을 위한 리포지토리를 각각 만들지 않는다.

Order가 애그리거트 루트고 OrderLine은 애그리거트에 속하는 구성요소이므로 Order를 위한 리포지토리만 존재한다.

새로운 애그리거트를 만들면 저장소에 애그리거트를 영속화하고 애그리거트를 사용하려면 저장소에서 애그리거트를 읽어야 하므로 리포지토리는 보통

다음의 두 메서드를 기본으로 제공한다
- save: 애그리거트 저장
- findById: ID로 애그리거트를 구함

이 두메서드 외에 필요에 따라 다양한 조건으로 애그리거트를 검색하는 메서드나 애그리거트를 삭제하는 메서드를 추가할 수 있다.

애그리거트는 개념적으로 하나이므로 리포지터리는 애그리거트 전체를 저장소에 영속화해야한다. 예를 들어 Order 애그리거트와 관련된 테이블이

세 개라면 Order애그리거트를 저장할 때 애그리거트 루트와 매핑되는 테이블뿐만 아니라 애그리거트에 속한 모든 구성요소에 매핑된 테이블에

데이터를 저장해야 한다.

```java
//리포지터리에 애그리거트를 저장하면 애그리거트 전체를 영속화해야한다.
orderRepository.save(order)
```

동일하게 애그리거트를 구하는 리포지터리 메서드는 완전한 애그리거트를 제공해야 한다. 즉 다음 코드를 실행하면 order애그리거트는 OrderLine,Orderer

등 모든 구성요소를 포함하고 있어야 한다.
```java
// 리포지터리는 완전한 order를 제공해야 한다.
Order order = orderRepository.findById(orderId);

// order가 온전한 애그리거트가 아니면
// 기능 실행 도중 NullPointerException과 같은 문제가 발생한다.
order.cancel();
```

리포지터리가 완전한 애그리거트를 제공하지 않으면 필드나 값이 올바르지 않아 애그리거트의 기능을 실행하는 도중에 NullPointerException과 같은 문제가 발생할 수 있다.

## ID를 이용한 애그리거트 참조

한 객체가 다른 객체를 참조하는 것처럼 애그리거트도 다른 애그리거트를 참조한다. 애그리거트 관리 주체는 애그리거트 루트이므로 애그리거트에서 다른 애그리거트를 참조한다는

것은 다른 애그리거트의 루트를 참조한다는 것과 같다. 애그리거트 간의 참조는 필드를 통해 쉽게 구현할 수 있다. 예를 들어 주문 애그리거트에 속해있는

Orderer는 주문한 회원을 참조하기 위해 회원 애그리거트 루트인 Member필드로 참조할 수 있다.

![image](https://user-images.githubusercontent.com/40031858/168503003-0056b218-4f61-4312-935a-a2f4bffa4f0b.png)

필드를 이용해서 다른 애그리거트를 직접 참조하는 것은 개발자에게 구현의 편리함을 제공한다. 예를 들어 주문 정보 조회 화면에서

회원 ID를 이용해 링크를 제공해야 할 경우 다음과 같이 Order로부터 시작해서 회원 ID를 구할 수 있다.
```java
order.getOrderer().getMember().getId();
```

JPA는 @ManyToOne, @OneToOne과 같은 애노테이션을 이용해 연관된 객체를 로딩하는 기능을 제공하고 있으므로 필드를 이용해 다른 애그리거트를 쉽게 참조할 수 있다.

ORM기술 덕에 애그리거트 루트에 대한 참조를 쉽게 구현할 수 있고 필드또는 getter를 이용한 애그리거트 참조를 사용하면 다른 애그리거트의 데이터를

쉽게 조회할 수 있다. 하지만 필드를 이용한 애그리거트 참조는 다음문제를 야기할 수 있다.
- 편한 탐색 오용
- 성능에 대한 고민
- 확장 어려움

애그리거트를 직접 참조할 때 발생할 수 잇는 가장 큰문제는 편리함을 오용할 수 있다는 것이다. 한 애그리거트 내부에서 다른 애그리거트 객체에 접근할 수 있으면

다른 애그리거트의 상태를 쉽게 변경할 수 있게 된다. 트랜잭션 범위에서 언급한 것처럼 한 애그리거트가 관리하는 범위는 자기 자신으로 한정해야 한다.

그런데 애그리거트 내부에서 다른 애그리거트 객체에 접근할 수 있으면 다음 코드처럼 구현의 편리함 때문에 다른 애그리거트를 수정하고자 하는 유혹에 빠지기 쉽다.

```java
public class Order{
    private Orderer orderer;

    public void changeShippingInfo(ShippingInfo newShippingInfo,
            boolean useNewShippingAddrAsMemberAddr){
                ...
        
            if(useNewShippingAddrAsMemberAddr){
                // 한 애그리거트 내부에서 다른 애그리거트에 접근할 수 있으면,
                // 구현이 쉬워진다는 것 때문에 다른 애그리거트의 상태를 변경하는
                // 유혹에 빠지기 쉽다.
                orderer.getMember().changeAddress(newShippingInfo.getAddress());
            }
        }
        ....
}
```

한 애그리거트에서 다른 애그리거트의 상태를 변경하는 것은 애그리거트 간의 의존결합도를 높여서 결과적으로 애그리거트의 변경을 어렵게 만든다.

두 번째 문제는 애그리거트를 직접 참조하면 성능과 관련된 여러가지 고민을 해야한다는 것이다. JPA를 사용하면 참조한 객체를 지연로딩과 즉시로딩의 두가지

방식으로 로딩할 수 있다.

세번째 문제는 확장이다. 초기에는 단일 서버에 단일 DBMS로 서비스를 제공하는 것이 가능하다. 문제는 사용자가 몰리기 시작하면서 발생한다.

사용자가 늘고 트래픽이 증가하면 자연스럽게 부하를 분산하기 위해 하위 도메인별로 시스템을 분리하기 시작한다.

이과정에서 하위 도메인마다 서로 다른 DBMS를 사용할 때도 있다. 심지어 하위 도메인마다 다른종류의 데이터 저장소를 사용하기도 한다. 

이런 세가지 문제를 완화할때 사용할 수 있느 것이 ID를 이용해서 다른 애그리거트를 참조하는 것이다.

DB테이블에서 외래키로 참조하는 것과 비슷하게 ID를 이용한 참조는 다른 애그리거트를 참조할때 ID를 사용한다.

![image](https://user-images.githubusercontent.com/40031858/168503932-7632932c-cccc-45f9-b3f6-cc8951c5d932.png)

ID참조를 사용하면 모든 객체가 참조로 연결되지 않고 한 애그리거트에 속한 객체들만 참조로 연결된다. 이는 애그리거트의 경계를 명확히 하고 애그리거트간 물리적인

연결을 제거하기때문에 모델의 복잡도를 낮춰준다. 또한 애그리거트 간의 의존을 제거하므로 응집도를 높여주는 효과도 있다.

구현 복잡도도 낮아진다. 다른 애그리거트를 직접 참조하지 않으므로 애그리거트간 참조를 지연로딩으로 할지 즉시 로딩으로 할지 고민하지 않아도 된다.

참조하는 애그리거트가 필요하면 응용서비스에서 ID를 이용해서 로딩하면 된다.

```java
public class ChangeOrderService{
    
    @Transactional
    public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo,
                boolean useNewShippingAddrAsMemberAddr){

            Order order = orderRepository.findbyId(id);
            if(order == null) throw new OrderNotFoundException();
            order.changeShippingInfo(newShippingInfo);
            if(useNewShippingAddrAsMemberAddr){
                //ID를 이용해서 참조하는 애그리거트를 구한다.
                Member member = memberRepository.findById(
                    Order.getOrderer().getMemberId());
                
                member.changeAddress(newShippingInfo.getAddress());
            }

        }
}
```

응용서비스에서 필요한 애그리거트를 로딩하므로 애그리거트 수준에서 지연로딩을 하는것과 동일한 결과를 만든다.

ID를 이용한 참조 방식을 사용하면 복잡도를 낮추는 것과 함께 한 애그리거트에서 다른 애그리거트를 수정하는 문제를 근원적으로 방지할 수 있다.

외부 애그리거트를 직접 참조하지 않기 때문에 애초에 한 애그리거트에서 다른 애그리거트의 상태를 변경할 수 없는 것이다.

애그리거트별로 다른 구현 기술을 사용하는 것도 가능해진다. 중요한 데이터인 주문 애그리거트는 RDBMS에 저장하고 조회 성능이 중요한 상품 애그리거트는

NoSQL에 저장할 수 있다. 또한 각도메인을 별도 프로세스로 서비스하도록 구현할 수도 있다.

### ID를 이용한 참조와 조회성능

다른 애그리거트를 ID로 참조하면 참조하는 여러 애그리거트를 읽을 때 조회 속도가 문제될 수 있다. 예를 들어 주문 목록을 보여주려면 상품 애그리거트와 회원

애그리거트를 함께 읽어야 하는데, 이를 처리할 때 다음과 같이 각 주문마다 상품과 회원 애그리거트를 읽어온다고 해보자.

한 DBMS에 데이터가 있다면 조인을 이용해서 한 번에 모든 데이터를 가져올 수 있음에도 불구하고 주문마다 상품 정보를 읽어오는 쿼리를 실행하게 된다.

```java
Member member = memberRepository.findById(ordererId);
List<Order> orders = orderRepository.findByOrderer(ordererId);
List<OrderView> dtos = orders.stream()
                .map(order -> {
                    ProductId prodId = order.getOrderLines().get(0).getProductId();
                    //각 주문마다 첫 번째 주문 상품 정보 로딩 위한 쿼리 실행
                    Product product = productRepository.findById(prodId);
                    return new OrderView(order,member,product);
                }).collect(toList());
```

위 코드는 주문 개수가 10개면 주문을 읽어오기 위한 1번의 쿼리와 주문 별로 각 상품을 읽어오기 위한 10번의 쿼리를 실행한다.

`조회 대상이 N개일때 N개를 읽어오는 한번의 쿼리와 연관된 데이터를 읽어오는 쿼리를 N번 실행한다` 해서 이를 `N+1`조회 문제라고 부른다.

ID를 이용한 애그리거트 참조는 지연로딩과 같은 효과를 만드는데 지연 로딩과 관련된 대표적인 문제가 N+1 조회 문제이다.

ID참조 방식을 사용하면서 N+1조회와 같은 문제가 발생하지 않도록 하려면 조회 전용 쿼리를 사용하면 된다. 예를 들어 데이터 조회를 위한

별도 DAO를 만들고 DAO의 조회 메서드에서 조인을 이용해 한번의 쿼리로 필요한 데이터를 로딩하면 된다.

```java
@Repository
public class jpaOrderViewDao implements OrderViewDao{
    
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<OrderView> selectByOrderer(String ordererId){
        String selectQuery = 
                "select new com.myshop.order.application.dto.OrderView(o,m,p) "+
                "from Order o join o.orderLines ol, Member m, Product p " +
                "where o.orderer.memberId.id = :orderId "+
                "and o.orderer.memberId = m.id "+
                "and index(ol) = 0 " +
                "and ol.productId = p.id "+
                "order by o.number.number desc";
        TypedQuery<OrderView> query = 
            em.createQuery(selectQuery, OrderView.class);
        query.setParameter("ordererId",ordererId);
        return query.getResultList();
    }
}
```

이런식 이외에도 애그리거트마다 서로 다른 저장소를 사용하면 한 번의 쿼리로 관련 애그리거틀르 조회할 수 없다. 이때는 조회 성능을 높이기 위해 캐시를 적용하거나

조회 전용 저장소를 따로 구성한다. 이 방법은 코드가 복잡해지는 단점이 있지만 시스템의 처리량을 높일 수 있다는 장점이 있다. 특히 한대의 DB장비로 대응할 수 없는

수준의 트래픽이 발생하는 경우 캐시나 조회 전용 저장소는 필수로 선택해야 하는 기법이다.

## 애그리거트 간 집합 연관
카테고리 입장에서 한 카테고리에 한 개 이상의 상품이 속할 수 있으니 카테고리와 상품은 1-N관계이다.

애그리거트간 1-N관계는 Set과 같은 컬렉션을 이용해서 표현할 수 있다. 예를 들어 다음 코드처럼 Category가 연관된 Product를 값으로 갖는 컬렉션을 필드로 정의할 수 있다.

```java
public class Category{
    private Set<Product> products; //다른 애그리거트에 대한 1-N연관
}

```

그런데 개념적으로 존재하는 애그리거트 간의 1-N 연관을 실제 구현에 반영하는 것이 요구사항을 충족하는 것과는 상관없을 때가 있다. 특정 카테고리에 속한

상품목록을 보여주는 요구사항을 생각해보자. 보통 목록 관련 요구사항은 한 번에 전체 상품을 보여주기보다는 페이징을 이용해 제품을 나눠서 보여준다.

이기능을 카테고리 입장에서 1-N 연관을 이용해 구현하면 다음과 같은 방식으로 코드를 작성해야 한다.

```java
public class Category{
    private Set<Product> products;

    public List<Product> getProducts(int page, int size){
        List<Product> sortedProducts = sortById(products);
        return sortedProducts.subList((page-1) * size, page * size);
    }
    ...
}
```
이 코드를 실제 DBMS와 연동해서 구현하면 Category에 속한 모든 Product를 조회하게 된다. Product개수가 수만 개 정도로 많다면 이 코드를 실행할 때마다

실행 속도가 급격히 느려져 성능에 심각한 문제를 일으킬 것이다. 개념적으로는 애그리거트 간에 1-N 연관이 있더라도 이런 성능 문제 때문에 애그리거트 간의

1-N 연관을 실제 구현에 반영하지 않는다. 카테고리에 속한 상품을 구할 필요가 있다면 상품 입장에서 자신이 속한 카테고리를 N-1로 연관지어 구하면 된다.

이를 구현 모델에 반영하면 Product에 다음과 같이 Category로의 연관을 추가하고 그 연관을 이용해서 특정 Category에 속한 Product 목록을 구하면 된다.

```java
public class Product{
    ...
    private CategoryId categoryId;
    ...
}
```

카테고리에 속한 상품목록을 제공하는 응용 서비슨느 다음과 같이 ProductRepository를 이용해서 categoryId가 지정한 카테고리 식별자인 Product 목록을 구한다.

```java
public class ProductLsitService{
    public Page<Product> getProductOfCategory(Long categoryId, int page, int size){
        Category category = categoryRepository.findById(categoryId);
        checkCategory(category);
        List<Product> products = 
            productRepository.findByCategoryId(category.getId(),page,size);
        int  totalCount = productRepository.countsByCategoryId(category.getId());
        return new Page(page, size, totalCount, products);
    }
}
```

M-N연관은 개념적으로 양쪽 애그리거트에 컬렉션으로 연관을 만든다. 상품이 여러 카테고리에 속할 수 있다고 가정하면 카테고리와 상품은 M-N연관을 맺는다.

앞서 1-N 연관처럼 M-N 연관도 실제 요구사항을 고려하여 M-N 연관을 구현에 포함시킬지를 결정해야 한다.

보통 특정 카테고리에 속한 상품 목록을 보여줄 때 목록 화면에서 각 상품이 속한 모든 카테고리를 상품 정보에 표시하지는 않는다. 제품이 속한 모든 카테고리가

필요한 화면은 상품 상세 화면이다. 이러한 요구사항을 고려할 때 카테고리에서 상품으로의 집합 연관은 필요하지 않다. 다음과 같이 상품에서 

카테고리로의 집합 연관만 존재하면 된다. 즉 개념적으로는 상품과 카테고리의 양방향 M-N 연관이 존재하지만 실제 구현에서는 상품에서 카테고리로의 단방향 M-N 연관만 적용하면 되는 것이다.

```java
public class Product{

    private Set<CategoryId> categoryIds;
    ...
}
```

RDBMS를 이용해서 M-N 연관을 구현하려면 조인 테이블을 사용한다. 상품과 카테고리의 M-N연관은 다음과 같이 연관을 위한 조인 테이블을 이용해서 구현한다.

![image](https://user-images.githubusercontent.com/40031858/168506711-c0026a50-d794-40d1-8beb-16b561642200.png)

JPA를 이용하면 다음과 같은 매핑 설정을 사용해서 ID 참조를 이용한 M-N 단방향 연관을 구현할 수 있다.
```java
@Entity
@Table(name = "product")
public class Product{
    @EmbeddedId
    private ProductId id;

    @ElementCollection
    @CollectionTable(name = "product_category",
            joinColumns= @JoinColumn(name="product_id"))
    private Set<CategoryId> categoryIds;
}
```

이 매핑은 카테고리 ID목록을 보관하기 위해 밸류 타입에 대한 컬렉션 매핑을 이용했다. 이 매핑을 사용하면 다음과 같이 JPQL의 member of 연산자를

이용해서 특정 Category에 속한 Product 목록을 구하는 기능을 구현할 수 있다.
```java
@Repository
public class JpaProductRepository implements ProductRepository{
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<Product> findByCategoryId(CategoryId catId, int page, int size){
        TypedQuery<Product> query = entityManager.createQuery(
            "select p from Product p "+
            "where :catId member of p.categoryIds order by p.id.id desc",
            Product.class);
        query.setParameter("catId",catId);
        query.setFirstResult((page-1) * size);
        query.setMaxResults(size);
        return query.getResultList();
    }
}
```

이 코드에서 ':catId member of p.categoryIds'는 categoryIds 컬렉션에 catId로 지정한 값이 존재하는지를 검사하기 위한 검색조건이다.

응용 서비스는 이기능을 사용해서 지정한 카테고리에 속한 Product 목록을 구할 수 있다.

## 애그리거트를 팩토리로 사용하기

고객이 특정 상점을 여러 차례 신고해서 해당 상점이 더 이상 물건을 등록하지 못하도록 차단한 상태라고 해보자. 상품 등록 기능을 구현한 응용 서비스는

다음과 같이 상점 계정이 차단 상태가 아닌 경우에만 상품을 생성하도록 구현할 수 있을 것이다.

```java
public class RegisterProductService{
    public ProductId registerNewProduct(NewProductRequest req){
        Store store = storeRepository.findById(req.getStoreId());
        checkNull(store);
        if(account.isBlocked()){
            throw new StoreBolckedException();
        }
        ProductId id = productRepository.nextId()
        Product product = new Product(id, store.getId(), ...생략);
        productRepository.save(product);
        return id;
    }
}

```

이 코드는 Product를 생성 가능한지 판단하는 코드와 Product를 생성하는 코드가 분리되어있다. 코드가 나빠 보이지는 않지만 중요한 도메인 로직 처리가 응용

서비스에 노출되었다. Store가 Product를 생성할 수 있는지를 판단하고 Product를 생성하는 것은 논리적으로 하나의 도메인 기능인데 이 도메인 기능을 

응용서비스에서 구현하고 있는 것이다. 이 도메인 기능을 넣기 위한 별도의 도메인 서비스나 팩토리 클래슬르 만들 수도 있지만 이 기능을 Store 애그리거트에 구현할 수도 있다.

Product를 생성하는 기능을 Store애그리거트에 옮겨보자
```java
public class Store{
    public Product createProduct(ProductId newProductId, ...생략){
        if(isBlocked()) throw new StoreBolckedException();
        return new Product(newProductId, getId(), ...생략);
    }
}
```

Store 애그리거트의 createproduct()는 Product애그리거트를 생성하는 팩토리 역할을 한다. 팩토리 역할을 하면서도 중요한 도메인 로직을 구현하고 있다.

팩토리 기능을 구현했으므로 이제 응용서비스는 팩토리 기능을 이용해서 Product를 생성하면 된다.

```java
public class RegisterProductService{
    public ProductId registerNewproduct(NewProductRequest req){
        Store store = storeRepository.findById(req.getStoreId());
        checkNull(store);
        ProductId id = productRepository.nextId();
        Product product = store.createProducT(id,...생략);
        productRepository.save(product);
        return id;
    }
}
```

앞선 코드와 차이점이라면 응용 서비스에서 더 이상 Store의 상태를 확인하지 않는다는 것이다. Store가 Product를 생성할 수 있는지를 확인하는 도메인 로직은

Store에서 구현하고 있다. 이제 Product 생성 가능 여부를 확인하는 도메인 로직을 변경해도 도메인 영역의 Store만 변경하면 되고 응용 서비스는 영향을 받지않는다.

`도메인의 응집도도 높아졌다. 이것이 바로 애그리거트를 팩토리로 사용할때 얻을 수 있는 장점이다.`

애그리거트가 갖고 있는 데이터를 이용해서 다른 애그리거트를 생성해야 한다면 애그리거트에 팩토리 메소드를 구현하는 것을 고려해보자. Product의 경우 제품을 생성한

Store의 식별자를 필요로 한다. 즉 Store의 데이터를 이용해서 Product를 생성한다. 게다가 Product를 생성할 수 있는 조건을 판단할 때 Store의 상태를 이용한다.

따라서 Store에 Product를 생성하는 팩토리 메서드를 추가하면 Product를 생성할 때 필요한 데이터의 일부를 직접 제공하면서 동시에 중요한 도메인 로직을

함께 구현할 수 있게된다. Store 애그리거트가 Product 애그리거트를 생성할 때 많은 정보를 알아야 한다면 Store 애그리거트에서 Product 애그리거트를 직접

생성하지 않고 다른 팩토리에 위임하는 방법도 있다.

```java
public class Store{
    public Product createProduct(ProductId newProductId, ProductInfo pi){
        if(isBlocked()) throw new StoreBlockedException();
        return ProductFactory.create(newProductId, getId(), pi);
    }
}
```

다른 팩토리에 위임하더라도 차단 상태의 상점은 상품을 만들 수 없다는 도메인 로직은 한곳에 계속 위치한다.