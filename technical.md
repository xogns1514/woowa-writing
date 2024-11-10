# 스프링 OSIV와 **데이터 소스 라우팅 오류 해결하기**

안녕하세요. 우아한테크코스 6기 백엔드 러쉬입니다. 

현재 `크루루`는 동아리 리크루팅 서비스를 개발하고 있습니다. 기존에 운영 중이던 크루루 서비스는 단일 데이터베이스 인스턴스에 의존하고 있었습니다. 단일 인스턴스 사용에는 `SPOF`의 위험이 있어, 이 문제를 해결하기 위해 데이터베이스를 스케일아웃 했습니다.

그러나 데이터베이스 스케일 아웃 과정에서 `OSIV(Open Session in View)`와 관련된 문제가 발생했습니다. 이번 글에서는 OSIV의 개념을 설명하고, 크루루 서비스에서 OSIV 문제를 해결하기 위한 트러블슈팅 과정을 공유하겠습니다.

<aside>
💡

OSIV 패턴을 이해하기 위해서는 하이버네이트(Hibernate) 매커니즘을 알아야 합니다. 해당 글을 읽기 전에 하이버네이트 엔티티의 생명주기에 대해 학습하신 뒤 읽는 것을 추천합니다.

</aside>

### OSIV란?

`OSIV(Open Session in View)`는 영속성 컨텍스트를 클라이언트 요청이 완료될 때까지, 즉 뷰 영역까지 유지하는 전략입니다. 여기서 뷰는 스프링 MVC의 뷰(View) 단계를 의미하며, OSIV는 영속성 컨텍스트를 뷰 단계까지 열어두어 뷰에서 엔티티를 사용할 수 있도록 합니다. 이 전략은 Hibernate와 JPA에서 데이터베이스 세션을 관리할 때 발생할 수 있는 지연 로딩 문제를 해결하기 위해 도입되었습니다.

이 설명만으로는 OSIV의 필요성과 동작 방식을 충분히 이해하기 어려울 수 있습니다. OSIV가 왜 등장하게 되었는지, 그리고 어떤 문제를 해결하기 위해 사용되는지, 그 배경을 예시와 함께 설명하겠습니다.

### OSIV 등장 배경

OSIV 패턴이 등장하기 전에는 영속성 컨텍스트가 뷰 렌더링 시점에 종료되기 때문에, 지연 로딩된 엔티티에 접근할 수 없었습니다. 이에 따라  `LazyInitializationException` 과 같은 예외가 발생했습니다. 아래 예시 코드는 이러한 문제 상황을 보여줍니다.

```java
// 도메인 계층
@Entity
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Comment> comments;
}

// 서비스 계층
@Service
public class PostService {
    private final PostRepository postRepository;

    public PostService(PostRepository postRepository) {
        this.postRepository = postRepository;
    }

    @Transactional
    public Post getPostById(Long id) {
        return postRepository.findById(id).orElse(null);
    }
}

// 컨트롤러 계층
@Controller
public class PostController {
    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @GetMapping("/post/{id}")
    public String getPost(@PathVariable Long id, Model model) {
        Post post = postService.getPostById(id);
        model.addAttribute("post", post);
        return "postView"; // postView.jsp 또는 postView.html
    }
}

// view - postView.html
<c:forEach items="${post.comments}" var="comment">
    <li>${comment.content}</li>
</c:forEach>                          
```

위 코드에서 Post 엔티티와 Comment 엔티티 간에 지연 로딩이 설정되어 있습니다. 이제 Post를 조회하는 API가 호출된다고 가정하겠습니다. 

 서비스 계층에서 Post를 조회하는 `getPostById` 메서드는 `@Transactional` 이 적용되어 있습니다. 따라서 메서드가 호출되면 세션이 열리고 트랜잭션이 시작됩니다. 이 트랜잭션이 종료되면 커밋과 함께 세션이 닫히고, 영속성 컨텍스트가 종료되면서 엔티티는 준영속 상태가 됩니다. 

 문제는 뷰 렌더링 시점에 발생합니다. `post.comments` 필드는 지연 로딩 설정으로 인해 실제 comments 데이터를 세션이 열려있는 동안에만 가져올 수 있습니다. 하지만, 트랜잭션 종료 후 세션이 닫힌 준영속 상태에서는 post.commnets에 접근할 수 없습니다. 결과적으로 `LazyInitializationException` 예외가 발생하게 됩니다. 

                                                   

