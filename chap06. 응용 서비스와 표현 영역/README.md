# Chap06.  응용 서비스와 표현 영역


도메인이 제 기능을 하려면 사용자와 도메인을 연결해주는 매개체가 필요하다. 응용 영역과 표현 영역이 사용자와 도메인을 연결해주는 매개체 역할을 한다.
<img width="875" alt="image" src="https://user-images.githubusercontent.com/40031858/169631671-fb831397-af34-4440-942c-b8d5a45a2e9b.png">


표현 영역은 사용자의 요청을 해석한다. 사용자가 웹 브라우저에서 폼에 ID와 암호를 입력한 뒤에 전송 버튼을 클릭하면 요청 파라미터를 포함한

HTTP 요청을 표현 영역에 전달한다. 요청을 받은 표현 영역은 URL, 요청 파라미터, 쿠키, 헤더 등을 이용해서 사용자가 실행하고 싶은

기능을 판별하고 그 기능을 제공하는 응용 서비스를 실행한다.

실제 사용자가 원하는 기능을 제공하는 것은 응용 영역에 위치한 서비스다. 사용자가 회원 가입을 요청했다면 실제 그 요청을 위한 기능을 제공하는 주체는

응용 서비스에 위치한다. 응용 서비스는 기능을 실행하는 데 필요한 입력 값을 메서드 인자로 받고 실행 결과를 리턴한다.

응용 서비스의 메서드가 요구하는 파라미터와 표현 영역이 사용자로부터 전달받은 데이터는 형식이 일치하지 않기 때문에 표현 영역은 응용 서비스가

요구하는 형식으로 사용자 요청을 변환한다. 예를 들어 표현 영역의 코드는 다음과 같이 폼에 입력한 요청 파라미터 값을 사용해서 응용 서비스가 요구하는 객체를 생성한 뒤 응용 서비스의 메서드를 호출한다.

```java
@PostMapping("/member/join")
public ModelAndView join(HttpServletRequest request){
    String email = request.getParameter("email");
    String password = request.getParameter("password");
    //사용자 요청을 응용 서비스에 맞게 변환
    JoinRequest joinReq = new JoinRequest(email,password);
    //변환한 객체(데이터)를 이용해서 응용 서비스 실행
    joinService.join(joinReq);
    ...
}
```

## 응용 서비스의 역할

응용 서비스는 사용자가 요청한 기능을 실행한다. 응용 서비스는 사용자의 요청을 처리하기 위해 리포지토리에서 도메인 객체를 가져와 사용한다.

응용 서비스의 주요 역할은 도메인 객체를 사용해서 사용자의 요청을 처리하는 것이므로 표현 영역 입장에서 보았을 때 응용 서비스는 도메인 영역과 표현영역을 연결해주는 창구 역할을한다.

등용 서비스는 주로 도메인 객체 간의 흐름을 제어하기 때문에 다음과 같이 단순한 형태를 갖는다.

```java
public Result doSomeFunc(SomeReq req){
    //1. 리포지토리에서 애그리거트를 구한다.
    SomeAgg agg = someAggRepository.findById(req.getId());
    checkNull(agg);

    //2. 애그리거트의 도메인 기능을 실행한다.
    agg.doFunc(req.getValue());

    //3. 결과를 리턴한다
    return createSuccessReulst(agg);
}
```

새로운 애그리거트를 생성하는 응용 서비스 역시 간단하낟.

```java
public Reulst doSomeCreation(CreateSomeReq req){
    //1. 데이터 중복 등 데이터가 유효한지 검사한다.
    validate(req);

    //2. 애그리거트를 생성한다
    SomeAgg newAgg = createSome(req);

    //3. 리포지토리에 애그리거트를 저장한다
    someAggRepository.save(newAgg);

    //4.결과를 리턴한다
    return createSuccessResult(newAgg);
}
```

응용 서비스가 복잡하다면 응용 서비스에서 도메인 로직의 일부를 구현하고 있을 가능성이 높다. 응용 서비스가 도메인 로직을 일부 구현하면 코드 중복, 로직 분산 등 코드 품질에

안좋은 영향을 줄 수 있다. 응용 서비스는 트랜잭션 처리도 담당한다. 응용 서비스는 도메인의 상태 변경을 트랜잭션으로 처리해야한다.

