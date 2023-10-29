---
title: 🐳 Docker로 Jenkins서버를 구축하고 CI/CD 맛보기
author: mingo
date: 2023-10-10 19:30:00 +0900
categories: [docker, jenkins]
tags: [docker, jenkins, ci/cd]
---

----

회사에서는 인프라팀이 따로 전담하고 있는 부분이라 Jenkins를 배포하는 용도로만 사용하고 구축 및 환경 설정에는 관여하지 않았다. 
알아두면 도움이 될 것 같아서 실제 Jenkins 서버 구축부터 Github 프로젝트 CI/CD 과정을 자동화를 해보았다. 

## 환경 및 구성 요소
가볍게 Docker로 모든 환경을 구성하고 실행하기로 결정했다.
1. Docker를 사용하여 스프링부트 서비스가 동작하기 위한 ubuntu 서버, 이하 service 서버 
2. Docker를 통한 Jenkins 서버

## service 서버 구축
우선 service 서버를 구축하기 위해 Dockerfile을 생성한다. 
Dockerfile이란 컨테이너 이미지를 생성하는 데 사용되는 텍스트 파일이며, 애플리케이션과 그 실행 환경을 포함하는 경량 가상화 단위이다. 
즉, Dockerfile은 컨테이너를 위한 일종의 명세서이고 이미지를 build하는데 필요한 모든 명령과 구성 정보를 갖추고 있다.

적당한 디렉토리에 boot_service 폴더를 생성하고 그 안에 Dockerfile을 생성한다.
```dockerfile
FROM ubuntu:20.04

ENV USER boot

RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y sudo net-tools ssh openssh-server openjdk-17-jdk-headless

RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -i 's/UsePAM yes/#UserPAM yes/g' /etc/ssh/sshd_config

RUN groupadd -g 999 $USER
RUN useradd -m -r -u 999 -g $USER $USER

RUN sed -ri '20a'$USER'    ALL=(ALL) NOPASSWD:ALL' /etc/sudoers

RUN echo 'root:root' | chpasswd
RUN echo $USER':boot' | chpasswd

ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64

ENTRYPOINT sudo service ssh restart && bash

USER $USER
```
 - `FROM ubuntu:20.04` : 컨테이너 베이스 이미지를 의미한다. 우분투 20.04 버전을 사용한다.
 - `ENV USER boot` : 환경변수 USER를 boot로 설정한다.
 - `RUN apt-get ...` : 컨테이너 내에서 실행될 명령어를 정의한다. ubuntu 패키지 관리자를 통해 시스템을 업데이트하고 필요한 패키지를 설치한다. `openssh-server` 패키지는 SSH서버를 호스팅하는데 사용되고 원격 접속 및 관리가 허용된다. `openjdk-17-jdk-headless` java 17 버전을 사용해서 스프링부트 서비스를 띄울 생각이다. headless 의미는 그래픽 사용자 인터페이스(GUI)를 지원하지 않는 JDK라는 의미다.
 - `RUN sed -i` : SSH 서버 파일을 수정해서 원격 로그인을 허용하는 `PermitRootLogin` 을 활성화하고 사용자 인증 및 보안 정책을 위한 `PAM`을 비활성화 한다.
 - `RUN groupadd || useradd ...` : 위에서 설정한 boot를 사용자 이름으로 가지고 사용자 그룹과 사용자를 추가한다.
 - `RUN sed -ri ...` : sudo 명령을 사용하여 사용자에 대한 비밀번호 입력을 필요로 하지 않도록 sudoers 파일을 수정한다. 
 - `RUN echo 'root:root' | chpasswd` : root 사용자의 비밀번호를 root로 설정한다. 
 - `RUN echo $USER':boot' | chpasswd` : 사용자의 비밀번호를 boot로 설정한다.
 - `ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64` : Java 환경 변수를 설정한다.
 - `ENTRYPOINT sudo service ssh restart && bash` : 컨테이너가 실행될 때 실행되는 명령어를 정의한다. SSH 서비스를 재시작하고, bash셀을 실행한다. 컨테이너에 사용자가 로그인할 수 있게 된다.
 - `USER $USER` : 사용자를 $USER, 즉 boot로 전환한다. 컨테이너 내에서 사용자는 `boot`

