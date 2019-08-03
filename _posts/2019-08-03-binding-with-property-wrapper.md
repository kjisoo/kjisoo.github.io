---
layout: post
title: PropertyWrapper, RxSwift를 이용한 MVVM Binding 구현
tags: [ios, swfit, propertywrapper, mvvm, binding, se-0258, rxswift, rxcocoa]
---
```
해당 구현은 Swift5.1 Xcode11 beta5에 의존합니다. 
Xcode 정식 버전이 나오거나 베타 버전이 변함에 따라 구현 방법이 달라질 수 있습니다.
https://forums.swift.org/t/se-0258-property-wrappers-third-review/26399 해당 쓰레드에서 변화와 관련된 자세한 내용을 확인할 수 있습니다. 
```

이번에 나올 예정인 SwiftUI는 아쉽게도 min SDK가 13이기 때문에 바로 적용하기엔 무리가 있습니다.  
대신, Swift UI를 지탱하고 있는 기술들이 Swift5.1 에 포함됨에 따라 비슷한 형태의 구현을 UIKit 위에서 할 수 있게 되었습니다.  
이번에는 Swift5.1에서 추가될 [PropertyWrapper(SE-0258)](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md)와 RxSwift+RxCocoa를 이용한 MVVM 패턴을 구현해보고자 합니다.  
  
