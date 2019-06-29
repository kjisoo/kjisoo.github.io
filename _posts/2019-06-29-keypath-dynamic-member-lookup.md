---
layout: post
title: SE-0252, Swift5.1에 추가된 keypath dynamic member lookup
tags: [ios, swfit, se-0252, mvvm]
---
이번 WWDC에서는 SwiftUI, Combine등 굵직한 기술들이 많이 소개가 되었습니다.  
Language level이 아니라 SDK level에서의 기술이고, min SDK가 13이기 때문에 실무에선 바로 사용하기에는 시간이 걸릴것같습니다.  
위 기술들을 지탱해주기 위해 swift5.1 에서도 여러 기능들이 추가되었습니다.  
#### 기존의 dynamic member lookup
swift4.2에서 소개된 기존의 dynamic member lookup은 제한적이었습니다. [SE-0195](https://github.com/apple/swift-evolution/blob/master/proposals/0195-dynamic-member-lookup.md) 
{% highlight swift %}
@dynamicMemberLookup
struct Person {
  subscript(dynamicMember member: String) -> String {
    switch member {
      case "age":
        return "20"
      case "name":
        return "hello"
      default:
        return ""
    }
  }
}
{% endhighlight %} 
리턴 타입이 한가지로 제한되는 부분, 어떤 멤버변수가 존재하는지 알 수 없는 부분들이 있습니다.  
동적언어와의 호환성에는 무리가 없을지 몰라도 실제 사용성에 있어서는 그렇게 도움이 되진 않았습니다.
  
#### Swift5.1 keypath dynamic member lookup
swift5.1에서 진보된 keypath dynamic member lookup이 추가되었습니다. [SE-0252](https://github.com/apple/swift-evolution/blob/master/proposals/0252-keypath-dynamic-member-lookup.md)  
{% highlight swift %}
@dynamicMemberLookup
struct Proxy<T> {
  private var value: T
  
  init(value: T) {
    self.value = value
  }
 
  subscript<U>(dynamicMember keyPath: WritableKeyPath<T, U>) -> U {
    return value[keyPath: keyPath]
  }
}
 
struct User {
    var name: String = "hello"
    var age: Int = 20
    let someType: SomeType = .foo
}
 
let proxyModel = Proxy(value: User())
print(proxyModel.name)
print(proxyModel.age)
{% endhighlight %}
KeyPath를 이용해 동적으로 접근을 할 수 있게되었습니다.
기존에 있던 리턴타입이 제한되는 부분과 멤버변수문제도 해결되었습니다.  
proxyModel. 에서 T Type 인 User 의 name, age, someType 이 목록으로 나옵니다.  

#### 어디에 쓸 수 있나
BindableObject나 다른 클래스들이 기존 C#의 MVVM에서 사용되던 이름과 동일하고, SwiftUI등을 보면 
Apple에서는 MVVM 형태의 아키텍처를 공식적으로 지지하는듯 보입니다.  
프리젠테이션 레이어를 두고 설계할때 문제가 되는 부분이, 데이터만 있는 모델을 위한 프리젠테이션 모델을 만들어야 하나? 입니다.  
예를 들면,  
{% highlight swift %}
struct User {
  let name: String
  let age: Int
}
 
struct UserViewModel {
  private let user: User

  let name: String
  let age: Int
}
{% endhighlight %}
위와 같이 아무 역할도 하지 않는 ViewModel이 생길 수 있습니다.  
위와 같이 할때의 장점은, View Layer과 Business layer를 완벽하게 분리할 수 있습니다.  
하지만 iOS에서는 여러 타겟을 두는 형태로 개발하지 않고 따라서 View와 Model이 같은 타겟에 속하고 있어 편의상 Model이 immutable이면 별도의 ViewModel을 만들지 않고 View에서 Model을 참조하기도 합니다.  
위 문제를 Keypath dynamic member lookup 을 이용하면, 쉽게 해결이 가능합니다.  
{% highlight swift %}
@dynamicMemberLookup
struct ProxyViewModel<T> {
  private var value: T
  
  init(value: T) {
    self.value = value
  }
    
  subscript<U>(dynamicMember keyPath: WritableKeyPath<T, U>) -> U {
    return value[keyPath: keyPath]
  }
}
 
struct UserListViewModel {
  var userViewModels: [ProxyModel<User>] = []

  func update(user: [User]) {
    userViewModels = user.map { ProxyModel(value: $0) }
  }
}
{% endhighlight %}
위와 같이 별도의 프리젠테이션 로직이 없는 모델을을 ViewModel로 랩핑하여 전달할 수 있습니다.  
이를 통해 View layer, business layer의 물리적 분리도 쉽게 가능하게 됩니다.  

#### 마무리
SE-0252외에도, [0258-property-wrappers](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md), [0244-opaque-result-types](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md) 도 추가되었는데, iOS13에 추가된 기능들이 위 기능들을 적절히 이용하여 구현된것으로 보입니다.  

