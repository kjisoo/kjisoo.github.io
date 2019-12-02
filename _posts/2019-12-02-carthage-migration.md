---
layout: post
title: Cocoapod 프로젝트를 Carthage로 변환할때 하는 실수
tags: [ios, carthage, cocoapods, objectivec]
---
기존 프로젝트를 Carthage로 변환하는 이유는 많겠지만, 가장 큰 이점은 Prebuild로 인한 빌드타임 감소 입니다.  
대부분의 큰 프로젝트는 바로 변환이 가능하며, Realm과 같은 라이브러리는 바이너리도 제공해주고 있기 때문에 빌드 없이 사용도 가능합니다.  
(구 빌드 버전에 대해서도 swift5.1로 컴파일 된 바이너리를 제공해주고 있다.)  
문제는 기존 Pod로 사용하던 레거시 프로젝트를 Carthage로 변환하는것 입니다.  
변환하는 과정에서 만나기 쉬운 상황들을 정리해봤습니다.  
### 소스만 있는 프로젝트일 경우
Pod를 이용할 경우 podspec에 소스 파일의 리스트만 제공하면 되기때문에 별도의 xcodeproj가 없어도 됩니다.  
Carthage의 경우 프로젝트를 모두 생성해주어야 합니다.  
Xcode의 Framework로 프로젝트를 생성하면 됩니다.  
  
### Shared scheme
Manage schemes의 Shared를 체크해주지 않으면 `Dependency "XYZ" has no shared framework schemes` 에러가 나옵니다.  
프로젝트를 생성하고 shared에 체크가 되어있음에도 `/xcshareddata`가 생성되어있지 않습니다.  
이때는 shared에 체크를 해제하고, 다시 체크하면 해당 폴더가 생깁니다.  
혹은 `/xcshareddata`가 이미 생성되어 있는데, gitignore에 `/xcshareddata`가 등록되어 있을 수 있습니다.    
해당 폴더를 git에 추가해주면 해결됩니다.  
  
### Public header
Swift를 프레임워크로 만들때 접근제한자로 모두 해결되지만, Objective-c 파일을 변환할때 헤더 파일을 신경써야 합니다.  
헤더 파일이 1개일 경우 타겟 이름과 헤더파일의 이름을 동일하게 하면됩니다.  
헤더 파일이 2개 이상일 경우에는 타겟 이름과 동일안 헤더 파일에서 다른 헤더 파일을 임포트 해주어야 합니다.  
임포트 되는 헤더파일들은 target -> Build Phases -> Headers 에 Public에 위치해야 합니다.  
Xcode에서 프레임워크로 프로젝트를 생성하면 타겟 이름과 동일한 헤더파일이 자동으로 생성됩니다.  
{% highlight swift %}
#import <Foundation/Foundation.h>

//! Project version number for asdasdasdas.
FOUNDATION_EXPORT double testProjectVersionNumber;

//! Project version string for asdasdasdas.
FOUNDATION_EXPORT const unsigned char testProjectVersionString[];

// In this header, you should import all the public headers of your framework using statements like #import <Target/PublicHeader.h>
// 해당 위치에 프레임워크에서 퍼블릭으로 공개하고 싶은 헤더들을 임포트 합니다.  
#import <Target/foo.h>
#import <Target/bar.h>
{% endhighlight %}
  
### 사소한 것
프레임워크의 Deployment target이 프로젝트의 타겟보다 높을때 런타임 에러가 발생합니다.  

