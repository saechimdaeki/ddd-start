# Chap05. 스프링 데이터 JPA를 이용한 조회 기능

## 시작에 앞서
`CQRS`는 명령모델과 조회 모델을 분리하는 패턴이다. 명령 모델은 상태를 변경하는 기능을 구현할 때 사용하고 조회 모델은 데이터를 조회하는

기능을 구현할 때 사용한다. 예를 들어 회원 가입, 암호 변경, 주문 취소 처럼 상태를 변경하는 기능을 구현할 때 명령 모델을 사용한다.

주문 목록, 주문 상세 처럼 데이터를 보여주는 기능을 구현할 때는 조회 모델을 사용한다.

## 검색을 위한 스펙
검색 조건이 고정되어 있고 단순하면 다음과 같이 특정 조건으로 조회하는 기능을 만들면 된다.
```java
public interface OrderDataDao{
    Optional<OrderData> findById(OrderNo id);
    List<OrderData> findByOrderer(String ordererId, Date fromDate, Date toDate);
    ...
}
```

그런데 목록 조회와 같은 기능은 다양한 검색 조건을 조합해야 할 때가 있다. 필요한 조합마다 find메서드를 정의할 수도 있지만 이것은 좋은 방법은 아니다.

조합이 증가할수록 정의해야 할 find메서드도 함께 증가하기 때문이다.

이렇게 검색 조건을 다양하게 조합해야 할때 사용할 수 있는 것이 스펙이다. 스펙은 애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스다.

스펙 인터페이스는 다음과 같이 정의한다.

```java
public interface Speficiation<T>{
    public boolean isSatisfiedBy(T agg);
}
```

isSatisfiedBy() 메서드의 agg 파라미터는 검사 대상이 되는 객체다. 스펙을 리포지토리에 사용하면 agg는 애그리거트 루트가 되고, 스펙을 DAO에 

사용하면 agg는 검색 결과로 리턴할 데이터 객체가 된다. isSatisfiedBy() 메서드는 검사 대상 객체가 조건을 충족하면 true를 리턴하고, 그렇지 않으면 

false를 리턴한다. 예를 들어 Order 애그리거트 객체가 특정 고객의 주문인지 확인하는 스펙은 다음과 같이 구현할 수 있다.

```java
public class OrderSpec implements Specification<Order>{
    private String ordererId;

    public OrderSpec(String ordererId){
        this.ordererId = ordererId;
    }

    public boolean isSatisfiedBy(Order agg){
        return agg.getOrdererId().getMemberId().getId().equals(ordererId);
    }
}
```

리포지토리나 DAO는 검색 대상을 걸러내는 용도로 스펙을 사용한다. 만약 리포지토리가 메모리에 모든 애그리거트를 보관하고 있다면 다음과 같이 스펙을 사용할 수 있다.

```java
public class MemoryOrderRepository implements OrderRepository{
    public List<Order> findAll(Specification<Order> spec){
        List<Order> allOrders = findAll();
        return allOrders.stream()
                        .filter(order -> spec.isSatisfiedBy(order))
                        .toList();
    }
}
```

리포지토리가 스펙을 이용해서 검색 대상을 걸러주므로 특정 조건을 충족하는 애그리거트를 찾고 싶으면 원하는 스펙을 생성해서 리포지토리에 전달해 주기만 하면 된다.

```java
//검색 조건을 표현하는 스펙을 생성해서
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
//리포지토리에 전달
List<Order> orders = orderRepository.findAll(ordererSpec);
```

하지만 실제 스펙은 이렇게 구현하지 않는다. 모든 애그리거트 객체를 메모리에 보관하기도 어렵고 메모리에 다 보관할수 있다고 하더라도 조회성능에 심각한 문ㄷ제가 발생한다.

## 스프링 데이터 JPA를 이용한 스펙 구현

스프링 데이터 JPA는 검색 조건을 표현하기 위한 인터페이스인 Specification을 제공하며 다음과 같이 정의되어 있다.

```java
public interface Specification<T> extends Serializable{
    //not,where,and,or 메서드 생략
    @Nullable
    Predicate toPredicate(Root<T> root,
                        CriteriaQuery<?> query,
                        CriteriaBuilder cb);
}
```

