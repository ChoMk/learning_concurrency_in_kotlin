# 9장 코틀린의 동시성 내부

### 컴파일러가 일시 중단 함수를 상태 머신으로 변환하는 방법과 스레드 스위칭이 어떻게 발생하고 예외가 전파되는지 분석해보자.

- 연속체 전달 스타일(CPS)과 일시 중단 연산과의 관계
- 코루틴이 바이트 코드로 컴파일될 때 사용되는 다양한 내부 클래스
- 스레드 전환 방법을 포함한 코루틴 차단의 흐름
- CorotineExceptionHandler를 사용했을 때와 사용하지 않았을 때의 예외 전파

→ 상태머신 : 상태머신은 코틀린 컴파일러가 연속체에 대한 상태를 나타내기 위해 생성한다.

# 연속체 전달 스타일(CPS)

- 호출되는 함수에 연속체를 보내는 것을 전제로 하고 있고, 함수가 완료되는 대로 연속체를 호출하는 것. (연속체를 콜백으로 생각할 수 있다.)
- 일시 중단 연산이 다른 일시 중단 연산을 호출할 때마다 완료 또는 오류가 생겼을 때 호출돼야 하는 연속체를 전달한다.
- 일시 중단 연산은 재개할 때마다 중단된 위치에서 상태를 복구하고 실행을 지속한다.
- CPS와 상태머신이 결합하면 컴파일러는 다른 연산이 완료되기를 기다리는 동안 일시 중단될수 있는 연산을 생성한다.

## 연속체

```kotlin
interface Continuation<in T> {
    val context: CoroutineContext //Cotinuation과 함께 사용
    fun resume(value: T) // 일시 중단을 일으킨 작업의 결과
    fun resumeWithException(exception: Throwable) // 예외의 전파를 허용
}
```

## suspend 한정자

- 동시성을 지원하기 위해 가능한 언어 변환을 작게 가져가는 것이었다. 대신 코루틴 및 동시성의 지원에 따른 영향은 컴파일러, 표준 라이브러리, 코루틴 라이브러리에서 취하도록 했다.
- 언어 관점에서 관련된 유일한 변화는 suspend 한정자의 추가 부분이다.

```kotlin
suspend fun getUserSummary(id: Int): UserSummary {
    val profile = fetchProfile(id)//suspending fun
    val age = caculateAge(profile.dateOfBirth)
    val term = validateTerms(profile.coutry, age)//suspending fun
    return UserSummary(profile, age, terms)
}
```

## 상태머신

- 컴파일러가 코드를 분석하면 일시 중단 함수가 상태 머식느오 변환될 것이다.
- 일시 중단 함수가 현재 상태를 기초로 해서 매번 재개되는 다른 코드부분을 실행해 연속체로서 동작할 수 있다.

### 라벨

```kotlin
when(label){
    0->{
				fetchProfile(id)
        return
		}
		1->{
				calculateAge(profile.dateOfBirth)
        validateTerms(profile.country, age)
        return
		}
		2->{
				UserSummary(profile, age, terms)
		}
}
```

## 연속체

- 다른 지점에서 실행을 재개할 수 있는 기본 함수가 생겼으며, 이제 함수의 라벨을 나타내기 위한 방법을 찾아야 한다.

```kotlin
suspend fun getUserSummary(id: Int): UserSummary {
    val sm = object : CoroutineImpl {
        override fun doResume(data: Any?, exception: Throwable?) {
            //TODO:재개를 위해 getUserSummary호출
				}
    }
}
```

→ 파라미터로 Cotinuation<Any?>를 수신해야 한다. 그렇게 하면 다음과 같이 doResume()이 getUserSummary()로 볼백을 전달할 수 있다.

```kotlin
suspend fun getUserSummary(id: Int, cont: Continuation<Any?>): UserSummary {
    val sm = object : CoroutineImpl {
        override fun doResume(data: Any?, exception: Throwable?) {
            getUserSummary(id, this)
        }
    }
    val state = sm as CoroutineImpl//CoroutineImpl을 직접 받지 않는 이유는 호환성 이슈
    when (state.label) {

    }
}
```

## 콜백

- getUserSummary()로부터 호출된 다른 일시 중단 함수가 CoroutineImpl을 전달받도록 수정해야 한다.

```kotlin
when(state.label){
    0->{
				fetchProfile(id, sm)
        return
		}
		1->{
				calculateAge(profile.dateOfBirth)
        validateTerms(profle.country, age, sm)
        return
		}
		2->{
				UserSummary(profile, age, terms)
		}
}
```

## 라벨 증분

다른 일시 중단 함수를 호출하기 전에 라벨이 증분돼야 한다.

