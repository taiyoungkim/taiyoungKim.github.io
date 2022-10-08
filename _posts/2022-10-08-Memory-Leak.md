---
layout: post
title: 메모리 릭에 대해(Memory Leak)
tags: android
author: taeyoung kim
categories: android
date: 2022-10-08 19:10 +0900
---

# Memory Leak

Java(더 정확히 말하면 JVM)의 핵심 이점 중 하나는 바로  garbage collector (GC) 일 것이다.  garbage collector가 알아서 메모리를 할당하고 해제해주는 덕분에 우리는 메모리에 대해 걱정하지 않고 새로운 object를 만들 수 있다.

**그럼 정말 garbage collector가 알아서 메모리를 할당하고 해제해주니깐 걱정을 안해도 될까?**

절대 아니다!  우리는 가비지 컬렉터가 어떻게 작동하는지 제대로 이해해야, 가비지 컬렉터가 메모리를 해방시켜 out-of-memory exception과 앱에서 지연을 초래할 수 있는 Memory Leak를 방지할 수 있다.

### **Memory Leak이란?**

<aside>
📝 *Failure to release unused objects from the memory*

</aside>

Failure to release *unused objects from the* memory란 말은 애플리케이션에 GC가 메모리에서 지울 수 없는 미사용 개체가 있다는 말이다.

메모리에서 GC가 미사용 개체를 지우는데 실패했을때, 애플리케이션 또는 메서드가 끝날 때까지 메모리에서 지우지 못한 미사용 개체가 메모리를 점유하게 된다. (케바케일수있다.)

### ****Memory Leaks: 일시적 vs 영구적****

Memory Leak은 두 가지로 나눌수 있다:

**애플리케이션이 종료**될때까지 메모리를 차지하고 있는것(**영구적**)과 **메소드가 종료**될때까지 메모리를 차지하고 있는것(**일시적**)이 있다.

첫번째(**애플리케이션 종료**)는 간단하게 앱이 종료되면 GC는 앱이 사용한 모든 메모리를 해제합니다.

두번째(**메소드 종료**)는 설명이 조금 필요하다. 아래 예시를 보자

Method X가 있다고 가정해 보겠습니다. Method X는 완료하는 데 5분이 소요되는 백그라운드에서 작업을 수행하고 있으며 일부 문자열을 검색하기 위해 액티비티 컨텍스트를 제공했습니다. 1분 후에 activity를 닫기로 결정했지만 메서드는 여전히 실행 중이며 작업을 완료하는 데 4분이 더 필요합니다. 이 시간 동안 우리는 더 이상 사용되지 않는 activity 객체를 보유하고 있으며 GC는 이를 메모리에서 해제할 수 없습니다. 메서드가 작업을 완료하면 GC가 다시 실행되고 메모리에서 지우고 회수할 수 있다.

우선 memory leaks과 garbage collector에 대해 더 알아보기 전에 먼저 기본부터 알아보고 들어가는게 좋을거같다.

### RAM(Memory)이란?

RAM (Random access memory)은 현재 실행 중인 애플리케이션과 해당 데이터를 저장하는 데 사용되는 Android 장치 또는 컴퓨터의 메모리입니다.

RAM의 두가지 주요 특 으을 하명명면 첫번째는 Heap 그리고 두번째는 Stack입니다.

이제 이 두가지에 대해 알아봅시다.

**Heap 및 Stack**

![Untitled](/assets/20221008MemoryLeak/Untitled.png)

간단하게 설명하자면, Stack은 **정적** 메모리 할당에 사용되고 Heap은 **동적** 메모리 할당에 사용됩니다. Heap과 Stack이 모두 RAM에 저장된다는 점에 유의하십시오.

**Heap 메모리에 대한 추가 정보**

Java Heap 메모리는 가상 머신에서 개체를 할당하는데 사용됩니다. 객체를 생성할때 마다 항상 Heap에 생성이 됩니다. JVM또는 DVM과 같은 가상 머신은 정기적인 garbage collector (GC)를 수행하여 더 이상 참조되지 않는 모든 객체의 Heap 메모리를 향후 할당에 사용할 수 있도록 합니다.

원할한 사용자가 경험을 제공하기 위해 Android는 실행 중인 각 애플리케이션의 Heap 크기에 엄격한 제한을 설정합니다. Heap 크기 제한은 기기마다 다르며 장치의 RAM 용량에 따라 다릅니다. 앱이 이 Heap 제한에 도달하고 더 많은 메모리를 할당하려고 하면 앱이 수신하고 `[OutOfMemoryError](https://developer.android.com/reference/java/lang/OutOfMemoryError.html)`종료됩니다.

**애플리케이션의 Heap 크기가 얼마인지 궁금하신가요?**