스펙 인터페이스에서 제네릭 타입 파라미터 T는 JPA 엔티티 타입을 의미한다. toPredicate() 메서드는 JPA 크리테리아 API에서 조건을 표현하는 Predicate을 생성한다.

예를들어 다음에 해당하는 스펙은 다음과 같이 구현할 수 있다.
- 엔티티 타입이 OrderSummanry다.
- ordererId 프로퍼티 값이 지정한 값과 동일하다.

```java
public class OrdererIdSpec implements Specification<OrderSummanry>{
    private String ordererId;

    public OrdererIdSpec(String ordererId){
        this.ordererId = ordererId;
    }

    @Override
    public Predicate toPredicate(Root<OrderSummanry> root,
                                CriteriaQuery<?> query,
                                CriteriaBuilder cb){
            return cb.equal(root.get(OrderSummary_.ordererId),ordererId);
        }
}
```

스펙 구현 클래스를 개별적으로 만들지 않고 별도 클래스에 스팩 생성 기능을 모아도 된다. 예를 들어 OrderSummanry와 관련된 스펙 생성을 다음과 같이 한 클래스에 모을 수 있다.

```java
public class OrderSummanrySpecs{
    public static Specification<OrderSummanry> ordererId(String ordererId){
        return (Root<OrderSummary> root, CriteriaQuery<?> query,
                CriteriaBuilder cb) ->
                    cb.equal(root.<String>get("ordererId"), ordererId);
    }

    public static Specification<OrderSummanry> orderDateBetween(
        LocalDateTime from, LocalDateTime to){
            return (Root<OrderSummary> root, CriteriaQuery<?> query,
                    CriteriaBuilder cb) ->
                    cb.between(root.get(OrderSummary_.orderDate), from, to);
        }
}
```

스펙 인터페이스는 함수형 인터페이스이므로 람다식을 이용해서 객체를 생성할 수 있다. 스펙 생성이 필요한 코드는 스펙 생성 기능을 제공하는 클래스를 이용해서

조금 더 간결하게 스펙을 생성할 수 있다.

```java
Specification<OrderSummary> betweenSpec = 
        OrderSummarySpecs.orderDateBetween(from,to);
```

## 리포지토리/DAO에서 스펙 사용하기

스펙을 충족하는 엔티티를 검색하고 싶다면 findAll() 메서드를 사용하면 된다. findAll() 메서드는 스펙 인터페이스를 파라미터로 갖는다.

```java
public interface OrderSummanryDao extends Repository<OrderSummanry,String>{
    List<OrderSummanry> findAll(Specification<OrderSummanry> spec);
}
```

findAll() 메서드는 OrderSummanry에 대한 검색 조건을 표현하는 스펙 인터페이스를 파라미터로 갖는다. 이 메서드에 앞서 스펙 구현체를 사용하면 특정 조건을

충족하는 엔티티를 검색할 수 있다.

```java
//스펙 객체를 생성
Specification<OrderSummary> spec = new OrderIdSpec("user1");
//findAll()메서드를 이용해서 검색
List<OrderSummanry> results = orderSummanryDao.findAll(spec);
```

## 스펙 조합 
스프링 데이터 JPA가 제공하는 스펙 인터페이스는 스펙을 조합할 수 있는 두 메서드를 제공하고 있다. 이 두메서드는 and와 or다.

```java
public interface Specification<T> extends Serializable{
    ...생략

    default Specifiaction<T> and(@Nullable Specification<T> other){...}
    default Specification<T> or(@Nullable Specification<T> other){...}

    @Nullable
    Predicate toPredicate(Root<T> root,
                        CriteriaQuery<?> query,
                        CriteriaBuilder criteriaBuilder);
}
```

and()와 or() 메서드는 기본 구현을 가진 디폴트 메서드이다.  개별 스펙 조건마다 변수를 선언하지 않아도 된다. and()메서드를 사용하면 불필요한 변수 사용을 줄일 수 있다.
```java
Specification<OrderSummary> spec = OrderSummarySpecs.ordererId("user1")
            .and(OrderSummarySpecs.orderDateBetween(from,to));
```