```kotlin
when(state.label){
    0->{
				sm.label = 1
        fetchProfile(id, sm)
        return
		}
		1->{
				sm.label = 2
        calculateAge(profile.dateOfBirth)
        validateTerms(profle.country, age, sm)
        return
		}
		2->{
				UserSummary(profile, age, terms)
		}
}
```

## 다른 연산으로부터의 결과 저장

```kotlin
private class GetUserSummarySm : CoroutineImpl {
    var value: Any? = null
    var exception: Throwable? = null
    var cont: Cotinuation<Any?>? = null
    var id: Int? = null
    var profile: Profile? = null
    var age: Int? = null
    var terms: Terms? = null

    override fun doResume(data: Any?, exception: Throwable?) {//호출자에 의해 반환되는 데이터를 저장하기 위한 값을 추가.
        this.value = data
        this.exception = exception
        getUserSummary(id, this)//실행이 처음 시작될 때 보내진 초기 연속체를 저장하기 위한 값을 추가
    }
}
```

```kotlin
valsm= cont as? GetUserSummarySm ?: GetUserSummarySm()

when(sm.lavel){
    0->{
				sm.cont = cont
        sm.label = 1
        fetchProfile(id, sm)
        return
		}
		1->{
				sm.profile = sm.value as Profile
        sm.age = calculateAge(sm.profle!!.dateOfBurth)
        sm.label = 2
        validateTerms(sm.profle!!.country, sm.age!!, sm)
        return
		}
		2->{
				sm.terms = sm.value as Terms
        UserSummary(sm.profile!!, sm.age!!, sm.terms!!)
		}
}
```

→ 상태 머신으로부터 직접 전달받은 모든 변수들을 사용한다.

## 일시 중단 연산의 결과 반환

```kotlin
valsm= cont as? GetUserSummarySm ?: GetUserSummarySm()

when(sm.lavel){
    0->{
				sm.cont = cont
        sm.label = 1
        fetchProfile(id, sm)
        return
		}
		1->{
				sm.profile = sm.value as Profile
        sm.age = calculateAge(sm.profle!!.dateOfBurth)
        sm.label = 2
        validateTerms(sm.profle!!.country, sm.age!!, sm)
        return
		}
		2->{
				sm.terms = sm.value as Terms
        UserSummary(sm.profile!!, sm.age!!, sm.terms!!)
				sm.cont!!.resume(UserSummary(sm.profile!!, sm.age!!, sm.terms!!))
		}
}
```

- 함수로부터의 유형 반환도 제거됐다.
- 일시 중단 함수의 시그니처는 실제로 any?를 반환하는 것을 나타내는데, 이는 일시 중단 연산이 일시 중단이 발생했음을 나타내는 COROUTINE_SUSPENDED 값을 반환하거나, 일시 중지되지 않았을 때 직접 결과를 반환할 수 있어서 존재한다.

→ COROUTINE_SUSPENDED를 반환하는 경우에만 일시 중지하며 그렇지 않으면 함수의 결과를 예상된 유형으로 캐스팅하고 다음 라벨을 계속 실행한다. 불필요한 일시 중단이 발생하지 않는다는 것을 보장한다.

## 컨텍스트 전환

- 코루틴은 처음 시작한 컨텍스트가 아닌 다른 컨텍스트에서 다시 시작할 수 있다는 한가지 특징이 있다. CoroutineContext는 사용할 스레드와 스레드 풀에 대한 것뿐만 아니라 예외 처리와 같은 또 다른 중요한 구성을 포함할 수 있다.

## 스레드 전환

### ContinuationInterceptor

- CoroutineContext는 고유한 키와 함께 저장되는 서로 다른 CoroutineContext.Element를 가지는 맵처럼 동작한다.

```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(cont: Continuation<T>): Continuation<T>
}
```

- ContinuationInterceptor의 구현은 수신된 연속체를 올바른 스레드가 사용되도록 보장하기 위해 다른 연속체로 랩핑하는 것.

## CoroutineDispatcher

- CoroutineDispatcher는 commonPool, Unconfined 및 DefaultDispatcher와 같이 제공된 모든 디스패처의 구현을 위해 사용되는 ContinuationInterceptor의 추상 구현체다.
- dispatch() 함수는 필요할 때 실제로 스레드를 강제로 전환할 수 있는 함수라는 점을 명심하자.

```kotlin
abstract class CoroutineDispatcher:AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor{
    abstract fun dispatch(context:CoroutineContext, block:Runnable)
    override fun <T> interceptContinuation(continuation:Continuation<T>):Continuation<T> = DispatchedContinuation(this, continuation)
}
```

## CommonPool

- isDispatchNeeded()가 commonPool에서 재정의되지 않아서 항상 true
- ForkJoinPool을 사용하거나 newFixedThreadPool을 사용해 생성된다.

