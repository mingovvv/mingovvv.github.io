---
title: 😂 WSL2 + Docker에서 발생하는 메모리 이슈...!?
author: mingo
date: 2023-10-31 09:30:00 +0900
categories: [docker, wsl2]
tags: [docker, wsl2, vmmem]
---

-----

### vmmem 메모리 이슈
WSL2 기반으로 docker를 사용하다보면 노트북 팬 돌아가는 소리가 진짜 무섭게 날 때가 종종 있다.
작업관리자를 통해 프로세스를 확인해보면 `vmmem` 이라는 프로세스가 꽤나 무섭게 돌아가고 있는 것을 볼 수 있다.
![Desktop View](/assets/img/post/202310/2.png){: width="800" height="300" }
_vmmem 프로세스_

`vmmem` 프로세스는 운영 체제가 가상 머신에 의해 사용되는 메모리와 CPU 리소스를 나타내기 위해 합성하는 가성 프로세스이고 실제 사용하고 있는
WSL2와 docker의 사용으로 메모리가 점유된다.

그.런.데... 내가 직접 컨테이너로 띄워서 사용중인 메모리에 대해서는 할 말 없지만, 컨테이너를 종료해도 메모리가 돌아오지 않는 이슈가 있다.

### 원인은?
리눅스의 기본적으로 파일에 액세스 할 때, RAM 메모리를 최대한 사용하여 파일 정보를 캐시로 보존하는 동작을 반복한다. 
WSL2에 할당된 RAM이 부족해지면 호스트PC의 메모리를 추가적으로 전달하여 리눅스 캐시쪽에 메모리가 몰려있는 이슈가 발생한다.     
기본적으로 호스트 PC의 80%까지 메모리를 땡겨쓸수 있다고 하니 쾌적환 개발환경을 위해선 적절한 관리가 필요하다.

### 해결방법

#### 1. wsl --shutdown
`wsl --shutdown` 개인적으로 가장 많이 사용했던 방법이다. 컨테이너를 사용하지 않는다면 굳이 메모리를 점유하도록 놔둘 필요도 없다.

#### 2. 할당시킬 메모리 통제
`%USERPROFILE%/.wslconfig` 파일을 생성하고 할당 메모리 양 통제하는 방법이다.
```text
[wsl2]
memory=4GB
```
할당시킬 메모리를 4GB로 제한하는 방법이다. 로컬에서는 별 문제 없겠지만 사실 사용하는 메모리를 제한하는 것이 썩 좋아보이지 않고 귀찮을 것 같다. 
파일을 저장한뒤 `wsl --shutdown` 명령어를 통해 재기동해야 적용된다.

#### 3. autoMemoryReclaim=gradual
마지막 방법은 약간 감동이다. 메모리가 해제되지 않는 이 이슈는 1년이 넘도록 사람들 사이에서 문제가 제기되었지만 MS측에서도 간단하게 처리하지 못한 문제였다.
> https://github.com/microsoft/WSL/issues/4166

![Desktop View](/assets/img/post/202310/5.png){: width="1000" height="800" }
_2019년부터 ㅎㅎ_

그런데 2023년 9월 WSL 새로운 릴리즈 버전(2.0.0)에서 메모리 회수에 대한 실험적 기능을 출시했다.   
![Desktop View](/assets/img/post/202310/6.png){: width="800" height="450" }
_따끈 따끈 소식_

![Desktop View](/assets/img/post/202310/7.png){: width="800" height="450" }

간단하게 내용을 살펴보면 캐시화된 메모리를 회수하여 WSL VM의 메모리를 축소하는 기능이 실험적 기능으로 출시되었다고 한다.

유후 CPU 사용량을 감지한 후 캐시된 메모리를 자동으로 회수하는데 점진적인 방식으로 회수하는 `gradual`, 캐시 메모리를 드랍시키는 `dropcache` 두가지 옵션을 제공한다.
`gradual` 옵션은 유후 상태가 5분이상 지속 시, 메모리 회수를 시작한다고 하니 현재로선 가장 좋은 대안이 아닐까 싶다.

사용법은 `%USERNAME%/.wslconfig` 경로에 파일을 생성하고 아래 텍스트를 입력하면 된다. wsl 재기동은 필수~
```text
[experimental]
autoMemoryReclaim=gradual
```

> wsl 2023 release 유튭 : https://www.youtube.com/watch?v=OB2Rfzy5V0Y
> wsl 2023 release 블로그 : https://devblogs.microsoft.com/commandline/windows-subsystem-for-linux-september-2023-update/