이제 도커 이미지를 빌드하면 된다.

`docker build -t boot-service .`

![Desktop View](/assets/img/post/20230908/10.png){: width="500" height="150" }

이미지 태그에 식별을 위해 `boot-service`라는 이름을 붙여주었고 `.`은 현재 디렉토리에서 Dockerfile을 찾아 사용한다는 것을 나타낸다.

![Desktop View](/assets/img/post/20230908/26.png){: width="500" height="150" }
_docker images_

`docker images` 명령어를 통해 이미지가 생성된 것을 확인할 수 있다. 이제 생성된 이미지를 바탕으로 컨테이너를 실행시킨다.

`docker run --name boot-service -itd -p 9000:9000 -p 9022:22 boot-service`
 - 컨테이너 식별을 위해 동일하게 boot-service로 명명했다.
 - `-itd` 옵션을 통해 터미널 상호작용을 가능하게 하고 백그라운드 모드 실행을 지원했다.
 - 호스트PC port `9000`과 boot-service port `9000`을 매핑한다. 실제 애플리케이션은 `9000` port로 실행시켜 접근할 예정이다.
 - 호스트PC port `9022`과 boot-service port `22`을 매핑한다. Jenkins에서 ssh로 접근할 poer이다.

그럼 직접 컨테이너 내부로 진입하여 추가 작업을 해보자.

`docker exec -it boot-service /bin/bash`

![Desktop View](/assets/img/post/20230908/27.png){: width="500" height="150" }

정상적으로 진입되면 위의 Dockerfile을 통해 생성된 user명인 boot로 진입되어 있는 것을 확인할 수 있다.

Jenkins 서버로부터 깃 프로젝트를 build하고 실행시킬 스크립트를 미리 작성해두자. 경로는 /home/boot/boot_service/run.sh 로 잡아두었다.

```shell
JAR_PATH="/home/boot/boot_service/target/jenkins-with-docker-0.0.1-SNAPSHOT.jar"
PROCESS_NAME="jenkins-with-docker"

start() {
  echo "Deploy Project...."
  nohup java -jar $JAR_PATH > /dev/null 2>&1 &
  echo "Done"
}

stop() {
  echo "PID Check..."
  CURRENT_PID=$(ps -ef | grep java | grep $PROCESS_NAME | awk '{print $2}')
  echo "Running PID: $CURRENT_PID"

  if [ -z "$CURRENT_PID" ]; then
     echo "Project is not running"
  else
     echo "Project is stopping"
     kill -9 $CURRENT_PID
     sleep 3
  fi
}

restart() {
  stop
  sleep 5
  start
}

case $1 in
start)
    start
    ;;
stop)
    stop
    ;;
restart)
    restart
    ;;
*)
   echo "Usage: ./run.sh {start|stop|restart}"
esac
```
 - run.sh 에 start/stop/restart 인자를 넘겨 실행하는 간단한 스크립트  

스크립트가 완성 되었다면 Jenkins 서버에서 발급한 SSH key를 등록하는 일만 남았다. 그럼 우선 Jenkins 서버를 도커를 이용해서 구성해보자.

## Jenkins 서버 구축
Jenkins 서버는 Docker hub에서 이미지를 가져와 실행시키면 된다. Docker hub란 도커 컨테이너 이미지를 저장하고 공유하는데 사용되는 클라우드 기반 이미지 저장소이다. 
여러가지 이미지를 업로드, 저장, 공유, 검색, 다운로드 할 수 있는 중앙 저장소의 개념이다. https://hub.docker.com/search?q=&type=image

```text
// Docker hub에서 최신버전 Jenkins 이미지를 가져온다.
$ docker pull jenkins/jenkins:lts
```
![Desktop View](/assets/img/post/20230908/28.png){: width="500" height="150" }

Jenkins 이미지가 로컬에 들어온 것이 확인되면 컨테이너에 올려둔다.

