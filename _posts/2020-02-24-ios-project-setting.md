---
layout: post
title: 최근 적용한 iOS 프로젝트 구성
tags: [ios, carthage, cocoapods, propertywrapper, rxswift]
---
기존 프로젝트의 개발 환경과 리팩토링 경험을 공유합니다. 
### 빌드속도(Carthage, Rome)
모든 라이브러리들을 cocoapods를 통해 관리하고 있고 realm, couchbase와 같이 무거운 라이브러리들도 포함되어 있었습니다.  
클린 빌드시 약 30분 정도의 시간이 필요했고, 이후 다시 말하겠지만 프리뷰와 CI를 위해서도 빌드 시간을 줄이고자 했습니다.  
  
이를 위해 carthage를 도입 했는데, iOS의 Min Version을 11로 올린 상황이라 arm64만 지원해도 되었고, 사용하고 있는 대부분의 라이브러리들이 carthage를 지원하고 있었습니다.  
몇몇 라이브러리들은 직접 지원하도록 수정이 필요했는데, 그동안 업데이트 없이 사용했고 앞으로는 제거할 예정이었기 때문에 포크하여 사용했습니다.  
carthage로 pre-build이후 클린 빌드 시간은 4분 가량으로 줄어들었지만, pre-build단계가 30분 이상을 차지 했습니다.  
팀이 크지 않은 상황에서 큰 문제는 아니었지만, CI가 너무 오래걸렸고 master branch까지 머지되기 전 과도기 단계에서 불편을 줄 수 있는 부분을 인지하고 Rome을 도입했습니다.  
Carthage로 생성된 Pre-build frameworks를 local, git에 저장하도록 하고 CI와 그 외 개발자는 git에서 캐시를 받아 사용할 수 있게 수정했습니다. 
git에선 swiftToolchainVersion에 따라서 branch를 나눠 사용했습니다.  
S3나 storage를 사용하지 않은 이유는 보안이 중요한 상황에 앱과 직접적으로 관련없는 서비스의 권한을 받는게 번거롭고 팀 인원대비 효율이 적을것이라 판단 했습니다.  
  
Buck와 같은 방법도 있었지만, 팀이 적은 상황에서 적용으로 얻을 수 있는 득이 크지 않다고 판단했습니다.  
Airbnb에선 클린빌드가 50분 걸리던것을 좋은 사양의 디바이스로 교체하여 30분으로 줄이고, Buck를 도입하여 5분 이하로 줄였다고 합니다.  
[2.9 - Francisco Diaz - Working effectively at scale](https://www.youtube.com/watch?v=KhZcSRXJHFs) 해당 영상에서 확인할 수 있습니다.  
[Journey to Buck build system at Booking.com - Alexey Gaponov & Oleksandr Kolodii](https://vimeo.com/362205579) 해당 영상에선 Buck에 관련하여 더 많은 정보를 얻을 수 있습니다.  
  
### View
팀원이 증가해 같은 화면을 동시에 수정하는 빈도수가 증가하고 컴파일 속도상의 이점을 확보하고자 xib에서 SanpKit을 이용해 코드베이스로 변경했습니다.  
앞으로 정착하게 될 SwiftUI와 비슷한 모습이 될 수 있는점도 선택에 참고가 되었습니다.  
주로 언급되는 단점으로 실행전에 해당 화면을 볼 수 없다는 것이 있는데, 이를 SwiftUI의 Preview기능을 통해 해결했습니다.  
혹시 비슷하게 이용하실 분은 SwiftUI를 optional로 추가해주어야 합니다. 그렇지 않을경우 iOS13미만에서 런타임 크래시가 발생합니다.  
뷰에 존재하는 로직은 모두 뷰 외부로 이동시켰습니다.  
  
### ViewModel(MVVM)
Presentation Model는 ViewModel를 선택했습니다.  
View와 마찬가지로 SwiftUI와 비슷하게 할 수 있는점과 PropertyWrapper를 이용하면 Rx의 단점을 어느정도 보안할 수 있을거란 기대가 있었습니다.  
Rx로 ViewModel를 구성했을때 가장 큰 문제점이 데이터의 순환 참조와 속성들의 바인딩 방향이 명확하지 않다는 점 입니다.  
[kickstart/ios-oss](https://github.com/kickstarter/ios-oss/blob/master/Kickstarter-iOS/ViewModels/UpdateViewModel.swift) 의 형식으로 구현하면 Input, Output이 구별되지만 장황해지고 모든 데이터 스트림이 Reactive이기 때문에 개발측면에서 단점이 존재한다고 생각했습니다.  
[PropertyWrapper, RxSwift를 이용한 MVVM Binding 구현]({{ site.baseurl }}/2019/08/03/binding-with-property-wrapper.html) 이를 PropertyWrapper를 이용해 구현했습니다.  

### 마무리
기술을 도입하기전 지금 팀의 도움이 될것인지, 지금 팀 규모에서 해당 방법이 좋은 방법인지, 지금 상태에서 도입이 현실적으로 가능한지 세가지를 가장 중점적으로 살펴봤던거 같습니다.  
예를 들어 팀 규모가 더 컸다면 Ribs, VIPER와 같은 풀 아키텍쳐를 도입하거나 Module형식의 프로젝트를 구성해 DI에 더 많은 신경을 썼을거라 생각합니다.  