한번에 다수 회원을 차단 상태로 변경하는 응용 서비스를 생각해보자. 이 서비스는 차단 대상이 되는 Member 애그리거트 목록을 구하고 차례대로 차단 기능을 실행할 것이다.

```java
public void blockMembers(String[] blockingIds){
    if(blockingIds == null || blockingIds.length ==0) return;
    List<Member> members = memberRepository.findByIdIn(blockingIds);
    for(Member mem : members){
        mem.block();
    }
}
```

blockMembers() 메서드가 트랜잭션 범위에서 실행되지 않는다고 가정하면 Member 객체의 block()메서드를 실행해서 상태를 변경했는데

DB에 반영하는 도중 문제가 발생하면 일부 Member만 차단 상태가 되어 데이터 일관성이 깨지게 된다. 이런 상황이 발생하지 않으려면 트랜잭션 범위에서 응용 서비스를 실행해야 한다.

### 도메인 로직 넣지 않기

도메인 로직은 도메인 영역에 위치하고 응용 서비스는 도메인 로직을 구현하지 않는다고 했다. 암호 변경 기능을 예로 들어보자

암호 변경 기능을 위한 응용 서비스는 Member애그리거트와 관련 리포지토리를 이용해 다음코드처럼 도메인 객체 간 실행 흐름을 제어한다
```java
public class ChangePasswordService{
    public void changePassword(String memberId,String oldPw, String newPW){
        Member member = memberRepository.findById(memberId);
        checkMemberExists(member);
        member.changePassword(oldPw,newPw);
    }
}
```
Member 애그리거트는 암호를 변경하기 전에 기존 암호를 올바르게 입력했는지 확인하는 로직을 구현한다.

```java
public class Member{
    public void changePassword(String oldPw, String newPw){
        if(!matchPassword(oldPw)) throw new BadPasswordException();
        setPassword(newPw);
    }
    //현재 암호와 일치하는지 검사하는 도메인 로직
    public boolean matchPassword(String pwd){
        return passwordEncoder.matches(pwd);
    }

    private void setPassword(String newPw){
        if(isEmpty(newPw)) throw new IllegalArgumentException("no new password");
        this.password = newPw;
    }
}
```

기존 암호를 올바르게 입력했는지를 확인하는 것은 도메인의 핵심 로직이기 때문에 다음코드처럼 응용서비스이서 이 로직을 구현하면 안된다.

```java
public class ChangePasswordService{
    public void changePassword(String memberId, String oldPw, String newPw){
        Member member = memberRepository.findById(memberId);
        checkMemberExists(member);

        if(!passwordEncoder.matches(oldPw,member.getPassword()))
            throw new BadPasswordException();
        member.setPassword(newPw);
    }
}
```

도메인 로직을 도메인 영역과 응용 서비스에 분산해서 구현하면 코드 품질에 문제가 발생한다.
`첫번째 문제는 코드의 응집성이 떨어진다는 것이다`. `두번째 문제는 여러 응용 서비스에서 동일한 도메인 로직을 구현할 가능성이 높아진다는 것이다`

코드 중복을 막기위해 응용 서비스 영역에 별도의 보조 클래스를 만들 수 있지만, 애초에 도메인 영역에 암호 확인 기능을 구현했으면 응용 서비스는 

그 기능을 사용하기만 함녀 된다.

```java
public class DeactivationService{
    public void deactivate(String memberId, String pwd){
        Member member = memberRepository.findById(memberId);
        checkMemberExists(member);
        if(!member.matchPassword(pwd))
            throw new BadPasswordException();
        member.deactivate();
    }
}
```

이처럼 소프트웨어의 가치를 높이려면 도메인 로직을 도메인 영역에 모아서 코드 중복을 줄이고 응집도를 높여야 한다.


## 응용 서비스의 구현
응용 서비스는 표현 영역과 도메인 영역을 연결하는 매개체 역할을 하는데 이는 디자인 패턴에서 `facade`와 같은 역할을 한다.

### 응용 서비스의 크기

응용 서비스 자체의 구현은 어렵지 않지만 몇가지 생각할 거리가 있다. 그중 하나가 응용 서비스의 크기다. 회원 도메인을 생각해보자.

응용 서비스는 회원가입, 회원 탈퇴, 회원 암호 변경, 비밀번호 초기화 같은 기능을 구현하기 위해 도메인 모델을 사용하게된다.

