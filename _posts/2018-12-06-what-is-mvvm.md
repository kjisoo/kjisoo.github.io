---
layout: post
title: MVVM패턴에 대하여
tags: [ios, mvvm, pattern, architecture, di, mediator]
---
iOS에서 MVVM 패턴을 공부하면서 잘못 이해했던 부분, 잘못 알려진 부분들, 고민했던 부분들에 대해 적어봅니다.  
우선 MVVM에 대해 먼저 이야기 해보면 Model, ViewModel, View 로 마틴파울러의 [Presentation Model](https://martinfowler.com/eaaDev/PresentationModel.html) 변형입니다.  
#### Model
**Domain Model, Data access Layer, Business Logic**등이 Model에 해당됩니다.  
Model은 MVC, MVP등의 Model과 동일합니다.  
  
정의 부터 살펴보면,  
#### View
View는 UI와 관련된 부분들을 말합니다.  
iOS에서는 Controller도 View로 취급합니다.  
터치, 제스쳐, 애니메이션등도 이곳에 해당합니다.  
GUI만 View를 의미하는것은 아닙니다. CUI에서는 출력되는 텍스트가 View에 해당합니다.  
  
#### ViewModel
**추상화된 View의 Model(abstraction of the view)**입니다.  
View의 상태를 스테이트 머신처럼 다루며 UI의 의존적이지 않습니다.  
**Presentation Logic**이 ViewModel에 해당됩니다.  
Properties와 Commands를 통해서 ViewModel을 조작할 수 있습니다.  
  
MVVM에서 ViewModel은 스테이트 머신과 프리젠테이션 로직을 처리하게 됩니다.  
이로인해 View는 ViewModel만 알면 되고, ViewModel뒤에 있는 다른 레이어에 간/직접적인 의존성이 없게 됩니다.  
ViewModel은 여러 모델에 의존하여 요청을 처리합니다.  
ViewModel은 properties(text, items, ...)와 commands(send, select, ...)를 통해 View에 조작이 가능한 인터페이스를 제공합니다.  
ViewModel의 인터페이스를 통해 ViewModel과 의존적인 Model에 대해 테스트를 쉽게할 수 있습니다.  
View는 ViewModel의 인터페이스와 바인딩(Binding)되어야 합니다.  
바인딩을 통해 View와 ViewModel은 항상 동일한 상태를 갖게 됩니다.  
바인딩을 위해서는 기술적인 도움이 필요합니다. WPF에서는 data binding이 기본으로 제공되며, Android에서도 Data binding을 제공하고 있습니다.  
iOS에서는 바인딩을 위해 제공되는 부분이 없으므로 주로 Rx를 이용하게 됩니다.  
MVVM은 Data binding, Reactive, 그외 기술들에 의존적인 아키텍쳐지만 swift에서는 지원하지 않은 기능이 많기 때문에 더 어렵게 느껴질 수 있습니다.  
이제부터는 오해 했던 부분이나, 고민 했던 부분들에 대해 적어봅니다.  
  
### 잘못된 예제, 오해
#### ViewModel에서 UI 컴포넌트를 사용
인터넷에 돌아다니는 예제를 보면 ViewModel에서 UI와 관련된 데이터를 넘기는 경우가 있습니다.  
{% highlight swift %}
class ViewModel {
  let mainImage = PublishSubject<UIImage>()
}
{% endhighlight %}
위와 같이 UIImage 혹은 View에 값을 수정하는것을 ViewModel 에서 하면 안됩니다.  
  

#### everything observable

#### View에 대해 너무 많이 아는 ViewModel

#### Massive ViewModel

### 고민, 어려움
#### Confirm같은 UI와 밀접하게 연관된 부분
View에서 처리가 가능하면 View에서 처리하면 되지만 ViewModel에서 처리 해야 한다면,  
UI에 의존적이지 않은 Interface에 의존하게 합니다.  
{% highlight swift %}
protocol Confirmable {
  func show(title: String, message: String) -> Single<Bool>
}
{% endhighlight %}
ViewModel은 Confirmable에는 의존적이지만, UI와는 여전히 분리되어 있습니다.  
`Confirmable`을 따르는 Mock을 생성하여 테스트도 여전히 가능합니다.  
  
#### ViewModel 간에 의존성
ViewModel이 다른 ViewModel에 의존적일 수 있습니다.  
이때 ViewModel에 직접 접근하여 개발하다 보면, 의존성 그래프가 매우 복잡해질 수 있습니다.  
`Mediator`와 `EventBus`를 통해 어느정도 해결이 가능합니다.  
`Mediator`는 ViewModel간에 의존성을 명시적으로 표시할 수 있습니다. 의존성간에 Mediator를 작성하는것이 과도할 수 있고 의존적인 ViewModel이 많을경우 Mediator가 더 복잡해질 수 있습니다.  
`EventBus`는 ViewModel간에 의존성을 명시적으로 표시할 순 없지만, 간단하고 더 많은 ViewModel에게 이벤트를 전파할 수 있습니다.  

#### 데이터만 있는 ViewModel

#### one way, two way binding

#### Commands

#### Dependncy Injection 

#### Navigation
