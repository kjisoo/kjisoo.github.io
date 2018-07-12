---
layout: post
title: struct가 복사되는 여러 시나리오
tags: [swift, struct, iOS]
---
# WIP
struct와 class는 value type과 reference type으로 기본 동작에 큰 차이를 보입니다. 
문제가 없는 기본 동작에서 부터 문제가 생기는 상황까지 살펴보려고 합니다. 
### 기본
#### 값의 복사
struct와 class는 값을 할당시 값 복사와 레퍼런스 참조로 나뉘게 됩니다.
{% highlight swift %}
struct StructType {
  var id: String
}
 
class ClassType {
  var id: String
  init(id: String) {
    self.id = id
  }
}
 
var originalStruct = StructType(id: "original") // 생성
var copiedStruct = originalStruct
copiedStruct.id = "copied"
print(originalStruct.id, copiedStruct.id) // original copied
 
var originalClass = ClassType(id: "original")
var copiedClass = originalClass
copiedClass.id = "copied"
print(originalClass.id, copiedClass.id) // copied copied
{% endhighlight %}
위 예제에서 struct는 새로 값이 복사되어 id를 바꾸었을때 각각의 id를 출력합니다.  
하지만 class는 참조 포인터를 복사하기 때문에 한 변수에서 값을 바뀌면 다른 변수까지 영향이 생기게 됩니다.  
![1](https://www.uml-diagrams.org/examples/state-machine-example-water.png)
  
#### 멤버변수에 참조 타입이 있을때
멤버변수에 참조타입이 있을때 참조타입은 참조 포인터만 복사하게 됩니다.  
{% highlight swift %}
class ClassType {
  var id: String = "default"
  deinit {
    print("class type deinit!")
  }
}
 
struct StructType {
  var id: String = "default"
  let classType = ClassType()
}
 
var original: StructType! = StructType() // classType의 참조 카운트 1
var copied: StructType! = original // 참조 카운트 2
original.id = "touch"
original.classType.id = "touch"
print(original.id, original.classType.id)
print(copied.id, copied.classType.id)
original = nil // 참조 카운트 1
copied = nil // 참조 카운트 0, 출력 class type deinit!
{% endhighlight %}
struct의 값(id)은 복사 되지만, classType은 참조포인터가 복사되어 한쪽에서 수정하면 모두 영향이 생기게 됩니다.  
값이 복사되면서 참조 카운트도 증가하게 됩니다.  
![1](https://www.uml-diagrams.org/examples/state-machine-example-water.png)

 
### 고급
#### 함수 포인터를 저장
struct의 함수 포인터를 저장할 수 있습니다.  
{% highlight swift %}
struct StructType {
  var id: String = "default"
  
  func foo() {
    print(id)
  }
}
 
var original = StructType()
var foo = original.foo
original.id = "original"
original.foo() // original
foo() // default
{% endhighlight %}
original.foo 를 저장하는 부분에서 값의 복사가 일어납니다.  
original의 복사본이 생기게 되고, StructType의 foo와 바인딩된 함수 포인터를 받게 됩니다.  
따라서 original의 id를 바꾸어도 foo와는 독립되어 수정됩니다.  
복사본도 참조 포인터를 갖게 되므로 참조 타입의 모든 참조 카운트가 1씩 증가하게 됩니다.  
context를 공유해야 하는 상황이라면, 서로 다른 context를 갖게 되어 문제가 발생합니다.  
공유 되어야 하는 context를 class로 감싸서 해당 부분을 해결할 수 있습니다.  
![1](https://www.uml-diagrams.org/examples/state-machine-example-water.png)
  
  
#### 함수 포인터를 참조 타입 맴버 변수에게 전달
{% highlight swift %}
class ClassType {
  var f: (() -> ())?
  deinit {
    print("This is never called.")
  }
}
struct StructType {
  let classType = ClassType()
  init() {
    classType.f = foo // 문제 지점
  }
  func foo() {}
}
var structType: StructType? = StructType()
structType = nil
{% endhighlight %}
위 코드에서는 ClassType은 deinit가 되지 않습니다.  
문제 지점에서 함수를 전달할 때 self가 복사되어 foo와 바인딩된 함수 포인터가 전달됩니다.  
self가 복사되는 순간 classType의 참조 카운트가 1이 증가 하여 순환참조가 생기게 됩니다.  
structType이 nil이 되면서 참조카운트가 1이 감소되지만, classType의 f가 복사된 struct의 foo를 소유하고 있어 참조카운트가 1이 되어 릴리즈되지 못합니다.  
![1](https://www.uml-diagrams.org/examples/state-machine-example-water.png) 


#### lazy loading + 함수 포인터 사용
lazy loading + 함수포인터 저장할때 생기는 문제 


### 참조
[AutomaticReferenceCounting(ARC)](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html)
[partial-apply of SIL](https://github.com/apple/swift/blob/master/docs/SIL.rst#partial-apply)