이 경우 응용 서비스는 보통 다음의 두가지 방법 중 한가지 방식으로 구현한다
- 한 응용 서비스 클래스에 회원 도메인의 모든 기능 구현하기
- 구분되는 기능별로 응용 서비스 클래스를 따로 구현하기

회원과 관련된 기능을 한클래스에서 모두 구현하면 다음과 같은 모습을 갖는다
```java
public class MemberService{
    //각 기능을 구현하는 데 필요한 리포지토리, 도메인 서비스 필드 추가
    private MemberRepository memberRepository;

    public void join(MemberJoinRequest joinRequest){...}
    public void changePassword(String memberId,String curPw, String newPw){...}
    public void initializePassword(String memberId){...}
    public void leave(String memberId,String curPw){...}
}
```
한 도메인과 관련된 기능을 구현한 코드가 한 클래스에 위치하므로 각 기능에서 동일 로직에 대한 코드 중복을 제거할 수 있다는 장점이 있다.

하지만 한 클래스에 코드가 모이기 시작하면 엄연히 분리하는 것이 좋은 상황임에도 습관적으로 기존에 존재하는 클래스에 억지로 끼워 넣게 된다.

이것은 코드를 점점 얽히게 만들어 코드 품질을 낮추는 결과를 초래한다.

구분되는 기능별로 서비스 클래스를 구현하는 방식은 한 응용 서비스 클래스에서 한 개 내지 2~3개의 기능을 구현한다. 다음은 암호 변경 기능만을 위한 응용 서비스 클래스다.

```java
public class ChangePasswordService{
    private MemberRepository memberRepository;

    public void changePassword(String memberId, String curPw, String newPw){
        Member member = memberRepository.findById(memberId);
        if(member == null) throw new NoMemberException(memberId);
        member.changePassword(curPw,newPw);
    }
    ...
}
```
이 방식을 사용하면 클래스 개수는 많아지지만 한 클래스에 관련 기능을 모두 구현하는 것과 비교해서 코드 품질을 일정 수준으로 유지하는데 도움이 된다.

또한 각 클래스별로 필요한 의존 객체만 포함하므로 다른 기능을 구현한 코드에 영향을 받지 않는다.

각 기능마다 동일한 로직을 구현할 경우 여러 클래스에 중복해서 동일한 코드를 구현할 가능성이 있는데 다음과 같이 별도 클래스에 로직을 구현해서 중복을 방지할 수 있다.

```java
//각 응용 서비스에서 공통되는 로직을 별도 클래스로 구현
public final class MemberServiceHelper{
    public static Member findExistingMember(MemberRepository repo, String memberId){
        Member member = memberRepository.findById(memberId);
        if(member == null) throw new NoMemberException(memberId);
        return member;
    }
}

//공통 로직을 제공하는 메서드를 응용 서비스에서 사용
public class ChangePasswordService{
    private MemberRepository memberRepository;

    public void changePassword(String memberId, String curPw, String newPw){
        Member member = MemberServiceHelper.findExistingMember(memberRepository, memberId);
        member.changePassword(curPw,newPw);
    }
}
```

### 표현 영역에 이ㅡ존하지 않기
응용 서비스의 파라미터 타입을 결정할 때 주의할 점은 표현 영역과 관련된 타입을 사용하면 안된다는 점이다.

예를들어 다음과 같이 표현영역에 해당하는 HttpServletRequest나 HttpSession을 응용 서비스에 파라미터로 전달하면 안된다.

```java
@Controller
@RequestMapping("/member/changePassword")
public class MemberPasswordController{
    @PostMapping
    public String submit(HttpServletRequest request){
        try{
            //응용 서비스가 표현 영역을 의존하면 안됨!
            changePasswordService.changePassword(request);
        }catch(NoMemberException ex){
            //알맞은 익셉션 처리 및 응답
        }
    }
}
```

응용 서비스에서 표현 영역에 대한 의존이 발생하면 응용 서비스만 단독으로 테스트하기 어려워진다. 게다가 표현 영역의 구현이 변경되면 응용 서비스의 구현도 함께

변경해야 하는 문제도 발생한다. 심각한 문제는 응용 서비스가 표현 영역의 역할까지 대신하는 상황이 벌어질 수 있다는 것이다.

