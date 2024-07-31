---
title: mockkObject는 flaky한가?
date: 2024-06-30T18:08:29+09:00
draft: false
tags: ["kotlin", "test", "mockk"]
categories: ["development"]
---

# mockkObject 뭐가 문제일까?

Kotlin을 사용한다면 대체로 mock 라이브러리로 mockk를 많이 선택하게 됩니다.
그만큼 직관적이로 약간의 온보딩 과정만 거친다면 실제로 테스트 코드를 작성하는데 큰 어려움이 없기 때문입니다.
하지만 이면에는 mockk가 가지는 고질적인 속도 문제와 동기화 문제도 존재합니다.

mockk가 지원하는 mockkObject는 Kotlin의 object class를 쉽게 목(mock) 객체로 만들어 줍니다.
하지만 이런 마법같은 작용 뒤에는 생각보다 치명적인 동기화 이슈가 숨어져 있습니다.

## 문제의 발단
문제의 발단은 사내에서 사용하는 build 파이프라인이 테스트의 실패로 깨지기 시작한 것이었는데요.
억울한 것이 상황에 따라서는 성공을 때로는 실패를 하는 전형적인 flaky한 테스트 군집의 특성을 보였습니다.
간혹 실패하는 테스트를 살펴보면 아래와 같은 에러메시지를 확인할 수 있었는데요.

```
io.mockk.MockKException: can't find stub Companion(object Companion)
    at io.mockk.impl.stub.StubRepository.stubFor(StubRepository.kt:16)
    at io.mockk.impl.recording.states.AnsweringState.call(AnsweringState.kt:13)
```

실제로 실패가 발생한 코드 내부를 보면 프로파일을 추출하는 object class인 `ProfileIdentifier`에 해당하는 목 객체를 조회하는 과정에서 발생한 문제라는 것을 알 수 있었습니다.
하지만 놀라운 사실은 해당 테스트에서는 `ProfileIdentifier`를 목으로 변환하지 않았다는 점이었습니다.

최종적으론 다른 테스트 클래스에서 사용하는 mockkObject로 인해서 목 생성을 하지 않았음에도 mockk와 관련된 에러를 만났다는 것을 직관적으로 알 수 있었는데요.
좀 더 자세하게 왜 이런 일이 발생할 수 있었는지 mockkObject의 동작원리와 함께 이해해 봅시다.

## mockkObject의 동작 원리

mockkObject 메서드를 실행하면 SpyKStub이 생성되고 cancellation 람다가 같이 등록됩니다.
과정을 세부적으로 코드와 함께 살펴보겠습니다.

1. stub을 생성합니다.
    ```Kotlin
   // JvmObjectMockFactory.kt - objectMockk() 
   val stub = SpyKStub(cls, "object " + cls.simpleName, gatewayAcess, 
        recordPrivateCalls, MockType.OBJECT)
    ```
2. StubRepository에 stub 등록하고 전역적으로 관리합니다.
    ```Kotlin
   // JvmObejctMockFactory.kt - objectMockk()
   stubRepository.add(obj, stub)
   ```
3. cancellation 람다를 생성하여 목 해제시 함수가 실행될 수 있도록 합니다.
    ```Kotlin
   // JvmObejctMockFactory.kt - objectMockk()
   return {
       if (refCntMap.decrementRefCnt(obj)) {
           val stub = stubRepository.remove(obj)
           stub?.let {
               it.dispose()
           }
       }
   }
   ```
   이어서 취소 람다를 레지스트리에 등록합니다.
    ```Kotlin
   // API.kt - internalMockkObject()
   MockKCancellationRegistry
       .subRegistry(MockKCancellationRegistry.Type.OBJECT)
       .cancelPut(it, cancellation)
   ```
4. 목 해제 시점에서 레지스트리에 등록된 람다를 실행합니다.
   ```Kotlin
   // API.kt - internalUnmockkObject()
   MockKCancellationRegistry
       .subRegistry(MockKCancellationRegistry.Type.OBJECT)
       .cancel(it)
    ```
   
mockkObject로 생성된 목 객체가 전역적인 StubRepository **{{<color color="#FB6F92" text="내부에서 전역적으로 관리된다는 사실을 알 수 있습니다.">}}**
싱글톤인 object class의 특징상 전역적으로 목 객체를 관리하는 것이 뭐가 문제냐고 되물을 수 있겠지만, 
목 객체를 만든다는 것은 테스트 내부에서 제한적으로 사용하겠다는 의미를 가지기도 합니다.
테스트 환경을 위해서 만든 객체가 전역적으로 관리된다는 점이 의아할 수 밖에 없는 상황입니다.

