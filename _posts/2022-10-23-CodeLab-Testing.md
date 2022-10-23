---
layout: post
title: CodeLab Testing 5.2 정리
tags: android, testing
author: taeyoung kim
categories: android
date: 2022-10-23 18:40 +0900
---

안드로이드 테스트에 관련된 내용 정리

## Codelab **Introduction to Test Doubles and Dependency Injection**

이번 codelab에서는 Test Double과 의존성 주입 (Dependenct Injection)를 알아볼것이다.

이번 시간에 배우게 될 테스트 작성: 

1. Repository unit tests
2. Fragments and viewmodel intergration test
3. Fragment navigation test

배우게 될 것들:

- Testing 전략을 세우는 방법
- Test Double, fakse와 mocks를 만들고 사용하는 방법
- Android에서 unit과 integration tests를 위한 의존성 주입을 사용하는 방법
- Service Locator 패턴을 적용하는 방법
- repositories, fragments, viewmodel 그리고 Navigation component를 테스트 하는 방법

이제 계속 진행하기 전에 testing strategy를 먼저 보겠다.

testing strategy에 대해 생각할 때 세 가지 테스트 측면이 있다.

- Scope - 테스트가 얼마나 많은 코드에 영향을 주는지를 나타낸다. 테스트는 한개의 메소드에서부터 전체 어플리케이션까지 다양하다.
- Speed - 테스트가 얼마나 빠른지를 나타낸다. 테스트 속도는 밀리초에서 몇 분까지 다양하게 시간이 걸린다.
- Fidelity - 테스트가 얼마나 실제와 유사한지를 나타낸다. 예를 들어 테스트 중인 코드의 일부가 네트워크 요청을 해야하는 경우 테스트 코드가 실제로, 이 네트워크 요청을 하는지, 아니면 결과를 위조하는지에 따라 테스트의 충실도(Fidelity)는 달라진다. 테스트가 실제로 네트워크와 통신하는 경우 충실도가 더 높아진다는 의미이다. 그러나 단점은 테스트를 실행하는데 시간이 더 오래 걸리고 네트워크가 다운된 경우 오류가 발생하거나 사용 비용이 많이 들 수 있다.

이러한 측면 간에는 고유한 절충점이 있습니다. 예를 들어, 속도와 충실도는 트레이드 오프입니다. 테스트가 빠를수록 일반적으로 충실도가 낮고 그 반대도 마찬가지입니다. 자동화된 테스트를 나누는 일반적인 방법은 다음 세 가지 범주로 나누는 것입니다.

- **Unit Test** - 단일 클래스, 일반적으로 해당 글래스의 단일 메서드에서 실행되는 고도로 집중된 테스트이다. Unit test가 실패하면 코드에서 문제가 있는 위치를 정확히 알 수 있다. 현실 세계에서 앱이 하나의 메서드나 클래스를 실행하는 것보다 훨씬 더 많은 것을 포함하기 때문에 정확도가 낮다. 코드를 변경할 때마다 실행할 수 있을 만큼 빠르기 때문에 대부분 로컬에서 실행되는 테스트다. ex) ViewModel, repository의 단일 메서드
- **Integration Test** - 여러 클래스의 상호 작용을 테스트하여 함께 사용할 때 예상대로 작동하는지 확인합니다. integration test를 구성하는 한 가지 방법은 작업 저장 기능과 같은 단일 기능을 테스트하도록 하는것이다. unit test보다 넓은 범위를 테스트하지만 완전한 충실도를 갖는것보다 빠르게 실행하도록 최적화 되어있다. 상황에 따라 로컬로 실행하거나 계측 테스트로 실행할 수 있다. ex) fragment와 ViewModel의 모든 기능 테스트
- **End to end tests (E2e)** - 함께 작동하는 기능 조합을 테스트합니다. 앱의 많은 부분을 테스트하고 실제 사용을 밀접하게 시뮬레이션하므로 일반적으로 느리다. 그러나 가장 최고의 충실도를 가지고있고 어플리케이션의 전체적으로 제대로 작동하는지를 테스트한다. 대체로 이러한 테스트는 계측 테스트(androidTest 소스에서) ex) 전체 앱을 시작하고 몇 가지 기능을 함께 테스트한다.

이러한 테스트의 제안된 비율은 피라미드로 표시가 되고 대부분 아래 그림과 같이 나눈다.

![Untitled](/assets/20221023CodeLabTesting/Untitled.png)

이제 실제 프로젝트에서 테스트를 진행해보자.

