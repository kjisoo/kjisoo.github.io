---
layout: post
title: RxSwift를 이용한 method 호출 구독하기
tags: [iOS, Swift, RxSwift, swizzling]
redirect_from:
  - /blog/rxswift를-이용한-method-호출-관찰하기
  - /blog/rxswift를-이용한-method-호출-관찰하기/
---
RxSwift에서 swizzling을 통해 특정 메서드의 호출전, 후에 대한 이벤트를 쉽게 구독할 수 있습니다.  
RxSwift의 NSObject+Rx.swift에 두 메서드가 정의되어있습니다.  
{% highlight swift %}
public func sentMessage(_ selector: Selector) -> Observable<[Any]> 
public func methodInvoked(_ selector: Selector) -> Observable<[Any]>
{% endhighlight %}
sentMessages는 메서드가 호출 되기 전, methodInvoked는 호출된 후에 실행됩니다.

swift로 작성된 클래스/메서드에 적용하기 위해서는 @objc, dynamic  2가지 키워드와 NSObject의 상속이 적용되어 있어야 합니다.  
{% highlight swift %}
class Foo: NSObject {
  @objc dynamic func bar(_ param: String) {
    print(param)
  }
}

let foo = Foo()
foo.rx.sentMessage(#selector(Foo.bar(_:))).subscribe(onNext: { (params) in
  print("sentMessage \(params)")
})
foo.rx.methodInvoked(#selector(Foo.bar(_:))).subscribe(onNext: { (params) in
  print("methodInvoked \(params)")
})
foo.bar("call")

// sentMessage [call]
// call
// methodInvoked [call]
{% endhighlight %}
Cocoa class와 함께 사용하면 유용하게 사용할 수 있습니다.  

대부분을 기본 RxCocoa에서 지원 해주지만, 지원해주지 않는 메서드를 쉽게 변환할 수 있습니다.
{% highlight swift %}
let tableView = UITableView()
tableView.rx.sentMessage(#selector(tableView.reloadData)).subscribe(onNext: { (_) in
  print("reload")
})
tableView.reloadData()
{% endhighlight %}
위 코드와 같이 reloadData를 관찰할 수있습니다.

extension을 이용하여 기능을 추가할 수 있습니다.  
{% highlight swift %}
extension Reactive where Base: UITableView {
  var reloadData: Observable {
    return self.methodInvoked(#selector(Base.reloadData)).map { _ in Void() }
  }
}

tableView.rx.reloadData.subscribe(onNext: { (_) in
  print("reload")
})
{% endhighlight %}
