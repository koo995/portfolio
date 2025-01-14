[Spring]애플리케이션의 원자성과 다른 트랜잭션의 원자성(Feat. @Transactional)
=====================================================================================

혹시나 마크다운 문법이 보기 불편하시다면 원글에 해당하는 블로그 글을 [여기](https://medium.com/@gunhong951/spring-프로젝트에서-interceptor-와-argumentresolver-로-firebase-인증을-구현할-때-transactional과-unique-제약조건에-유의하자-e250b121c14e)에서 확인하실 수 있습니다.

지난 프로젝트를 진행할 때, 회원가입과 로그인 처리를 위하여 Firebase을 이용한 적이 있습니다.

안드로이드 앱에서 인증을 완료하면 백엔드는 로그인과 회원가입을 간단히 처리할 수 있고, 이 과정에서 Spring의 Interceptor와 ArgumentResolver을 이용하여 인증을 처리했습니다.

구현 요구사항의 특이한 점
==============

백엔드에서 로그인이나 회원가입 API를 만들 필요가 없었습니다.

프론트엔드에서 Firebase를 이용하여 이메일 또는 구글 로그인으로 인증을 진행하고, 그 결과로 얻은 idToken만 Authorization 헤더에 담아서 백엔드 서버에 요청을 보냅니다.

그러면 서버에서 토큰을 추출 후, Firebase SDK을 이용하여 유효한 토큰인지 확인합니다. 토큰이 유효하면 이후 DB에 요청을 보내서 **해당 회원이 있으면 회원을 반환(로그인), 없으면 회원 생성 후 반환(자동으로 회원가입)**합니다.

그리고 이후 로그인된 사용자 정보가 필요하면 ArgumentResolver를 통해 주입했습니다.

이러한 프로세스가 이상하다고 느끼실 수 있지만

여기서 핵심인 부분을 코드로 설명하면

> Repository의 find메서드로 Optional<회원> 객체를 반환하고,
> 없으면 orElseGet()을 통해서 새로운 회원객체의 생성과 저장 그리고 반환 로직을 수행합니다.

**이 로직이 하나의 과정으로 묶여(원자화)** 자동으로 회원가입이 이루어지고 로그인 처리가 되기를 원했습니다.

하지만 구현 도중 회원가입이 중복으로 처리되어 동일한 회원이 DB에 중복해서 저장되는 문제가 발생했고 이 문제를 트랜잭션 처리를 하지 않아 발생한 것으로 생각했습니다.

그러나 트랜잭션 처리를 했음에도 여전히 중복 저장 문제는 발생했습니다.
분명 같은 트랜잭션 내에서 조회와 저장을 수행했는데도 왜 이 과정이 원자성을 보장받지 못했는지 그리고 어떤 이유 때문인지 살펴보도록 하겠습니다.

먼저, 이야기의 이해를 위해 백엔드에서 Interceptor 을 이용하여 토큰을 검증하는 구조를 간단히 살펴보겠습니다
=================================================================

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*-kaDCmrfC2X_zYPfQPhEbA.png)
```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;
    
    @Column
    private String username;
    
    @Column(nullable = false)
    private String email;
    
    @Column
    private String uid;
}
```

Member 엔티티는 위와 같이 구성했습니다.

uid 필드는 Firebase 가 사용자를 식별하기 위해 제공하는 id 값입니다.
프로젝트의 DB에서도 uid 를 통해 각 멤버를 구분합니다.

```java
public class FirebaseTokenInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        
        // 헤더값 추출
        String token = getAuthorizationToken(request);
        
        // 토큰 검증
        FirebaseToken decodedToken = decodeToken(token);
        
        // 검증된 토큰을 HttpServletRequest 객체에 저장함. 
        request.setAttribute("decodedToken", decodedToken);
        return true;
    }
    
    // 요청에서 Authorization 헤더 값(idToken)을 추출합니다.
    private static String getAuthorizationToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) {
            throw new IllegalArgumentException("No Token or Invalid Token.");
        }
        return header.split(" ")[1];
    }

    // firebase 서버에 토큰을 보내어 검증합니다.
    private static FirebaseToken decodeToken(String token) {
        FirebaseToken decodedToken;
        try{
            // idToken 검증.
            decodedToken = FirebaseAuth.getInstance().verifyIdToken(token);
        } catch (FirebaseAuthException e) {
            throw new IllegalArgumentException("Invalid Token.");
        }
        return decodedToken; // 검증된 토큰의 결과 반환.
    }
}
```

