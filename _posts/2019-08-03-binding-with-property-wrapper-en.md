---
layout: post
title: Implementation of MVVM Binding using PropertyWrapper and RxSwift
tags: [ios, swfit, propertywrapper, mvvm, binding, se-0258, rxswift, rxcocoa]
---
```
This article based on Swift5.1 and Xcode11.
If the version of Xcode changes, the implementation may vary.
See https://forums.swift.org/t/se-0258-property-wrappers-third-review/26399 for a thread on the Property Wrapper.
```
SwiftUI's min SDK is iOS13, it is difficult to apply it to existing project.  
However, some of techniques needed to implement SwiftUI are included in Swift5.1.  
So we can implement similar SwiftUI on UIKit
This article show you how to implement binding in MVVM with [PropertyWrappers(SE-0258)](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md), Rx.
  
I wrote about [(KR)MVVM패턴에 대하여]({{ site.baseurl }}/2018/12/09/what-is-mvvm.html). 
In that article I mentioned the difficulty of binding.
When you use Rx, View is easy to bind but Implementing ViewModel is difficult.  
Because all the properties of the ViewModel are Observable, it is difficult to modify or configure the values.  
Also If you give property as Subject, View can change Data.  
I want to solve problems with PropertyWrapper and Rx.
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
I used `BaseSubject <Element>` as the projectedValue to prevent direct access to the internal properties of the SubjectProperty in View.  
View only be able to access attributes of ObservableType, ObserverType(BaseSubject).  
if there is a change in the PropertyWrapper, it does not receive the event through the didSet in the ViewModel, so we implement it by placing a delegate.
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
ObservableProperty used Observable because it flows in one direction.
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
When the value of the property changes, you can receive the event via didChanged and change the value.  
If you use _ to access a property, you can access it as a SubjectProperty Type.  
If you use $ to access a property, you will access it with projectedValue Type.  
so, the ViewModel has access to all the public interfaces of the SubjectProperty.  
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
Finally, you can bind these properties in the Controller.  
View is accessing properties with $, it can be used as Observer, Observable type.  
View can not access its properties as an access level in _.  
so You can only expose the interface you want to expose in View.
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
You can create a ViewModel of the form Properties + Action (Commaned) as above.  
Repos, which is a List, uses ObservableProperty because it is a one-way stream.  
You can access Observable from the outside, but you can still change the value by accessing [Repo] type inside.  
<br/>
I have seen a simple example that makes it easier to implement bindings with PropertyWrapper.
