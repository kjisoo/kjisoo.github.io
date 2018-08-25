---
layout: post
title: iOS ViewController Transition 적용과 이해
tags: [iOS, Swift, transition, animator]
---
iOS에서는 UIViewController(VC)에서 다른 VC로 넘어가는 효과를 커스텀할 수 있는 프로토콜 및 클래스를 제공하고 있습니다.  
이를 이용해 기본으로 제공되는 UINavigation push, pop 효과나, modal style과 별도로 새로운 애니메이션을 비교적 쉽게 적용할 수 있습니다.  
   
Transition은 Non-Interactive와 Interactive로 나뉘게 됩니다.  
Non-Interactive는 present와 같이 새로운 VC가 나오는 애니메이션을 중간에 멈추거나, 되돌릴 수 없습니다.  
반면에 Interactive는 UINavigationController에서 왼쪽을 쓸어내면, 사용자의 제스쳐에 따라서 애니메이션을 진행시키거나 취소할 수 있습니다.  


#### 프로토콜 
VC는 `UIViewControllerTransitioningDelegate` 프로토콜의 `transitioningDelegate` 속성이 있습니다.  
A VC에서 B VC로 이동(present)한다면, B VC의 `transitioningDelegate` 가 있어야 합니다.  
B VC에서 A BV로 다시 돌아갈때도(dismiss) 마찬가지로 B VC의 `transitioningDelegate`가 필요 합니다.  
`transitioningDelegate`은 트랜지션을 담당하는 오브젝트를 리턴하는데,  
Non-Interactive를 정의하는 `UIViewControllerAnimatedTransitioning` 프로토콜과,  
Interactive를 정의하는 `UIViewControllerInteractiveTransitioning` 프로토콜이 있습니다.  
`UIViewControllerInteractiveTransitioning`의 경우에는 `UIPercentDrivenInteractiveTransition`의 구체클래스도 추가로 제공하고 있습니다.  


#### 동작 방식
1.  StartVC에서 NextVC를 present 합니다.
1.  Application은 NextVC의 `transitionDelegate`의 값이 nil인지 확인합니다. nil이라면, 기본 효과
1.  `transitionDelegate`에게 트랜지션 오브젝트를 요청합니다. nill이라면, 기본 효과
1.  Application은 트랜지션 오브젝트에게 `UIViewControllerContextTransitioning` 프로토콜을 따르는 오브젝트를 제공 합니다. 
1.  트랜지션 오브젝트는 nextVC의 view를 containerView에 추가하고, 효과 애니메이션을 진행합니다.
1.  Interactive라면, 효과가 진행되는 동안 현재 상태를 업데이트 합니다.
1.  트랜지션 오브젝트는 효과가 끝난 뒤 효과가 끝난것을 알립니다.  


#### 예제
결과 gif 추가, 


실제 코드를 바탕으로 서술, 
duration 에서 coordinate 언급


#### 마무리
collectionview layout animation 존재 언급


#### 참고
[Transition Topics - Apple document](https://developer.apple.com/documentation/uikit/animation_and_haptics/view_controller_transitions)  
[Customizing the Transision Animations - Apple document](https://developer.apple.com/library/archive/featuredarticles/ViewControllerPGforiPhoneOS/CustomizingtheTransitionAnimations.html#//apple_ref/doc/uid/TP40007457-CH16-SW1)  
[Custom Transition Using View Controllers - WWDC2013-218](https://developer.apple.com/videos/play/wwdc2013/218/)  
[Advances in UIKit Animations and Transitions - WWDC2016-216](https://developer.apple.com/videos/play/wwdc2016/216/)  