```kotlin
override fun dispatch(context: CoroutineContext, block: Runnable) =
    try {
        (pool ?: getOrCreatePoolSync())
            .execute(timeSource.trackTast(block))
    } catch (e: RejectedExecutionException) {
        timeSource.unTrackTask()
        DefaultExecutor.execute(block)
    }

```

## Unconfined

- unconfined는 특정 스레드나 스레드 풀 사용을 강제하지 않는다. isDispatchNeeded()에 재정의해야 한다.

## 안드로이드 ui(HandlerContext)

- dispatch 함수는 runnable 핸들러의 post 함수로 간단히 전달
- ui라는 인스턴스가 생성된다. 메인 루퍼와 ui라는 이름을 생성자에 전달한다.
- val UI = HandlerContext(Handler(Looper.getMainLooper()), "UI")

## DispatchedContinuation

- resume 혹은 resumeWithException()이 호출 될 때마다 DispatchedContinuation은 디스패처를 사용한다.

```kotlin
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : Continuation<T> by continuation, DispatchedTask<T> {
    override fun resume(value: T) {
        val context = continuation.context
        if (dispatcher.isDispatchNeeded(context)) {
            _state = value
            resumeMode = MODE_ATOMIC_DEFAULT
            dispatcher.dispatch(context, this)
        } else {
            resumeUndispatched(value)
        }
    }

    override fun resumeWithException(exception: Throwable) {
        val context = continuation.context
        if (dispatcher.isDispatchNeeded(context)) {
            _state = CompletedExceptionally(exception)
            resumeMode = MODE_ATOMIC_DEFAULT
            dispatcher.dispatch(context, this)
        } else {
            resumeUndispatchedWithException(value)
        }
    }
}
```

## DispatchedTast

- DispatchedContinuation은 또한 DispatchedTask를 구현한다. 인터페이스는 연속체에 있는 resume()과 resumeWithException()을 트리거할 수 있는 run()의 기본 구현을 추가하는 것으로 runnable을 확장한다.

```kotlin
override fun run() {
    try {
        val delegate = delegate as DispatchedContinuation<T>
        val continuation = delegate.continuation
        val context = continuation.context
        val job = if (resumeMode.isCancellableMode) context[Job] else null
        val state = takeState()
        withCoroutineContext(context){
						if (job != null && !job.isActive) {
                continuation.resumeWithException(job.getCancellationException())
            } else {
                val exception = getExceptionalResult(state)
                if (exception != null) {
                    continuation.resumeWithException(exception)
                } else {
                    continuation.resume(getSuccessfulResult(state))
                }
            }
				}
} catch (e: Throwable) {
        throw DispatchException("Unexpected exception running $this", e)
    }
}
```

## 정리

- 실제 스레드 변경 작업은 CoroutineDispatcher에서 일어나지만, 실행 전에 연속체를 가로챌 수 있는 전체 파이프라인 덕분에 가능한 것이다.

## 예외처리

### handleCoroutineException() 함수

- 예외가 발생할 때마다 handleCoroutineException() 함수로 다시 보내진다.

```kotlin
public fun handleCoroutineException(context: CoroutineContext, exception: Throwable) {
    try {
        context[CoroutineExceptionHandler]?.let{
						it.handleException(context, exception)
            return
		}
				if (exception is CancellationException) return
	      context[Job]?.cancel(exception)
    } catch (handlerException: Throwable) {
        if (handlerException === exception) throw exception
        throw RuntimeException(
            "Exception while trying to handle coroutine exception",
            excetion
        ).apply{
						addSuppressed(handlerException)
				}
		}
}
```

- CoroutineExceptionHandler를 찾게 되면 handleException() 함수가 호출되며, 컨텍스트와 예외를 모두 전달한다.
- CancellationException 코루틴을 취소하는데 사용되기 때문에 무시된다.
- CoroutineContext에 잡이 존재하면 해당 cancel 함수가 호출된다.
- jvm의 경우 serviceLoader를 사용하는 예외 처리를 찾고 예외를 발견할 수 있는 모든 대상에게 전달한다.

# 요약

- 일시중단 연산은 상태 머신으로 변환되며 cps의 사용을 통해 다른 일시 중단 함수를 위한 콜백이 된다.
- 연속체는 런타임에 DispatchedContinuations으로 감싸인다. CoroutineDispatcher이 코루틴이 시작되거나 재개될 때 가로채는 것을 허용한다. 이때 스레드가 강제로 적용되는데 UNCONFINED의 경우는 예외다.
- CoroutineContext가 CoroutineExceptionHander가 없고, 포착되지 않은 예외가 CancellationException이 아니라면, 설사 있다 하더라도 프레임워크는 플랫폼별로 예외를 처리하기 위한 코드를 허용하고 잡을 취소한다.