Interceptor에서 Firebase SDK를 이용하여 Authorization헤더에 있는 토큰의 유효성을 검증합니다.

검증된 FirebaseToken타입의 decodedToken 에는 **유저의 정보**가 들어있고 그 값을 HttpServletRequest 에 저장합니다.
HttpServletRequest 에 저장한 decodedToken 은 ArgumentResolver 에서 필요하면 꺼내쓰기 위해 저장하였습니다.

```java
@RequiredArgsConstructor
public class LoginMemberArgResolver implements HandlerMethodArgumentResolver {
    
    private final MemberRepository memberRepository;
    
    // @Login 애너테이션이 있으면 값을 주입한다.
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        boolean hasLoginAnnotation = parameter.hasParameterAnnotation(Login.class);
        boolean hasMemberType = Member.class.isAssignableFrom(parameter.getParameterType());
        return hasLoginAnnotation && hasMemberType;
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
      HttpServletRequest request = (HttpServletRequest) webRequest.getNativeRequest();
      
      // request 객체에서 decodedToken 획득
      FirebaseToken decodedToken = (FirebaseToken) request.getAttribute("decodedToken");
      
      // AuthService를 호출
      return authService.joinAndLogin(decodedToken); 
    }
}
```

```java
@RequiredArgsConstructor
@Service
public class AuthService {

    private final MemberRepository memberRepository;

    public Member joinAndLogin(FirebaseToken decodedToken) throws InterruptedException {
        String uid = decodedToken.getUid();

        // uid 로 멤버를 조회하고 DB에 등록되어 있지 않은 uid 라면,
        // 새로운 멤버를 만들어 자동 회원가입이 됩니다.
        return memberRepository.findByUid(uid)
                .orElseGet(() -> memberRepository.save(Member.builder()
                        .username(decodedToken.getName())
                        .email(decodedToken.getEmail())
                        .uid(uid)
                        .build()));
    }
}
```

```java
// Controller
// 파라미터에 @Login 애너테이션이 있으니
// HandlerMethodArgumentResolver을 이용하여 Member을 초기화한다.
@PostMapping("/post/new")
public Post createPost(@RequestBody Post post, @Login Member member) {
 ...
}
```

ArgumentResolver 은 위와 같이 컨트롤러의 매개변수에 @Login 애너테이션이 있으면 로그인된 멤버를 주입하도록 했습니다.

decodedToken에는 Firebase에서 관리하는 유저식별 Id(앞서 설명한 uid) 정보가 있습니다.

이것을 이용하여 프로젝트의 DB에서 **사용자를 조회하고 반환. 만약, 해당하는 사용자가 없다면 새로운 Member 엔티티를 생성하여 DB 테이블에 멤버를 저장하고 반환합니다.**

즉, 사용자가 애플리케이션을 “처음" 사용할 때 회원가입이 자동으로 이루어집니다.

애플리케이션은 모든 요청에 로그인 된 멤버만 사용할 수 있다는 요구사항을 가집니다. 따라서 모든 API 요청에는 Authorization 헤더값이 있습니다.

문제 상황
=====

안드로이드 client 와 연결해서 애플리케이션을 실제로 사용하는 도중 때때로 로그인이 제대로 안되는 문제가 발생했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*8qzc68VSgKK4vVnWKTeNjg.png)

왜 그런지 요청 로그를 모두 살펴보았을 때 위와 같은 로그가 나타났습니다.

(이미지는 과거 프로젝트의 로그라서 현재의 구성과 조금 다릅니다!)

문제의 상황은

“**한명의 사용자가 첫 애플리케이션 실행에서 여러개의 요청을 동시에 발생시켰을 때**” 였습니다.

그리고 그 결과

> **DB 에 멤버가 존재하면 그대로 반환하고 없으면 해당 멤버 엔티티를 생성, DB저장 후 반환한다.**

라는 작업이 **하나의 작업**으로 처리되지 못했기 때문에 데이터베이스에는 아래와 같이 2개의 동일한 uid를 가진 멤버가 저장되었습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Eqrf67CvAX0q0YmqwxZO3w.png)

