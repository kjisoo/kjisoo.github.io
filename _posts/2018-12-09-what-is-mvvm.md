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
GUI만 View를 의미하는것은 아닙니다. CLI에서는 출력되는 텍스트가 View에 해당합니다.  
  
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
ViewModel를 개발하고 View를 디자이너의 영역이라고 가정하고 ViewModel과 View가 각자 완성되었을때 이 두를 잇는게 바인딩입니다.  
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
혹은 View의 lifecycle 에 의존적이거나, (viewDidLoad, viewDidAppear, ...)  
View를 레퍼런스로 받아 뷰의 값을 직접 변경하는 일도 UI와 관련된것으로 간주합니다.   
  
#### Everything observable
MVVM예제를 처음 접했을때 모든것을 Observable 로 처리한 예제를 봤습니다.  
{% highlight swift %}
class ViewModel {
  // MARK: Input
  let password = PublishSubject<String>()
  let confirm = PublishSubject<Void>()
 
  // MARK: Output
  let isConfirm: Observable<Bool>
}
{% endhighlight %}
실제 프로젝트에선 이정도로만 구성해도 문제는 없을거라 생각이 듭니다. [kickstarter/ios-oss](https://github.com/kickstarter/ios-oss)가 위와 같이 구성되어 있습니다.  
위와 같이 구성할때 문제점 혹은 어려운점은,  
input에 대한 output이 init에서 모두 구성하거나, 별도의 observer type을 생성 해야 한다는점,  
input이 Action인지 Property의 업데이트인지 명확하지 않은점,  
Output의 초기값이 있는지, 없는지 혹은 ObserverType으로 Output을 구성했을때 외부에서 값을 변경할 수 있는점,  
Rx의 의존도가 매우 상승하는 부분이 있습니다.  
Observable property없이 생성하는 예제는 아래에서 다루겠습니다.  
  
#### View에 대해 너무 많이 아는 ViewModel
MVVM을 처음 적용해볼때 View에 코드를 최대한 남기지 않게 하다보니 ViewModel이 View에 대해 너무 많이 알게 되는 경우가 생겼습니다.  
예를 들면, ViewModel의 Observable<CGPoint> 프로퍼티를 View의 좌표에 바인딩 한다고 하면, ViewModel은 UI에 의존적인건 아니지만, UI에 대해 간접적인 지식을 알게 됩니다.(View가 어떤식으로 구성될 건지 구체적인 내용)  
해당 내용은 View의 추상화에서 더 다루겠습니다.  
  
#### Massive ViewModel
ViewModel이 Massive해지는건 아키텍쳐가 아닌 설계의 문제로 자세한 내용은, [iOS에서 MVC는 왜 망가질까](https://blog.jisoo.net/2018/11/24/why-mvc-destroyed.html)
프리젠테이션 로직을 담당하는 ViewModel이 비지니스 로직까지 책임지면서 ViewModel이 비대해지는 상황입니다.  
그래도 MVC보다 긍정적인 부분은, View가 ViewModel 인터페이스에 의존적이기 때문에 인터페이스를 유지한채 ViewModel의 리팩토링이 자유로운 점 입니다.  
  
### 고민, 어려움
#### View의 추상화
ViewModel은 추상화된 View입니다.  
추상화 된 View란 무엇인가에 대해 고민할 필요가 있었습니다.  
어떤 기능을 수행하는 ViewModel이 있고 이를 이용해 여러 형태의 View를 만들 수 있다면 이 ViewModel은 추상화가 잘 되어있다고 할 수 있을거 같습니다.  
View가 iOS의 View뿐만 아니라 MacOS 심지어 CUI로도 구현하는것도 여허 형태의 View에 속합니다.  
ViewModel은 수행하고자 하는 기능에 문제가 없는 수준의 인터페이스를 제공하기만 하고 실제 View가 어떤 식으로 구성되던지 신경쓰지 않아야 합니다.  
예를 들어,  
{% highlight swift %}
class ViewModel {
  var volume: Int
}
{% endhighlight %}
volume 만 있는 ViewModel 이 있으면 해당 volume을 변경하기 위해서 slider를 이용하건 textfield를 이용하건 ViewModel은 신경쓰지 않고 작동해야 합니다.  
  
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
Mediator와 EventBus를 통해 어느정도 해결이 가능합니다.  
Mediator는 ViewModel간에 의존성을 명시적으로 표시할 수 있습니다. 의존성간에 Mediator를 작성하는것이 과도할 수 있고 의존적인 ViewModel이 많을경우 Mediator가 더 복잡해질 수 있습니다.  
EventBus는 ViewModel간에 의존성을 명시적으로 표시할 순 없지만, 간단하고 더 많은 ViewModel에게 이벤트를 전파할 수 있습니다.  
  
#### 데이터만 있는 ViewModel
데이터의 변형이 없이 Entity의 데이터를 그대로 사용할때가 있습니다.  
Cell들이 대부분 이에 해당하는데,  
{% highlight swift %}
struct Memo {
  let title: String
  let content: String
  let id: Int
}
{% endhighlight %}
위와 같이 메모의 title, content를 그대로 사용하고 id은 사용하지 않는다고 가정할 때  
ViewModel은 각 데이터들을 전달만 하게 됩니다.  
Mirror를 활용해 Proxy형태의 ViewModel을 만들고자 했는데 좋은 결과를 얻지 못했습니다.  
가능한 옵션은 3가지가 있는데,  
1. Entity를 전달하기
2. ViewModel에서 중복으로 필드를 구현하기
3. ViewModel에서 Entity를 전달하기 단, Entity는 Immutable
{% highlight swift %}
// 1. Entity를 전달하기
class ListViewModel {
  let memos: Observable<Memo>
}
 
// 2. ViewModel에서 중복으로 필드를 구현하기
class ListViewModel {
  let memos: Observable<MemoViewModel>
}
 
class MemoViewModel {
  let title: String
  let content: String
 
  init(memo: Memo) {}
}
 
// 3. ViewModel에서 Entity를 전달하기 단, Entity는 Immutable
class MemoViewModel {
  let memo: Observable<Memo>
}
{% endhighlight %}
저는 가능하면 2, 3번을 사용합니다.  
각 셀마다 ViewModel을 만드는 수고에 비해 추가적인 동작이나 값의 변환이 필요할 때 수정이 쉽다는 장점이 더 크다고 생각됩니다.  
추가로 ViewModel 스스로 값을 변경할 수 있습니다. `MemoViewModel`에서 Memo의 id를 이용해 entity의 변경을 구독하고 값이 변경되었을때 ListViewModel와 상관없이 스스로 값을 변경할 수 있습니다.  
[ReactiveMvvm-github](https://github.com/envicase/ReactiveMvvm), [Reactive MVVM(Model-View-ViewModel) 모바일 응용프로그램 아키텍쳐](https://justhackem.wordpress.com/2015/03/19/rmvvm-architecture/) 에 좋은 내용이 많이 있습니다.  
  
#### Properties, Commands
속성은 Property로 액션은 Command로 분류할 수 있습니다. [.NET Command interface](https://docs.microsoft.com/ko-kr/dotnet/api/system.windows.input.icommand?view=netframework-4.7.2)
Command에는 canExecute, Execute의 속성이 있습니다. 이를 Reactive로 구성하여 UIControl에 바로 바인딩할 수 있게 확장할 수 있습니다.  
아이디와 비밀번호를 입력하고 로그인을 하는 ViewModel 이라면, 
{% highlight swift %}
// Observable로 구성된 ViewModel
class ViewModel {
  // MARK: Input
  let id = PublishSubject<String>()
  let password = PublishSubject<String>()
  let login = PublishSubject<Void>()
 
  // MARK: Output
  let result: Observable<Bool>
  let validation: Observable<Bool>
}
 
// Command와 함께 구성된 ViewModel
class ViewModel {
  let id = PublishSubject<String>()
  let password = PublishSubject<String>()
  let loginCommand = Command<Void>(canExecute: Observable<Bool>, execute: func)
}
{% endhighlight %}
Command 로 구성할 경우 property와 Action이 명확하게 구분이 됩니다.  
Command를 UIControl(UIButton, Switch, ..)에 바로 바인딩 할 수 있게 확장하고 Command의 canExecute의 속성을 이용해 버튼의 Enabled를 컨트롤할 수 있습니다.  
Properties를 keypath와 Observe를 이용하여 바인딩할 수 있습니다.  
{% highlight swift %}
class ViewModel {
  @obj dynamic var id = ""
  @obj dynamic var password = ""
  @obj dynamic private(set) var someProperty = ""
  let loginCommand = Command()
}
{% endhighlight %}
위 방법의 장점은 one way, two way가 명확하단 점과 데이터의 이전 데이터 참조가 쉬운 부분이 있습니다.  
@obj, dynamic을 통해서 런타임 메시징이 되어 swift의 성능보단 떨어지는점, enum이나 @objc타입이 불가능한 타입은 사용이 불가능한 단점이 있습니다.  
[kjisoo/RxObserve](https://github.com/kjisoo/RxObserve/) 해당 방법을 쉽게 사용할 수 있는 라이브러리를 제작했습니다.  
  
#### Navigation & Dependency Injection
화면과 화면 사이에 이동을 어떻게 처리 해야할지, 어디서 처리해야 할지에 대해 정답은 없습니다.  
Coordinator pattern을 사용해 MVVM-C 형태를 취하는 방법이 있습니다. [Coordinators Essential tutorial. Part I](https://medium.com/blacklane-engineering/coordinators-essential-tutorial-part-i-376c836e9ba7)  
Coordinator에서 화면 전환과 DI 를 함께 진행할 수 있습니다.  
혹은 Controller가 Dependency resolver가 되어 Navigation을 함께 진행할 수 있습니다.  
Coordinator혹은 Navigator와 같이 별도의 객체가 이를 담당하여 처리할 경우 역할이 명확하다는 장점이 있지만 프로젝트 구성이 장황해지는 단점이 있습니다.  
Controller에서 처리할 경우 단순하다는 장점이 있지만 프로젝트가 커지면 흐름을 추적하기 힘든 부분이 있습니다.  
Storyboard + [Swinject](https://github.com/Swinject/Swinject), [SwinjectStoryboard](https://github.com/Swinject/SwinjectStoryboard)를 이용해 Storyboard에서 Navigation을 처리하고 SwinjectStoryboard에서 필요한 값을 inject 하는 방법도 좋았습니다. 해당 방법에서는 생성자 주입보단 프로퍼티 주입이 편해지는점, SnapKit등을 활용해 Code base UI를 구성하기 힘들다는점등의 단점이 있습니다.  
  
### MVVM은 좋은가?
MVVM을 구성하기 위해선 많은 기술적인 준비가 되어 있어야 합니다.  
혹은 현재 있는 기술로도 충분히 좋은 설계도 가능합니다. VIPER, [uber/RIBs](https://github.com/uber/RIBs), 혹은 Unidirectional Architecture 등 많은 대안들이 있습니다.  
준비를 선택했다면 많은 바퀴들을 만들어야 프로젝트를 잘 굴릴 수 있을거라 생각합니다.  
Android, WPF의 DataBinding 비슷하게 아래 처럼 뷰를 구성할 수 있다면 좋겠다는 생각을 했습니다.  
{% highlight swift %}
func setupUI() {
  view.addSubViews([
    UITextField(text: viewModel.text, constraint: []),
    UIButton(title: "Done", command: viewModel.doneCommand, constraint: [])
  ])
}
{% endhighlight %}
MVVM이 제대로 정착되기 위해선 필요한 바퀴들이 만들어져 있는 라이브러리 혹은 프레임워크가 만들어 지거나 만들어야 하는데 단 시간내에는 힘들어 보입니다.  
그럼에도 여전히 테스트에 매우 좋고 역할이 잘 분리되어 있는 패턴이기에 이곳저곳에서 부족한 부분을 보충하고 해결방법들이 나올것이라 기대합니다.  
MVVM은 좋은가에 대해선 iOS에서 제대로 적용된 MVVM을 적용하기 힘들다가 대답이 될거 같습니다.  
