# OSIV(Open Session in View) 란 무엇인가?

---

안녕하세요. 우아한테크코스 6기 백엔드 러쉬입니다. 

현재 `크루루` 라는 리크루팅 서비스를 개발하고 있습니다. 기존 운영환경에 배포되어 있던 크루루 서비스는 단 하나의  데이터베이스 인스턴스를 사용하고 있었습니다. 하나의 데이터베이스 인스턴스만 이용했을 경우 SPOF라는 치명적인 단점이 존재합니다. 이를 해결하기 위해 기존 데이터베이스를 `스케일-아웃` 했습니다. 

데이터베이스를 스케일 아웃하는 과정에서 OSIV와 관련된 문제가 발생했습니다. 

이번 글에서 OSIV란 무엇이며, 크루루 서비스를 개발하면서 겪은 문제에 대해 살펴보겠습니다.

---

## 1. OSIV 이해를 위한 사전지식

OSIV 패턴을 이해하기 위해서는 하이버네이트(Hibernate) 매커니즘을 알아야합니다. OSIV를 이해하기 위한 사전지식을 간단히 짚고 넘어가겠습니다. 

### 1.1 하이버네이트 엔티티의 생명주기

하이버네이트 엔티티에는 다음 4가지의 상태가 존재합니다. 

- 비영속(new/trasient)
- 영속(managed)
- 준영속(detached)
- 삭제(removed)

<img width="563" alt="image" src="https://github.com/user-attachments/assets/4363cb54-ed4f-4db5-8427-30eee288bd59">

각 상태에 대해 간략히 설명하겠습니다. 

- 비영속 상태는 영속성 컨텍스나 데이터베이스와 전혀 관련이 없는 상태를 의미합니다.

```java
// 데이터베이스 트랜잭션 시작
EntityTransaction tx = entityManager.getTrasaction();
tx.begin();

// 객체를 생성한 상태(비영속)
Dashboard dashboard = new Dashboard();
dashboard.setId = 1L;
dashboard.setClub = aClub;
```

위처럼 순수한 객체 상태이고 아직 데이터베이스에 저장하지 않은 객체의 상태를 비영속 상태라고 합니다.

- 영속 상태는 영속성 컨텍스트가 관리하는 엔티티의 상태를 의미합니다.

```java
// 데이터베이스 트랜잭션 시작
EntityTransaction tx = entityManager.getTrasaction();
tx.begin();

// 객체를 생성한 상태(비영속)
Dashboard dashboard = new Dashboard();
dashboard.setId = 1L;
dashboard.setClub = aClub;

// 객체 영속화(영속)
entityManager.persist(dashboard);

// 데이터베이스 트랜잭션 커밋
tx.commit()
```

엔티티 매니저를 통해서 엔티티를 영속성 컨텍스트에 저장했습니다. 이제 Dashboard 객체는 비영속 상태에서 영속 상태가 되었습니다. 즉, 영속 상태라는 것은 영속성 컨텍스트에 의해 관리되는 것을 의미합니다. 

- 준영속 상태는 영속성 컨텍스트가 관리하던 영속 상태의 엔티티를 영속성 컨텍스트가 관리하지 않는 상태를 의미합니다.

```java
// 데이터베이스 트랜잭션 시작
EntityTransaction tx = entityManager.getTrasaction();
tx.begin();

// 객체를 생성한 상태(비영속)
Dashboard dashboard = new Dashboard();
dashboard.setId = 1L;
dashboard.setClub = aClub;

// 객체 영속화(영속)
entityManager.persist(dashboard);

// 데이터베이스 트랜잭션 커밋
tx.commit()

// 엔티티 매니저 종료. 영속성 컨텍스트의 모든 엔티티들이 준영속 상태가 된다.
entityManager.detach(Dashboard)

// 밑의 방법을 통해서도 엔티티를 준영속 상태로 만들 수 있다. 
// entityManager.close();
// entityManager.clear()
```