### (참고) 일반 object 객체와 목 객체를 어떻게 구분할 수 있을까?
여기서 잠깐 궁금증이 드는 부분은 테스트 실행시 실행 환경에서 접근한 객체가 목 객체인지 원래 객체인지 어떻게 알 수 있을까 인데요.
object class의 경우 똑같이 싱글톤 클래스의 레퍼런스를 참조할텐데, 어디서 이를 구분할 수 있는지 확인해 봅시다.

코드를 추적하다보면 mockk에서는 어드바이스를 등록해서 객체에 접근할 때 프록시 핸들러가 존재하는지 확인 후, 인터셉터를 이용해서 stub(목 객체)이 결과로 나오도록 처리한다는 것을 알 수 있습니다.

```Java
// JvmMockKProxyAdive.java
public class JvmMockKProxyAdvice extends BaseAdvice {
   @OnMethodEnter(skipOn = OnNonDefaultValue.calss)
   private static Callable<?> enter(...)
}
```

mockk에서 어드바이스가 동작하는 과정을 자세하게 살펴보겠습니다.

1. JvmMockDispatcher에서 적절한 JvmMockDispatcher를 가져옵니다.
   ```Java
   // JvmMockKDispatcher.java
   public class JvmMockKDispatcher get(long id, Object obj) {
      if (obj == DISPATCHER_MAP) {
         return null;
      }
      return DIPATCHER_MAP.get(id);
   }
   ```
2. 추출한 디스패처의 핸들러를 동작시킵니다.
   ```Java
   // JvmMockKProxyAdvie.java - enter()
   JvmMockKDispacher  dispatcher = JvmMockKDispatcher.get(id, self);
   
   if (dispatcher == null || !dispatcher.isMock(self)) {
      return null;
   }
   
   return dispatcher.handler(self, method, arguments);
   ```
3. 핸들러가 실행되면 인터셉터가 호출되어 목 객체를 반환합니다.
   ```Kotlin
   // BaseAdvice.java - handler()
   fun handler(self: Any, method: Method, arguments: Array<Any?>): Callable<*>? {
      val handler - handlers[self] ?: return null
      
      retrn if (SelfCallEliminator.isSelf(self, method) {
         null
      } else {
         Intercaptor(handler, self, method, arguments)
      }
   }
   ```
4. 이때 인터셉터는 목 객체의 존재 여부에 따라 원본 객체를 반환할지 stub을 호출할지 결정합니다.
   ```Kotlin
   // Intercpetor.kt - call()
   fun call(): Any? {
      val callOriginalMethod = SelfCallEliminatorCallable(
         MethodCall(self, method, arguments),
         self,
         method
      )
      return handler.invocation(self, method, callOriginalMethod, arguments)
         ?.boxedValue // unbox value calss objects
   }
   ```
   JvmMockFactoryHelper의 invocation 메서드를 호출합니다.

   ```Kotlin
   // JvmMockFactoryHelper.kt - invocation()
   object JvmMockFactoryHelper {
      fun mockHandler(stub: Stub) = object : MockKInvocationHandler {
         override fun invocation(self: Any, method: Method?, originalCall: Callble<*>?, args: Array<Any?>) =
            stdFunctions(self, method!!, args) {
               stub.handlerInvocation(
                  self,
                  method.toDescription(),
                  {
                     handlerOriginalCall(originalCall, method)
                  },
                  args,
                  findBanckingField(slef, method)
               )
            }
      }
   }
   ```

구성이 복잡하니 다이어그램으로 나타낸다면 아래와 같습니다. 
다이어그램을 살펴보면 전형적인 프록시 패턴으로 구현된 reflect 패키지와 유사한 구조를 가진다는 것을 알 수 있습니다.
![mockkObject flow](mockkObject_flow.png "40rem")

## mockkObject와 동시성 이슈

다시 돌아와 object class 목 객체를 전역적으로 관리하는 mockk 특성으로 인해 동기화 과정에서 문제가 생길 수 있다는 의심이 들었습니다.
이제는 테스트를 통해서 mockk를 사용하였을 때, 발생할 수 있는 동기화 문제를 재현해 봅시다.