### Fake Data Source 만들기

클래스의 일부에 대한 unit test를 작성할때 목표는 **해당 클래스의 코드만 테스트**하는 것이다.

특정 클래스의 코드만 테스트하는 것은 까다로울수 있다. 한번 예를 들어보자. 공부하는 codelab의 data.source.DefaultTaskRepository에서 클래스를 보자. 우리의 목표는 해당 클래스 코드만 테스트 하는것이다. 그러나 DefaultTaskRepository은 LocalTaskDataSource나 RemoteTaskDataSource 같은 다른 클래스에 따라 작동한다. 이런것을 다른말로 **종속성**이라고 한다.

따라서 모든 메서드는 DefaultTaskRepository 데이터 소스 클래스의 메서드를 호출하고, 다른 클래스의 메서드를 호출하여 정보를 데이터베이스에 저장하거나 네트워크와 통신한다.

![Untitled](/assets/20221023CodeLabTesting/Untitled%201.png)

예시로 아래 소스를 보자.

```kotlin
suspend fun getTasks(forceUpdate: Boolean = false): Result<List<Task>> {
    if (forceUpdate) {
        try {
            updateTasksFromRemoteDataSource()
        } catch (ex: Exception) {
            return Result.Error(ex)
        }
    }
    return tasksLocalDataSource.getTasks()
}
```

getTasks는 repository에 대해 수행할 수있는 가장 기본적인 호출 중 하나이다. 이 방법에는 SQLite 데이터베이스에서 읽고 네트워크 호출이 포함된다. 여기에는 저장소 코드보다 훨씬 더 많은 코드가 포함된다.

저장소 테스트가 어려운 몇 가지 구체적인 이유는 다음과 같다.

1. 이 repository에 대한 가장 간단한 테스트를 수행하기 위해 데이터베이스를 만들고 관리하는 것에 대한 생각을 처리해야 합니다. 그러면 “로컬 테스트여야 하나 아니면 계측 테스트여야 하나?” 같은 질문이 생긴다.
2. 네트워킹 코드와 같은 코드의 일부는 실행하는데 오랜 시간이 걸리거나 때로는 실패하여 장기간 실행되는 불안정한 테스트를 생성할 수 있다.

> **Flaky** 테스트는 동일한 코드에서 반복적으로 실행될 때 통과할 때도 있고 실패할 때도 있는 테스트입니다. 가능하면 피하십시오 .
> 
1. 테스트 실패로 인해 어떤 코드에 결함이 있는지 진단하는 기능이 테스트에서 손실될 수 있다. 테스트는 비저장소 코드 테스트를 시작할 수 있으므로 예를 들어 데이터베이스 코드와 같은 일부 종속 코드의 문제로 인해 가정된 “저장소” unit test가 실패할 수 있다.

### Test Double

이에 대한 해결책은 저장소를 테스트할 때 *실제 네트워킹* 또는 *데이터베이스 코드*를 **사용하지 않고**

대신 테스트 더블을 이용하는 것이다. 테스트 더블은 테스트를 위해 특별히 제작된 클래스이다. 테스트에서 클래스의 실제 버번을  대체하기 위한것이다. 영화에서 위험한 액션을 스턴트 배우가 대신 하는거랑 비슷하다.

다음은 몇 가지 유형의 테스트 더블이다.

- Fake - • 복잡한 로직이나 객체 내부에서 필요로 하는 다른 외부 객체들의 동작을 단순화하여 구현한 객체. 테스트에는 적합하지만 프로덕션에는 부적합한 방식으로 구현되는 테스트 더블
- Mock - 어떤 메서드가 호출되었는지 추적하는 테스트 더블입니다. 그런 다음 메서드가 올바르게 호출되었는지 여부에 따라 테스트를 통과하거나 실패한다.
- Stub - 로직을 포함하지 않고 프로그래밍한 내용만 반환하는 테스트 더블이다. (하드코딩된 내용으로 테스트를 하는걸 예로 들수있다.)
- Dummy - 객체는 전달되지만 사용되지 않는 테스트 더블이다. 단지 인스턴트화 된 객체가 필요하고,  해당 객체의 기능까지 필요하지 않은 경우에 사용
- Spy - 몇 가지 추가정보를 추적하는 테스트 더블이다.

안드로이드에서 가장 일반적으로 사용되는 테스트 더블은 Fake와 Mock이다.

위 내용으로 DefaultTasksRepository에 대한 테스트를 요약하면 아래와 같다.