스펙 인터페이스는 not() 메서드도 제공한다. not() 은 정적 메서드로 조건을 반대로 적용할 때 사용한다.

```java
Specification<OrderSummary> spec=
    Specification.not(OrderSummarySpecs.ordererId("user1"));
```

null 가능성이 있는 스펙 객체와 다른 스펙을 조합해야 할 때가 있다. 이 경우 다음 코드처럼 null여부를 판단해서 `NPE` 가 발생하는 것을 방지해야 하는데 null여부를 매번 검사하려면

다소 귀찮다.

```java
Specification<OrderSummary> nullableSpec = createNullableSpec(); //null 일수있음
Specification<OrderSummary> otherSpec = createOtherSpec();

Specification<OrderSummary> spec = 
        nullableSpec == null ? otherSpec : nullableSpec.and(otherSpec);
```

where() 메서드를 사용하면 이런 귀찮음을 줄일 수 있다. where() 메서드는 스펙 인터페이스의 정적 메서드로 null을 전달하면 아무 조건도 생성하지 않는

스펙 객체를 리턴하고 null이 아니면 인자로 받은 스펙 객체를 그대로 리턴한다. 이 메서드를 사용하면 위 코드를 다음과 같이 간단하게 변경할 수 있다.

```java
Specification<OrderSummary> spec =
    Specification.where(createNullableSpec()).and(createOtherSpec());
```

## 정렬 지정하기
스프링 데이터 JPA는 두가지 방법을 사용해서 정렬을 지정할 수 있다
- 메서드 이름에 OrderBy를 사용해서 정렬 기준 지정
- Sort를 인자로 전달

특정 프로퍼티로 조회하는 find 메서드는 이름 뒤에 OrderBy를 사용해서 정렬 순서를 지정할 수 있다. 다음 코드를 보자
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String>{
    List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererId);
}
```

findByOrdererIdOrderByNumberDesc 메서드는 다음 조회 쿼리를 생성한다
- ordererId 프로퍼티 값을 기준으로 검색 조건 지정
- number 프로퍼티 값 역순으로 정렬

메서드 이름에 `OrderBy`를 사용한느 방법은 간단하지만 정렬 기준 프로퍼티가 두 개 이상이면 메서드 이름이 길어지는 단점이 있다. 

또한 메서드 이름으로 정렬 순사거 정해지기 때문에 상황에 따라 정렬 순서를 변경할 수도 없다. 이럴때는 Sort타입을 사용하면 된다.

```java
public interface OrderSummaryDao extends Repository<OrderSummary, String>{
    List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
    List<OrderSummary> findAll(Specification<OrderSummary> spec, Sort sort);
}
```

find메서드에 마지막 파라미터로 Sort를 추가했다. 스프링 데이터 JPA는 파라미터로 전달 받은 Sort를 사용해서 알맞게 정렬 쿼리를 생성한다. find메서드를

사용하는 코드는 알맞은 Sort 객체를 생성해서 전달하면 된다.

```java
Sort sort = Sort.by("number").ascending();
List<OrderSummary> results = orderSummaryDao.findByOrdererId("user1",sort);
```

위 코드는 "number" 프로퍼티 기준 오름차순 정렬을 표현하는 sort객체를 생성한다. 만약 두개 이상의 정렬 순서를 지정하고 싶다면 Sort#and 메서드를 사용해서

두 Sort객체를 연결하면 된다

```java
Sort sort1 = Sort.by("number").ascending();
Sort sort2 = Sort.by("orderDate").descending();

