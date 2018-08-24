---
layout: post
title: iOS ViewController Transition 적용과 이해
tags: [iOS, Swift, transition, animator]
---
iOS에서는 UIViewController(VC)에서 다른 VC로 넘어가는 효과를 커스텀할 수 있는 프로토콜 및 클래스를 제공하고 있습니다.  
이를 이용해 기본으로 제공되는 UINavigation push, pop 효과나, modal style과 별도로 새로운 애니메이션을 비교적 쉽게 적용할 수 있습니다.  
   
Transition은 Non-Interactive와 Interactive로 나뉘게 됩니다.  
Non-Interactive는 present와 같이 새로운 VC가 나오는 애니메이션은 중간에 멈추거나, 되돌릴 수 없습니다.  
반면에 Interactive는 UINavigationController에서 왼쪽을 쓸어내면, 사용자의 제스쳐에 따라서 애니메이션이 달라집니다.  

#### 프로토콜 
A VC에서 B VC로 이동한다면, A VC는 `UIViewControllerTransitioningDelegate`프로토콜 따라야 합니다.  
Non-Interactive를 정의하는 `UIViewControllerAnimatedTransitioning` 프로토콜과,  
Interactive를 정의하는 `UIViewControllerInteractiveTransitioning` 프로토콜이 있습니다.  
`UIViewControllerInteractiveTransitioning`의 경우에는 `UIPercentDrivenInteractiveTransition`의 구체클래스도 추가로 제공하고 있습니다.  
구체 클래스에는 애니메이션의 업데이트, 캔슬등 편의 기능을 제공하고 있습니다.  

### 동작 방식?
가나다라마바사


### 예제?
foo bar baz

### Coordinate 추가할지 말지
### collectionview layout animation 존재 언급

### 참고
[Transition Topics - Apple document](https://developer.apple.com/documentation/uikit/animation_and_haptics/view_controller_transitions)  
[Customizing the Transision Animations - Apple document](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/CustomizingtheTransitionAnimations.html#//apple_ref/doc/uid/TP40007457-CH16-SW1)  
[Custom Transition Using View Controllers - WWDC2013-218](https://developer.apple.com/videos/play/wwdc2013/218/)  
[Advances in UIKit Animations and Transitions - WWDC2016-216](https://developer.apple.com/videos/play/wwdc2016/216/)  
