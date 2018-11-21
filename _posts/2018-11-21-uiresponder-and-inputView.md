---
layout: post
title: UIResponder와 inputView, inputAccessoryView
thumbnail: "assets/img/posts/2018-11-21-uiresponder-and-inputView/2.png"
tags: [swift, iOS, UIResponder, inputView, inputAccessoryView]
---
이번 글에서는 `First Responder`와 `inputView`, `inputAccessoryView` 에 대해 알아보고,  
다음글 에서는 위 내용을 바탕으로 메시지 앱의 데모를 제작해봅니다.  
  
### Responder
First Responder 을 이해하기 위해서는 [UIResponder](https://developer.apple.com/documentation/uikit/uiresponder)를 먼저 살펴봐야 합니다.  
UIResponder에는 모션, 터치, 입력에 관련하여 인터페이스가 정의되어 있는데 앞으로 알아볼 `inputView`, `inputAccessoryView` 또한 이곳에 정의되어 있습니다.  
UIResponder는 위 나열한 모션, 터치, 입력 등 에 관련하여 이벤트를 처리합니다.  
그중 현재 상태에서 이벤트를 가장 먼저 받게 되는 Responder가 First Responder가 됩니다.  
예를 들면, UITextField는 UIResponder를 따르고 있습니다. 아이디와 비밀번호 2가지 TextField가 있고,  
아이디 텍스트 필드를 누른 후 키보드를 타이핑 하면 아이디의 TextField에 글자가 채워집니다.  
이때 아이디 TextField가 `First Responder`가 됩니다.  
![text]({{ site.baseurl }}/assets/img/posts/2018-11-21-uiresponder-and-inputView/1.png)
  
이벤트를 First Responder가 받았지만, 처리가 불가능 하면 Responder chain에 따라 next responder로 이벤트를 넘기게 됩니다.  
해당 내용에 대해서는 [Understanding cocoa and cocoa touch responder chain](https://medium.com/ios-os-x-development/understanding-cocoa-and-cocoa-touch-responder-chain-12fe558ebe97), [한글번역](https://medium.com/@audrl1010/cocoa-와-cocoa-touch-responder-chain에-이해하기-5121d8d707d2) 에 자세히 나와있습니다.   
  
First Responder는 Action을 전송할때에도 이용됩니다.   
버튼을 터치 했을때, TextField의 키보드를 내리는`resignFirstResponder`를 아래와 같이 버튼에 등록할 수 있습니다.  
`button.addTarget(textField1, action: #selector(UIResponder.resignFirstResponder), for: .touchUpInside)`  
위 코드의 버튼은 textField1에만 작동하지만,  
`button.addTarget(nil, action: #selector(UIResponder.resignFirstResponder), for: .touchUpInside)`  
만약 target이 위와 같이 nil이라면, First Responder에게 액션이 먼저 전달됩니다. 따라서 현재 first Responder인 TextField, responder을 저장하지 않고 있어도 resign 하는게 가능합니다.  
`NSObjectProtocol`의 `perform`혹은 비슷한 런타임 메시징 방식에서도 동일하게 작동합니다.  


### inputView,  inputAccessoryView 
해당 뷰들은 Responder object가 first Responder가 되었을때 나오는 뷰입니다.  
inputView의 대표적인것은 키보드입니다. UITextField, UITextView 가 First Responder가 되었을때 시스템 키보드가 디폴트로 나오게 됩니다.  
아마 UITextField의 inputView에 PickerView를 넣어 사용해본적이 있을 수 있습니다. 해당 경우에는 디폴트인 키보드를 사용하는것이 아니라 inputView로 PickerView를 사용하는것을 의미 합니다.  
UIResponder에서 intputView는 get-only property지만, UITextView, UITextField에서는 get, set property로 재정의 되어 있습니다.  
inputAccessoryView는 inputView(일반적으로 키보드)위에 뜨는 보조적인 뷰 입니다.  
UITextField가 많을때 prev, next를 두거나 보조적인 버튼을 추가하여 사용합니다.  

![input]({{ site.baseurl }}/assets/img/posts/2018-11-21-uiresponder-and-inputView/2.png)  
위 이미지는 아이디 텍스트 필드의 inputView를 PickerView로 변경했을때 모습입니다.  
inputAccessoryView로는 빨간색 뷰를 추가했습니다.  

다음글에서는, TableView를 First Responder로 사용하여, iOS의 메시지앱과 비슷한 레이아웃을 만들어볼 예정입니다.  