### 목 객체 생성 과정에서의 critical section

먼저 목 객체가 생성하는 과정에서 발생하는 동기화 문제가 있습니다.
재현을 위해 JUnit 설정을 아래와 같이 변경합시다.

```
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.config.strategy=dynamic
junit.jupiter.execution.parallel.mode.default=same_thread
junit.jupiter.execution.parallel.mode.classes.default=concurrent
```

동시성 옵션을 켜두고 두 개의 테스트를 만들어 봅시다.
테스트는 간단하게 mockkObject를 사용하는 테스트 2개로 구성됩니다.

```Kotlin
object Sample {
   fun get() = true
}

class MockKObjectTest {
   @Test
   fun test1() {
      mockkObject(Sample)
      every { Sample.get() } returns false
      unmockkObject(Sample)
   }
   
   @Test
   fun test2() {
      mockkObject(Sample)
      every { Sample.get() } returns false
      unmockkObject(Sample)
   }
}
```

테스트를 실행하여 동시에 test1과 test2를 실행할 경우 높은 확률로 아래와 같은 에러를 확인할 수 있습니다.
```
can't find stub Sample@598a46d0
io.mock.MockException: can't find stub Sample@598a46d0
   at io.mock.impl.stub.StubRepository.stubFor(StubRepository.kt:16)
   at io.mockk.impl.stub.CommonClearer.clear(CommonClearer.kt:17)
```

mockkObject는 기존에 목 객체가 이미 존재하는지 확인하고 존재한다면 이미 만들어진 목 객체를 초기화합니다.
문제가 발생한 곳을 따라가 보면 stub을 조화하고 초기화 하는 과정에서 stub을 발견하지 못했다는 에러라는 사실을 알 수 있습니다.

```Kotlin
objects.forEach {
   val cancellation = factory.objectMockk(it, recordPrivateCalls)
   
   internalClearMocks(it, emptyArray())
   
   MockKCancellationRegistrty
      .subRegistry(MockKCancellationRegistry.Type.OBJECT)
      .cancelPut(it, cancellation)
}
```

문제가 발생한 부분은 시간에 따라 순차적으로 그려보면 아래와 같습니다.
![critical1](critical1.png)

test1이 먼저 refCount를 올리면서 목 객체를 생성을 하겠다는 사실을 동기화하였습니다.
test2는 이에 목 생성을 스킵하고 목 객체를 초기화하고 사용하는 단계로 넘어갑니다.
하지만 test1이 아직 목을 생성하지 않았기 때문에 문제가 발생합니다.

결론적으로 위 문제는 mockkObject를 통해서 **{{<color color="#FB6F92" text="목 생성이 느리다보니 병렬적으로 실행되는 테스트에 영향을 준 상황으로 정리할 수 있습니다.">}}**
따라서 목 생성 과정 자체를 동기화 시킨다면 위 문제를 쉽게 해결할 수 있습니다.

```Kotlin
object Sample {
   fun get() = true
}

class MockKObjectTest {
   private val monitor = ""
   @Test
   fun test1() {
      synchronized(monitor) { mockkObject(Sample) }
      every { Sample.get() } returns false
      unmockkObject(Sample)
   }
   
   @Test
   fun test2() {
      synchronized(monitor) { mockkObject(Sample) }
      every { Sample.get() } returns false
      unmockkObject(Sample)
   }
}
```

### 목 객체가 아니더라도 목 객체를 참조하는 문제

두 번째 상황은 목 객체의 프록시가 생성되면서 목킹(mocking)을 적용하지 않았지만 목 객체에 접근하여 값을 가져오는 상황입니다.
똑같이 실험으로 알아봅시다.

```
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.config.strategy=dynamic
junit.jupiter.execution.parallel.mode.default=same_thread
junit.jupiter.execution.parallel.mode.classes.default=concurrent
```

두 번째 실험에는 독립된 두 테스트 클래스를 정의해서 한 쪽은 목을 한 쪽을 목을 적용하지 않도록 합니다.