1. 단위 테스트는 일반적으로 단일 클래스를 테스트하는 고도로 집중된 테스트이다.
2. `DefaultTasksRepository`은 **두 가지 복잡한 종속성** `TasksLocalDataSource` `TasksRemoteDataSource`이 있으므로 단위 테스트가 어렵다.
3. 이 문제를 해결하기 위해 테스트할 때 와 대체할 **테스트 더블** `TasksLocalDataSource` `TasksRemoteDataSource`를 만든다.
4. 이렇게 하면 `DefaultTasksRepository`코드만 테스트하는 단위 테스트를 작성할 수 있다.

이제 직접 코드에 테스트를 추가해보자.

먼저 테스트를 위한 DataSource가 필요하니깐 FakeDataSource를 만든다.

그리고 기존 소스의 TasksDataSource 인터페이스를 각각 Local과 Remote에서 어떻게 사용되는지를 확인하고 메서드를 구현한다.

그러나 `DefaultTasksRepository` 에서는 TasksDataSource를 직접 선언해서 사용하고 있다. 이렇게 될 경우 테스트 더블로 교환 할 방법이 없기 때문에 종속성 주입을 해야한다.

기존의 선언되는 부분은 지우고 외부에서 주입 하도록 수정을 하면 준비는 됐다.

이제 `DefaultTasksRepository`의 테스트 코드를 만들고 @Before 어노테이션을 통해 repository를 FakeDataSource로 주입하여 테스트 준비를 한다.

그리고 @Test 어노테이션을 이용해 실제 테스트 하려는 코드를 추가하고 테스트를 하면 된다.

이제까지 repository를 테스트 하는 방법이였고 자세한 예시와 설명은 codelab에서 볼수있다. (꼭 손으로 직접 해보는걸 추천한다.)

### Set Up Fake Repository

이제 viewmodel에서 unit test와 integration test를 해볼것이다.

unit test는 관심 있는 클래스나 메서드**만 테스트**해야 한다.이를 **isolation(격리)** 테스트 라고 하며 “Unit”을 명확하게 격리하고 해당 단위의 일부인 코드만 테스트한다.

따라서 TasksViewModel 만 테스트해야 한다. TasksViewModel에 연결된 데이터베이스, 네트워크 또는 repository 클래스에서 테스트를 해서는 안된다. 따라서 위에 repository를 테스트 할때와 같은 문제이니 다시 동일하게 Fake Repository를 만들어서 테스트에 사용하면 된다.

![Untitled](/assets/20221023CodeLabTesting/Untitled%202.png)

생성자 종속성 주입을 사용하는 첫번째 단계는 fake 클래스 간에 공유되는 공통 인터페이스를 만드는 것이다.

기존 `DefaultTasksRepository` 를 안드로이드 스튜디오를 통해 interface로 변경한다.

이제 인터페이스가 있으므로 테스트 더블을 만들수있게된다.

`FakeTestRepository` 를 만들고 `DefaultTasksRepository` 를 인터페이스화 시킨 `TasksRepository`를 상속받는다.

그리고 이제 메서드를 모두 선언하고 viewmodel에서도 `TasksRepository` 로 종속성을 주입해서 테스트와 동일하게 작동하게 만든다.

이제 동일하게 만든 테스트 더블로 테스트를 해볼 시간이다.

fragment 및 viewmodel 테스트를 위한 unit test를 작성한다. 이때 ServiceLocator 패턴과 에스프레소 및 Mockito 라이브러리의 도움을 받는다.

Unit test는 여러 클래스의 상호 작용을 테스트하여 함께 사용할 때 예상대로 작동하는지 확인합니다. 이런한 테스트는 로컬(test) 또는 integration test(androidTest)로 실행 할 수있다.

먼저 상대적으로 기본적인 기능 위주로 가지고 있는 `TaskDetailFragment` 를 테스트 작성한다.

`TaskDetailFragment` 를 우클릭해서 테스트를 생성하고 unit test가 아닌 integration test 이므로 androidTest 경로에 생성한다.

그 후 2가지 주석을 추가해주는데

```
@MediumTest
@RunWith(AndroidJUnit4::class)
```

이 두가지를 상단에 추가해준다.

이것은 각각 

- `**@MediumTest**`는 “중간 런타임” Integration test로 표시한다. (`@SmallTest` 는 Unit test `@LargeTest`는 end-to-end tests를 뜻한다.) 이렇게 하면 실행할 테스트 크기를 그룹화하고 선택하는데 도움이 된다.
- **`@RunWith(AndroidJUnit4::class)`**— AndroidX 테스트를 사용하는 모든 클래스에서 사용됩니다.