```text
docker run --name jenkins-docker -d -p 8080:8080 -p 50000:50000 -v /home/jenkins:/var/jenkins_home -u root jenkins/jenkins:lts
```
 - jenkins 서버 컨테이너 명을 `jenkins-docker` 로 지정하두었다.
 - 호소트 PC의 8080 port를 컨테이너 por 8080과 매핑 -> 젠킨스 웹 인터페이스 접근
 - 호스트 PC의 /home/jenkins 디렉토리와 젠킨스 컨테이너 /var/jenkins_home 디렉토리에 마운팅한다. Jenkins 설정을 호스트 PC에서 제어할 수 있도록 한다.

이제 호스트 PC에서 `localhost:8080` 경로에 접속해보자.

![Desktop View](/assets/img/post/20230908/1.png){: width="600" height="600" }
_localhost:8080_

젠킨스 초기 서버 접속을 위한 비밀번호를 입력해야한다. 비밀번호는 컨테이너 기동 시 로그를 보면 확인할 수 있다. (docker logs jenkins-docker)
![Desktop View](/assets/img/post/20230908/4.png){: width="600" height="600" }
_docker logs jenkins-docker_

또는 jenkins 서버에서도 확인할 수 있다.
![Desktop View](/assets/img/post/20230908/3.png){: width="600" height="600" }

비밀번호를 입력하면 이어서 플러그인 설치를 요청하는 페이지가 노출된다. 설치하자. 
![Desktop View](/assets/img/post/20230908/2.png){: width="600" height="600" }
_plugin 설치_

![Desktop View](/assets/img/post/20230908/6.png){: width="600" height="600" }
설치가 완료되면 대시보드로 이동된다. 이어서 Jenkins 서버 -> boot 서버로 접근하기 위한 플러그인 `Publish Over SSH` 패키지를 설치한다.
Jenkins 관리 -> System configuration -> Plugins -> Publish Over SSH 검색
![Desktop View](/assets/img/post/20230908/7.png){: width="600" height="600" }

![Desktop View](/assets/img/post/20230908/8.png){: width="600" height="600" }

플러그인 까지 설치가 완료되면 Github와 부트서비스 서버에 넘겨줄 공개키를 얻기 위하여 Jenkins 서버에서 SSH 키 생성을 진행한다.

> SSH 키는 개인 키(Private Key)와 공개 키(Public Key) 페어로 구성된다. SSH 프로토콜은 네트워크 프로토콜 및 암호화 기술로 원격 컴퓨터나 서버와 안정하게 통신하기 위한 표준 프로토콜이다. 서버 사이에서 SSH 프로토콜을 사용할 수 있는 인증수단으로 SSH 키 페어가 사용되는 것이다. 개인 키는 서비스 주체가 소유해야하고 안정하게 보관되어야 한다. 여기서는 젠킨스 서버가 주체가 된다. 반면 공개 키는 대상 서비스에게 제공이 되며 대상은 이 공개키를 사용하여 주체를 인증한다. Github 및 부트서비스는 공개 키를 등록하여 접근하는 신원을 확인하는데 사용된다.
{: .prompt-tip }

Jenkins 서버를 컨테이너 띄울때 마운트를 해둔 상태이므로 호스트 PC에서 ssh키를 생성하자.
```text
$ sudo mkdir /home/jenkins/.ssh
$ sudo chmod 700 /home/jenkins/.ssh
$ sudo ssh-keygen -t rsa
```
마지막 명령어를 수행하면 키 페어 파일을 저장하는 경로 설정을 하라는 문구가 나오는데 적당한 곳에 저장해두자.

![Desktop View](/assets/img/post/20230908/29.png){: width="600" height="600" }

개인 키는 `id_rsa` 이며, 공개 키는 `id_rsa.pub` 로 저장된다.

1. 깃허브에 접속하고 우측 상단에 프로필을 누른 뒤 Settings에 접속한다. `SSH and GPG keys` -> `New SSH key` 선택 -> 공개 키(id_rsa.pub) 등록
![Desktop View](/assets/img/post/20230908/13.png){: width="600" height="600" }