Sort sort = sort1.and(sort2);
```

다음과 같이 짧게 표현할 수도 있다.

```java
Sort sort = Sort.by("number").ascending().and(Sort.by("orderDate").descending());
```

## 페이징 처리하기

스프링 데이터 JPA는 페이징 처리를 위해 Pageable타입을 이용한다. Sort타입과 마찬가지로 find 메서드에 Pageable 타입 파라미터를 사용하면 페이징을 자동으로 처리해준다.

```java
public interface memberDataDao extends Repository<MemberData, String>{
    List<MemberData> findByNameLike(String name, Pageable pageable);
}
```

위 코드에서 findByNameLike() 메서드는 마지막 파라미터로 Pageable 타입을 갖는다. Pageable타입은 인터페이스로 실제 Pageable타입 객체는

PageRequest 클래스를 이용해서 생성한다. 다음 코드는 findByNameLike() 메서드를 호출하는 예를 보여준다.

```java
PageRequest pageReq = PageRequest.of(1,10);
List<MemberData> user = memberDataDao.findByNameLike("사용자%",pageReq);
```

이는 Sort와 사용해서 정렬 순서를 지정할 수도 있다.

## 스펙 조합을 위한 스펙 빌더 클래스

스펙을 생성하다 보면 다음 코드처럼 조건에 따라 스펙을 조합해야 할때가 있다.
```java
Specification<MemberData> spec = Specification.where(null);
if(searchRequest.isOnlyNotBlocked()){
    spec = spec.and(MemberDataSpecs.nonBlocked());
}

if(StringUtils.hasText(searchRequest.getName())){
    spec = spec.and(MemberDataSpecs.nameLike(searchRequest.getName()));
}

List<MemberData> results = memberDataDao.findAll(spec,PageRequest.of(0,5));
```

이 코드는 if와 각 스펙을 조합하는 코드가 섞여있어서 실수하기 좋고 복잡한 구조를 갖는다. 이 점을 보완하기 위해 스펙빌더를 사용하곤 한다. 
```java
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
    .ifTrue(searchRequest.isOnlyNotBlocked(),
            () -> MemberDataSpecs.nonBolcked())
    .ifHasText(searchRequest.getName(),
            name -> MemberDataSpecs.nameLike(searchRequest.getName()))
    .toSpec();
List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0,5));
```
이렇게 메소드를 사용해 조건을 표현하고 메서드 호출 체인으로 연속된 변수할당을 줄여서 가독성을 높일 수 있다.

## 동적 인스턴스 생성
JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공하고 있다.

```java
public interface OrderSummaryDao extends Repository<OrderSummary, String>{
    @Query("
            select new com.myshop.order.query.dto.OrderView(
                o.number,o.state,m.name,m.id,p.name
            )
            from Order o join o.orderLines ol, Member m, Product p 
            where o.orderer.memberId.id = :ordererId
            and o.orderer.memberId.id = m.id
            and index(ol) = 0
            and ol.productId.id = p.id
            order by o.number.number desc
            "
        )
    List<OrderView> findOrderView(String ordererId);
}
```
```java
public class OrderView{
    private final String number;
    private final OrderState state;
    private final String memberName;
    private final String memberId;
    private final String productName;