이제 테스트를 작성하면 테스트 작성된 내용은 저장이 되지 않는다.

### ServiceLocator 만들기

이 작업에서는 ServiceLocator를 사용하여 fragment에 fake repository를 제공한다. 이렇게 하면 fragment를 작성하고 integration test를 할 수 있게 된다.

viewmodel이나 repository에 종속성을 제공해야 할 때, 이전처럼 여기에서 Constructor 종속성 주입을 사용할 수 없다. Constructor 종속성 주입을 위해서는 클래스를 생성해야 하는데 fragment나 activity는 일반적으로 Constructor에 액세스 할 수 없는 클래스이기 때문이다.

그래서 constructor 종속성 주입을 사용하여 repository 테스트 더블(FakeTestRepository)를 fragment로 바꿀 수 없다. 대신 ServiceLocator 패턴을 사용하면 된다.

ServiceLocator 패턴은 의존성 주입의 대안이다. ServiceLocator라는 싱글톤 클래스를 만들면 일반 코드와 테스트 코드 모두 종속성을 제공할 수 있다. 일반 앱 코드에서 종속성은 일반 종속성이고 테스트의 경우 테스트 더블로 제공된다.

이제 codelab에 적용하면 싱글톤이기 때문에 object 파일로 생성하고 database와 `tasksRepository` 를 선언한다. 그리고 각각 database와 repository에 필요한 datasourece나 repository들을 추가로 정의하면 된다.

이제 repository 클래스를 하나만 만드는것이 중요하기 때문에 이를 보장하기 위해 application에서 serviceLocator를 사용한다.

그리고 integretion 테스트 코드를 추가할 `FakeAndroidTestRepository` 을 추가한다. 이때 기존의 `FakeTestRepository` 가 있지만 AndroidTest에서 다시 만드는 이유는 test와 androidTest 소스 세트 간 파일 공유가 안되기 때문에 따로따로 만들어준다.

이제 `FakeAndroidTestRepository` 까지 만들었다면 마지막 `ServiceLocator`를 설정해줄 시간이다. `ServiceLocator` 에서 tasksRepository에 설정자를 @VisibleForTesting을 해준다. 테스트를 unit test를 실행하든 integretion test로 실행하든 정확히 동일하게 실행이 되어야 한다. 이 말이 의미하는 바는 테스트에서 서로 종속적인 동작이 없어야 한다는 것이다. (테스트 간에 개체 공유를 피하는것을 의미)

싱글톤이기 때문에 ServiceLocator 테스트 간에 실수로 공유될 가능성이 있다. 이를 방지하려면 ServiceLocator 테스트 사이에 적절하게 재설정하는 메서드를 만들어야한다.

### Espresso로 Integration test 하기

espresso로 테스트를 하기 전에 먼저 espresso에 대해 알아보자.

Espresso는 간단하게 말하자면 UI test를 쉽게 작성하게 해주는 테스팅 프레임워크이다.

Espresso를 사용하면 내 어플리케이션의 UI와 테스트를 자동으로 동기화를 해주고 또한, 테스트를 실행하기 전 Activity가 먼저 실행되는것을 보장해준다. 한마디로 Activity의 모든 백그라운드 작업이 끝날때까지 기다렸다가 테스트를 수행한다.

espresso 테스트는 실제 장치에서 실행되므로 본질적으로 integration test이다. 발생하는 한가지 문제는 애니메이션이다. 애니메이션이 지연되고 보기가 화면에 있는지 테스트하려고 하지만 여전히 애니메이션이 진행 중인 경우 espresso가 실수로 테스트에 실패할 수 있다. 그러므로 espresso 테스트를 하기 전에는 애니메이션 옵션을 끄는것이 좋다.

이제 espresso 코드를 한번 살펴보자

```kotlin
onView(withId(R.id.task_detail_complete_checkbox)).perform(click()).check(matches(isChecked()))
```

대부분의 espresso는 네 부분으로 구성된다.

1. Static Espresso method
    
    `onView`는 Espresso 문을 시작하는 Static Espresso method의 예시이다. onView가 가장 일반적인 옵션 중 하나이지만 onData와 같은 다른 옵션도 있다.
    
2. ViewMatcher
    
    `withId(R.id.task_detail_title_text)` withId 는 ID로 view를 얻는 예시이다. 공식 문서에서 찾을 수 있는 다른 ViewMatcher 또한 있다.
    
