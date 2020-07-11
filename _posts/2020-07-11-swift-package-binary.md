---
layout: post
title: Swift Package Manager(SPM)에 새롭게 추가된 바이너리 배포
tags: [ios, carthage, cocoapods, spm, swiftpackagemanager, se-0272]
---
Swift5.3에서 Swift Package Manager(SPM)의 Binary 배포가 추가되었습니다.  
배포를 위해서는 바이너리를 Xcode11에서 소개된 xcframework로 만들어야 합니다.  
만들어진 xcframework 를 사용하는 방법은 2가지가 있습니다.  
원격 소스에 있는 xcframework의 url, checksum이 필요합니다.  
{% highlight swift %}
static func binaryTarget(name: String, url: String, checksum: String) -> Target
{% endhighlight %}
로컬에 있는 xcframework의 경우 path만 필요합니다.  
{% highlight swift %}
static func binaryTarget(name: String, path: String) -> Target
{% endhighlight %}
  
### Pod, Carthage와는 무엇이 다를까  
특별하게 다른 부분은 없다고 봅니다.  
포럼을 참고해도 다른 디펜던시 툴에서 이미 지원하고 있는 기능을 SPM을 통해 이용할 수 있다는 장점만 있어보입니다.  
SPM의 해당 기능이 carthage의 use-binaries와 비슷하다고 느낄 수 있는데, 포럼의 논의를 보면 해당 옵션과는 별개이며 binary "url" 과 같은 기능으로 보는것이 맞아보입니다.  
checksum이 필요해 carthage처럼 간편하게 버전을 변경하기도 힘들어보입니다.  
  
### 언제, 누가 사용해야 할까  
SPM을 이미 사용하고 있고 로컬에서 frameworks 사용하고 있거나,  
외부 바이너리파일(주로 gogole, firebase, ..) 를 위해 3rd party 의존성 툴을 이용하고 있다면 SPM으로 통합할 수 있을거 같습니다.  
해당 기능을 빌드속도 향상의 목적을 갖고 프로젝트 코드 일부를 binary로 받아오거나 외부 라이브러리를 이용한다면 부적절해 보입니다.  
prebuilt를 통한 빌드 속도 향상은 Carthage + Rome 조합을 이용하는게 합당해보입니다.  
  
### 참조
[SE-0272](https://github.com/apple/swift-evolution/blob/master/proposals/0272-swiftpm-binary-dependencies.md)  
[WWDC20 - Distribute binary frameworks as Swift packages](https://developer.apple.com/wwdc20/10147)  
[WWDC19 - Binary Frameworks in Swift](https://developer.apple.com/wwdc19/416)  
[Apple Document - Distributing binary frameworks as swift packages](https://developer.apple.com/documentation/swift_packages/distributing_binary_frameworks_as_swift_packages)