이후 사용자가 **새로운 요청**을 보내면 MemberRepository의 findByUid() 메서드는 한명의 멤버가 아닌, 2개의 멤버 레코드를 조회하여 로그인이 제대로 안되는 **에러상황**이 발생했습니다.

원인을 하나씩 살펴보겠습니다
===============

아래의 코드는 위에서 보여드린 AuthService 클래스입니다.

```java
@RequiredArgsConstructor
@Service
public class AuthService {

    private final MemberRepository memberRepository;

    public Member joinAndLogin(FirebaseToken decodedToken) throws InterruptedException {
        String uid = decodedToken.getUid();

        // memberRepository에서 uid 로 멤버를 조회하고 DB에 등록되어 있지 않은 uid 라면,
        // 새로운 멤버를 만들어 자동 회원가입이 됩니다.
        return memberRepository.findByUid(uid)
                .orElseGet(() -> memberRepository.save(Member.builder()
                        .username(decodedToken.getName())
                        .email(decodedToken.getEmail())
                        .uid(uid).build()));
    }
}
```

Spring 프레임워크는 하나의 request 요청을 하나의 스레드가 담당합니다.
따라서 앞선 스크린 샷에 나타난 2개의 요청은 서로 다른 스레드가 처리를 합니다.

2개의 요청이 모두 동일한 클라이언트의 요청인 경우 예를 들겠습니다.

**첫 번째 스레드**가 memberRepository.findByUid() 메서드를 통해 데이터를 조회하는 동안 **다른 스레드**도 memberRepository.findByUid() 메서드를 실행하고 동일한 uid로 findByUid() 을 수행할 수 있습니다.

이를 방지하기 위해서는

> **DB 에 멤버가 존재하면 그대로 반환하고 없으면 해당 멤버 엔티티를 생성, DB저장 후 반환한다.**

과정을 하나의 트랜잭션으로 묶고 uid컬럼은 멤버를 유일하게 식별하기위해 **Uique 제약**이 필요하다는 판단이 들었습니다.

> 그렇다면 @Transactional 애너테이션을 붙이면 되겠구나?

```java
@RequiredArgsConstructor
@Service
public class AuthService {
    private final MemberRepository memberRepository;
     
    // @Transactional 추가!
    @Transactional
    public Member joinAndLogin(FirebaseToken decodedToken) throws InterruptedException {
        String uid = decodedToken.getUid();
        // uid 로 멤버를 조회하고 DB에 등록되어 있지 않은 uid 라면,
        // 새로운 멤버를 만들어 자동 회원가입이 됩니다.
        return memberRepository.findByUid(uid)
                .orElseGet(() -> memberRepository.save(Member.builder()
                        .username(decodedToken.getName())
                        .email(decodedToken.getEmail())
                        .uid(uid).build()));
    }
}
```

하지만 동시성을 테스트해보니 여전히 똑같은 문제가 해결되지 않았습니다.

트랜잭션이 먼저 원자적으로 처리되는 것을 확인하기 위해 Unique 제약조건은 아직 추가하지 않았습니다. Unique 제약 조건이 없더라도 원자적으로 처리된다면 중복해서 저장되지 않을 것이라 생각했습니다.

트랜잭션이 동작하는 것은 분명히 확인했지만, 문제가 해결되지 않았습니다.

> 아! 트랜잭션이 원자성을 제공하지만 이것이 하나의 스레드가 findByUid() 를 호출할 때, 다른 스레드가 findByUid() 호출을 막는 것은 아니였지!

라는 생각도 들어 **synchronized 키워드**도 추가해 보았습니다.

> 그리고 지금부터 로그를 찍어보며 자세히 살펴보기 위해 findByUid() ~ orElseGet() 메서드를 아래와 같이 Optional<Member>로 풀어서 작성하겠습니다.