안드로이드에는 Dalvik VM(DVM)이 있습니다. DVM은 모바일 장치에 최적화된 고유한 Java 가상 머신입니다. 메모리, 배터리 수명 및 성능을 위해 가상 머신을 최적화하고 각 애플리케이션에 대한 메모리 양을 분배하는 역할을 합니다.

DVM의 두 줄에 대해 이야기해 보겠습니다.

1. *dalvik.vm.**heapgrowthlimit**:*
    
    이 줄은 응용 프로그램의 Heap 크기에서 Dalvik이 시작되는 방식을 기반으로 합니다. 각 애플리케이션의 기본 Heap 크기입니다. 앱이 도달할 수 있는 최대값!
    
2. dalvik.vm.**heapsize**:
    
    이 줄은 더 큰 Heap에 대한 최대 Heap 크기를 나타냅니다. 애플리케이션 매니페스트(android:largeHeap=”true”)에서 더 큰 Heap을 Android에 요청하여 이를 달성할 수 있습니다.
    

앱에서 더 큰 Heap을 사용하지 마세요. 이 단계의 부작용을 정확히 알고 있는 경우에만 수행하십시오. 여기에서 주제를 계속 연구하는 데 충분한 정보를 제공할 것입니다.

다음은 장치 RAM에 따라 얻은 Heap 크기를 보여주는 표입니다.

```
+==================================================== ==================+
|DVM |1GB RAM |2GB RAM |3GB RAM 이상 |
+==================================================== ==================+
|DEFAULT(Heap 성장 제한) |64m |128m |256m |
+--------------------------+---------+---------+-- ------------------+
|LARGE(Heap 크기) |128m |256m |512m |
+--------------------------+---------+---------+-- ------------------+
```

RAM이 많을수록 Heap 크기가 커집니다. 높은 RAM을 가진 모든 장치가 512m를 초과하는 것은 아닙니다. 장치 RAM이 3GB가 넘는 경우 Heap 크기가 512m보다 큰지 확인하기 위해 장치에 대한 조사가 필요하다.

**기기에서 heap size를 확인 하는 방법은?**

ActivityManager를 사용한다. 메서드`[getMemoryClass()](https://developer.android.com/reference/android/app/ActivityManager.html#getMemoryClass())` 나 `[getLargeMemoryClass()](https://developer.android.com/reference/android/app/ActivityManager.html#getLargeMemoryClass())`를 사용하여 런타임 시 최대 힙 크기를 확인할 수 있습니다. (large heap이 설정되어 있을때)

- *getMemoryClass():* 기본 최대 heap size를 반환한다.
- *getLargeMemoryClass():* manifest에서 large heap flag를 사용한 후 최대 사용 가능한 heap size를 반환한다.

```
ActivityManager am = getSystemService(ACTIVITY_SERVICE);
Log.d("XXX", "dalvik.vm.heapgrowthlimit: " + am.getMemoryClass());
Log.d("XXX", "dalvik.vm.heapsize: " + am.getLargeMemoryClass());
```

### **그럼 실제 코드에서는 어떻게 작동 될까?**

샘플 코드를 보면서 어떻게 데이터가 heap과 stack에 저장되는지 확인하자.

```java
public class Memory {

    public static void main(String[] args) { // Line 1
        int i = 1; // Line 2
        Object obj = new Object(); // Line 3
        Memory memo = new Memory(); // Line 4
        memo.foo(obj); // Line 5
    }

    private void foo(Object param) { // Line 6
        String str = param.toString(); // Line 7
        System.out.println(str);
    } // Line 8
} // Line 9
```

아래 이미지는 앱의 heap과 stack을 나타내며, 앱 실행 시 각 객체가 가리키는 위치와 저장되는 위치를 보여줍니다.

![Untitled](/assets/20221008MemoryLeak/Untitled1.png)

코드를 한 줄씩 살펴보고 heap 또는 stack에서 개체가 할당, 저장 및 해제되는 시기를 보겠습니다.

```java
public static void main(String[] args) { // Line 1
```

1번째 줄 - JVM의 기본 메소드에 대한 stack 메모리 **블록을 생성**한다.

 

```java
int i = 1; // Line 2
```

2번째 줄 - 기본 지역 변수가 생성된다. 변수가 **생성**되어 **메인 메소스**의 **stack** 메모리에 **저장**된다.

```java
Object obj = new Object(); // Line 3
```

3번째 줄 - 새로운 **객체**가 생성되어 heap에 저장되고 **객체 heap 메모리 주소**가 stack에 저장된다.

*Note: 객체 heap 메모리 주소는 **main method’s**에 저장된다**.***

```java
Memory memo = new Memory(); // Line 4
```

4번째 줄 - 3번째 줄과 동일하게 작동한다.

```java
memo.foo(obj); // Line 5
```

5번째 줄 - JVM은 foo 메소드에 대한 stack 메모리 **블록을 생성합니다.** 

```java
private void foo(Object param) { // Line 6
```