2. boot-service에 다시 접속해서 `~/.ssh/authorized_keys` 파일을 생성 -> 공개 키(id_rsa.pub) 등록

깃허브와 부트서비스 양쪽 모두 공개 키(id_rsa.pub) 설정을 완료 하였다면 이제 젠킨스 서버에서 SSH 개인 키(id_rsa)를 등록한다.

![Desktop View](/assets/img/post/20230908/15.png){: width="600" height="600" }
Jenkins 대시보드 -> Jenkins 관리 -> System Configuration -> System -> Publish over SSH 항목

![Desktop View](/assets/img/post/20230908/17.png){: width="600" height="600" }
 - Name : 식별자
 - Hostname : Host PC ip 주소
 - Username : 부트서비스 사용자 계정 `boot` 
 - Remote Directory : 부트서비스 사용자 루트 `/home/boot`

![Desktop View](/assets/img/post/20230908/18.png){: width="600" height="600" }
 - Passphrase / Password : 부트서비스 사용자 패스워드 `boot`
 - Port : 호스트 PC로 개방한 `9022` port

하단의 `Test Configuration`을 클릭 했을 때, 정상적으로 수행된다면 접속 상태가 온전한 것으로 판단할 수 있다.

이제 Github 저장소랑 SSH 연결을 해보자.
![Desktop View](/assets/img/post/20230908/19.png){: width="600" height="600" }
_github page_
SSH 링크를 복사해 두고 Jenkins 프로젝트에 등록한다.

![Desktop View](/assets/img/post/20230908/20.png){: width="600" height="600" }
_Jenkins 대시보드 -> 새로운 item -> 프로젝트 명 입력_

프로젝트 명을 입력하면 깃허브 SSH 링크를 넣는 부분이 나온다. 작성해주고 Credentials add 버튼을 클릭하자.

![Desktop View](/assets/img/post/20230908/21.png){: width="600" height="600" }
_Credentials > add_
Kind 항목을 `SSH Username with private key`로 바꾸고 

![Desktop View](/assets/img/post/20230908/22.png){: width="600" height="600" }
_개인 키_
Private key(개인 키) 부분을 Jenkins 서버에서 생성한 개인 키(id_rsa)를 입력한다.
Crudentials가 정상적으로 등록되면 다음 스텝을 진행할 수 있다.
 - Name : origin
 - Refspec : +refs/pull/\*:refs/remotes/origin/pr/\*
 - Branch : master

Build Steps 항목에서 `Invoke Gradle script`를 선택하고 실행시킬 Tasks를 작성한다.
![Desktop View](/assets/img/post/20230908/25.png){: width="600" height="600" }

(참고로 빌드 유발 항목에서 `GitHub hook trigger for GITScm polling`를 통해 github 웹훅과 연동시켜 원격 repository에 소스코드가 push가 되면 github가 jenkins 서버로 push된 소스코드가 있다는 알림을 던져주고 젠킨스 서버는 자동으로 배포까지 진행시킬수도 있다.)

빌드 후 조치 항목에서 부트서비스로 빌드된 파일을 전달하기 위해 `Send files or excute commands over SSH` 를 선택한다.
 - Source files : build/libs/*.jar
 - Remove prefix : build/libs
 - Remote directory : /boot_service/target
 - Exec command : /home/boot/boot_service/run.sh restart
   - 부트서비스 서버를 만들 때 작성해둔 스크립트 파일을 시작하는 명령

마지막 저장을 누르면 설정은 완료되었다. 간단한 시간 출력 메서드를 작성한 스프링부트 프로젝트를 github repository로 올려둔 상태로 `지금 빌드`를 클릭하여 배포하였다.
![Desktop View](/assets/img/post/20230908/30.png){: width="600" height="600" }

![Desktop View](/assets/img/post/20230908/31.png){: width="600" height="600" }
![Desktop View](/assets/img/post/20230908/32.png){: width="600" height="600" }

빌드가 성공적으로 완료되었고 부트 서비스의 로또번호 API 호출도 정상적으로 수행되는 것을 확인했다. 정리 끝!