3. ViewAction
    
    `perform(click())` ViewAction은 view에 대해 수행할 수 있는 작업이다. 예를 들어 위에 코드에서는 view를 클릭하는것이다.
    
4. ViewAssertion
    
    `check(matches(isChecked()))` ViewAssertion은 view에 대해 확인하거나 view에 대해 어떤것을 assert한다. 가장 일반적으로 사용하는것은 matches이다. 
    

![Untitled](/assets/20221023CodeLabTesting/Untitled%203.png)

이제 위에 예시를 토대로 테스트 코드를 작성하면 된다.

### Mockito를 사용하여 Navigation test를 하기

이제 마지막으로 mock이라고 하는 다른 유형의 테스트 더블을 테스트 라이브러리인 Mockito를 사용하는 법을 배워보자.

이 codelab에서는 fake라는 테스트 더블을 사용했다. fake는 여러 유형의 테스트 더블 중 하나이다. 그럼 Navigation 테스트를 할때는 어떤 테스트 더블을 사용해야할까?

Navigation이 어떻게 작동되는지를 생각해보자. `TasksFragment` 중 하나의 task를 눌러 세부 정보 화면으로 이동한다고 상상해보자.

```kotlin
private fun openTaskDetails(taskId: String) {
    val action = TasksFragmentDirections.actionTasksFragmentToTaskDetailFragment(taskId)
    findNavController().navigate(action)
}
```

지금 현재 이동하는 코드 부분이다.

Navigation은 navigate 메서드 호출로 인해 발생한다. assert문을 작성해야 하는 경우 `TaskDetailFragment` 로 이동했는지 여부를 테스트할 수 있는 간단한 방법은 없다. Navigation은 초기화 외에 명확한 출력이나 상태 변경을 일으키지 않는 복잡한 작업이다.

그래서 확인 할 수 있는 방식은 navigate 메소드가 올바르게 호출이 되었다는 것을 확인하는 것이다. 이것이 바로 mock 테스트 더블이 하는 일이다. 특정 메서드가 호출되었는지 여부를 확인하는 일 말이다.

Mockito는 테스트 더블을 만들기 위한 프레임워크이다. mock이라는 단어는 API와 이름에 사용되지만 단지 mock을 만들기 위한 것은 아니다. stub과 spy도 만들수 있다.

이제 codelab 코드에서 Mockito를 사용 `NavigationController`하여 탐색 메서드가 올바르게 호출되었음을 주장할 수 있는 모의 객체를 만든다.

이렇게가 Codelab에서 테스트 코드 공부 두번째인 테스트 더블 및 종속성 주입 소개 부분이다.

자세한 설명이나 실제 코드를 넣는 부분은 해당 글보다는 Codelab에 들어가서 직접 해보면서 하는게 가장 좋다.

이제 마지막으로 요약을 하고 글을 마무리 한다.

- 테스트하려는 항목과 테스트 전략에 따라 앱에 구현할 테스트 종류가 결정됩니다.
    
    **Unit 테스트** 는 집중적이고 빠릅니다. **Integration 테스트**는 프로그램 부분 간의 상호 작용을 확인합니다. End to end 테스트는 기능을 확인하고, 충실도가 가장 높으며, 자주 계측되며 실행하는 데 시간이 더 오래 걸릴 수 있습니다.
    
- 앱의 아키텍처는 테스트의 난이도에 영향을 미칩니다.
- 테스트를 위해 앱의 일부를 분리하기 위해 테스트 더블을 사용할 수 있습니다. **테스트 더블**은 테스트를 위해 특별히 제작된 클래스 버전입니다. 예를 들어, 데이터베이스나 인터넷에서 데이터를 가져오는 것을 가장합니다.
- **종속성 주입**을 사용 하여 실제 클래스를 리포지토리 또는 네트워킹 계층과 같은 테스트 클래스로 대체합니다.
- i**nstrumented testing** (`androidTest`)를 사용하여 UI 구성 요소를 시작합니다.
- 예를 들어 프래그먼트를 시작하기 위해 생성자 종속성 주입을 사용할 수 없는 경우 서비스 로케이터를 사용할 수 있습니다. **Service Locator 패턴**은 의존성 주입의 대안입니다 . 여기에는 "서비스 로케이터"라는 싱글톤 클래스를 만드는 것이 포함되며, 그 목적은 일반 코드와 테스트 코드 모두에 종속성을 제공하는 것입니다.