    public OrderView(OrderNo number, OrderState state,
                    String memberName, MemberId memberId,
                    String productName){
            this.number = number.getNumber();
            this.state = state;
            this.memberName = memberName;
            this.memberId = memberId.getId();
            this.productName = productName;
        }
}
```

조회 전용 모델을 만드는 이유는 표현 영역을 통해 사용자에게 데이터를 보여주기 위함이다. 많은 웹 프레임워크는 새로 추가한 밸류 타입을 알맞은 형식으로 출력하지 못하므로

위 코드처럼 기본타입으로 변환하면 편리하다. 동적 인스턴스의 장점은 JPQL을 그대로 사용하므로 객체 기준으로 쿼리를 작성하면서도 동시에 지연/즉시 로딩과 같은

고민 없이 원하는 모습으로 데이터를 조회할 수 있다는 점이다.

## 하이버네이트 @Subselect 사용
하이버네이트는 JPA 확장 기능으로 `@Subselect`를 제공한다. `@Subselect`는 쿼리 결과를 @Entity로 매핑할 수 있는 유용한 기능으로 예를보자.
```java
@Entity
@Immutable
@Subselect(
    "
    select o.order_number as number,
    o.version, o.orderer_id, o.orderer_name,
    o.total_amouns, o.receiver_name, o.state, o.order_date,
    p.product_id, p.name as product_name
    from purchase_order o inner join order_line ol
        on o.order_number = ol.order_number
        cross join product p
    where
    ol.line_idx = 0
    and ol.product_id = p.product_id
    "
)
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary{
    @Id
    private String number;
    private long vewrsion;
    @Column(name = "orderer_id")
    private String ordererId;
    @Column(name= = "orderer_name")
    private String ordererName;
    ...
    protected OrderSummary(){}
}
```

@Immutable, @Subselect, @Synchronize는 하이버네이트 전용 애노테이션인데 이 태그를 사용하면 테이블이 아닌 쿼리 결과를

@Entity로 매핑할 수 있다. @Subselect는 조회 쿼리를 값으로 갖는다 하이버네이트는 이 select 쿼리의 결과를 매핑할 테이블처럼 사용한다. DBMS가

여러 테이블을 조인해서 조회한 결과를 한 테이블처럼 보여주기 위한 용도로 뷰를 사용하는 것처럼 @Subselect를 사용하면 쿼리 실행 결과를 매핑할 테이블처럼 사용한다.

뷰를 수정할 수 없듯이 @Subselect로 조회한 @Entity 역시 수정할 수 없다. 실수로 @Subselect를 이용한 @Entity 역시 수정할 수 없다. 실수로 @Subselect를

이용한 @Entity의 매핑 필드를 수정하면 하이버네이트는 변경 내역을 반영하는 update 쿼리를 실행할 것이다. 그런데 매핑한 테이블이 없으므로 에러가 발생한다

이런 문제를 방지하기 위해 @Immutable을 사용한다. @Immutable 을 사용하면 하이버네이트는 해당 엔티티의 매핑 필드/ 프로퍼티가 변경되도 DB에 반영하지 않고 무시한다.

```java
//purchase_order 테이블에서 조회
Order order = orderRepository.findById(orderNumber);
order.changeShippingInfo(newInfo); //상태변경

//변경내역이 DB에 반영되지 않았는데 purchase_order 테이블에서 조회
List<OrderSummary> summaries = orderSummaryRepository.findByOrdererId(userId);
```

위 코드는 Order의 상태를 변경한 뒤에 OrderSummary를 조회하고 있다. 특별한 이유가 없으면 하이버네이트는 트랜잭션을 커밋하는 시점에 변경사항을 DB에 반영하므로,

Order의 변경 내역을 아직 purchase_order 테이블에 반영하지 않은 상태에서 purchase_order 테이블을 사용하는 OrderSummary를 조회하게 된다.

즉 , OrderSummary에는 최신 값이 아닌 이전 값이 담기게 된다.

이런 문제를 해소하기 위한용도로 사용한 것이 @Syncronize이다. @Synchronize는 해당 엔티티와 관련된 테이블 목록을 명시한다. 하이버네이트는 엔티티를 로딩하기 전에

지정한 테이블과 관련된 변경이 발생하면 플러시를 먼저한다. OrderSummary의 @Synchroize는 'purchas_order'테이블을 지정하고 있으므로 OrderSummary를

로딩하기 전에 purchase_order 테이블에 변경이 발생하면 관련 내역을 먼저 플러시한다. 따라서 OrderSummary를 로딩하는 시점에서는 변경 내역이 반영된다.

@Subselect를 사용해도 일반 @Entity와 같기때문에 EntityManger.find(), JPQL 등을 사용해서 조회할 수 있다는 것이 @Subselct의 장점이다.

@Subselect는 이름처럼 @Subselect의 값으로 지정한 쿼리를 from 절의 서브쿼리로 사용한다. 즉 실행하는 쿼리는 다음과 같은 형식을 갖는다.

```sql
select osm.number as number1_0_, ..생략
    from(
        select o.order_number as number,
        o.version,
        ..생략
        p.name as product_name
        from purchase_oder o inner join order_line ol
            on o.order_number = ol.order_number
            cross join product p
            where
            ol.line_idx = 0
            and ol.product_id = p.product_id
    ) osm
    where osm.orderer_id = ? order by osm.number desc
```

@Subselect를 사용할 때는 쿼리가 이러한 형태를 갖는다는 점을 유념해야 한다. 서브 쿼리를 사용하고 싶지 않다면 네이티브 SQL 쿼리를 사용하거나

마이바티스와 같은 별도 매퍼를 사용해서 조회기능을 구현해야 한다.