6번째 줄 - 5번째 줄에서 전달한 객체의 **heap** **메모리 주소를 저장**하는 foo 메서드의 **stack** 메모리에 새 객체를 만듭니다.

*Note: Java는 항상 매개변수 변수를 **값**으로 전달합니다 . **Java**의 객체 변수는 항상 메모리 heap의 실제 객체를 가리킵니다.*

```java
String str = param.toString(); // Line 7
```

7번째 줄 - **문자열 pool**의 heap 메모리 주소를 **저장**하는 foo 메소드 stack에 새 객체를 생성한다.

```java
} // Line 8
```

8번째 줄 - 마지막 줄에 도달했을 때, foo 메소스는 **종료**되고 foo 메소드의 **stack 블록** 안에 **객체**는 메모리가 **해제**된다.

```java
} // Line 9
```

9번째 줄 - 8번째와 같다. 마지막 줄에 도달했을 때, main 메소스는 **종료**되고 main 메소드의 **stack 블록** 안에 **객체**는 메모리가 **해제**된다.

****메소드가 종료되면 어떻게 됩니까?****

각 메서드에는 고유한 범위가 있습니다. 메서드가 완료되면 개체가 자동으로 제거되고 스택에서 회수됩니다.

![Untitled](/assets/20221008MemoryLeak/Untitled2.png)

foo 메소드가 종료되었을때, 위 그림처럼 foo 메소드의 stack 블록은 **자동으로** 제거되고 재활용된다.

![Untitled](/assets/20221008MemoryLeak/Untitled3.png)

그림2 또한 그림1과 같습니다. main 메소드가 종료되었을때, 위 그림처럼 main 메소드의 stack 블록은 **자동으로** 해제되고 재활용된다.

### **Conclusion**

이제 **stack에** 있는 객체는 **한정된 시간**으로만 존재한다는게 명확해졌습니다. 메소드가 한번 완료되면 객체는 해제되고 재활용된다.

stack은 LIFO (Last-In-First-Out) 데이터 구조이다. 

그래서 stack을 **사용하는 프로그램은 push** 및 **pop** 의 두 가지 간단한 작업만으로 모든 작업을 쉽게 관리할 수 있습니다 .

![Untitled](/assets/20221008MemoryLeak/Untitled4.png)

위에 들었던 예시처럼 변수나 메소드가 선언 될때 마다 stack에 쌓이고 메소드를 나갈때 stack에서 제거와 재활용이 됩니다.

### 그렇다면 Heap은 어떨까?

heap은 stack과 다르다. heap 메모리에서 객체의 제거와 재활용에는 도움이 필요하다. JVM에는 이미 이 일을 도와주는 슈퍼 히어로가 있다. 우리가 부르는 **Garbage Collector**가 그 역할을 한다**.** GC는 우리를 위해 힘든 일을 할 것이다. 그리고 사용하지 않는 객체를 감지하고 제거하고 메모리에서 더 많은 공간을 확보하는 데 신경을 써 줄것이다.

![Untitled](/assets/20221008MemoryLeak/Untitled5.png)

### **garbage collector는 어떻게 일 할까?**

간단하다. garbage collector는 unused 나 unreachable 객체를 찾고 있다가 heap에 참조가 없는 객체가 있으면  garbage collector가 메모리에서 객체를 제거하고 더 많은 공간을 회수합니다.

![Untitled](/assets/20221008MemoryLeak/Untitled6.png)

GC roots는 JVM에서 참조하는 객체이다. GC root는 트리의 최상위 객체이다. 트리 안의 모든 객체는 한개 이상의 객체를 갖게 된다. 애플리케이션이나 GC root가 도달할 수 있는 한 모든 루트, 객체, 트리는 도달할 수 있다. 애플리케이션이나 GC 루트에서 연결할 수 없게 되면 사용하지 않는 개체 또는 연결할 수 없는 개체로 간주됩니다.

### GC 작업 전 후의 메모리 상태는?

![Untitled](/assets/20221008MemoryLeak/Untitled7.png)

위 그림은 애플리케이션 메모리의 현재 상태이다. heap에 미사용 객체가 가득 차 있고 stack은 비워져있다. 

![Untitled](/assets/20221008MemoryLeak/Untitled8.png)

GC 작업 후 heap 안에 있던 모든 미사용 객체는 GC에 의해 제거되었다. 

### Memory leak**는 언제 어떻게 발생하나?**

stack이 여전히 heap에서 미사용 개체를 참조할 때 memory leak이 발생합니다.

이해를 위해 아래 이미지를 봅시다.

![Untitled](/assets/20221008MemoryLeak/Untitled9.png)

위 그림에서 stack에서 참조된 객체가 있지만 더 이상 사용되지 않는 경우를 볼 수 있습니다. Garbage collector는 해당 객체가 사용되지 않는 동안 사용 중임을 보여주기 때문에 메모리에서 객체를 해제하거나 제거하지 않습니다.