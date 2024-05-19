---
title: 'Enum 톺아보기(1)'
date: 2024-01-19T16:49:39+09:00
draft: false
tags: ["kotlin"]
categories: ["development"]
---

Kotlin 혹은 Java를 사용하면서 Enum은 정말 흔하게 사용되는 키워드입니다. 하지만 흔히 사용하는 만큼 정작 Enum이 무엇인지 깊이 고민하지 않고 넘어가는 경우가 많았었습니다. 그만큼 편하게 쓸 수 있기 때문이라는 생각도 듭니다. 이 글은 Enum이 어떠한 형태로 정의되는지 부터 Enum을 사용하면서 놓질 수 있는 성능적인 부분까지 다뤄보겠습니다.

# Enum이란 무엇인가?
Java에서는 `enum`이라는 키워드로 Kotlin에서는 `enum class`라는 키워드로 Enum을 쉽게 정의할 수 있습니다.

그렇다면 Enum은 무엇일까요? 이를 확인하기 위해서는 Java가 Enum을 어떻게 다루는지부터 이해해야 합니다. 먼저 아주 간단한 Enum을 정의하도록 하겠습니다.

```kotlin
enum class EnumTest(code: String) {
    ZERO("00"),
    ONE("01"),
    TWO("02"),
    THREE("03"),
    FOUR("04"),
    FIVE("05"),
    SIX("06"),
    SEVEN("07"),
    EIGHT("08"),
    NINE("09"),
    TEN("10"),
    ETC_ERROR("99"),
}
```

위 코드를 바이트코드로 변환해 보겠습니다. 

```java
public final enum EnumTest extends java.lang.Enum {
  public final static enum LEnumTest; ZERO
  public final static enum LEnumTest; ONE
  public final static enum LEnumTest; TWO
  public final static enum LEnumTest; THREE
  public final static enum LEnumTest; FOUR
  public final static enum LEnumTest; FIVE
  public final static enum LEnumTest; SIX
  public final static enum LEnumTest; SEVEN
  public final static enum LEnumTest; EIGHT
  public final static enum LEnumTest; NINE
  public final static enum LEnumTest; TEN
  public final static enum LEnumTest; ETC_ERROR
}
```

변환된 내용에서 알 수 있는 내용을 정리해보면 다음과 같습니다.
1. 바이트코드에서도 Enum을 특수한 `enum`으로 간주하고 있습니다.
2. `java.lang.Enum`을 상속받아서 사용하고 있습니다.
2. Enum의 각 엘리먼트는 정적 변수(`statis`)로 정의되어 있습니다.

하지만 아직도 명확히 Enum이 무엇인지는 잘 모르겠습니다. 오라클의 Java 문서에서는 Enum 타입을 다음과 같이 정의합니다.
>An enum type is a special data type that enables for a variable to be a set of predefined constants. The variable must be equal to one of the values that have been predefined for it.

결국 Enum(enum type)은 상수로 미리 정의한 특수한 데이터 타입으로 정의하고 있다는 것을 알 수 있습니다. 

## Enum의 특징
Enum이 특수한 타입이라는 것을 알았지만 아직 Enum을 이해했다고 보기는 어렵습니다. 먼저 바이트코드를 통해 알 수 있는 사실은 내부 변수를 정적인 상수로 관리한다는 특징을 가지고 있습니다. 그렇기 때문에 Enum의 엘리먼트가 같다면 항상 같은 대상을 가리키게 됩니다.

```kotlin
val a = EnumTest.ONE
val b = EnumTest.ONE

// true
System.identityHashCode(a) == System.identityHashCode(b) 
```

시스템 상의 해시코드가 같다는 것은 같은 객체이자 변수가 같은 주소를 바라보고 있다는 것을 의미합니다. 연장해서 생각한다면 Enum의 엘리먼트는 `copy()` 메서드를 이용하여 복제본을 만들 수도 없습니다. 

또한 모든 Enum 타입은 `java.lang.Enum`을 상속하고 있다는 특징에 따라 항상 `Enum<*>`의 하위타입이 됩니다. 

```kotlin
// true
EnumTest.ONE is Enum<*>
```

## Enum의 정의방식(recursive generics)
이번에는 좀 더 깊게 Enum을 어떻게 정의하는지 알아보겠습니다. `kotlin.Enum.kt` 내부에 정의된 Enum 클래스의 정의를 살펴보면 다음과 같습니다.

