---
title: 'Enum 톺아보기(2)'
date: 2024-01-20T16:51:32+09:00
draft: false
tags: ["kotlin"]
categories: ["development"]
---

이번 시간에는 Enum을 사용하면서 쉽게 놓질 수 있는 성능적인 측면에 대해서 알아보도록 하겠습니다.

# values() vs entires
Enum 내부의 엘리먼트를 조회하는 방식은 흔히 `values()`를 사용합니다. 
`values()`는 정의된 모든 엘리먼트를 배열 형태로 반환합니다.
따라서 일반적으로는 `values()`와 스트림을 조합하여 원하는 엘리먼트를 찾을 수 있습니다.

```kotlin
SomeEnum.values().filter { /* condition */ }
```

## values() 의 문제
하지만 `values()`를 사용하는 데에는 몇가지 치명적인 단점이 있습니다.

* `values()` 호출과 동시에 메모리에 새로운 배열을 할당하여 엘리먼트 배열을 반환합니다.
* `values()`가 반환하는 `Array` 타입은 수정이 가능한(mutable) 타입입니다.
* 활용을 위해서 배열을 리스트로 바꾸는 반복적인 작업이 동반됩니다.

성능적인 측면에서 봤을 때 `values()`는 매번 새롭게 메모리를 할당하기 때문에 미리 정의된 엘리먼트 리스트를 사용하는 것 보다 오버헤드가 큰 작업이 될 수 있습니다.
만약 한 Enum에 정의한 엘리먼트가 많고 내부 프로퍼티도 다양하다면 그만큼 부하가 큰 작업이 될 것입니다.
이를 보완하기 위해서 Kotlin 1.9 부터는 `entries`라는 새로운 프로퍼티를 제공합니다.

## entries의 사용

`entries`는 `EnumEntries` 타입을 반환합니다. 
공식 문서에 나와있는 코드를 참고하면 리스트를 상속한 인터페이스라는 것을 알 수 있습니다.

```kotlin
@SinceKotlin("1.9")
@WasExperimental(ExperimentalStdlibApi::class)
public sealed interface EnumEntries<E : Enum<E>> : List<E>
```

또한 기본적으로 `entries`는 미리 정의한 리스트를(pre-allocated) 반환합니다. 
따라서 매 호출마다 메모리를 정의해야 했던 `values()`가 가졌던 문제점을 개선시켰습니다.
추가로 `List` 타입으로 반환하기 때문에 수정이 불가능하고(immutable) 활용이 넓다는 장점이 생겨 IDE에서도 `values()`보다 `entries`를 사용하는 것을 권장하고 있습니다.

### 참고😅
실제 테스트를 돌린 결과는 의외로 `values()`의 속도가 더 빠른 것으로 확인이 되었습니다. 
현재로는 배열과 리스트의 차이로 인한 결과이거나 테스트로 사용한 `EnumSut`의 엘리먼트 개수(10개)가 두 작업의 차이를 극대화 시키지 못한게 아닐까 추측을 하고 있습니다. 

```kotlin
// 10억번 반복 

// values() 20958 ms
fun ofCodeWithValues(code: String): EnumSut {
    return values().firstOrNull { it.code == code } ?: DEFAULT
}
// entries 21682 ms
fun ofCodeWithEntries(code: String): EnumSut {
    return entries.firstOrNull { it.code == code } ?: DEFAULT
}
```

# Enum을 조회하는 방식
Enum을 사용하면서 원하는 엘리먼트에 해당하는 변수를 직접 사용할 수 있지만 Enum이 가지는 프로퍼티에 따라서 엘리먼트를 가져와야 하는 상황이 생길 수 있습니다.
예를 들어 Enum을 `name`을 기준으로 필터링할 때처럼 말입니다.

보통은 위에서 소개한 `values()`나 `entries`를 `filter` 함수와 같이 사용하여 원하는 엘리먼트를 비교적 쉽게 추출할 수 있습니다.
하지만 이 방식은 $$O(n)$$의 복잡도를 가지기 때문에 단시간에 많은 요청을 처리하는 상황에서는 순회하여 엘리먼트를 추출하는 방식은 적합하지 않을 수 있습니다.

## 실험🧪
간단한 테스팅을 통해서 순회하는 방식과 프로퍼티를 `Map`으로 변형하여 조회하는 방식을 비교해 봅시다.

>가설은 "Map을 이용한 조회는 $$O(1)$$로 $$O(n)$$인 순회 방식보다 더 효율적일 것이다"는 겁니다.

실험할 케이스를 정리해 보면 name을 사용할 때와 커스텀 프로프터를 조회할 때로 나눴습니다. 
`ordinal`을 제외한 이유는 보통 실제 상황에서 `ordinal`을 기반으로 조회하는 것은 위험할 수 있어 이 케이스는 배제했습니다.

### Map 만들기
Map을 만드는 방식은 entries와 associatedBy를 활용하였습니다.
```kotlin
val codeMap: Map<String, EnumSut> = entries.associateBy { it.code }
private val nameMap: Map<String, EnumSut> = entries.associateBy { it.name }
```

`name`의 경우는 `valueOf()`라는 메서드를 지원하고 있기 때문에 `nameMap`과 `valueOf()` 방식을 테스트하였습니다.
`valueOf()`의 소스코드를 살펴보면 내부에 `Map` 형태로 저장하고 있는 `enumConstantDirectory`에서 원하는 엘리먼트를 반환하는 방식입니다. 

```java
// Enum.java
public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                            String name) {
    T result = enumType.enumConstantDirectory().get(name);
    if (result != null)
        return result;
    if (name == null)
        throw new NullPointerException("Name is null");
    throw new IllegalArgumentException(
        "No enum constant " + enumType.getCanonicalName() + "." + name);
}

// Class.java
Map<String, T> enumConstantDirectory() {
    Map<String, T> directory = enumConstantDirectory;
    if (directory == null) {
        T[] universe = getEnumConstantsShared();
        if (universe == null)
            throw new IllegalArgumentException(
                getName() + " is not an enum type");
        directory = new HashMap<>((int)(universe.length / 0.75f) + 1);
        for (T constant : universe) {
            directory.put(((Enum<?>)constant).name(), constant);
        }
        enumConstantDirectory = directory;
    }
    return directory;
}
```

## 실험 결과🧪
실험 결과는 예상대로 Map 형태를 구성해서 사용하는 방식이 람다를 사용한 방식보다 훨씬 빠르다는 것을 알 수 있었습니다.

| 메서드            | 시간     | 비교 | 예상 복잡도 | 비고           |
| ----------------- | -------- | ---- | ----------- | -------------- |
| ofNameByMap       | 1108 ms  | 1    | O(1)        | `nameMap` 사용 |
| valueOf           | 3415 ms  | 3    | O(1)        | `valueOf` 사용 |
| ofCodeByMap       | 4215 ms  | 3.8  | O(1)        | `codeMap` 사용 |
| ofCodeWithEntries | 21624 ms | 19.5 | O(n)        | `filter` 사용  |

실험 결과를 요약하면 다음과 같습니다.
* Enum의 경우 `name`을 기반으로 조회를 하는 것이 가장 효율적입니다.
* 커스텀 프로퍼티를 사용하더라도 `Map`을 미리 만들어서 사용하는 것이 `filter`를 사용하는 방식보다 더 효율적입니다.

# 결론
지금까지 Enum에 대한 분석을 정리하면 아래와 같습니다.


# References
* [Kotlin Enums — Replace values() with entries](https://engineering.teknasyon.com/kotlin-enums-replace-values-with-entries-bbc91caffb2a)