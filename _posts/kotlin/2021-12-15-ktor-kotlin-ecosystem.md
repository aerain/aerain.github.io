---
title: "ktor로 클린 아키텍쳐 구현하기. (with kotlin ecosystem)"
category: "architecture"
tags: ["kotlin", "ktor", "exposed", "koin"]
---

# 배경
이번 포스팅은 사실 별건 없고 문득 [if(kakao) 2021](http:/?if.kakao.com/) 를 보다가 우연하게 [카카오톡 서버가 스프링을 걷어내고 ktor로 서버 프레임워크를 전환한다는 얘기](https://if.kakao.com/session/49)를 보면서 시작되었다. 세션을 볼때는 그렇구나... 하고 넘어가다가 어느순간 술 먹다가 얼마나 괜찮은 넘이길래? 라는 궁금증에 평화로운 토요일 주말 새벽 두시에 취중 코딩(?)을 하며 시작 되었다.

# [Ktor](https://ktor.io/)?
> Create asynchronous client and server applications.

ktor에서 매력을 느꼈던 부분중 하나는 Coroutine을 정식 지원한다는 점이었다.

기존 스프링 webflux 에서 사용하던 코루틴 같은 경우에는 Mono나 Flux를 스프링 내부에서 정지함수의 반환 값으로 암시적으로 바꿔주거나, 혹은 명시적으로 우리가 직접 Publisher를 await해야 하는 불편한 reactor와의 동거가 있었는데, 이부분에서 매력적인 부분을 느낄 수 있었다.

하지만 뭐니뭐니해도 ktor가 스프링보다 경량이라는 점이 필자를 더더욱 매력적으로 만들었다. 

```kt
fun main() {
	embeddedServer(Netty, port = 8000) {
		routing {
			get ("/") {
				call.respondText("Hello, world!")
			}
		}
	}.start(wait = true)
}
```

불과 10줄도 되지 않는 코드로 뭔가를 할 수 있다는 점이 놀랍지 않은가? 마치 node.js 진영의 express.js나 koa.js를 보는 느낌이 들었다.

하지만 우리는 항상 코드의 변경 가능성을 염두에 두고, 유지보수를 위해 노력하는 개발자가 되어야 하지 않겠는가? 결국은 비즈니스 로직과 서버 계층 의존성과 떼어 놓아야 할것이다. 즉 간단하게 말하면 이 서버를 구축하기 위한 아키텍쳐를 하나 정해야 하는 셈이었다.

고민 끝에 클린 아키텍쳐를 도입해 보기로 했다. 다니는 직장에서는 계층화 아키텍쳐를 사용중에 있었는데, 한번 토이 프로젝트로 클린 아키텍쳐를 접목해보는것도 어떨까 해서 
ktor + clean architecture를 사용한 정말 간단한 crud 프로젝트를 도입해 보기로 하였다.


# Clean Architecture

사실 클린 아키텍쳐에 대한 설명은 다른 게시글들에서 더 많이 접할 수 있으니, 여기에선 정말 간략하게만 설명하고자 한다.

![clean-architecture](https://miro.medium.com/max/2400/1*_HxTmyiDQNCfACBZle67rw.jpeg)

위 그림을 설명하면, 각 계층 별로 레이어를 정의하고, 우리의 비즈니스 로직을 도메인 계층 (유스케이스 + 엔티티)로 응집시켜 관리하고자 하는 것이 최종 목표이다.

어째서 분리해야 할까? 사실 스프링을 기반으로 비즈니스 로직에서 RestTemplate을 사용하여 데이터를 받아온 뒤에 이렇게 예를 들어서 설명하면 다음과 같은 예시코드를 들면 정말 이해하기 편해진다.

## AS-IS
```kt
// Business logic
class PurchaseService(
    private val template: RestTemplate,
    private val repository: PurchaseRepository
) {


    fun purchaseItem(item: Item) {
        val purchaseResponse: PurchaseResponse = restTemplate.postForObject("/purchase/{id}", item.id)

		checkValidity(purchaseResponse)
		...
		someLogic()
		...
		val purchase = purchaseResponse.toEntity()
		repository.save(purchase)
    }
    ...
}
```

그런데 이 RestTemplate는 deprecated 된 코드이며, WebClient로 바꿔야 한다고 해보자. 당연한 일이지만 http 라이브러리의 변경이 우리의 비즈니스 로직을 수정하게 만든다. 

어째서 이런 일이 생겨야만 할까? 이 PurchaseService라는 서비스는 너무 외부 라이브러리에 종속적이다. PurchaseService 입장에서는 단순히 PurchaseResponse라는 녀석을 받는 것만 알길 원하는데, PurchaseResponse를 받기위해 **어떤 라이브러리를 사용할까?** 라는 관심사까지 전부 들어가 있는 것이다. 요약하면 [SoC](https://en.wikipedia.org/wiki/Separation_of_concerns)가 덜 된것이다. 이처럼 변하기 쉬운 것들은 밖으로 빼버리고, 우리는 비즈니스 로직에만 집중하면 된다.

## TO-BE

```kt
// Business logic
class PurchaseService(
	private val purchaseClient: PurchaseClient,
	private val repository: PurchaseRepository
) {
	fun purchaseItem(item: Item) {
		val purchaseResponse: PurchaseResponse = purchaseClient.purchase(item.id)
		checkValidity(purchaseResponse)
		...
		someLogic()
		...
		val purchase = purchaseResponse.toEntity()
		repository.save(purchase)
	}
}

// purchase client interface
interface PurchaseClient {
	fun purchase(id: String): PurchaseResponse
}

// 어떤 라이브러리를 사용할지에 대한 전략으로 원하는 곳에서 적절히 주입하여 사용 가능하다.
class RestTemplatePurchaseClient(
	private val restTemplate: RestTemplate
) : PurchaseClient {
	override fun purchase(id: String) = restTemplate
		.postForObject(...)
}

class WebClientPurchaseClient(
	private val webClient: WebClient
) : PurchaseClient {
	override fun purchase(id: String) = webClient
		.post()
		.uri("...")
		...
}
```

dip를 통해 외부와 통신하는 http 구현체를 PurchaseService로부터 분리하니, 추후에 OKHttp, Retrofit, Apache HttpClient 등의 신규 http 라이브러리 도입 요청으로 인한 코드 변경이 들어오더라도, 우리는 그냥 PurchaseClient를 구현한 신규 구현체를 PurchaseService에서 알아서 주입하면 그만이다. Business Logic(PurchaseService)는 변하지 않기 때문에 안전하고, 테스트도 더 비즈니스 로직에 중점적으로 할 수 있다.

# Architecture
![architecture](https://www.plantuml.com/plantuml/png/LT1H2iCW383XzvmYz7qzmkXLGWr3S39CEYYZTv_fLamUy_iPn4MKccxF0erNfVeeZ9DmUtERy0FeQifdyObU6GljPacmJ_6OgsRTdVY5Y3RXbOIT-fV84YavOoFWHV5s7xjFESyzZKNsq10CXLk01nomS4tzCxu0)

다음 형태를 기조로 패키지를 분리하였다. 각 패키지의 설명을 한줄로 간단히 요약하자면 다음과 같다.

* server : ktor 서버 설정 관련 모듈 설정이 들어있으며, di 컨테이너를 직접 바라본다.
* di : 내부 레이어에 대한 모든 의존성을 관리한다.
* api : ktor 기반의 api 라우터가 들어있는 녀석들이다.
* usecase : 핵심 비즈니스 로직이 들어있는 녀석들이다.
* entity : 우리의 도메인 모델이 들어가게 될 녀석이다.
* data : 유스케이스의 녀석을 구현하고 (ex: Repository), entity layer의 도메인 모델을 직접 의존한다.

설명만 하는 것은 그러니, 실제로 각 레이어에 들어갈 코드들을 예를 들어서 설명하고자 한다.

# Entity Layer
똑같은 내용에 대한 내용에 대한 얘기지만, 우리의 도메인 모델이 직접 들어갈 녀석이다. 해당 도메인에 대한 비즈니스 룰이 전부 이 도메인 모델에 들어가게 된다고 보면 되겠다. 클린 아키텍쳐 그림상 노란색에 해당한다.

```kt
class Post(
    val id: Long,
    title: String,
    val author: String,
    val postAt: Instant
) {
    var title = title
        private set

    fun changeTitle(title: String) {
        this.title = title
    }
}
```

# UseCase Layer
도메인 모델을 사용하는 사용사례에 대한 녀석들을 한 사례씩 정의 해 놓은 녀석이다. 클린 아키텍쳐 그림상 주황색에 해당한다. 여기에 UseCase와 Repository등의 인터페이스를 두고자 하였다.

다음 예제는 간단하게 Post 도메인 모델을 pk 기반으로 조회하는 사용 사례이다.

```kt
@UseCase
class GetPostUseCase(
    private val repository: PostRepository
) {
    suspend operator fun invoke(id: Long): Post {
        val entity = repository.findById(id) ?: throw NotFoundException(ID_NOT_FOUND)
        return entity.toDto()
    }

    companion object {
        private const val ID_NOT_FOUND = "id"
    }
}
```

# Data Layer
이 레이어를 통해 외부와 소통하거나, 외부 라이브러리에 강하게 의존적인 코드들을 정리하도록 하였다. 클린 아키텍쳐 그림상 초록색(게이트웨이)에 해당한다. 여기에 UseCase Layer에 정의된 Repository 인터페이스 등을 직접 구현한다.

다음 예제는 data layer에 작성된 녀석들의 예시이다.

```kt
// Exposed SQL Framework 기반으로 MariaDB와 소통하는 레포지토리 구현체이다.
class ExposedMariaDbPostRepository(
    private val dispatcher: CoroutineDispatcher
) : PostRepository {

    override suspend fun findById(id: Long) = withContext(dispatcher) {
        Posts.select { Posts.id.eq(id) }
    }.singleOrNull()?.toEntity()

    private fun ResultRow.toEntity() = Post(
        this[Posts.id].value,
        this[Posts.title],
        this[Posts.author],
        this[Posts.postAt]
    )
}

// Exposed의 트랜잭션을 지원하기 위해 신규 생성한 UseCase의 프록시 객체
class TransactionalGetPostUseCase(
    repository: PostRepository,
    private val dispatcher: CoroutineDispatcher
): GetPostUseCase(repository) {
    override suspend fun invoke(id: Long) = 
		newSuspendedTransaction(dispatcher) { super.invoke(id) }
} 
```

## Api Layer
이 레이어를 통해 클라이언트가 rest api를 통해 직접 통신한다. 클린 아키텍쳐 그림상 초록색 (컨트롤러)에 해당한다.
```kt
abstract class Api(val route: Routing.() -> Unit) {
    operator fun invoke(app: Application) = app.routing {
        route()
    }
}

class PostApi(
    private val getUseCase: GetPostUseCase
) : Api({
    route("/posts") {
		...
        get("/{id}") {
            val id: Long by call.parameters
            call.respond(getUseCase(id))
        }
		...
	}
})
```

# DI Layer - Koin(https://insert-koin.io/)
Ktor은 기본적으로 di container를 제공하지 않는다. 따라서 기존에 잘 성숙되어 있는 di framework인 dagger, koin 중, 코틀린 생태계에 최적화된 koin 이라는 녀석을 도입해보기로 하였다.

각 도메인 별로 컨테이너에 등록되어야할 녀석들을 별도로 관리하여 저장하였다. 그 밖에 Dependency Lookup에 사용될 몇몇 녀석들 또한 해당 레이어에 관리하여 저장하였다.

```kt
// 모듈 선언부
val postFactory = factory {
    single<PostApi> { PostApi(get()) } bind Api::class
    single<GetPostUseCase> { TransactionalGetPostUseCase(get(), Dispatchers.Default) }
    single<PostRepository> { MariaDbPostRepository(Dispatchers.IO) }
}

// ktor 모듈 추가를 위한 함수 선언
fun Application.installKoin() {
	install(Koin) {
		modules(postFactory)
	}
}

// Api 타입으로 등록되어있는 모든 인스턴스를 Lookup 한다.
val apis get() = getKoin().getAll<Api>().toSet()
```

# Server Layer
여기에서 Ktor 서버 설정에 필요한 모든 config가 들어가게 된다. DI Layer를 직접 의존하고, api 등록 및 기타 설정에 DI Layer로부터 인스턴스를 룩업하여 제공하였다.

다음은 API 등록을 진행하는 부분의 예시이다. 요렇게 정의함으로써 이 레이어에서는 API Layer에서 새로운 api가 추가되더라도 변경이 일어나지 않는다.

```kt
// 이 함수에서 사용중인 apis 녀석은 di layer에 정의되어있는 apis의 그것이다.
fun Application.configureRouting() {
    routing {
        log.info("registered api. {}", apis.map { it.javaClass.simpleName })
        apis.forEach { api -> api(this@configureRouting) }
    }
}
```

# 후기

Koin이나 Ktor에 대해서 익숙하지 않아서 이리저리 문서를 뒤져보느라 시간이 더 걸리기도 했다. 기존 스프링의 어노테이션 매직(?)에 깊이 적응되어버린 필자에겐 koin 모듈 등록 같은 작업이 너무 노가다 같고 귀찮은 작업이라 여겼지만, 적응되고 나니 해당 방식도 이것저것 좋은 장점이 있다고 생각했다. (모듈만 보고 해당 도메인 클래스의 연결고리를 확실히 알 수 있었다.)

여기에선 고려하지 않았지만, 메트릭 가시성 확보로 Observability를 굳건히 가져갈 수 있다면 정말 강력한 상용 서비스 하나로 발돋움 할 수 있지 않을까 하는 기대를 조금은 하게 되었다.


소스코드는 여기에서 참고할 수 있다.

https://github.com/aerain/ktor-clean-architecture