```java
@RequiredArgsConstructor
@Service
public class AuthService {
    private final MemberRepository memberRepository;
    
    // syschronized 적용
    @Transactional
    public synchronized Member joinAndLogin(FirebaseToken decodedToken) throws InterruptedException {
        String uid = decodedToken.getUid();

        // uid 로 멤버를 조회하고 DB에 등록되어 있지 않은 uid 라면,
        // 새로운 멤버를 만들어 자동 회원가입이 됩니다.
        Optional<Member> memberOptional = memberRepository.findByUid(uid);
        if (memberOptional.isPresent()) {
            log.info("기존 회원입니다.");
            return memberOptional.get();
        }

        log.info("새로운 회원입니다.");
        Member member = Member.builder()
                .username(decodedToken.getName())
                .email(decodedToken.getEmail())
                .uid(uid).build();
        memberRepository.save(member);
        return member;
    }
}
```

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*9h6afD8H1cOZ65WOKz0tiQ.png)
![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*igpmG-cUwvehvgVLDLw8ng.png)

**synchronized 키워드를 추가해도** 여전히 문제가 해결되지 않았습니다.
==============================================

이 과정에서 락도 걸어보고 여러 삽질 끝에 생각난 의문들…
--------------------------------

> 1. MySQL 은 레코드 기반으로 락(lock)을 거는데, 레코드가 없으면 락 자체를 못거는 것이 아닌가? 쉽게 말해, 첫 findByUid() 에 해당하는 레코드가 없으니 트랜잭션을 시작하더라도 락을 걸거나 레코드의 변경 로그를 저장할 대상 자체가 없는 것이 아닌가?
> 
> 2. findByUid() 메서드를 호출할 때 트랜잭션이 시작한다고 생각했는데… 로그가 찍힌 것을 보니 무언가 이상하다… 그러면 트랜잭션의 시작이 어디지? 5개의 요청을 동시에 보냈을 때, 트랜잭션이 동시에 시작하는건가? 그렇다면… JPA 는 DB의 기본 트랜잭션 Isolation level을 선택하고 나는 MySQL 을 사용하니까 Repeatable Read 에 해당하는 Isolation Level 이 기본으로 적용되었을 것이고…
> 
> 3. 트랜잭션의 시작 시점이 AuthService 의 joinAndLogin() 이 아니라 AOP **프록시**의 invoke() 에 의해 트랜잭션이 실행된다면… 실제 joinAndLogin() 메서드가 실행되기 전에 5개의 요청이 모두 동시에 진행되다가 synchronized 키워드에 의해 실제 객체의 메서드만 순차적으로 실행된 것인가…?

생각난 의문점들을 정리해서 살펴보겠습니다.
=======================

1. 트랜잭션이라는 것은 하나의 DB 트랜잭션 세션 내에서 ACID 를 보장하는 것입니다.

이 말은 곧, Java 애플리케이션 코드의 원자성을 보장하지는 않습니다.

> **DB 에 멤버가 존재하면 그대로 반환하고 없으면 해당 멤버 엔티티를 생성, DB저장 후 반환한다.**

위의 Java 코드의 메서드를 synchronized 키워드로 원자화하는 것은 DB 트랜잭션의 원자성과는 별개의 과정입니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*BELe6IrRy4GlU110GvxxUg.png)

MySQL의 **Repeatable Read Isolation Level** 에서는 MVCC(Multi Version Concurrency Control) 를 이용하여 트랜잭션 시작 시점(5번)의 레코드 스냅샷(undo 로그)을 가지고 있습니다.

5개의 트랜잭션이 동시에 시작할 때, 하나의 트랜잭션이 9번에서 커넥션 획득 이후 findByUid()의 실제 SQL쿼리를 날리기도 전에 모두 트랜잭션을 시작했습니다.(5번 작업)

따라서 findByUid() 메서드의 결과는 "**해당하는 레코드가 존재하지 않습니다**"라는 동일한 결과를 나타낼 것입니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*tt2ahLjqb_pL8DhVSQcFHA.png)

MySQL의 MVCC는 하나의 트랜잭션이 먼저 조회 후 멤버 객체를 저장하고 커밋을 하더라도, 다른 트랜잭션들의 결과는 **본인이 시작하던 시점 기준**으로 그 이전에 기록된 데이터들의 **스냅샷**을 바라봅니다.

![synchronized 적용](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*P_-Ws8dBq-rhJ855bgyitw.png)

MySQL의 MVCC와 Transaction의 고립 레벨을 조금 더 실감하기 위해서 joinAndLogin() 메서드에 synchronized 키워드를 적용한 경우를 살펴보겠습니다.