```kotlin
public abstract class Enum<E : Enum<E>>(name: String, ordinal: Int): Comparable<E>
```

꽤나 복잡한 정의이기 때문에 끊어서 살펴보겠습니다.

1. 타입파라미터를 `E : Enum<E>`로 정의하고 있습니다.
2. `name`과 `ordinal`을 기본 프로퍼티로 정의하고 있습니다.
3. `Comparable<E>`를 상속(구현)하여 비교가능한 대상으로 정의하고터 있습니다.

2번과 3번은 Java나 Kotlin을 사용해 보신 분들이라면 쉽게 이해가 가실거라 생각합니다. 하지만 1번은 다소 낯설게 느껴집니다. 제네릭스가 가지는 모호함도 있겠지만 새롭게 정의한 타입인 `E`의 조건을 `Enum<E>`의 서브타입으로 정의하고 있기 때문입니다.

### 거꾸로 생각하기
그렇다면 `E : Enum<E>`라는 표현은 어디서 나오게 된 것일까요? `E : Enum<E>`에 대한 생각은 잠시 접고 맨 처음 Enum을 도입하려는 개발자 입장에서 앞서 언급한 Enum을 구현하기 위해 어떠한 요구사항이 있었을지 생각해 봅시다.

1. 먼저 Enum 타입이라고 부를 수 있는 **공통된 타입 혹은 추상화된 템플릿**이 있어야 할 것입니다. 
2. 같은 타입의 Enum 끼리는 비교가 가능하지만 Enum 자체로 캐스팅하여 비교할 수는 없어야 합니다.

2번이 약간 이해하기 어렵습니다. 기본적으로 Java나 Kotlin은 다형성을 제공하기 때문에 서브클래스는 언제든 상위 클래스를 대체할 수 있습니다. 
1에 언급한 대로라면 모든 Enum 타입은 `Enum`을 상속하기 때문에 상위 클래스로 캐스팅할 수 있습니다. 
하지만 그럼에도 서로 다른 Enum 끼리는 비교할 수 있어서는 안됩니다. 

```kotlin
class Custom : Comparable<Custom>
class SubCustom1 : Custom()
class SubCustom2 : Custom()

val sub1 = SubCustom1()
val sub2 = SubCustom2()

(sub1 as Custom) < (sub2 as Custom)
```

Kotlin에서는 상위입이 `Comparable`을 구현하고 있을 경우 상위타입으로 캐스팅을 통해서 서로 비교가 가능합니다. 
하지만 위의 코드에서 `class`가 `enum class`이라고 가정해 봅시다. 
서로 다른 Enum 타입으로 묶인 엘리먼트 간에는 비교가 불가능해야 합니다. 
한 마디로 다음 코드는 잘못된 코드라는 것이지요.

```kotlin
enum class Enum1 { SAMPLE }
enum class Enum2 { SAMPLE }

val sample1 = Enum1.SAMPLE
val sample2 = Enum2.SAMPLE

// not valid
(sample1 as Enum) < (sample2 as Enum)
```

정리하면 Enum은 비교 가능한 클래스 타입이지만 상위타입으로 캐스팅하여 서로 다른 Enum끼리 비교 가능해서는 안됩니다. 
Enum으로 정의된 클래스는 **Enum이라는 공통된 조상은 있지만 캐스팅에 있어서는 폐쇄적이어야** 합니다. 
이를 위해 트릭으로 생각해낸 것이 바로 `E : Enum<E>`와 같은 재귀적인 방식으로 정의하는 타입파라미터입니다.

## E : Enum<E>의 쓰임
본론으로 넘어와서 우리가 답해야할 질문은 다음과 같습니다.
> 그래서 `E : Enum<E>`는 도대체 왜 사용하는 것인가?

바로 답을 하기전에 인내심을 가지고 먼저 최대한 코드로 Enum과 비슷한 타입의 클래스를 정의해보고, 
재귀적인 형태의 타입파라미터를 정의했을 때의 효과를 분석해 봅시다.

```kotlin
abstract class MyEnum<E : MyEnum<E>>(
    val name: String,
    val ordinal: Int,
) : Comparable<E> {
    final override fun compareTo(other: E): Int {
        return if (this.ordinal < other.ordinal) {
            1
        } else if (this.ordinal > other.ordinal) {
            -1
        } else {
            0
        }
    }
}
```