Post 조회 요청 실행 경로와, 세션 및 트랜잭션의 유지를 그림으로 나타내면 다음과 같습니다.

![트랜잭션 범위의 영속성 컨텍스트](https://github.com/user-attachments/assets/ab844a30-a722-45cb-b2c3-26e7406ba8fe)

OSIV는 이와 같은 문제를 해결하고자 도입된 전략입니다. 뷰 렌더링 시점까지 영속성 컨텍스트를 유지하여 필요한 데이터를 조회할 수 있도록합니다. 

## 스프링 OSIV

 스프링 프레임워크의 OSIV는 트랜잭션을 비즈니스 계층에서만 사용하도록 설정됩니다. 즉, OSIV로 영속성 컨텍스트는 요청 전반에 걸처 열려 있지만, 실제 데이터 변경 작업(트랜잭션)은 비즈니스 계층에서만 이루어집니다. 

### 스프링 프레임워크 OSIV 동작과정

![스프링 OSIV 패턴이 적용된 영속선 컨텍스트](https://github.com/user-attachments/assets/2dad6093-e1f4-48d2-b1de-7a5e0bfbc833)

다음은 스프링 프레임워크에서 OSIV가 동작하는 과정입니다. 

1. 클라이언트의 HTTP 요청이 들어오면 서블릿 필터 또는 스프링 인터셉터에서 요청을 가로채 영속성 컨텍스트를 생성합니다. 이때 트랜잭션은 시작되지 않습니다.
2. 서비스 계층의 `@Transactional`이 적용된 메서드가 실행되면 영속성 컨텍스트에 트랜잭션이 연결됩니다.
3. 서비스 계층의 비즈니스 로직 실행이 끝나면 트랜잭션이 커밋되고 영속성 컨텍스트가 플러시됩니다. 트랜잭션은 종료되지만, 영속성 컨텍스트는 남아있습니다.
4. 컨트롤러와 뷰까지 영속성 컨텍스트가 유지되어, 뷰에서 지연 로딩을 시도해도 안전하게 데이터를 가져올 수 있습니다.
5. 서블릿 필터 또는 인터셉터가 요청을 마무리할 때 영속성 컨텍스트를 종료합니다.

이와 같은 OSIV 동작 방식으로 인해, 지연 로딩이 필요한 데이터를 뷰 단계에서도 안전하게 사용할 수 있게 되었습니다. 트랜잭션이 종료된 이후에도 영속성 컨텍스트가 유지되므로, 뷰에서 엔티티의 지연 로딩 필드에 접근할 때 LazyInitializationException과 같은 예외가 발생하지 않으며, OSIV가 지연 로딩 문제를 해결해 줍니다.

## 스프링 OSIV와 데이터 소스 라우팅 문제

### 데이터 소스 라우팅 문제 상황

```java
public class DataSourceRouter extends AbstractRoutingDataSource {

    public static final String READ_DATASOURCE_KEY = "read";
    public static final String WRITE_DATASOURCE_KEY = "write";

    @Override
    protected Object determineCurrentLookupKey() {
        if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            return READ_DATASOURCE_KEY;
        }
        return WRITE_DATASOURCE_KEY;
    }
}
```

크루루 서비스에서는 단일 데이터베이스 인스턴스에 의존할 경우 발생할 수 있는 `SPOF` 문제를 해결하기 위해 데이터베이스를 읽기 전용과 쓰기 전용 인스턴스로 분리했습니다. 이를 통해, 읽기 전용 트랜잭션은 읽기 전용 데이터베이스로 라우팅 되고, 쓰기 트랜잭션은 쓰기 전용 데이터베이스로 라우팅 되도록 설정하였습니다.

운영 서버에 배포한 뒤 API를 호출하여 테스트했을 때, 읽기 전용 데이터베이스에서 쓰기 작업을 시도하여 다음과 같은 오류가 발생했습니다.

![databaseError.png](https://github.com/user-attachments/assets/3a28f309-1c3d-4108-8bcc-dd59e5fedab9)

```bash
[The MySQL server is running with the --read-only option so it cannot execute this statement
[insert into evaluation (appicant_id, content, created_date, process_id, score, updateed_date)
caluse (?,?,?,?,?,?)]]
```

운영 서버에 배포 후 API를 호출하여 테스트를 진행하던 중, `The MySQL server is running with the --read-only option so it cannot execute`라는 오류가 발생했습니다. 이 오류는 읽기 전용 데이터베이스에서 쓰기 작업을 시도하면서 발생한 것입니다. 문제를 명확히 파악하기 위해 디버깅을 진행했습니다.

### 문제 분석

아래와 같이 `determineCurrentLookupKey` 메서드에서 데이터 소스가 선택될 때마다 콘솔에 현재 라우팅 된 데이터 소스를 출력하도록 설정했습니다.

```java
  @Override
    protected Object determineCurrentLookupKey() {
        if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
            System.out.println("--------READ------------");
            return READ_DATASOURCE_KEY;
        }
        System.out.println("--------WRITE------------");
        return WRITE_DATASOURCE_KEY;
    }
```

![image](https://github.com/user-attachments/assets/0c22894a-f97f-4681-ab54-40c90c36e652)

이후, 쓰기 작업이 포함된 대시보드 만들기 API를 호출해 보았습니다. 로그를 확인한 결과, 처음에 시작된 read-only 트랜잭션이 읽기 전용 데이터베이스로 라우팅 되었으며, 이후 insert 작업이 시작되었음에도 데이터베이스 라우팅이 변경되지 않았음을 확인할 수 있었습니다.

즉, 요청 시작 시 설정된 데이터베이스 커넥션이 요청이 끝날 때까지 유지되었고, 중간에 발생한 쓰기 작업에 대해서도 여전히 읽기 전용 데이터베이스에 연결된 상태로 남아 있었습니다. 이는 트랜잭션이 매번 새로운 커넥션을 생성하지 않고, **처음에 설정된 커넥션을 재사용**하기 때문에 발생한 문제로 보였습니다. 그렇다면 왜 트랜잭션마다 커넥션이 새로 생성되지 않고, 초기 커넥션이 그대로 유지되는 걸까요?

```java
@RestController
@RequestMapping("/v1/dashboards")
@RequiredArgsConstructor
public class DashboardController {

    private final DashboardFacade dashboardFacade;

    @PostMapping
    // 커스텀 어노테이션을 통해 인가 대상을 알려준다.
    @RequireAuthCheck(targetId = "clubId", targetDomain = Club.class)
    public ResponseEntity<DashboardCreateResponse> create(
            @RequestParam(name = "clubId") Long clubId,
            @RequestBody @Valid DashboardCreateRequest request,
            LoginProfile loginProfile
    ) {
        DashboardCreateResponse dashboardCreateResponse = dashboardFacade.create(clubId, request);
        return ResponseEntity.created(URI.create("/v1/dashboards/" + dashboardCreateResponse.dashboardId()))
                .body(dashboardCreateResponse);
    }
```

```java
@Aspect
@Component
@RequiredArgsConstructor
public class AuthCheckAspect {

    private static final String SERVICE_IDENTIFIER = "Service";

    private final ApplicationContext applicationContext;  // 서비스 빈을 동적으로 가져오기 위해 ApplicationContext 사용
    private final MemberService memberService;

		// Controller 계층에서 해당 요청에 대한 인가를 진행합니다. 
    @Before("@annotation(com.cruru.auth.annotation.RequireAuthCheck)")
    public void checkAuthorization(JoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        RequireAuthCheck authCheck = method.getAnnotation(RequireAuthCheck.class);
				
				... 생략
        authorize(domainClass, targetId, loginProfile);
    }
    
     private void checkAuthorizationForTarget(
            Class<? extends SecureResource> targetDomain,
            Long targetId,
            LoginProfile loginProfile
    ) throws Exception {
		    // 현재 로그인한 회원을 조회합니다.
        Member member = memberService.findByEmail(loginProfile.email());
        ... 
    }
```

크루루 서비스에서는 AOP를 사용하여 전역적인 인가 처리를 컨트롤러 계층에서 수행하고 있습니다. 인가 처리의 첫 단계는 현재 로그인한 사용자를 조회하는 작업이며, 이 작업은 읽기 전용 트랜잭션으로 수행됩니다. 따라서 요청의 초기 데이터베이스 커넥션은 읽기 전용 데이터베이스로 설정됩니다.

문제는 이 읽기 전용 트랜잭션이 종료된 후에도 영속성 컨텍스트가 계속 유지되면서, 요청 중간에 쓰기 작업이 발생해도 데이터베이스 커넥션이 처음 설정된 상태로 유지된다는 점입니다. 결과적으로, 쓰기 작업을 시도할 때도 여전히 읽기 전용 데이터베이스에 연결된 상태에서 작업이 이루어지게 됩니다.

이러한 현상은 트랜잭션이 시작될 때 데이터 소스가 결정되기 때문에 발생합니다. 즉, 요청 중간에 트랜잭션의 유형이 변경되더라도 데이터 소스는 새로 지정되지 않고 초기 설정을 그대로 유지하므로, 쓰기 작업이 읽기 전용 데이터베이스에 연결되는 문제가 발생하는 것입니다.

### 해결 방법

이 문제를 해결하기 위해 두 가지 접근 방안을 고려했습니다.

1. OSIV 활성화 상태에서 트랜잭션 전파 속성을 설정하기
2. OSIV 비활성화

**1. OSIV 활성화 상태에서 트랜잭션 전파 속성을 설정하기**

OSIV를 활성화한 상태에서, 트랜잭션 전파 속성을 Propagation.REQUIRES_NEW로 설정하여 쓰기 작업을 별도의 트랜잭션으로 분리하는 방법입니다. 이 설정은 항상 새로운 트랜잭션을 시작하므로, 기존 트랜잭션과 상관없이 데이터 소스를 새로 지정할 수 있습니다. 이로써, 인가 과정이 끝난 후 시작되는 쓰기 작업에서 읽기 전용 데이터베이스가 아닌 쓰기 전용 데이터베이스에 연결할 수 있습니다.

아래는 Propagation.REQUIRES_NEW 속성을 적용한 예시 코드와 데이터 소스 라우팅 결과입니다.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
    public Dashboard create(Club club) {
        Dashboard savedDashboard = dashboardRepository.save(new Dashboard(club));

        List<Process> initProcesses = ProcessFactory.createInitProcesses(savedDashboard);
        processRepository.saveAll(initProcesses);

        return savedDashboard;
    }
```

![image](https://github.com/user-attachments/assets/5d43b56b-235b-4826-a1ea-9bdefc2f9765)

이 설정을 통해 쓰기 작업 시 새로운 커넥션을 얻어, 쓰기 전용 데이터베이스로 라우팅되는 것을 확인할 수 있습니다. 이 방법은 OSIV를 활성화한 상태로 유지하기 때문에, 뷰에서 지연 로딩된 엔티티에 안전하게 접근할 수 있는 장점이 있습니다.

**2. OSIV 비활성화**

OSIV를 비활성화하면 영속성 컨텍스트가 서비스 계층까지만 유지되므로, 각 트랜잭션이 종료될 때마다 커넥션이 닫히고 이후의 작업에서는 새로운 트랜잭션을 통해 커넥션이 새로 생성됩니다. 이 방법을 적용하면 읽기 트랜잭션과 쓰기 트랜잭션이 각각 올바른 데이터베이스로 연결되므로 데이터 소스 라우팅이 제대로 작동하게 됩니다.

![image](https://github.com/user-attachments/assets/87a8eaf7-9eb1-4b83-8e71-071403019fba)

![image](https://github.com/user-attachments/assets/b4e84a00-7ac9-4e42-a32c-9c47e14d6e24)

하지만 OSIV를 비활성화할 경우 뷰에서의 지연 로딩이 제한되므로, `LazyInitializationException` 문제가 발생했습니다. 이를 해결하기 위해, OSIV를 비활성화한 상태에서도 엔티티를 로드할 수 있는 몇 가지 대안을 고려했습니다.

2-1. **Fetch Join**

Fetch Join은 JPQL에서 사용하는 기법으로, 연관된 엔티티를 한 번의 쿼리로 함께 가져오는 방식입니다. 이 방식은 복잡한 객체 관계를 가진 엔티티를 한 번에 로드할 수 있어 성능상 유리합니다.

```java
  	@Query("""
           SELECT d FROM Dashboard d 
           JOIN FETCH d.club c 
           JOIN FETCH c.member 
           WHERE d.id = :id
           """)
    Optional<Dashboard> findByIdFetchingMember(@Param("id") long id);
```

위 예시에서는 Dashboard 엔티티를 조회할 때 club과 member 연관 엔티티를 함께 로드하여, 프록시가 아닌 실제 객체로 데이터를 가져올 수 있습니다.

2-2. **@EntityGraph**

@EntityGraph는 JPA 애너테이션으로, JPQL 쿼리를 작성하지 않아도 특정 엔티티의 연관 데이터를 즉시 로드하도록 설정할 수 있습니다. 

```java
@EntityGraph(attributePaths = {"club.member"})
    @Query("SELECT d FROM Dashboard d WHERE d.id = :id")
    Optional<Dashboard> findByIdFetchingMember(long id);
```

위 코드에서는 @EntityGraph를 적용해 Dashboard를 조회할 때 club.member 관계를 명시적으로 로드하도록 설정했습니다.

2-3. **Eager Fetching**

Eager Fetching은 엔티티를 로드할 때 관련된 모든 엔티티를 즉시 로드하는 설정입니다. 예를 들어, @OneToMany나 @ManyToOne 관계에서 fetch = FetchType.EAGER로 설정하면 연관된 데이터를 한 번에 가져옵니다. 하지만 Eager Fetching은 모든 연관 데이터를 항상 로드하므로, 실제로 필요하지 않은 데이터까지 불필요하게 가져와 리소스가 낭비될 수 있습니다.


**최종 선택: @EntityGraph** 

 위의 두 가지 해결 방안(1. OSIV 활성화 상태에서 트랜잭션 전파 설정, 2. OSIV 비활성화)을 비교한 결과, **OSIV 비활성화** 방안을 선택했습니다. OSIV를 활성화한 상태에서 트랜잭션 전파 속성을 설정하는 방법도 유효했지만, 트랜잭션을 분리하고 관리하는 복잡도가 올라간다고 판단해 최종적으로 적용하지 않기로 했습니다.

 OSIV를 비활성화한 상태에서도 지연 로딩 문제를 해결할 수 있는 여러 방법을 추가로 비교한 결과, @EntityGraph 방식을 선택했습니다.

 @EntityGraph는 코드 내에서 복잡한 JPQL 쿼리를 작성하지 않아도 되므로 가독성이 높고, 필요한 연관 데이터만 명시적으로 지정해 로드할 수 있어 데이터베이스 성능 부하도 상대적으로 낮습니다. 또한, 유지보수와 코드의 단순성 측면에서도 Fetch Join보다 더 유리하다고 판단했습니다.

![image](https://github.com/user-attachments/assets/f39c3d81-1122-4168-bb30-f7931551cae3)

이전과 달리 실제 객체를 가져오는 것을 확인할 수 있습니다. 이로써 OSIV 비활성화 시 발생하는 Lazy Loading 문제를 해결했습니다.

## 결론

OSIV는 트랜잭션 종료 이후에도 영속성 컨텍스트를 유지하여, 뷰 단계에서 지연 로딩된 엔티티에 안전하게 접근할 수 있게 해주는 유용한 전략입니다. 하지만 데이터베이스 커넥션을 요청 완료 시점까지 유지해야 하므로, 예상치 못한 부작용이 발생할 수 있습니다.  크루루 서비스의 데이터 소스 라우팅 오류도 이러한 예 중 하나입니다. 

이번 트러블슈팅을 통해 OSIV의 동작 방식과 그에 따른 문제를 깊이 이해할 수 있었고, OSIV는 충분한 이해를 바탕으로 신중히 사용해야 한다는 것을 깨닫게 되었습니다.
