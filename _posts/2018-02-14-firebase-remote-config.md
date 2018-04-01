---
layout: post
title: Firebase로 A/B 테스트 하기 (remote config)
tags: [Swift, iOS, AB test, firebase, remote config]
redirect_from:
  - /blog/firebase로-ab-테스트-하기
  - /blog/firebase로-ab-테스트-하기/
---
서비스를 운영/개발 하는 사람에게 a/b 테스트는 강력한 기능입니다.  
실험적 기능에 대해 실제 사용자에 대해 통계데이터를 확보할 수 있습니다.  
a/b 테스트중 특정 케이스가 지지되었을때 업데이트 없이 모든 사용자를 이동시킬 수 있는 점도 장점입니다.  
네이티브 레벨에서 테스트를 진행하기 위해서 파이어베이스의 원격 설정기능을 이용했습니다.  
다른 툴로는 린플럼, 옵티마이즈 등 여러 툴들이 있습니다.  
파이어베이스를 선택한 이유는, 많은 기능이 필요하지 않아서, 이미 붙어있기 때문에 모듈만 추가해주면 되는점, 무료인 부분이 있습니다.  
통계부분과 추가적인 기능, 웹과의 연동, 실시간성이 필요하다면 다른 툴을 찾아보아야합니다.  
파이어베이스를 통해 가능한 부분들은,
  - 이벤트를 발생시키고, 대조군과의 이벤트 발생 빈도를 비교
  - 사용자에게 값을 할당하고, 특정 값인 사용자만 이벤트를 발생
  - 배포 사용자를 점진적을 증가
  - 제가 알지 못하는 더 많은 기능들

### 설정
2가지 방법으로 진행할 수 있습니다.  
1. 일부 사용자에게 y값을 주고, 나머지 모든 사용자에게 x값을 준다.  
2. 전체중 일부 사용자에게만 x, y, z 특정 값을 준다. (일부에 포함되지 못한 사용자는 값 자체가 없음)  

#### 1번 케이스
![1]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/1.png)  
remote config 탭에 들어가 새로운 파라미터를 만듭니다.  

![2]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/2.png)  
테스트용으로 파라미터 이름은 first_param, 값으로는 test 를 할당했습니다.  

![3]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/3.png)  
이후 변경한 부분을 발행하게 되면 클라이언트에서는 해당 값을 즉시 접근할 수 있습니다.

![4]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/4.png)  
만들어진 파라미터에 새로운 조건들을 추가할 수 있습니다.

![5]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/5.png)  
이름은 해당 조건의 이름입니다. 재사용할 수 있기 때문에 범용적인 조건을 미리 만들어놓을 수 있습니다.  
컬러는 단순히 구분을 위한 컬러입니다.  
조건에는 여러 옵션이 있지만, 랜덤 퍼센트를 사용합니다. 위 케이스에서는 한가지 조건이지만, AND 로 여러 조건을 할당할 수 있습니다.  
조금 혼란스러웠던 부분을 적어보면,  
X > 50 AND X <= 60 의 조건일 경우 X는 50보다 크고, 60보다 같거나 작은 유저입니다.  
전체의 50-60% 라 생각했지만 위 조건은 전체의 10%만 해당하는 조건입니다.  
이유는, 파이어베이스를 초기화 하게되면 사용자는 랜덤한 숫자를 1가지 배정받습니다. 값은 1-100 사이입니다.  
위 조건은 해당 값의 범위를 지정합니다.  
따라서 모든 사용자가 10보다 작은 랜덤한 값을 배정받고, 조건이 X <= 10 이라면, 모든 사용자가 해당 조건에 충족하게 됩니다.  

위 내용은 구글 서포트를 통해 얻은 답변입니다.
```
you are asking if having a user in random percentile condition  
between > 50% and <=60% targets 10% of your users is that correct?  
If so, then yes that is correct.  
Your users are randomly assigned a percentile value  
so you may target any random grouping.  
If you wanted to target 50% of your users, you would only need <=50%.
```

![6]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/6.png)
이제 위 조건으로 새로운 값을 지정할 수 있습니다.

이제 30%의 사용자는 condition test 가 나오고, 나머지 사용자는 test가 나오게 됩니다.  
##### 장점
 - 간편하다.
 - 조건을 새로 할당해 배포 비율을 줄일 수 있다. 30% 에서 20%, .. 

##### 단점
 - 동일한 퍼센트로 진행하면 항상 같은 사용자에게 할당됩니다. 0-10% 유저는 항상 0-10%
 - 비교군의 이벤트 결과를 보지 못한다. 별도로 통계를 쌓아 비교해야합니다.