트랜잭션1이 커밋(commit)을 했음에도 트랜잭션2의 조회결과는 “결과 없음” 이라는 결과가 나타납니다. 결과적으로 트랜잭션2는 자신의 스냅샷 기반으로 데이터를 조회하므로, 새로 삽입된 데이터를 볼 수 없습니다.

이때, 트랜잭션1이 커밋되고 트랜잭션2가 조회할 때 데이터가 보이게 하기 위해 Repeatable Read Isolation Level보다 한 단계 낮은 Read Committed Isolation Level을 적용하면 새로 삽입된 데이터를 확인할 수 있을 것이라 예상할 수 있습니다.

하지만 synchronized 키워드는 프록시의 메서드에 상속되지 않습니다.

따라서 스레드1이 실제 joinAndLogin()메서드의 임계영역(critical section)을 빠져나오고 프록시 객체에서 트랜잭션을 커밋하기 전에 또 다른 스레드(트랜잭션을 시작하고 synchronized 영역을 기다리는)가 실제 joinAndLogin()메서드에 진입할 수 있습니다. 이러한 이유로 Read Committed Isolation Level을 적용하더라도 커밋한 데이터를 읽지 못할 수 있습니다.

만약, 그럼에도 커밋된 데이터를 읽는 결과를 확인해 보고 싶다면 제일 낮은 트랜잭션 고립 레벨인 **Read Uncommitted Isolation Level**을 적용하여 추가된 멤버 데이터를 볼 수 있습니다.

이 경우 데이터가 저장되면 커밋하지 않았더라도 다른 트랜잭션에서 저장된 데이터를 바로 볼 수 있습니다.

2. 동일한 uid 값에 대한 조회와 저장에서 Lock을 사용할 수 있지 않나?

MySQL 의 InnoDB 는 **조회** 시 기본적으로 락(Lock)이 아닌 MVCC 로 Repeatable Read 을 처리하지만 아래와 같이 락을 걸 수 있습니다.

```sql
SELECT * FROM MEMBER WHERE uid="ASDW12SD3" FOR UPDATE; // 쓰기 잠금
SELECT * FROM MEMBER WHERE uid="ASDW12SD3" FOR SHARE;  // 읽기 잠금
```

따라서 현재 트랜잭션에서 락을 걸고 조회를 할 때, 다른 트랜잭션에서 임의의 데이터가 추가되지 않도록 MySQL의 레코드락, 넥스트키락등이 적용될 수 있습니다. 하지만 InnoDB의 락은 인덱스를 기준으로 락을 거는데 조회할 레코드가 없다? 인덱스도 존재하지 않습니다. 즉 조회할 때 락을 걸 수 없는 상황입니다.

그렇다면 어떻게 해결할 것인가?
=================

사실 지금까지의 과정은 모든 것이 정상적으로 동작하면서 발생한 것입니다. 단지, 프로젝트의 요구사항을 만족시키는 과정에서 애플리케이션 레벨에서 특정 로직을 하나의 과정으로 처리하려 했으나 의도한 결과가 나타나지 않았습니다. 그리고 그 과정을 단순히 @Transactional을 추가하여 해결하려 했던 것이 문제였습니다.

약간의 테스트로 애플리케이션 레벨에서 원자성을 보장하는 것을 확인하려면, AuthService클래스에 @Transactional 애너테이션을 제거하여 Spring AOP가 적용되지 않도록하고 synchronized 키워드를 사용하여 애플리케이션 코드를 원자화하면 됩니다.

```java
@RequiredArgsConstructor
@Service
public class AuthService {

    private final MemberRepository memberRepository;
    
    // synchronized 적용
    public synchronized Member joinAndLogin(FirebaseToken decodedToken) throws InterruptedException {
        String uid = decodedToken.getUid();

        // uid 로 멤버를 조회하고 DB에 등록되어 있지 않은 uid 라면,
        // 새로운 멤버를 만들어 자동 회원가입이 됩니다.
        Optional<Member> memberOptional = memberRepository.findByUid(uid);
        if (memberOptional.isPresent()) {
            log.info("기존 회원입니다.");
            return memberOptional.get();
        }

        log.info("새로운 회원입니다.");
        Member member = Member.builder()
                .username(decodedToken.getName())
                .email(decodedToken.getEmail())
                .uid(uid).build();
        memberRepository.save(member);
        return member;
    }
}
```