```java
public class AuthenticationService{
    public void authenticate(HttpServletRequest request){
        String id = request.getParameter("id");
        String password = request.getParameter("password");
        if(checkIdPasswordMatching(id,password)){
            //응용 서비스에서 표현 영역의 상태 처리
            HttpSession session = request.getSession();
            session.setAttribute("auth",new Authentication(id));
        }
    }
}
```
HttpSession이나 쿠키는 표현 영역의 상태에 해당하는데 이 상태를 응용 서비스에서 변경해버리면 표현 영역의 코드만으로 표현 영역의 상태가

어떻게 변경되는지 추적하기 어려워진다. 이런 문제가 발생하지 않도록 하려면 철저하게 응용 서비스가 표현 영역의 기술을 사용하지 않도록 해야한다.

### 트랜잭션 처리
회원가입에 성공했다고 하면서 실제로 회원 정보를 DB에 삽입하지 않으면 고객은 로그인을 할 수 없다. 비슷하게 배송지 주소를 변경하는 데

실패했다는 안내 화면을 보여줬는데 실제로는 DB에 변경된 배송지 주소가 반영되어 있다면 고객은 물건을 제대로 받지 못하게 된다.

이 두가지는 트랜잭션과 관련된 문제로 트랜잭션을 관리하는 것은 응용서비스의 중요한 역할이다. 스프링과 같은 프레임워크가 제공하는

트랜잭션 관리 기능을 이용하면 쉽게 트랜잭션을 처리할수있다.
```java
public class ChangePasswordService{
    @Transactional
    public void changePassword(ChangePasswordRequest req){
        Member member = findExistingMember(req.getMemberId());
        member.changePassword(req.getCurrentPassword(),req.getNewPassword());
    }
}
```

## 표현 영역
표현 영역의 책임은 크게 다음과 같다
- 사용자가 시스템을 사용할 수 있는 흐름을 제공하고 제어한다
- 사용자의 요청에 알맞은 응용 서비스에 전달하고 결과를 사용자에게 제공한다
- 사용자의 세션을 관리한다

표현 영역이ㅡ 첫번째 책임은 사용자가 시스템을 사용할 수 있도록 알맞은 흐름을 제공하는 것이다. 예를들어 웹애플리케이션에서 사용자가 게시글

쓰기를 표현 영역에 요청하면 표현영역은 다음과 같이 게시글을 작성할 수 있는 폼화면을 응답으로 제공한다.

<img width="687" alt="image" src="https://user-images.githubusercontent.com/40031858/169635040-7aa7d5ec-dd69-4f42-81c5-ed617edf42b5.png">

사용자는 표현 영역이 제공한 폼에 알맞은 값을 입력하고 다시 폼을 표현 영역에 전송한다. 표현 영역은 응용 서비스를 이용해서 표현 영역의 요청을 처리하고

그 결과를 응답으로 전송한다. 표현 영역의 두 번째 책임은 사용자의 요청에 맞게 응용 서비스에 기능 실행을 요청하는 것이다.

화면을 보여주는데 필요한 데이터를 읽거나 도메인의 상태를 변경해야 할 때 응용 서비스를 사용한다. 이과정에서 표현 영역은 사용자의 요청 데이터를

응용 서비스가 요구하는 형식으로 변환하고 응용 서비스의 결과를 사용자에게 응답할 수 있는 형식으로 변환한다.

예를들어 암호 변경을 처리하는 표현 영역은 다음과 같이 HTTP요청 파라미터로부터 필요한 값을 읽어와 응용 서비스의 메서드가 요구하는 객체로 변환해서 요청을 전달한다

```java
@PostMapping()
public String changePassword(HttpServletRequest request, Errors errors){
    //표현 영역은 사용자 요청을 응용 서비스가 요구하는 형식으로 변환한다.
    String curPw = request.getParamater("curPw");
    String newPw = request.getParameter("newPw");
    String memberId = SecurityContext.getAuthentication().getId();
    ChangePasswordRequest chPwdReq = new ChangePasswordRequest(memberId,curPw,newPw);

    try{
        //응용 서비스를 실행
        changePasswordService.changePassword(chPwdReq);
        return successView;
    }catch(BadPasswordException | NoMemberException ex){
        //응용 서비스의 처리 결과를 알맞은 응답으로 변환
        errors.reject("idPasswordNotMatch");
        return formView;
    }
}

```

MVC 프레임워크는 HTTP 요청 파라미터로부터 자바 객체를 생성하는 기능을 지원하므로 이 기능을 사용하면 다음 코드처럼 응용 서비스에 전달할