#### 2번 케이스
![ab1]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/ab1.png)  
remote config의 오른쪽에는 a/b test 버튼이 있습니다. 새로운 실험을 만들 수 있습니다.  

![ab2]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/ab2.png)  
새로운 실험의 초기 타겟 유저를 설정합니다.  
해당 값은 늘릴수는 있지만 줄일수는 없습니다.  
![ab3]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/ab3.png)  
control group, Variant A, Variant B는 그룹입니다. 그룹은 최대 8개까지 늘릴 수 있습니다.  
second_param, third_param 은 해당 그룹에 해당하는 파라미터들입니다. 파라미터 역시 8개까지 늘릴 수 있습니다.  
![ab4]({{ site.baseurl }}/assets/img/posts/2018-02-14-firebase-remote-config/ab4.png)  
설정이 완료되면, 실험 시작을 할수 있습니다.  
위 조건으로 진행하면, 전체 유저의 5%만 해당 테스트를 진행하게 되며,  
5%중 33.3%(전체의 1.666%)는 Control group, 33.3%는 Variant A, 나머지 33.3%는 Variant B에 할당됩니다.  
해당 그룹에 속하게 된 이후에는 그룹이 변경되지 않습니다.  

Control group에 속한 유저는 second_param 값이 1, third_param 값이 4가 나옵니다.  
위 케이스에서 파라미터의 값은 1,4 혹은 2,5 혹은 3,6 입니다. 다른 교차된 값은 나오지 않습니다.  
A/B 테스트가 완료되면, 특정 값을 파라미터로 옮겨올 수 있습니다. (특정 값으로 고정)  
실험을 시작한 후 5%를 원하는 수치로 증가시킬 수 있습니다. 증가 시킨 후에는 감소시킬 수 없습니다.  
만약, 꼭 감소를 시키고 싶다면 실험을 제거하고 같은 파라미터 이름으로 다시 생성하면 됩니다.  
하지만 이렇게 했을 경우 랜덤 % 가 재 할당 되기 때문에 기존에 테스트 그룹 유저가 바뀌게 됩니다.  

#### 코드
설정을 마치면 앱에서 해당 값들을 접근할 수 있습니다.
{% highlight swift %}
FirebaseApp.configure() // 1
let remoteConfig = RemoteConfig.remoteConfig() // 2
let remoteConfigSettings = RemoteConfigSettings(developerModeEnabled: true) // 3
remoteConfig.configSettings = remoteConfigSettings! // 4
remoteConfig.fetch(withExpirationDuration: 60 * 60 * 4) { (status, error) in // 5
  if status == .success {
    print(remoteConfig["first_param"]) // 6
    remoteConfig.activateFetched() // 7
    print(remoteConfig["second_param"]) // 8
  }
}
{% endhighlight %}
1. 파이어 베이스 설정
2. 원격 설정에 인스턴스를 얻어옵니다. 싱글톤객체 입니다.
3. 디버그 모드입니다. expiration duration을 짧게 하고 fetch 를 자주하게 되면, 서버에서 쓰로틀링을 거는데, 디버그모드를 활성화 할경우 쓰로틀링을 피해갈 수 있습니다. (디버그 모드에서만 사용하세요.)
4. 3과 동일합니다. 
5. fetch해옵니다. fetch를 할때 유효시간을 지정하거나, 지정하지 않을 수 있습니다. 지정하지 않을경우 기본값은 12시간 입니다. 
6. fetch는 성공했지만, 값은 아직 읽을 수 없는 상태입니다. 출력하면 빈 값이 출력됩니다. (최초 요청임을 가정) 
7. fetch한 값을 적용합니다.
8. 이제 원격으로 얻어온 값이 출력됩니다.

ExpirationDuration는 캐시 유지 시간으로 보면 적절합니다. 만약 1시간으로 설정 후 , 값을 받아오면  
원격 설정에서 값이 변경되거나 심지어 제거되어도 1시간동안은 캐시에 남아있는 값을 가저오게 됩니다.  
하지만 시간을 너무 짧게 설정할경우 서버에서 쓰로틀링을 걸어 값을 얻어오지 못합니다.  
적절한 시간은 3시간 이상입니다.  아래는 구글 서포트를 통해 얻은 답변. 
```
We can't share any numbers for the limits on fetchs for Remote Config since the limits can change in the future.
As a general rule, deployed apps with a cache timeout less than 3 hours have a strong chance of running into our server-side throttling. 
If you think your app will need to pull more requests than that, Remote Config may not be the best choice for your use case and the Realtime Database may be a better option.
```

참조: [firebase 원격구성](https://firebase.google.com/docs/remote-config/)
