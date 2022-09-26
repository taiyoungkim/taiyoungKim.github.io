---
layout: post
title: 코틀린의 널 안전성(Null Safety)
tags: kotlin
author: taeyoung kim
categories: kotlin
date: 2022-09-26 20:10 +0900
---

*이 글은 [코틀린 공식문서](https://kotlinlang.org/docs/reference/null-safety.html)를 공부하며 번역한 글입니다. 틀린 부분이나 어색한 부분을 댓글로 알려주시면 감사하겠습니다.*

## Null이 될 수 있는 타입과 Null이 될 수 없는 타입(Nullable types and Non-Null Types)
코틀린의 타입(type) 시스템은 [Billion Dollar Mistake](http://en.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions)라고도 알려진 null 참조 코드의 위험성을 없애기 위한 것입니다.

Java를 포함한 많은 프로그래밍 언어에서 가장 일반적인 함정 중 하나는, null 참조의 멤버에 접근하면 null 참조 예외(null reference exception)가 발생한다는 것입니다. Java에서는 이것을 `NullPointerException` 또는 `NPE`라고 합니다.

코틀린의 타입(type) 시스템은 코드에서 `NullPointerExeption`을 제거하기 위한 것입니다.
코틀린에서 NullPointerException이 발생할 수 있는 경우:
- `throw NullPointerException`을 명시적으로 호출하는 것;
- 아래에서 설명하는 것과 같이 `!!` 연산자를 사용하는 것;
- 초기화와 관련하여 아래와 같은 특성으로 데이터의 불일치가 발생할 때:
  - 생성자에서 `this`를 초기화하지 않고도 사용할 수 있으며, 다른 곳으로 전달되어 사용할 수 있음("leaking `this`" 라고 함);
  - [수퍼 클래스 생성자는 파생 클래스의 구현에서 초기화되지 않은 상태를 사용하는 open member를 호출함]((https://kotlinlang.org/docs/reference/classes.html#derived-class-initialization-order));
- Java interoperation:
  - [플랫폼 유형](https://kotlinlang.org/docs/reference/java-interop.html#null-safety-and-platform-types)의 `null` 참조에서 멤어에 접근하려고 시도함
  - Generic types used for Java interoperation with incorrect nullability, e.g. a piece of Java code might add null into a Kotlin MutableList<String>, meaning that MutableList<String?> should be used for working with it;
  - 외부의 자바 코드로 인한 기타 문제들

코틀린에서는 타입 시스템이 `null`이 가능한 참조와 그렇지 않은 참조를 구분합니다. 예를 들어, `String`은 `null`을 참조할 수 없습니다.

```kotlin
var a: String = "abc"
a = null // 컴파일 에러 발생
```

null을 참조하기 위해서는 `String?`와 같이 선언해야 합니다.

```kotlin
var b: String? = "abc"
b = null // ok
print(b)
```

만약 NPE를 발생시키지 않도록 보장된 `a`라는 메소드를 호출하거나 프로퍼티에 접근한다면, 안전하게 아래와 같이 선언할 수 있습니다:

```kotlin
val l = a.length
```

하지만 만약 NPE에 안전하지 안흔 `b`와 같은 프로퍼티에 접근한다면, 컴파일 에러가 발생할 것입니다.

```kotlin
val l = b.length // error: variable 'b' can be null
```

하지만 우리는 여전히 `b` 프로퍼티에 접근해야합니다. 어떻게 해야 할까요? 이 문제를 해결하기 위한 몇 가지 방법이 아래에 있습니다.

## 조건에서 null을 확인하기(Checking for null in conditions)
우선, 명시적으로 `b`가 null인지를 체크하여 두 가지 옵션을 구분하여 다루어야 합니다:

```kotlin
val l = if (b != null) b.length else -1
```

컴파일러는 작성된 코드가 수행하는 검사를 추적하여 `if` 내부의 `length`를 호출하는 것을 허용합니다. 이보다 더 복잡한 조건도 지원됩니다:

```kotlin
val b: String? = "Kotlin"
if (b != null && b.length > 0) {
    print("String of length ${b.length}")
} else {
    print("Empty string")
}
```

이는 `b`가 변경 불가능한 경우일때에만(즉, 검증(check)과 사용(usage) 사이에서 수정되지 않은 지역 변수 혹은, backing field가 있고 재정의할 수 없는 멤버 변수) 동작합니다. 그렇지 않으면 `b`가 검증 이후에 `null`이 될 수 있기 때문입니다.

## 안전한 호출(Safe Calls)
두 번째 방법은 안전한 호출 연산자인 `?.`을 사용하는 것입니다:

```kotlin
val a = "Kotlin"
val b: String? = null
println(b?.length)
println(a?.length) // Unnecessary safe call
```

위의 코드는 `b`가 null이 아니면 `b.length`를 리턴하고, 그렇지 않으면 null을 반환합니다. 이 표현식의 타입은 `Int?`입니다.

안전한 호출(safe calls)은 체인(chain)에서 유용합니다. 예를 들어, 직원인 Bob이 부서에 배정된 경우(혹은 그렇지 않은 경우), 다른 직원을 부서장으로 둔 다음, Bob의 부서 장(만약 있다면)의 이름을 얻기 위해 아래와 같은 코드를 작성합니다:

```kotlin
bob?.department?.head?.name
```

이 체인에서 프로퍼티가 null인 것이 하나다로 있다면, 이 체인은 `null`을 리턴합니다.

null이 아닌 값에 대해서만 특정 연산을 수행하려면 아래와 같이 안전한 호출 연산자인 [let](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/let.html)을 사용하면 됩니다:


```kotlin
val listWithNulls: List<String?> = listOf("Kotlin", null)
for (item in listWithNulls) {
    item?.let { println(it) } // prints Kotlin and ignores null
}
```

A safe call can also be placed on the left side of an assignment. 그 다음, 만약 안전한 호출 체인의 리시버중 하나가 null이면 할당을 건너뛰고, 오른쪽의 표현식은 고려되지 않습니다.

```kotlin
// If either `person` or `person.department` is null, the function is not called:
person?.department?.head = managersPool.getManager()
```

## 엘비스 연산자(Elvis Operator)
null을 참조할 수 있는 `r`이 있을 때, "만약 `r`이 null이 아니면 해당 값을 사용하고, `r`이 null이면 `x`라는 null이 아닌 값을 사용한다"라고 표현할 수 있습니다:

```kotlin
val l: Int = if (b != null) b.length else -1
```

위의 if 표현식을 엘비스 연산자인 `?:`를 사용하여 아래와 같이 표현할 수도 있습니다.
```kotlin
val l = b?.length ?: -1
```

`?:` 연산자는 결과값이 null이 아니면 연산자 왼쪽의 값을 반환하고, 값이 null이면 오른쪽의 값을 반환합니다. 오른쪽 표현식은 왼쪽의 값이 null인 경우에만 계산된다는 것을 꼭 기억하세요.

코틀린에서는 `throw`와 `return`이 표현식이므로 엘비스 연산자의 오른쪽에서도 사용할 수 있습니다. 이는 함수의 인수를 확인할때와 같은 경우에 매우 유용합니다.

```kotlin
fun foo(node: Node): String? {
    val parent = node.getParent() ?: return null
    val name = node.getName() ?: throw IllegalArgumentException("name expected")
    // ...
}
```

## !! 연산자(The !! Operator)
세번쨰 방법은 NPE 애호가를 위한 것입니다: not-null을 선언하는 연산지인 `!!`는 모든 값을 null이 아닌 타입으로 변환하고, 값이 null인 경우에는 예외를 throw합니다. 예를 들면, `b!!`와 같이 쓸 수 있으며 이것은 null이 아닌 `b`값(예, `String`)을 리턴하거나 `b`가 null인 경우 NPE를 throw합니다:

```kotlin
val l = b!!.length
```

따라서, NPE를 원한 경우 NPE를 발생시킬 수 있지만 명시적으로 요청해야하며 파란색으로 표시되지 않습니다.

## 안전한 캐스팅(Safe Casts)
객체가 target 타입이 아닌 경우 규칙적인 타입 캐스잉으로 인해 `ClassCastException` 발생할 수 있습니다. 이 떄의 방법은, null을 리턴하는 안전한 캐스팅을 사용하는 것입니다.

```kotlin
val aInt: Int? = a as? Int
```

## 널이 가능한 타입 컬렉션(Collections of Nullable Type)
null을 입력할 수 있는 타입의 요소 컬렉션이 있을 때, null이 아닌 요소를 필터링하고 싶다면 `filterNotNull`을 사용할 수 있습니다:

```kotlin
val nullableList: List<Int?> = listOf(1, 2, null, 4)
val intList: List<Int> = nullableList.filterNotNull()
```

### 참고 문헌
[코틀린 공식문서](https://kotlinlang.org/docs/reference/null-safety.html)