자바 객체를 보다 손쉽게 생성할 수 있다.
```java
// 프레임워크가 제공하는 기능을 사용해서
// HTTP요청을 응용 서비스의 입력으로 쉽게 변경 처리
@PostMapping
public String changePassword(ChangePasswordRequest chPwdReq, Errors errors){
    String memberId = SecurityContext.getAuthentication().getId();
    chPwdReq.setMemberId(memberId);
    try{
        //응용 서비스를 실행
        changePasswordService.changePassword(chPwdReq);
        return successView;
    }catch(BadPasswordException | NoMemberException ex){
        //응용 서비스의 처리 결과를 알맞은 응답으로 변환
        errors.reject("idPasswordNotMatch");
        return formView;
    }
}
```

응용 서비스의 실행 결과를 사용자에게 알맞은 형식으로 제공하는 것도 표현 영역의 몫이다. 이 코드는 응용 서비스에서 익셉션이 발생하면

에러 코드를 설정하는데 표현 영역의 뷰는 이 에러코드에 알맞은 처리를 하게된다.

표현 영역의 다른 주된 역할은 사용자의 연결 상태인 세션을 관리하는 것이다. 웹은 쿠키나 서버 세션을 이용해서 사용자의 연결 상태를 관리한다.

## 값 검증
값 검증은 표현 영역과 응용 서비스 두곳에서 모두 수행할 수 있다. 원칙적으로 모든 값에 대한 검증은 응용 서비스에서 처리한다. 예를 들어 회원 가입을 처리하는 

응용 서비스는 파라미터로 전달받은 값이 올바른지 검사해야한다.
```java
public class JoinService{
    @Transactional
    public void join(JoinRequest joinReq){
        //값의 형식 검사
        checkEmpty(joinReq.getId(),"id");
        checkEmpty(joinReq.getName(),"name");
        checkEmpty(joinReq.getPassword(),"password");
        if(joinReq.getPassword().equals(joinReq.getConfirmPassworD()))
            throw new InvalidPropertyException("confirmPassword");
        
        //로직검사
        checkDuplicateId(joinReq.getId());
    }
    private void checkEmpty(String value, String propertyName){
        if(value == null || value.isEmpty())
            throw new EmptyPropertyException(propertyName);
    }
    private void checkDuplicateId(String id){
        int count = memberRepository.countsById(id);
        if(count > 0) throw new DuplicateIdException();
    }
}
```

그런데 표현 영역은 잘못된 값이 존재하면 이를 사용자에게 알려주고 값을 다시 입력받아야 한다. 스프링 MVC는 폼에 입력한 값이 잘못된 경우 에러메시지를 보여주기 위한 용도로

Errors나 BindingResult를 사용하는데, 컨트롤러에서 위와 같은 응용 서비스를 사용하면 폼에 에러메시지를 보여주기 위해 다음과 같이 번잡한 코드를 작성해야한다.

```java
@Controller
public class Controller{
    @PostMapping("/member/join")
    public String join(JoinRequest joinRequest, Errors errors){
        try{
            joinService.join(joinRequest);
            return successView;
        }catch(EmptyPropertyException ex){
            errors.rejectValue(ex.getPropertyName(),"emtpy");
            return formView;
        }catch(InvalidPropertyException ex){
            errors.rejectValue(ex.getPropertyName(),"invalid");
            return formView;
        }catch(DuplicateIdException ex){
            errors.rejectValue(ex.getPropertyName(),"duplicate");
            return formView;
        }
    }
}
```

하지만 이는 값을 검사하는 시점에 첫 번째 값이 올바르지 않아 익셉션을 발생시키면 나머지 항목에 대해서는 값을 검사하지 않게 된다.

이는 사용자가 같은 폼에 값을 여러번 입력하게 만든다. 이런 사용자 불편을 해소하기 위해 응용 서비스에서 에러 코드를 모아 하나의 익셉션으로 

발생시키는 방법도 있다. 아래코드는 그 예다.
```java
@Transactional
public OrderNo placeOrder(OrderRequest orderRequest){
    List<ValidationError> errors = new ArrayList<>();
    if(orderRequest == null)
        errors.add(ValidationError.of("empty"));
    else{
        if(orderRequest.getOrdererMemberId() == null)
            errors.add(ValidationError.of("ordererMemberId","empty"));
        if(orderRequest.getOrderProducts() == null)
            errors.add(ValidationError.of("orderProducts","empty"));
        if(orderRequest.getOrderProducts().isEmpty())
            errors.add(ValidationError.of("orderProducts","emtpy"));
    }
    //응용 서비스가 입력 오류를 하나의 익셉션으로 모아서 발생
    if(!errors.isEmpty()) throw new ValidationErrorException(errors);
}
```