특정 엔티티를 준영속 상태로 만들기 위해서는 entityManager.detach() 를 호출하면 됩니다. entityManager.close() 또는 entityManager.clear()를 호출해서 영속성 컨텍스트를 초기화를 통해서 엔티티들을 준영속 상태로 만들 수 있습니다. 

하이버네이트가 작업단위를 마치면 영속성 컨텍스트가 종료됩니다. 하지만 어플리케이션은 사용 중이던 엔티티 객체들을 참조하고 있습니다. 영속성 컨텍스트가 종료되기 때문에 데이터베이스와의 연동은 보장하지 않지만, 어플리케이션 메모리에 여전히 존재하는 상태를 의미합니다. 

- 삭제 상태는 영속성 컨텍스트에 엔티티가 삭제될 것이라고 기록한 상태이다. 즉, 영속성 컨텍스트와 데이터베이스에서 삭제한다는 것을 의미합니다.

### 1.2 하이버네이트 세션과 작업 단위

“””

## 2. OSIV의 정의 및 개념

---

 `OSIV(Open Session in View)`는 영속성 컨텍스트를 뷰 영역까지 열어둔다는 것이다. 여기서 뷰는 스프링 MVC의 뷰(View)를 의미합니다.  해당 개념은 Hibernate와 JPA에서 데이테베이스 세션을 관리하는 전략 중 하나입니다. 

### 2.1 OSIV 등장 배경

 OSIV 패턴 등장 이전에는 뷰를 렌더링하는 시점에 영속성 컨텍스트가 존재하지 않아 준영속 상태가 된 객체의 프록시를 초기화 할 수 없는 문제가 있었습니다. 예시 코드와 함께 문제를 살펴보겠습니다. 

```java
// 도메인 계층

// -- Post
@Entity
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<Comment> comments;

    // Getters and Setters
}

// -- Comment
@Entity
public class Comment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String content;

    // Getters and Setters
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
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Post Details</title>
</head>
<body>
    <h1>Post: ${post.title}</h1>
    <h2>Comments:</h2>
    <ul>
        <c:forEach items="${post.comments}" var="comment">
            <li>${comment.content}</li>
        </c:forEach>
    </ul>
</body>
</html>
```

Post를 조회하는 api가 호출되었다고 가정하겠습니다. 서비스 계층의 Post를 조회하는 메서드에 `@Transactional` 이  붙어있습니다. 따라서 `getPostById` 메서드를 호출하면 세션을 열고, 트랜잭션을 시작합니다. 해당 메서드가 종료될 때 트랜잭션을 커밋하고, 세션을 닫습니다.

세션이 닫히면 영속성 컨텍스트가 함께 종료되면서 엔티티들은 준영속 상태가 됩니다. View를 렌더링 할때 `post` 엔티티의 `comments` 필드에 접근할 때 지연 로딩을 시도합니다. 하지만 `comments` 엔티티는 이미 준영속 상태가 되었기 때문에 지연 로딩을 수행할 수 없습니다. 결과적으로 `LazyInitializationException` 에러가 발생하게 됩니다.

Post 조회 요청 실행 경로와, 세션 및 트랜잭션의 유지를 그림으로 나타내면 다음과 같습니다.

<img width="868" alt="image" src="https://github.com/user-attachments/assets/48430799-725b-427c-b1f1-68ecd32c4527"> 

위 문제를 해결하기 위해 등장한 개념이 OSIV입니다. OSIV는의 목적은 뷰에서도  `Lazy Loading`을 지원하여, 필요한 데이터만 효율적으로 가져올 수 있도록 하는 것입니다. 사용자의 요청마다 필요한 데이터가 다를 수 있으므로, 모든 데이터를 한 번에 가져오는 대신에 사용자가 요청한 데이터만을 가져옴으로써 성능을 향상시키는데 기여합니다. 