이 경우 트랜잭션을 사용하지 않고도 대부분 정상 동작하겠지만, 예기치 않은 상황에서 데이터 무결성을 보장하기 위해 트랜잭션을 사용하는게 좋습니다. 적절한 해결책은 아닙니다.

저의 경우 간단한 해결책으로 uid 로 멤버를 구분한다는 요구사항에 맞게 데이터베이스 “uid 컬럼”에 대한 유니크(Unique) 제약 조건을 추가했습니다.

그리고 중복 uid값을 갖는 멤버 저장이 발생하면 예외를 던지도록 처리를 했습니다. 이후 예외 발생 시 재시도 하는 방식으로 처리하여 에러상황 없이 자동 회원가입과 로그인이 정상적으로 작동하도록 했습니다.

추가로 Unique 제약조건은 조회 시 인덱스를 활용하여 더 빠른 성능을 낼 수 있습니다!

```gradle
// build.gradle 에 추가
implementation "org.springframework.retry:spring-retry"
```

```java
@EnableRetry // 이 부분 추가
@SpringBootApplication
public class FirebaseApplication {}

/.../

@RequiredArgsConstructor
@Service
public class AuthService {

    private final MemberRepository memberRepository;

    // 요 부분도 추가!
    @Retryable(
            retryFor = {DataIntegrityViolationException.class},
            backoff = @Backoff(delay = 1000)
    )    
    @Transactional
    public Member joinAndLogin(FirebaseToken decodedToken) throws InterruptedException {
        String uid = decodedToken.getUid();

        // uid 로 멤버를 조회하고 DB에 등록되어 있지 않은 uid 라면,
        // 새로운 멤버를 만들어 자동 회원가입이 됩니다.
        Optional<Member> memberOptional = memberRepository.findByUid(uid);
        if (memberOptional.isPresent()) {
            log.info("기존 회원입니다.");
            return memberOptional.get();
        }

        log.info("새로운 회원입니다.");
        Member member = Member.builder()
                .username(decodedToken.getName())
                .email(decodedToken.getEmail())
                .uid(uid).build();
        memberRepository.save(member);
        return member;
    }
}
```

하지만 정말 문제가 해결되었을까요?
===================

먼저, 재시도를 하는 과정에 문제점이 있습니다.
--------------------------

DataIntegrityViolationException은 Unique 키 위반 예외가 아닌 다른 예외의 경우도 발생할 수 있습니다.

예를 들어 외래키 제약 조건 위반, 데이터 타입 불일치, Not NULL 제약 조건 위반 등에서도 DataIntegrityViolationException이 발생할 수 있습니다.

이 경우 계속해서 재시도 로직이 수행될 수 있습니다.

사실 더 근본적인 문제가 있습니다.
-------------------

위의 과정은 프로세스 자체가 잘못되었습니다.

문제가 되는 상황에서 어떻게든 요구사항을 구현 레벨에서 풀려고 하다보니 불필요한 재시도 로직까지 처리하게 되었습니다.

지금의 API 들은 회원을 검증하는 부분과 비즈니스를 수행하는 부분이 하나의 API에서 처리되고 있습니다. 따라서 검증에 실패하면 회원 검증을 재시도하고 비즈니스 로직을 수행하도록 되어있습니다.

이 부분에서 프로세스를 분리한다면 불필요한 재시도 로직도 필요하지 않았을 것입니다.

먼저 정상적인 케이스로 프로세스를 바꾼다면, 기존에는 자동 회원가입 또는 로그인 API가 필요없다 생각하여 만들지 않았지만 이제는 만들어야합니다.

그리고 자동 회원가입 또는 로그인 API를 호출하고, 동시에 들어온다면 하나는 실패처리를 하면 됩니다.

그 후 비즈니스 로직 API를 호출하면 되는 문제였습니다.

수 개월이 지난 프로젝트에서 나타난 문제지만 해결을 위해 트랜잭션과 DB 등을 공부해 나가다 뒤늦게 근본적인 문제점까지 발견했을 땐, 너무나 비정상적인 프로세스여서 그 때 당시 제대로 만들지 못한 것에 대한 아쉬움이 남기도 합니다.

그럼에도 의의를 두자면 애플리케이션 레벨의 원자성과 트랜잭션에 대해서 조금 더 이해하게된 계기를 만들어준 문제였습니다.