표현 영역은 응용 서비스가 ValidationException을 발생시키면 다음 코드처럼 익셉션에서 에러 목록을 가져와 표현 영역에서 사용할 형태로 변환 처리한다.

```java
@PostMapping("/orders/order")
public String order(@ModelAttribute("orderReq") OrderRequest orderRequest,
                    BindingResult bindingResult,
                    ModelMap modelMap){
        User user = (User) SecurityContextHolder.getContext()
                    .getAuthentication().getPrincipal();
        orderRequest.setOrdererMemberId(MemberId.of(user.getUsername()));
        try{
            OrderNo orderNo = placeOrderService.placeOrder(orderRequest);
            modelMap.addAttribute("orderNo",orderNo.getNumber());
        }catch(ValidationErrorexception e){
            e.getErrors().forEach(err -> {
                if(err.hasName()){
                    bindingResult.rejectValue(err.getName(), err.getCode());
                }else{
                    bindingResult.reject(err.getCode());
                }
            });
            populateProductsModel(orderRequest,modelMap);
            return "order/confirm";
        }
    }
``` 

스프링과 같은 프레임워크는 값 검증을 위한 Validator 인터페이스를 별도로 제공하므로 이 인터페이스를 구현한 검증기를

따로 구현하면 위 코드를 다음과 같이 간결하게 줄일 수 있다.

```java
@Controller
public class Controller{
    @PostMapping("/member/join")
    public String join(JoinRequest joinRequest, Errors errors){
        new JoinRequestValidator().validate(joinRequest,errors);
        if(errors.hasErrors()) return formView;
        try{
            joinService.join(joinRequest);
            return successView;
        }catch(DuplicateIdException ex){
            errors.rejectvalue(ex.getPropertyName(),"duplicate");
            return formView;
        }
    }
}
```
이렇게 표현 영역에서 필수 값과 값의 형식을 검사하면 실질적으로 응용 서비스는 ID 중복 여부와 같은 논리적 오류만 검사하면 된다.

즉 표현 영역과 응용 서비스가 값 검사를 나눠서 수행하는 것이다. 응용 서비스를 사용하는 표현 영역 코드가 한곳이면 구현의 편리함을 위해

다음과 같이 역할을 나누어 검증을 수행할 수도 있다.
- 표현 영역: 필수 값, 값의 형식, 범위 등을 검증한다
- 응용 서비스: 데이터의 존재 유무와 같은 논리적 오류를 검증한다.

## 조회 전용 기능과 응용 서비스
앞서 조회화면을 위한 조회 전용 모델과 DAO를 만드는 내용을 다뤗다. 서비스에서 이들 조회전용 기능을 사용하면 서비스 코드가 다음과 같이

단순히 조회 전용 기능을 호출하는 형태로 끝날 수 있다.

```java
public class OrderListService{
    public List<OrderView> getOrderList(String ordererId){
        return orderViewDao.selectByOrderer(ordererId);
    }
}
```

서비스에서 수행하는 추가적인 로직이 없을뿐더러 단일 쿼리만 실행하는 조회 전용 기능이어서 트랜잭션이 필요하지도 않다.

이 경우라면 굳이 서비스를 만들 필요 없이 표현 영역에서 바로 조회 기능을 사용해도 문제가 없다
```java
public class OrderController{
    private OrderViewDao orderViewDao;

    @RequestMapping("/myorders")
    public String list(ModelMap model){
        String ordererId = SecurityContext.getAuthentication().getId();
        List<OrderView> orders = orderViewDao.selectByOrderer(ordererId);
        model.addAttribute("orders",orders);
        return "order/list";
    }
}
```

응용 서비스를 항상 만들었던 개발자는 컨트롤러와 같은 표현 영역에서 응용 서비스없이 조회 전용 기능에 접근하는 것이 이상하게 느껴질 수 있다.

하지만 응용서비스가 사용자 요청 기능을 실행하는 데 별다른 기여를 하지 못한다면 굳이 서비스를 만들지 않아도 된다.

<img width="682" alt="image" src="https://user-images.githubusercontent.com/40031858/169636062-3a64732b-92a4-48f8-b66d-594bbcdfd682.png">