## 3. 스프링 프레임워크 OSIV

 스프링 프레임워크가 제공하는 OSIV는 비즈니스 계층에서 트랜잭션을 사용하는 OSIV입니다. 즉, OSIV를 사용하지만 트랜잭션은 비즈니스 계층에서만 사용한다는 뜻입니다. 

### 3.1 스프링 프레임워크 OSIV 동작과정

<img width="870" alt="osiv2" src="https://github.com/user-attachments/assets/d17ae427-bebd-4791-b0fe-9ce84bc11263">

다음은 스프링 프레임워크에서 OSIV가 동작하는 과정입니다. 

1. 클라이언트로부터 HTTP 요청이 들어오면 서블릿 필터 또는 스프링 인터셉터에서 요청을 가로챕니다. 요청을 가로챈 이후, 영속성 컨텍스트를 생성합니다. 이때, 트랜잭션은 시작하지 않습니다. 
2. 서비스 계층에서 `@Transactional` 이 붙은 메서드가 실행되면, 위에서 미리 생성해둔 영속성 컨텍스트를 찾아와 트랜잭션을 시작합니다. 
3. 서비스 계층의 비즈니스 로직 실행이 끝나면 트랜잭션을 커밋합니다. 트랜잭션이 커밋되면 영속성 컨텍스트를 플러시합니다. 이때 트랜잭션은 끝나지만, 영속성 컨텍스트는 끝나지 않습니다. 
4. 컨틀롤러와 뷰까지 영속성 컨텍스트가 유지되기 때문에 조회하는 엔티티는 영속 상태를 유지합니다. 
5. 서블릿 필터 또는 스프링 인터셉터로 요청이 돌아오면 영속성 컨텍스트를 종료합니다. 이때 플러시를 호출하지 않고 바로 종료합니다.

스프링에서는 다음 방식들로 OSIV를 제공합니다.

1. JPA OEIV 서블릿 필터 - OpenEntityManagerInViewFilter
2. JPA OEIV  스프링 인터셉터 - OpenEntityManagerInViewInterceptor
3. 하이버네이트 OSIV 서블릿 필터 - OpenSessionInViewFilter
4. 하이버네이트 OSIV 스프링 인터셉터 - OpenSessionInViewInterceptor

서블릿 필터는 디스패처 서블릿 이전에 존재하는 필터를 기반으로 동작하는 OSIV이고, 인터셉터는 디스패처 서블릿 이후에 존재하는 인터셉터를 기반으로 동작하는 OSIV입니다. 

스프링 부트에서는 OSIV가 기본적으로 활성화 되어있습니다. application 설정을 통해 비활성화 할 수 있습니다.

```yaml
spring:
  jpa:
    open-in-view: false
```

## 3.1 OSIV 장점과 단점

OSIV를 통해 얻을 수 있는 장점에 대해 알아보겠습니다. 

장점

- LazyLoading 문제 해결

OSIV의 가장 큰 장점은 LazyLoading(지연로딩) 문제를 해결하는 것입니다. JPA를 사용하는 엔티티 매핑에서, 관계된 엔티티를 실제로 사용할 때까지 데이터베이스에서 로드하지 않는 LazyLoading은 자주 사용됩니다. 그러나 트랜잭션이 종료된 이후 뷰 단계에서 엔티티의 필드에 접근하려고 할 때, 지연 로딩된 필드에 접근하려 하면 `LazyInitializationException`이 발생합니다. OSIV는 HTTP 요청이 완료될 때까지 영속성 컨텍스트를 열어두므로, 뷰에서 지연 로딩된 엔티티를 안전하게 사용할 수 있습니다.

- 개발 편의성 증가

OSIV는 개발자가 트랜잭션 범위와 데이터 로딩 타이밍을 신경 쓰지 않아도 되는 장점을 제공합니다. 특히 복잡한 객체 연관 관계가 많을 때 LazyLoading이 적용된 엔티티를 뷰에서 자유롭게 사용할 수 있으므로 개발이 수월해집니다. 따라서 개발자는 비즈니스 로직에만 집중할 수 있게 됩니다. 

