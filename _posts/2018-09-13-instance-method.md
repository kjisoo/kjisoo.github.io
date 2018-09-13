---
layout: post
title: swift에서 메서드를 파라미터로 넘기기
tags: [swift, class, struct, method]
---
swift에서는 인스턴스 메서드(instance method)를 클로저를 대신해 전달할 수 있습니다.  
reference type인 class와 value type인 struct간의 차이도 존재합니다.  

### class 에서의 기본 동작
class에서는 강한 참조를 발생시킵니다.  
많은 경우에 순환 참조를 발생시켜 메모리 릭을 발생시킵니다.  
{% highlight swift %}
class Foo {
  func emptyFunc() {
  }
  
  deinit {
    print("foo deinit")
  }
}

var foo: Foo? = Foo()
var closure: (() -> Void)? = foo?.emptyFunc // 1
foo = nil // 2
closure = nil // 3 foo deinit
{% endhighlight %}
1.  클로저에 foo의 emptyFunc를 할당합니다. 이때 foo의 reference count가 1 증가합니다.  
2.  foo 에 nil 을 할당에 reference count 를 1 줄입니다.  1에서 카운트가 1 증가 했기때문에 foo는 해제되지 않습니다.  
3.  클로저에 nil을 할당함과 동시에 foo가 메모리에서 해제됩니다.  
RxSwift의 onNext등에 메서드를 전달하면 위와 같은 이유로 레퍼런스 카운트가 증가하게 됩니다.  

### struct 에서의 기본 동작
reference count가 없는 struct에서는 값의 복사를 발생시킵니다.  
{% highlight swift %}
struct Foo {
  var state = false

  func emptyFunc() {
  }
  
  mutating func toggleState() {
    self.state = true
  }
}

var foo: Foo? = Foo()
var closure: (() -> Void)? = foo?.emptyFunc // 1
// var closure: (() -> Void)? = foo?.toggleState // 2
foo = nil // 3
{% endhighlight %}
1.  struct에서는 값의 복사가 일어나게 됩니다. 따라서 클로저가 갖고 있는 foo와 기존에 만든 foo는 다른 foo를 참조합니다.  
2.  mutating func는 컴파일 에러를 발생시킵니다.  
3.  값 이기 때문에 nil 을 넣은 순간 기존의 foo는 사라집니다.

1에서 foo가 복사되어 넘어가기 때문에 내부적으로 참조타입을 갖고있다면, 모든 참조타입에 대해 reference count가 1 증가합니다.  
  
값이 복사되는 것을 확인하기 위해
{% highlight swift %}
struct Foo {
  var state = false

  func emptyFunc() {
    print(state)
  }
}

var foo: Foo? = Foo()
var closure: (() -> Void)? = foo?.emptyFunc
foo?.state = true
foo?.emptyFunc() // true
closure?() // false
{% endhighlight %}
코드를 실행해보면, 각각 true, false가 출력됩니다.  
  
#### mutating func
mutating func는 클로저로 대신할 수 없기 때문에, mutating을 제거하는 작업이 필요합니다.  
기존 스테이트를 레퍼런스 타입으로 변경하면 mutating을 제거할 수 있습니다.  
{% highlight swift %}
struct Foo {
  private class State {
    var name = ""
  }
  private let state = State()
  
  func updateName(name: String) {
    self.state.name = name
  }
}

var foo: Foo = Foo()
var closure: ((String) -> Void) = foo.updateName
closure("hello")
{% endhighlight %}
struct내에서 자신의 상태가 아닌, 레퍼런스 타입의 값을 변경할때에는 mutating이 필요가 없기 때문에 위와 같은 형태를 적용할 수 있습니다.  
  
### 참조
[AutomaticReferenceCounting(ARC)](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html)  
[partial-apply of SIL](https://github.com/apple/swift/blob/master/docs/SIL.rst#partial-apply)
