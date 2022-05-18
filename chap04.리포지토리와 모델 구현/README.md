# Chap04. 리포지토리와 모델 구현

## JPA를 이용한 리포지토리 구현

![image](https://user-images.githubusercontent.com/40031858/168939233-1a823d7c-80f3-4378-9484-88f59686b88c.png)

가능하면 리포지토리 구현 클래스를 인프라스트럭처 영역에 위치 시켜서 인프라스트럭처에 대한 의존을 낮추는 것이 좋다.

### 리포지토리 기본 기능 구현
리포지토리가 제공하는 기본 기능은 다음 두가지다
- ID로 애그리거트 조회하기
- 애그리거트 저장하기

두 메서드를 위한 리포지토리 인터페이스는 다음과 같은 형식을 갖는다

```java
public interface OrderRepository{
    Order findById(OrderNo no);
    void save(Order order);
}
```

인터페이스는 애그리거트 루트를 기준으로 작성한다. 주문 애그리거트는 Order 루트 엔티티를 비롯해 OrderLine, Orderer, ShippingInfo 등 다양한 객체를 포함하는데,

이 구성요소 중에서 루트 엔티티인 Order를 기준으로 리포지토리 인터페이스를 작성한다.

애그리거트를 조회하는 기능의 이름을 지을 때 특별한 규칙은 없지만, 널리 사용되는 규칙은 `findBy프로퍼티이름(프로퍼티 값)` 형식을 사용하는 것이다.

위 인터페이스는 ID로 애그리거트를 조회하는 메서드 이름을 findById()로 지정했다.

이 인터페이스를 구현한 클래스는 JPA의 EntityManager를 이용해서 기능을 구현한다. 스프링 프레임워크에 기반한 리포지토리 구현 클래스는 다음과 같다.

```java
@Repository
public class JpaRepository implements OrderRepository{
    
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public Order findById(OrderNo id){
        return entityManager.find(Order.class,id);
    }

    @Override
    public void save(Order order){
        entityManager.persist(order);
    }
}
```

애그리거트를 수정한 결과를 저장소에 반영하는 메소드를 추가할 필요는 없다. JPA를 사용하면 트랜잭션 범위에서 변경한 데이터를 자동으로 DB에 반영하기 떄문이다. 예를들어 다음 코드를 보자

```java
public class ChangeOrderService{
    @Transactional
    public void changeShippingInfo(OrderNo no, ShippingInfo newShippingInfo){
        Optional<Order> orderOpt = orderRepository.findById(no);
        Order order = orderOpt.orElseThrow(() -> new OrderNotFoundException());
        order.changeShippingInfo(newShippingInfo);
    }
    ...
}
```

changeShippingInfo() 메서드는 스프링 프레임워크의 트랜잭션 관리 기능을 통해 트랜잭션 범위에서 실행된다. 메서드 실행이 끝나면 트랜잭션을 커밋하는데 이때 JPA는 트랜잭션 

범위에서 변경된 객체의 데이터를 DB에 반영하기 위해 UPDATE쿼리를 실행한다. order.changeShippingInfo() 메서드를 실행한 결과로 애그리거트가 변경되면 JPA는 변경 데이터를

DB에 반영하기 위해 UPDATE 쿼리를 실행한다. ID가 아닌 다른 조건으로 애그리거트를 조회할 때는 findBy 뒤에 조건 대상이 되는 프로퍼티 이름을 붙인다. 예를 들어 특정 ID가 주문한

Order 목록을 구하는 메서드는 다음과 같이 정의할 수 있다.

```java
public interface OrderRepository{
    ...
    List<Order> findByOrdererId(String ordererId,int stratRow, int size);
}
```

findByOrdererId 메서드는 한 개 이상의 Order객체를 리턴할 수 있으므로 컬렉션 타입 중 하나인 List를 리턴 타입으로 사용했다.

ID외에 다른 조건으로 애그리거트를 구현할때는 JPA의 Criteria나 JPQL등 사용할 수 있다.

jpql을 사용한 예시는 다음과 같다.

```java
@Override
public List<Order> findByOrdererId(String ordererId, int startRow, int fetchSize){
    ThpedQuery<Order> query = entityManager.createQuery(
        "select o from Order  o "+
                    "where o.orderer.memberId.id = :ordererId " +
                    "order by o.number.number desc",
                Order.class);
        query.setParameter("ordererId",ordererId);
        query.setFirstResult(startRow);
        query.setMaxResults(fetchSize);
        return query.getResultList();
}
```

애그리거트를 삭제하는 기능이 필요할 수도 있다. 삭제 기능을 위한 메서드는 다음과 같이 삭제할 애그리거트 객체를 파라미터로 전달받는다.

```java
public interface OrderRepository{
    ...
    public void delete(Order order);
}
```

구현 클래스는 EntityManager의 remove()메서드를 이용해서 삭제 기능을 구현한다.

```java
public class JpaOrderRepository implements OrderRepository{
    @PersistenceContext
    private EntityManager entityManager;

    ...

    @Override
    public void delete(Order order){
        entityManager.remove(order);
    }
}
```

## 스프링 데이터 JPA를 이용한 리포지토리 구현

`스프링 데이터 JPA`는 다음 규칙에 따라 작성한 인터페이스를 찾아서 인터페이스를 구현한 스프링 빈 객체를 자동으로 등록한다.
- org.springframework.data.repository.Repository<T,Id> 인터페이스 상속
- T는 엔티티 타입을 지정하고 ID는 식별자 타입을 지정

예를 들어 Order엔티티 타입의 식별자가 OrderNo 타입이라고 하자
```java
@Entity
@Table(name="purchase_order")
@Access(AccessType.FIELD)
public class Order{
    @EmbeddedId
    private OrderNo number; //OrderNo가 식별자 타입
}
```

Order를 위한 OrderRepository는 다음과 같이 작성할 수 있다.
```java
public interface OrderRepository extends Repository<Order,OrderNo>{
    Optional<Order> findById(OrderNo id);

    void save(Order order);
}
```

스프링 데이터 JPA는 OrderRepository를 리포지토리로 인식해서 알맞게 구현한 객체를 스프링 빈으로 등록한다. OrderRepository가 필요하면 다음처럼 주입받아 사용한다

```java
@Service
public class CallOrderService{
    private OrderRepository orderRepository;
    ..

    public CancelOrderService(OrderRepository orderRepository,...){
        this.orderRepository=orderRepository;
        ..
    }
    @Transactional
    public void cancel(OrderNo orderNo, Canceller canceller){
        Order order = orderRepository.findById(orderNo)
                .orElseThrow(()->new NoOrderException());
        if(!cancelPolicy.hasCancellationPermission(order,canceller)){
            throw new NoCancellablePermission();
        }
        order.cancel();
    }
}
```


## 매핑 구현
애그리거트와 JPA 매핑을 위한 기본 규칙은 다음과 같다
- 애그리거트 루트는 엔티티이므로 @Entity로 매핑 설정한다.

한 테이블에 엔티티와 밸류 데이터가 같이 있다면
- 밸류는 @Embeddable로 매핑 설정한다.
- 밸류 타입 프로퍼티는 @Embedded로 매핑 설정한다.

주문 애그리거트를 예로 들어보자. 주문 애그리거트의 루트 엔티티는 Order이고 이 애그리거트에 속한 Orderer와 ShippingInfo는 밸류이다.

이 세 객체와 ShippingInfo에 포함된 Address 객체와 Receiver객체는 한테이블에 매핑할 수 있다. 루트 엔티티와 루트 엔티티에 속한 밸류는 한 테이블에 매핑할 때가 많다.

![image](https://user-images.githubusercontent.com/40031858/168944333-3f963c7a-95b4-485f-abf5-0ae1d013979c.png)

주문 애그리거트에서 루트 엔티티인 Order는 JPA의 @Entity로 매핑한다.

```java
@Entity
@Table(name="purchase_order")
public class Order{
    ...
}
```

Order에 속하는 Orderer는 밸류이므로 @Embeddable로 매핑한다.

```java
@Embeddable
public class Orderer{
    
    // MemberId에 정의된 칼럼 이름을 변경하기 위해
    // @AttributeOverride 애노테이션 사용
    @Embedded
    @AttributeOverrides(
        @AttributeOverride(name="id", column = @Column(name="orderer_id"))
    )
    private MemberId memberId;

    @Column(name="orderer_name")
    private String name;
}
```


Orderer의 memberId는 Member 애그리거트를 ID로 참조한다. Member의 ID타입으로 사용되는 MemberId는 다음과 같이 id 프로퍼티와 매핑되는 테이블 칼럼 이름으로 "member_id"를 지정하고 잇다.

```java
@Embeddable
public class MemberId implements Serializable{
    @Column(name="member_id")
    private String id;
}
```

위의 그림에서 Orderer의 memberId 프로퍼티와 매핑되는 칼럼이름은 'orderer_id'이므로 MemberId에 설정된 'member_id'와 이름이 다르다. @Embeddable 타입에 설정한

칼럼 이름과 실제 칼럼 이름이 다르므로 @AttributeOverrides 애노테이션을 이용해서 Orderer의 memberId프로퍼티와 매핑할 칼럼 이름을 변경햇다.

Orderer와 마찬가지로 ShippingInfo 밸류도 또 다른 밸류인 Address와 Receiver를 포함한다. Address의 매핑 설정과 다른 칼럼 이름을 사용하기 위해 @AttributeOverride

애노테이션을 사용한다.

```java
@Embeddable
public class ShippingInfo{
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name= "zipCode",
                        column = @Column(name="shipping_zipcode")),
        @AttributeOverride(name = "address1",
                        column = @Column(name="shipping_addr1")),
        @AttributeOverride(name= "address2",
                        column = @Column(name="shipping_addr2"))
    })
    private Address address;

    @Column(name="shipping_message")
    private String message;

    @Embedded
    private Receiver receiver;
}
```

루트 엔티티인 Order 클래스는 @Embedded를 이용해서 밸류 타입 프로퍼티를 설정한다.

```java
@Entity
public class Order{
    ...
    @Embedded
    private Orderer orderer;

    @Embedded
    private ShippingInfo shippingInfo;
}
```

### 기본 생성자
엔티티와 밸류의 생성자는 객체를 생성할 때 필요한 것을 전달받는다. 예를 들어 Receiver 밸류 타입은 생성 시점에 수취인 이름과 연락처를 생성자 파라미터로 받는다.

```java
public class Receiver{
    private String name;
    private String phone;

    public Receiver(String name, String phone){
        this.name = name;
        this.phone = phone;
    }
}
```
Receiver가 불변 타입이면 생성 시점에 필요한 값을 모두 전달받으므로 값을 변경하는 set 메서드를 제공하지 않는다. 이는 Receiver 클래스에 기본 생성자를

추가할 필요가 없다는 것을 의미한다.

하지만 JPA에서 @Entity와 @Embeddable로 클래스를 매핑하려면 기본 생성자를 제공해야 한다. DB에서 데이터를 읽어와 매핑된 객체를 생성할 때 기본 생성자를 사용해서

객체를 생성하기 때문이다. 이런 기술적인 제약으로 Receiver와 같은 불변 타입은 기본 생성자가 필요 없음에도 불구하고 다음과 같이 기본생성자를 추가해야한다.

```java
@Embeddable
public class Receiver{
    @Column(name="receiver_name")
    private String name;
    @Column(name="receiver_phone")
    private String phone;

    protected Receiver(){}

    public Receiver(String name, String phone){
        this.name = name;
        this.phone = phone;
    }
}

```

기본 생성자는 JPA 프로바이더가 객체를 생성할 때만 사용한다. 기본 생성자를 다른 코드에서 사용하면 값이 없는 온전하지 못한 객체를 만들게 된다.

이런 이유로 다른 코드에서 기본 생성자를 사용하지 못하도록 protected로 선언한다.

### 필드 접근 방식 사용

JPA는 필드와 메서드의 두 가지 방식으로 매핑을 처리할 수 있다.
```java
@Entity
@Access(AccessType.PROPERTY)
public class Order{
    @Column(name="state")
    @Enumerated(EnumType.STRING)
    public OrderState getState(){
        return state;
    }

    public void setState(OrderState state){
        this.state = state;
    }
}
```

엔티티에 프로퍼티를 위한 공개 get/set메서드를 추가하면 도메인의 의도가 사라지고 객체가 아닌 데이터 기반으로 엔티티를 구현할 가능성이 높아진다. 

특히 set메서드는 내부 데이터를 외부에서 변경할 수 있는 수단이 되기 때문에 캡슐화를 깨는 원인이 될 수 있다.

밸류 타입을 불변으로 구현하려면 set메서드 자체가 필요 없는데 JPA의 구현 방식 때문에 공개 set 메서드를 추가하는 것도 좋지 않다.

객체가 제공할 기능 중심으로 엔티티를 구현하게끔 유도하려면 JPA 매핑 처리를 프로퍼티 방식이 아닌 필드 방식으로 선택해서 불필요한 get/set 메서드를 구현하지 말아야 한다

```java
@Entity
@Access(AccessType.FIELD)
public class Order{
    @EmbeddedId
    private OrderNo number;

    @Column(name="state")
    @Enumerated(EnumType.STRING)
    private OrderState state;

    ... // cancel(), changeShippingInfo() 등 도메인 기능 구현
    ... // 필요한 get 메서드 제공
}

```

### AttributeConverter를 이용한 밸류 매핑 처리

밸류 타입의 프로퍼티를 한 개 컬럼에 매핑해야할 때가 있다. 예를 들어 Length가 길이값과 단위의 두 프로퍼티를 갖고있는데 DB테이블에는 한 개 칼럼에 '1000mm'와 같은

형식으로 저장할 수 있다. 두 개 이상의 프로퍼티를 가진 밸류 타입을 한 개 컬럼에 매핑하려면 @Embeddable 애노테이션으로 처리할 수 없다. 이럴 때 사용할 수 있는 것이

`AttributeConverter`이다. `AttributeConverter`는 다음과 같이 밸류 타입과 칼럼 데이터 간의 변화를 처리하기 위한 기능을 정의하고 있다.

```java
public interface AttributeConverter<X,Y>{
    public Y convertToDatabaseColumn(X attribute);
    public X convertToEntityAttribute(Y dbData);
}
```

타입 파라미터 X는 밸류 타입이고 Y는 DB타입이다. convertToDatabaseColumn()메소드는 밸류 타입을 DB칼럼 값으로 변환하는 기능을 구현하고 convertToEntityAttribute() 

메서드는 DB칼럼 값을 밸류로 변환하는 기능을 구현한다.

```java
@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, Integer>{
    @Override
    public Integer convertToDatabaseColumn(Money money){
        return money == null ? null : money.getValue();
    }

    @Override
    public Money convertToEntityAttribute(Integer value){
        return value == null ? null : new Money(value);
    }
}
```

AttributeConverter 인터페이스를 구현한 클래스는 @Converter 애노테이션을 적용한다. autoApply속성값을 true로 설정하면 모델에 출현하는 모든 Money

타입의 프로퍼티에 대해 MoneyConverter를 적용한다.

### 밸류 컬렉션: 별도 테이블 매핑
Order엔티티는 한 개 이상의 OrderLine을 가질 수 있다. OrderLine에 순서가 있다면 다음과 같이 List타입을 이용해서 컬렉션을 프로퍼티로 지정할 수 있다.

```java
public class Order{
    private List<OrderLine> orderLines;

}
```

밸류 컬렉션을 별도 테이블로 매핑할 때는 `@ElementCollection`과 `@CollectionTable` 을 함께 사용한다. 관련 매핑 코드는 다음과 같다

```java
@Entity
@Table(name = "purchase_order")
public class Order{
    @EmbeddedId
    private OrderNo number;

    ...

    @ElementCollection(fetch = FecthType.EAGER)
    @CollectionTable(name="order_line",
                    joinColumns = @JoinColumn(name= "order_number"))
    @OrderColumn(name="line_idx")
    private List<OrderLine> orderLines;
    ...
}

@Embeddable
public class OrderLine{
    @Embedded
    private ProductId productId;

    @Column(name = "price")
    private Money price;

    @Column(name = "quantity")
    private int quantity;

    @Column(name="amounts")
    private Money amounts;
    ...
}
```

OrderLine의 매핑을 함께 표시했는데 OrderLine 에는 list의 인덱스 값을 저장하기 위한 프로퍼티가 존재하지 않는다. 그 이유는 List 타입 자체가

인덱스를 가지고 있기 때문이다 . JPA는 `@OrderColumn` 애노테이션을 이용해서 지정한 칼럼에 리스트의 인덱스 값을 저장한다.

`@CollectionTable`은 밸류를 저장할 테이블을 지정한다. name 속성은 테이블 이름을 지정하고 joinColumns 속성은 외부키로 사용할 칼럼을 지정한다.

### 밸류 컬렉션 : 한 개 컬럼 매핑

밸류 컬렉션을 별도 테이블이 아닌 한 개 칼럼에 저장해야 할 때가 있다. 예를 들어 도메인 모델에는 이메일 주소 목록을 Set으로 보관하고 DB에는 한 개 컬럼에 콤마로

구분해서 저장해야할 때가 있다. 
```java
public class EmailSet{
    private Set<Email> emails = new HashSet<>();

    public EmailSet(Set<Email> emails){
        this.emails.addAll(emails);
    }

    public Set<Email> getEmails(){
        return Collections.umnodifiableSet(emails);
    }
}

```

밸류 컬렉션을 위한 타입을 추가했다면 AttributeConverter를 구현한다.
```java
public class EmailSetConverter implements AttributeConverter<EmailSet, String>{
    @Override
    public String convertToDatabaseColumn(EmailSet attribute){
        if(attribute == null) return null;
        return attribute.getEmails().stream()
                .map(email -> email.getAddress())
                .collect(Collectors.joining(","));
    }

    @Override
    public EmailSet convertToEntityAttribute(String dbData){
        if(dbData == null) return null;
        String[] emails = dbData.split(",");
        Set<Email> emailSet = Arrays.stream(emails)
                    .map(value -> new Email(value))
                    .collect(toSet());
        return new EmialSet(emailSet);
    }
}
```
이제 남은것은 EmailSet 타입 프로퍼티가 Converter로 EmailSetConverter를 사용하도록 지정하는 것이다
```java
@Column(name="emails")
@Convert(converter = EmailSetConverter.class)
private EmailSet emailSet;
```

### 밸류를 이용한 ID매핑
식별자라는 의미를 부각시키기 위해 식별자 자체를 밸류타입으로 만들 수도 있다. 밸류 타입을 식별자로 매핑하면 @Id 대신 @EmbeddedId 애노테이션을 사용한다

```java
@Entity
@Table(name = "purchase_order")
public class Order{
    @EmbeddedId
    private OrderNo number;
    ...
}

@Embeddable
public class OrderNo implements Serializable{
    @Column(name="order_number")
    private String number;
}
```
JPA에서 식별자 타입은 Serializable 타입이어야 하므로 식별자로 사용할 밸류 타입은 Serializable 인터페이스를 상속받아야 한다.

### 별도 테이블에 저장하는 밸류 매핑
애그리거트에서 루트 엔티티를 뺀 나머지 구성요소는 대부분 밸류이다. 루트 엔티티 외에 또 다른 엔티티가 있다면 진짜 엔티티인지 의심해 봐야 한다.

밸류가 아니라 엔티티가 확실하다면 해당 엔티티가 다른 애그리거트는 아닌지 확인해야 한다. 특히 자신만의 독자적인 라이프 사이클을 갖는다면 구분되는

애그리거트일 가능성이 높다. 애그리거트에 속한 객체가 밸류인지 엔티티인지 구분하는 방법은 고유 식별자를 갖는지를 확인하는 것이다. 하지만 식별자를 찾을 때 매핑되는

테이블의 식별자를 애그리거트 구성요소의 식별자와 동일한 것으로 착각하면 안된다. 별도 테이블로 저장하고 테이블에 PK가 있다고 해서

테이블과 매핑되는 애그리거트 구성요소가 항상 고유 식별자를 갖는 것은 아니기 때문이다.

![image](https://user-images.githubusercontent.com/40031858/168996244-8b4e502b-bc88-4e39-85fd-04cba1980deb.png)

ArticleContent는 밸류이므로 @Embeddable로 매핑한다. ArticleContent와 매핑되는 테이블은 Article과 매핑되는 테이블과 다르다.

이때 밸류를 매핑한 테이블을 지정하기 위해 `@SecondaryTable` 과 `@AttributeOverride`을 사용한다.

```java
@Entity
@Table(name="article")
@SecondaryTable(
    name = "article_content",
    pkJoinColumns = @PrimaryKeyJoinColumn(name = "id")
)
public class Article{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    
    @AttributeOverrides({
        @AttributeOverride(
            name="content",
            column = @Column(table="article_content", name ="content")),
        @AttributeOverride(
            name="contentType",
            column = @Column(table="article_content", name ="content_type"))
    })
    @Embedded
    private ArticleContent content;
}
```

`@SecondaryTable`의 name 속성은 밸류를 저장할 테이블을 지정한다. pkJoinColumns 속성은 밸류 테이블에서 엔티티 테이블로 조인할 때 사용할 칼럼을 지정한다.

content 필드에 `@AttributeOverride`를 적용했는데 이 애노테이션을 사용해서 해당 밸류 데이터가 저장된 테이블 이름을 지정한다.

`@SecondaryTable`을 이용하면 아래 코드를 실행할 때 두 테이블을 조인해서 데이터를 조회한다
```java
// @SecondaryTable로 매핑된 article_content 테이블을 조인
Article article = entityManager.find(Article.class,1L);
```

### 밸류 컬렉션을 @Entity로 매핑하기

개념적으로 밸류인데 구현기술의 한계나 팀 표준 때문에 @Entity를 사용해야 할 때도 있다. 예를들어 제품의 이미지 업로드 방식에 따라 이미지 경로와 섬네일 이미지 제공

여부가 달라진다고 해보자. 이부분은 바로 코드로 보자.

```java
@Entity
@Imheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name="image_type")
@Table(name = "image")
public abstract class Image{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name ="image_id")
    private Long id;

    @Column(name ="image_path")
    private String path;

    @Temporal(TemporalType.TIMESTAMP)
    @Column(name="upload_time")
    private Data uploadTime;

    protected Image(){}

    public Image(String path){
        this.path = path;
        this.uploadTime = new Date();
    }

    protected String getPath(){
        return path;
    }

    public Date getUploadTime(){
        return uploadTime;
    }

    public abstract String getURL();
    public abstract boolean hasTumbnail();
    public abstract String getThumbnailURL();
}
```

Image를 상속받은 클래스는 @Entity와 @Discriminator를 사용해서 매핑을 설정한다
```java
@Entity
@DiscriminatorValue("II")
public class InternalImage extends Image{
    ...
}

@Entity
@DiscriminatorValue("EI")
public class ExternalImage extends Image{
    ...
}
```

Image가 @Entity이므로 목록을 담고 있는 Product는 @OnetoMany를 이용해서 매핑을 처리한다. Image는 밸류이므로 독자적인 라이프 사이클을 갖지 않고 Product에 

완전히 의존한다. 따라서 Product를 저장할 때 함께 저장되고 Product를 삭제할 때 함께 삭제되도록 cascade속성을 지정한다.

```java
@Entity
@Table(name = "product")
public class Product{
    @EmbeddedId
    private ProductId id;
    private String name;

    @Convert(converter = MoneyConverter.class)
    private Money price;
    private String detail;

    @OneToMany(cascade = {CascadeType.PERSIST, CascadeType.REMOVE},
                orphanRemoval = true)
    @JoinColumn(name = "product_id")
    @OrderColumn(name="list_idx")
    private List<Image> images = new ArrayList<>();
    ...

    public void changeImages(List<Image> newImages){
        images.clear();
        images.addAll(newImages);
    }
}
```

하이버네이트는 @Embeddable 타입에 대한 컬렉션의 clear() 메서드를 호출하면 컬렉션에 속한 객체를 로딩하지 않고 한 번의 delete 쿼리로 삭제 처리를 수행한다.

따라서 애그리거트의 특성을 유지하면서 이 문제를 해소하려면 결국 상속을 포기하고 @Embeddable로 매핑된 단일 클래스로 구현해야 한다. 물론 타입에 따라 다른 기능을

구현하려면 다음과 같이 if-else를 써야한다
```java
@Embeddable
public class Image{
    @Column(name="image_type")
    private String imageType;
    @Column(name="iamge_path")
    private String path;

    @Temporal(TemporalType.TIMESTAMP)
    @Column(name="upload_time")
    private Date uploadTime;
    ...

    public boolean hasTumbnail(){
        if(iamgeType.equals("II"))
            return true;
        else
            return false;
    }    
}
```

### ID참조와 조인 테이블을 이용한 단방향 M-N 매핑

애그리거트간 집합 연관은 성능 상의 이유로 피하는게 좋은데 그럼에도 불구하고 요구사항을 구현하는데 집합 연관을 사용하는 것이 유리하다면

ID참조를 이용한 단방향 집합 연관을 적용해 볼 수 있다.
```java
@Entity
@Table(name = "product")
public class Product{
    @EmbeddedId
    private ProductId id;

    @ElementCollection
    @CollectionTable(name="product_category",
            joinColumns = @JoinColumn(name = "product_id"))
    private Set<CategoryId> categoryIds;
}
```

이 코드는 Product에서 Category로의 단방향 M-N 연관을 ID참조 방식으로 구현한 것이다.

ID참조를 이용한 애그리거트 간 단방향 M-N 연관은 밸류 컬렉션 매핑과 동일한 방식으로 설정한 것을 알 수 있다. 차이점이 있다면 집합의 값에 밸류 대신

연관을 맺는 식별자가 온다는 점이다. @ElementCollection을 이용하기 때문에 Product를삭제할 때 매핑에 사용한 조인 테이블의 데이터도

함께 삭제된다. 애그리거트를 직접 참조하는 방식을 사용했다면 영속성 전파나 로딩 전략을 고민해야하는데 ID참조 방식을 사용함으로써 이런 고민을 없앨 수 있다.
## 애그리거트 로딩 전략

JPA 매핑을 설정할 때 항상 기억해야 할 점은 애그리거트에 속한 객체가 모두 모여야 완전한 하나가 된다는 것이다. 즉 다음과 같이 애그리거트 루트를 로딩하면

루트에 속한 모든 객체가 완전한 상태여야 함을 의미한다.

```java
//product는 완전한 하나여야 한다
Product product = productRepository.findById(id);
```

조회 시점에서 애그리거트를 완전한 상태가 되도록 하려면 애그리거트 루트에서 연관 매핑의 조회 방식을 즉시로딩으로 설정하면 된다. 다음과 같이 컬렉션이나

@Entity에 대한 매핑의 fetch속성을즉시로딩으로 설정하면 EntityManager#find() 메서드로 애그리거트 루트를 구할 때 연관된 구성요소를 DB에서 함께 읽어온다.

```java
// @Entity 컬렉션에 대한 즉시 로징 설정
@OneToMany(cascade = {CascadeType.PERSIST, CascadeType.REMOVE},
            orphanRemoval = true, fetch = FetchType.EAGER)
@JoinColumn(name="product_id")
@OrderColumn(name="list_idx")
private List<Image> images = new ArrayList<>();

// @Embeddable 컬렉션에 대한 즉시 로딩 설정
@ElementCollection(fetch = FecthType.EAGER)
@CollectionTable(name = "order_line",
        joinColumns = @JoinColumn(name="order_number"))
@OrderColumn(name="line_idx")
private List<OrderLine> orderLines;
```

즉시 로딩 방식으로 설정하면 애그리거트 루트를 로딩하는 시점에 애그리거트에 속한 모든 객체를 함께 로딩할 수 있는 장점이 있지만 이것이 항상좋은것은 아니다.

특히 컬렉션에 대해 로딩 전략을 FetchType.EAGER로 설정하면 오히려 즉시 로딩방식이 문제가 될 수 있다.

예를 들어 Product 애그리거트 루트가 @Entity로 구현한 Image와 @Embeddable로 구현한 Option 목록을 갖고 있다고 해보자
```java
@Entity
@Table(name="product")
public class Product{
    ...
    @OnetoMany(cascade = {CascadeType.PERSIST,CascadeType.REMOVE},
            orphanRemoval = true,
            fetch = FetchType.EAGER)
    @JoinColumn(name = "product_id")
    @OrderColumn(name="list_idx")
    private List<Image> images = new ArrayList<>();

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "product_option",
            joinColumns = @JoinColumn(name="product_id"))
    @OrderColumn(name="list_idx")
    private List<Option> options = new ArrayList<>();
}
```

이 매핑을 사용할 때 EntityManger#find() 메서드로 Product를 조회하면 하이버네이트는 다음과 같이 Product를 위한 테이블, Image를 위한 테이블,

Option을 위한 테이블을 조인한 쿼리를 실행한다. `카타시안(Cartesian)` 조인을 사용하고 이는 쿼리 결과에 중복을 발생시킨다. 조회하는 Product의 image가 2개고

option이 2개면 쿼리 결과로 행 개수는 4개가 된다. product 테이블의 정보는 4번 중복되고 image와 product_option 테이블의 정보는 2번 중복된다.

보통 조회 성능 문제 때문에 즉시 로딩 방식을 사용하지만 이렇게 조회되는 데이터 개수가 많아지면 즉시 로딩 방식을 사용할때 성능을 검토해 봐야 한다.

애그리거트는 개념적으로 하나여야한다. 하지만 루트 엔티티를 로딩하는 시점에 애그리거트에 속한 객체를 모두 로딩해야 하는 것은 아니다. 애그리거트가 완전해야 이유는

두가지 정도로 생각해볼 수 있다. 첫 번째 이유는 상태를 변경하는 기능을 실행할 때 애그리거트 상태가 완전해야 하기 때문이고, 두 번째 이유는 표현 영역에서

애그리거트의 상태 정보를 보여줄 때 필요하기 때문이다.

이 중 두번째는 별도의 조회 전용 기능과 모델을 구현하는 방식을 사용하는 것이 더 유리하기 때문에 애그리거트의 완전한 로딩과 관련된 문제는 상태변경과 더 관련이 있다.

상태 변경 기능을 실행하기 위해 조회 시점에 즉시 로딩을 이용해서 애그리거트를 완전한 상태로 로딩할 필요는 없다. JPA는 트랜잭션 범위 내에서 지연 로딩을 허용하기

때문에 다음 코드처럼 실제로 상태를 변경하는 시점에 필요한 구성요소로만 로딩해도 문제가 되지 않는다.

```java
@Transactional
public void removeOptions(ProductId id, int optIdxToBeDeleted){
    //Product를 로딩. 컬렉션은 지연로딩으로 설정했다면, Option은 로딩하지 않음
    Product product = productRepository.findById(id);
    //트랜잭션 범위이므로 지연 로딩으로 설정한 연관 로딩 가능
    product.removeOption(optIdxToBeDeleted);
}

@Entity
public class Product{
    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(name="product_option",
            joinColumns = @JoinColumn(name="product_id"))
    @OrderColumn(name="list_idx")
    private List<Option> options = new ArrayList<>();

    public void removeOption(int optIdx){
        //실제 컬렉션에 접근할 때 로딩
        this.options.remove(optIdx);
    }
}
```

게다가 일반적인 애플리케이션은 상태 변경 기능을 실행하는 빈도보다 조회 기능을 실행하는 빈도가 훨씬 높다. 그러므로 상태 변경을 위해 지연 로딩을 사용할 때 발생하는

추가 쿼리로 인한 실행 속도 저하는 보통 문제가 되지 않는다.  이런 이유로 애그리거트 내의 모든 연관을 즉시 로딩으로 설정할 필요는 없다. 지연 로딩은 동작 방식이

항상 동일하기 때문에 즉시 로딩처럼 경우의 수를 따질 필요가 없는 장점이 있다. 물론 지연로딩은 즉시로딩보다 쿼리 실행 횟수가 많아질 가능성이 더 높다.

따라서 무조건 즉시 로딩이나 지연 로딩으로만 설정하기보다는 애그리거트에 맞게 즉시로딩과 지연로딩을 선택해야 한다.

## 식별자 생성 기능
식별자는 크게 세 가지 방식 중 하나로 생성한다
- 사용자가 직접 생성
- 도메인 로직으로 생성
- DB를 이용한 일련번호 사용
- 이메일 주소처럼 사용자가 직접 식별자를 입력하는 경우는 식별자 생성 주체가 사용자이기 때문에 도메인 영역에 식별자 생성 기능을 구현할 필요가 없다.

식별자 생성 규칙이 있다면 엔티티를 생성할 때 식별자를 엔티티가 별도 서비스로 식별자 생성 기능을 분리해야 한다. 식별자 생성 규칙은 도메인 규칙이므로 도메인 영역에

식별자 생성 기능을 위치시켜야 한다. 예를 들어 다음과 같은 도메인 서비스를 도멩니 영역에 위치시킬 수 있다.

```java
public class ProductIdService{
    public ProductId nextId(){
        //정해진 규칙으로 식별자 생성
    }
}
```

응용 서비스는 이 도메인 서비스를 이용해서 식별자를 구하고 엔티티를 생성한다.
```java
public class CreateProductService{
    @Autowired
    private ProductIdService idService;
    @Autowired
    private ProductRepository productRepository;

    @Transactional
    public ProductId createProduct(ProductCreationCommand cmd){
        //응용 서비스는 도메인 서비스를 이용해서 식별자 생성
        ProductId id = idService.nextId();
        Product product = new Product(id, cmd.getDetail(,...));
        productRepository.save(product);
        return id; 
    }
}
```

특정 값의 조합으로 식별자가 생성되는 것 역시 규칙이므로 도메인 서비스를 이용해서 식별자를 생성할 수 있다. 예를 들어 주문번호가 고객 ID와 타임스탬프로 구성된다면

다음과 같은 도메인 서비슬르 구현할 수 있다.

```java
public class OrderIdService{
    public OrderId createId(UserId userId){
        if(userId==null)    throw new IllegalArgumentException("invalid userid: "+userId);
        return new OrderId(userId.toString()+"_"+timestamp());
    }
    private String timestamp(){
        return Long.toString(System.currentTimeMillis());
    }
}
```

식별자 생성 규칙을 구현하기에 적합한 또 다른 저장소는 리포지토리다. 다음과 같이 리포지토리 인터페이스에 식별자를 생성하는 메소드를 추가하고 리포지토리 구현 클래스에서 

알맞게 구현하면 된다

```java
public interface ProductRepository{
    ..// save() 등 다른 메서드

    // 식별자를 생성하는 메소드
    ProductId nextId();
}
```

DB자동 증가 칼럼을 식별자로 사용하면 식별자 매핑에서 `@GeneratedValue`를 사용한다.
```java
@Entity
@Table(name="article")
public class Artice{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    public Long getId(){
        return id;
    }
}
```

자동 증가 칼럼은 DB의 insert 쿼리를 실행해야 식별자가 생성되므로 도메인 객체를 리포지토리에 저장할 때 식별자가 생성된다. 이 말은 도메인 객체를 생성하는 시점에는

식별자를 알 수 없고 도메인 객체를 저장한 뒤에 식별자를 구할 수 있음을 의미한다.

```java
public class WriteArticleService{
    private ArticleRepository articleRepository;

    public Long write(NewArticleRequest req){
        Article article = new Article("제목",new ArticleContent("content","type"));
        articleRepository.save(article);
        return article.getId();
    }
}
```
JPA는 저장 시점에 생성한 식별자를 @Id로 매핑 한 프로퍼티/필드에 할당하므로 위 코드처럼 저장 이후에 엔티티의 식별자를 사용할 수 있다.

## 도메인 구현과 DIP

앞서 구현한 리포지토리는 DIP원칙을 어기고 있다. 먼저 엔티티는 아래 코드처럼 구현 기술인 JPA에 특화된 @Entity, @Table, @Id, @Column등의 애노테이션을 사용하고 있다

```java
@Entity
@Table(name="article")
@SecondaryTable(
    name="article_content",
    pkJoinColumns = @PrimaryKeyJoinColumn(name="id")
)
public class Artice{
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

DIP에 따르면 @Entity, @Table은 구현 기술에 속하므로 Article과 같은 도메인 모델은 구현 기술인 JPA에 의존하지 말아야 하는데 이 코드는 도메인 모델인

Article이 영속성 구현 기술인 JPA에 의존하고 있다.  리포지토리 인터페이스도 마찬가지이다. 즉 도메인이 인프라에 의존하는 것이다.

구현 기술에 대한 의존 없이 도메인을 순수하게 유지하려면 스프링 데이터 JPA의 Repository인터페이스를 상속받지 않도록 수정하고 ArticleRepository

인터페이스를 구현한 클래스를 인프라에 위치시켜야 한다. 또한 Article 클래스에서 @Entity나 @Table과 같이 JPA에 특화된 애노테이션을 모두 지우고

인프라에 JPA를 연동하기 위한 클래스를 추가해야 한다.

![image](https://user-images.githubusercontent.com/40031858/169017251-42d18943-137f-4e1f-acff-bda7b1522652.png)

특징 기술에 의존하지 않는 순수한 도메인 모델을 추구한느 개발자는 위와 같은 구조로 구현한다. 이 구조를 가지면 구현 기술을 변경하더라도 도메인이 받는 영향을

최소화할 수 있다. DIP를 적용하는 주된 이유는 저수준 구현이 변경되더라도 고수준이 영향을 받지 않도록 하기 위함이다. 하지만 리포지토리와 도메인 모델의

구현 기술은 거의 바뀌지 않는다. JPA 전용 애노테이션을 사용하긴 했지만 도메인 모델을 단위 테스트하는데 문제는 없다.

물론 JPA에 맞춰 도메인 모델을 구현해야 할 때도 있지만 이런 상황은 드물다. 리포지토리도 마찬가지다. 스프링 데이터 JPA가 제공하는 Repository

인터페이스를 상속하고 있지만 리포지토리 자체는 인터페이스이고 테스트 가능성을 해치지 않는다.

DIP를 완벽하게 지키면 좋겠지만 개발 편의성과 실용성을 가져가면서 구조적인 유연함은 어느 정도 유지했다. 복잡도를 높이지 않으면서 기술에 따른 구현제약이 낮다면 합리적인 선택인것같다.