단점

- 긴 DB 커넥션 유지

OSIV의 가장 큰 단점 중 하나는 **데이터베이스 커넥션을 장시간 유지**해야 한다는 점입니다. HTTP 요청이 시작되고 뷰 렌더링이 끝날 때까지 영속성 컨텍스트를 열어두기 때문에, 트랜잭션이 끝난 이후에도 데이터베이스 연결이 유지됩니다. 이는 특히 트래픽이 많은 애플리케이션에서 **DB 커넥션 자원 부족**을 초래할 수 있습니다.

- 트랜잭션 외부에서 데이터 조작

OSIV가 활성화된 상태에서는 **트랜잭션이 종료된 후에도 데이터베이스에 접근**할 수 있습니다. 컨트롤러 계층에서 엔티티를 수정한 직후에 트랜잭션을 시작하는 서비스 계층을 호출할 경우 문제가 생길 수 있습니다. 예시 코드와 함께 살펴보겠습니다.

```java
@Controller
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/user/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        // DB에서 User 엔티티를 조회 (OSIV가 활성화된 상태에서 영속성 컨텍스트 유지)
        User user = userService.findUserById(id);

        // 컨트롤러에서 트랜잭션 밖에서 엔티티 수정
        user.setName("Updated Name");

        // 수정한 엔티티를 다시 서비스 계층에 전달 (트랜잭션을 시작)
        userService.updateUser(user);

        return "userView";

```

컨트롤러 계층에서 엔티티를 수정한뒤, `updateUser()` 호출했습니다. 이때 서비스 계층에서 새로운 트랜잭션이 시작됩니다. 트랜잭션이 커밋되면 변경 감지가 동작하면서 엔티티의 수정 사항을 데이터베이스에 반영합니다. 

컨트롤러에서 엔티티를 수정하고 즉시 뷰를 호출한 것이 아니라 트랜잭션이 동작하는 비즈니스 로직을 호출했으므로 문제가 발생한 것이다. 스프링 OSIV는 같은 영속성 컨텍스트를 여러 트랜잭션이 공유할 수 있으므로 위와 같은 문제가 발생할 수 있다. 

## 프로젝트에서 OSIV 관련 트러블 슈팅

### read-only 과 write transaction 분리

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

크루루 서비스에서는 SPOF의 문제를 해결하기 위해 데이터베이스 인스턴스를 두개로 분리했습니다. read-only 옵션이 적용된 트랜잭션은 읽기 전용 데이터베이스로 라우팅하고, 이 외의 트랜잭션들은 쓰기 전용 데이터베이스로 라우팅했습니다. 

<img width="1228" alt="databaseError" src="https://github.com/user-attachments/assets/1ed1d08a-d22f-4d71-9cdf-39a1226610f8">

운영서버에 배포한뒤 test를 위해 API를 호출했습니다. 하지만 `The MySQL server is running with the --read-only option so it cant execute`  에러가 발생했습니다. 현재 읽기 전용 옵션이 적용된 데이터베이스에 쓰기 작업을 시도해서 발생한 에러입니다. 트랜잭션에 따라 데이터베이스를 분리해주었지만, 제대로 라우팅이 되지 않는 것이었습니다. 이를 정확히 확인하기 위해 디버깅을 진행했습니다. 

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

<img width="531" alt="image" src="https://github.com/user-attachments/assets/d66e1d3c-c468-45df-91e7-10419913b950">


데이터베이스가 지정될 때마다, 문구를 출력하게 했습니다. 그뒤 쓰기 작업에 해당하는 `대시보드 만들기` API를 호출했습니다. 처음에 인가를 위한 ReadOnly 트랜잭션이 시작되고