```Kotlin
object Sample {
   fun get() = true
}

class WithMockingTest {
   @Test
   fun test() {
      mockkObject(Sample)
      every { Sample.get() } returns false
      // 목킹을 적용하고 일정 시간 블록
      Thread.sleep(3_000)
      unmockkObject(Sample)
   }
}

class WithoutMockingTest {
   @Test
   fun test() {
      // WithMockingTest가 먼저 시작되는 상황을 가정
      Thread.sleep(1_000)
      every { Sample.get() } returns false
      Sample.get() shouldBe true
   }
}
```

테스트 결과는 WithMockingTest에 의해 생성된 목 객체를 WithoutMockingTest가 참조하면서 테스트에 실패합니다.
단순하게 그림을 그려보면 아래와 같습니다.

![critical1](critical2.png)

# 올바른 object class 테스트

문제 원인을 정리해보면 mockkObject의 문제는 아래와 같이 정리할 수 있습니다.

* mockkObject는 목 객체를 생성하는데 많은 시간이 걸립니다. 이는 목 객체를 생성하는 클래스 간의 동기화 문제를 야기합니다.
* mockkObject는 전역적으로 object class를 관리하기 때문에 원치않은 상황에서 목 객체를 참조에 따른 문제가 발생할 수 있습니다.

실제로도 mockk가 thread-safe 하지 않다는 이슈는 계속 존재하는 것으로 보입니다.
* [Suggestion: Recommendations for running parallel tests](https://github.com/mockk/mockk/issues/860)
* [mockk is not threadsafe!!!!](https://github.com/mockk/mockk/issues/1062)

## mockkObject 그대로 사용해도 될까?

mockkObject는 일관된 테스트 결과를 보장하지 않을 수 있다는 결론을 내릴 수 있습니다.
넓은 마음으로 바라본다면 테스트가 가끔 실패하는 상황은 그리 심각한 상황은 아닐 수 있습니다. 
실패 이후에 다시 빌드하다 보면 언젠가 성공한 빌드가 생길 수 있기 마련이기 때문이죠.

하지만 개발의 생산적인 측면에서는 고민해볼 필요가 있습니다.
목 객체를 생성을 위해 mockk라는 아주 편리한 라이브러리를 사용했지만 그 결과는 오히려 빈번한 빌드 실행이라는 불편함을 낳았습니다.
개발의 생산성을 높이려고 사용한 라이브러리가 오히려 개발 생산성의 발목을 잡는 상황이 되버린 겁니다.

이 상황에서 택할 수 있는 선택지를 고민해보면 **{{<color color="#FB6F92" text="과감하게 해당 라이브러리 사용을 포기하거나">}}** 혹은 **{{<color color="#FB6F92" text="잘못 사용되고 있는 습관적인 부분을 개발 조직 내의 규칙으로 사전에 예방">}}**하는 것이 있을 것입니다.
하지만 기껏 잘 사용중이던 라이브러리 사용을 금지한다는 것은 아키텍처의 상당 부분을 변경해야하는 대규모 공사를 야기할 수 있을 만큼 신중할 필요가 있습니다.

## object class mocking은 smell 일 수 있다.👃

사건의 발단은 object class를 목 객체로 만드는 과정이었습니다. 
가장 쉬운 방법인 mockkObject를 사용하면서 생각하지 못했던 부작용과 조우하였는데요.
지금까지의 결론만 놓고 본다면 object class를 목 객체로 만든느 상황을 이제는 불편하게 느끼고 있다고 판단할 수 있을 것 같습니다.
이는 object class를 목 객체로 만드는 과정이 어색하고 불편한 작업이라는 것으로 해석할 수 있습니다.

항상 불편하다는 것이 code smell이라는 법은 없지만, 
현재까지의 상황을 놓고 보면 테스트를 불편하게 만들면서 테스트를 불완전하게 만드는 mockkObject는 어찌보면 code smell의 조건이라고 할 수 있는 부분을 이미 상당 부분 가지고 있다고 볼 수 있을 것입니다.

만약 object class가 목킹이 필요하다면 이는 어쩌면 더 이상 object class가 아닌 인터페이스 추상화가 필요한 단계라는 것을 암시하는 상황일 수 있습니다.
이미 익히 알려진 방법이지만 LocalDate를 직접 사용하는 것 대신에 인터페이스로 시계(clock)을 구현하여 정적 클래스 사용을 우회하는 것 처럼,
object class를 직접 목킹하는 방법 대신에 가짜(fake) 객체를 구현하는 방식을 선택해 보는 것은 어떨지 간단한 제언을 해봅니다.