* `MyEnum`은 하나의 공통된 템플릿이라고 할 수 있습니다.
* `MyEnum`의 타입파라미터(`E`)가 `MyEnum<E>`의 서브타입이기 때문에 내부 공통 필드(`ordinal`)를 기준으로 비교 구현이 가능합니다.

만약 `E : MyEnum<E>`이 아닌 단순 `E`로 정의한 경우라면 `compareTo()` 메서드를 구현하는데 있어 `MyEnum`에서 공통적으로 사용하는 `name`과 `ordinal`를 사용하지 못하게 됩니다.
재귀적으로 정의한 덕분에 `MyEnum`의 템플릿 내부에 공통된 로직을 정의하여 수정없이 서브타입에서 사용이 가능해졌습니다.

이번엔 추상 클래스로 정의한 `MyEnum`을 상속 받아서 엘리먼트를 생성해 봅시다. 
실제 Enum에서 엘리먼트는 정적이기 때문에`companion object` 내부에 엘리먼트를 정의하도록 합시다.

```kotlin
class SubEnum1(
    name: String,
    ordinal: Int,
) : MyEnum<SubEnum1>(
    name = name,
    ordinal = ordinal,
) {
    companion object {
        val ONE = SubEnum1("ONE", 0)
        val TWO = SubEnum1("TWO", 0)
    }
}

class SubEnum2(
    name: String,
    ordinal: Int,
) : MyEnum<SubEnum2>(
    name = name,
    ordinal = ordinal,
) {
    companion object {
        val ONE = SubEnum2("ONE", 0)
        val TWO = SubEnum2("TWO", 0)
    }
}
```

이제 `MyEnum`이 실제 Enum 처럼 동작하는지 검사하는 일이 남았습니다. 
먼저 당연하게도 `SubEnum1`은 `MyEnum<SubEnum1>`의 서브클래스입니다.

```kotlin
val sub1 = SubEnum1.ONE

// true
sub1 is MyEnum<SubEnum1>
```

또한 상위타입으로 변환 이후의 비교 연산이 불가능해야 합니다.

```kotlin
val sub1 = SubEnum1.ONE as MyEnum<*>
val sub2 = SubEnum2.ONE as MyEnum<*>

// error!!
sub1 < sub2 
```

위의 코드에서 에러가 발생하는 이유는 무엇일까요? 
각 변수를 캐스팅한 결과는 `MyEnum<*>` 입니다. 
비교 우변에 오는 `other`는 추상 클래스에서 정의하기를 `MyEnum`의 타입파라미터로 정의한 `E`가 와야 한다고 했습니다. 
당장은 와일드카드(`*`)로 타입파라미터를 뭉게버렸으니 `Nothing` 타입을 요구하게 되는 것입니다.

재귀적으로 타입파라미터를 정의했기 때문에 이를 상속하여 만든 클래스는 오로지 같은 클래스의 타입끼리만 비교가 가능해졌습니다. SubEnum1은 SubEnum1 끼리, SubEnum2는 SubEnum2 끼리만 비교가 가능하게 된 것이지요.
> `E : Enum<E>`으로 정의한 덕분에 컴파일 시점에 이미 타입을 고정할 수 있습니다.

상위타입으로 캐스팅을 하여도 고정이 됩니다.
```kotlin
val sub1 = SubEnum1.ONE
println(sub1) // SubEnum1@ae13544
println(sub1 as MyEnum<SubEnum1>) // SubEnum1@ae13544
```

## 결론
지금까지 Enum에 대한 분석을 정리하면 아래와 같습니다. 

* Enum은 폐쇄적인 정적 타입의 성질을 가지고 있습니다. 
* 재귀적인 제네릭스 정의를 이용해서 다형성을 의도적으로 차단하였습니다.
* 재귀적으로 타입파라미터를 정의하여 Enum 템플릿 내부에서 Enum에 해당하는 공통 메서드를 정의할 수 있습니다.

# References
* [The Java™ Tutorials - Enum Type](https://docs.oracle.com/javase/tutorial/java/javaOO/enum.html)
* [[열거타입] Enum 공부하다 생긴 의문점 2가지](https://hwan33.tistory.com/30)
* [Why in java enum is declared as E extends Enum<E>?](https://stackoverflow.com/questions/3061759/why-in-java-enum-is-declared-as-enume-extends-enume)