기존에 작성한 [MVVM패턴에 대하여](https://blog.jisoo.net/2018/12/09/what-is-mvvm.html)에서 언급한것처럼 iOS에서 Binding은 어려운 존재입니다.  
Rx를 활용했을때의 문제점은 데이터의 순환 참조가 어렵다는 문제, Input과 Output이 분리되는등의 문제점이 있었습니다.  
이를 PropertyWrapper와 Rx를 통해 해결해보고자 했습니다.   
최종적인 목표는, ViewModel 내에서 내부 데이터 참조를 쉽게 하고 `ex) viewModel.property.value() (x), viewmodel.property (o)`, binding시에는 이미 잘 개발되어 있는 풍부한 Rx의 기능을 사용하는것 입니다.  
<br/>
{% highlight swift %}
class BaseSubject<Element>: ObservableType, ObserverType {
  func subscribe<Observer>(_ observer: Observer) -> Disposable where Observer : ObserverType, Element == Observer.Element { fatalError() }
  func on(_ event: Event<Element>) { fatalError() }
}
 
@propertyWrapper
class SubjectProperty<Element>: BaseSubject<Element> {
  var wrappedValue: Element {
    didSet {
      subject.onNext(wrappedValue)
      delegate?.didChanged(key: key)
    }
  }
  weak var delegate: BindableObject?
  
  private let key: AnyHashable
  private let subject: BehaviorSubject<Element>
  
  init(wrappedValue: Element, key: AnyHashable) {
    self.wrappedValue = wrappedValue
    self.key = key
    self.subject = BehaviorSubject<Element>(value: wrappedValue)
  }
  
  var projectedValue: BaseSubject<Element> {
    return self
  }
  
  override func subscribe<Observer>(_ observer: Observer) -> Disposable where Observer : ObserverType, Element == Observer.Element {
    return subject.subscribe(observer)
  }
   
  override func on(_ event: Event<Element>) {
    switch event {
    case .next(let element):
      wrappedValue = element
    default:
      break
    }
  }
}
{% endhighlight %}
SubjectProperty의 내부 속성에 직접적인 접근을 막기 위해 projectedValue로 `BaseSubject<Element>`를 사용했습니다.  
이렇게 하면 View에서는 ObservableType, ObserverType의 속성만 접근할 수 있기 때문에 wrappedValue, delegate 등에 접근할 수 없게 됩니다.  
PropertyWrapper내에서 변화는 해당 Property를 사용하는 곳의 didSet을 통해 이벤트를 받지 못하기 때문에 delegate 를 두어 이를 구현합니다.  
<br/><br/> 
{% highlight swift %}
@propertyWrapper
class ObservableProperty<Element>: ObservableType {
  var wrappedValue: Element {
    didSet {
      subject.onNext(wrappedValue)
    }
  }
  
  private let subject: BehaviorSubject<Element>
  
  init(wrappedValue: Element) {
    self.wrappedValue = wrappedValue
    self.subject = BehaviorSubject<Element>(value: wrappedValue)
  }
  
  var projectedValue: Observable<Element> {
    return subject.asObservable()
  }
  
  func subscribe<Observer>(_ observer: Observer) -> Disposable where Observer : ObserverType, Element == Observer.Element {
    return subject.subscribe(observer)
  }
}
{% endhighlight %}
ObservableProperty는 단방향으로 데이터가 흐르기 때문에 Observable를 사용했습니다. 
<br/><br/> 
{% highlight swift %}
protocol BindableObject: class {
  func didChanged(key: AnyHashable)
}
 
class ViewModel: BindableObject {
  @SubjectProperty(wrappedValue: "", key: \ViewModel.email)
  var email: String
  @SubjectProperty(wrappedValue: "", key: \ViewModel.password)
  var password: String
  
  lazy var loignAction = CocoaAction(enabledIf: canExecuteLogin,
                                                  workFactory: { (_) -> Observable<Void> in
                                                    return Observable<Void>.just(()).delay(.seconds(5), scheduler: MainScheduler.asyncInstance)
  })
  
  private let canExecuteLogin = BehaviorSubject<Bool>(value: false)
  
  init() {
    _email.delegate = self
    _password.delegate = self
  }
  
  func didChanged(key: AnyHashable) {
    switch key.hashValue {
    case (\ViewModel.email).hashValue,
         (\ViewModel.password).hashValue:
      canExecuteLogin.onNext(email.count >= 10 && password.count >= 4)
    default:
      break
    }
  }
}
{% endhighlight %}
property의 값이 변화되면, didChanged를 통해 이벤트를 수신하게 되고 값을 변경할 수 있습니다.  
`init`에서 _를 이용해 프로퍼티를 접근하면, SubjectProperty type으로서 해당 프로퍼티를 접근할 수 있습니다.  
$를 사용해서 접근하면, projectedValue Type으로 접근하게 됩니다. 따라서 ViewModel에서는 SubjectProperty의 공개되어 있는 인터페이스에 모두 접근할 수 있습니다.  
<br/><br/>
{% highlight swift %}
class ViewController: UIViewController {
  @IBOutlet private weak var email: UITextField!
  @IBOutlet private weak var password: UITextField!
  @IBOutlet private weak var confirm: UIButton!

  let viewModel = ViewModel()
  
  private let disposeBag = DisposeBag()
  
  override func viewDidLoad() {
    super.viewDidLoad()
    
    disposeBag.insert(viewModel.$email.bind(to: email.rx.text),
                      viewModel.$password.bind(to: password.rx.text),
                      email.rx.text.orEmpty.bind(to: viewModel.$email),
                      password.rx.text.orEmpty.bind(to: viewModel.$password))
    confirm.rx.action = viewModel.loignAction
  }
}
{% endhighlight %}
최종적으로 Controller에서 해당 속성들을 바인딩할 수 있습니다.  
View에서는 프로퍼티들을 $로 접근하고 있기 때문에, Observer, Observable의 타입으로서 사용이 가능합니다.  
_의 접근 제한자 덕분에 View에서는 해당 속성에 접근할 수 없기 때문에 View에 노출하고 싶은 인터페이스만 노출이 가능합니다. 
<br/><br/> 
{% highlight swift %}
struct Repo {
  let name: String
  let author: String
}
  
class ListViewModel {
  @ObservableProperty
  private(set) var repos: [Repo] = []
  
  lazy var loadMoreAction = CocoaAction(enabledIf: canExecuteLoadMore,
                                        workFactory: { [weak self] _ -> Observable<Void> in
                                          // fetch data and append
                                          return .just(())
  })
  
  private let canExecuteLoadMore = BehaviorSubject<Bool>(value: true)
  
  private func appendRepo(_ repos: [Repo]) {
    if repos.isEmpty {
      canExecuteLoadMore.onNext(false)
    } else {
      self.repos.append(contentsOf: repos)
    }
  }
}
{% endhighlight %}
위와 같이 Properties + Action(Commaned)형태의 ViewModel 를 만들 수 있습니다.  
List인 Repos는 단방향 스트림이기 때문에 ObservableProperty를 사용합니다.  
외부에선 Observable로 접근할 수 있지만 내부에선 여전히 [Repo] 타입으로 접근하여 값을 변경할 수 있습니다.  
<br/><br/>
간단한 예제를 통해 PropertyWrapper를 통해 바인딩을 더 편하게 구현할 수 있는 방법을 보았습니다.  
앞으로 더 많은 사람이 이러한 방법을 연구하여 기존 UIKit에서도 유연한 설계를 위한 방법들이 추가되지 않을까 기대해봅니다.  

