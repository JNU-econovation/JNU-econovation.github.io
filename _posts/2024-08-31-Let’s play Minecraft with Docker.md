---
layout: post
title: Let’s play Minecraft with Docker
subtitle: Let’s play Minecraft with Docker
author: capDoYeonLee
categories: TECH
banner: 
  image: "https://github.com/user-attachments/assets/ac46f789-ab9f-4568-993f-ae25c5101d10"
tags: TECH
sidebar: []
---

# Let’s play Minecraft with Docker

안녕하세요, 에코노베이션에서 백엔드와 인프라를 공부하는 이도연입니다.
이번 기술 블로그는 조금 라이트하게 흥미를 유발할 수 있는 주제이니 재밌게 읽어주시면 감사하겠습니다.


에코노베이션의 성과공유회 SUMMER DEV가 끝난 직후 회원들은 조금 여유로워져 그동안 하지 못한 게임을 다 같이 즐겨 하는 중인데요. 이 게임은 바로 어릴 적 모두가 한 번씩(?) 즐겨 해보았던 마인크래프트입니다.

그런데 마인크래프트를 하는데 왜 Docker가 등장했을까요? 잠시 뒤 이유가 나옵니다!

마인크래프트는 싱글 플레이와 멀티 플레이가 있습니다. 싱글 플레이는 혼자 게임을 진행하지만, 멀티플레이는 다수의 플레이어가 하나의 서버에서 함께 게임을 진행합니다. 

즉, 누군가는 서버를 호스팅해야 멀티플레이가 가능합니다. 아래 사진은 서버 호스트에게 마인크래프트 서버 오픈을 요청하는 사진입니다.

- !여기서 잠깐! ‘서버는 게임을 할 수 있는 공간’, ‘호스팅은 공간을 열다’라는 느낌으로 이해하시면 편할 듯합니다.

![image](https://github.com/user-attachments/assets/ff6607ea-2c72-46a0-8a15-cd25e37168c0)

위 사진과 같이 알 수 있듯이 서버 호스트는 항시 서버 오픈 요청이 있다면 번거롭게 서버를 직접 열어야 한다는 불편한 점이 존재했습니다. 저희는 다음과같이 불편한 점을 개선하였습니다.

Docker를 이용해, 기존 마인크래프트 world의 데이터 손실 없이 AWS EC2 서비스 내부에 마인크래프트 서버를 띄워 원활한 멀티플레이 환경을 구축하였습니다.

그럼 어떻게 Docker를 통해 EC2 내부에 마인크래프트 서버를 구축하였는지 소개해 드리기 전 Docker가 생소하신 분이 계실 수 있으니 간단히 소개하고 넘어가겠습니다.

## Docker란 무엇일까!?

> Docker is an open platform for developing, shipping, and running applications. Docker enables you to separate your applications from your infrastructure so you can deliver software quickly.


도커 공식 문서에서 도커를 소개하는 문장 중 일부를 발췌해 왔습니다.

“Docker는 애플리케이션을 개발, 제공 및 실행하기 위한 오픈 플랫폼입니다. Docker를 사용하면 애플리케이션을 인프라에서 분리하여 소프트웨어를 신속하게 제공할 수 있습니다.”라고 설명되어 있습니다. 

이렇게 설명하면 Docker의 장점이 정확히 무엇인지 이해하기 힘들고 왜 사용해야 하는지 이해가 잘 안 가실 수 있습니다. 아래에서 조금 더 구체적으로 알아보죠!

Docker가 주목받는 이유는 다름 아닌 기존 가상화 기술과 비교할 때 훨씬 가볍고 작으며 빠르기 때문입니다.

기존 가상화 기술과 Docker에서 사용하는 가상화 기술이 어떤 차이점이 있는지 아래 사진을 통해 비교해 보겠습니다.

![image](https://github.com/user-attachments/assets/d3cc8f68-ebef-40fb-9b0a-124014ccb2a6)

(좌 : 가상머신의 가상화 / 우 : 도커 컨테이너의 가상화)

두 가상화 모두 실행하고자 하는 애플리케이션 프로세스 및 종속 요소와 소스 등을 패키지, 즉 이미지화하여 Host OS와 격리된 환경을 제공합니다. 

하지만 **VM 가상화**는 실제 Host OS와 같이 별도의 Guest OS를 두고 원하는 애플리케이션을 설치하는 하드웨어 수준의 가상화이며, **Container 가상화**는 VM 가상화에 비해 경량이면서 Host OS의 Kernel을 공유하는 운영체제 수준의 가상화를 구현합니다.
이를 통해 Docker는 VM에 비해 훨씬 가벼운 이점이 있고, 원하는 애플리케이션을 빠르게 구축할 수 있는 장점이 있습니다.

(Docker 관련해서 실습을 희망한다면 아래 링크를 추천드립니다 🙂)

[https://www.docker.com/play-with-docker/](https://www.docker.com/play-with-docker/)

Docker 소개는 여기서 마무리하고, 이제 본격적으로 마인크래프트 서버를 어떻게 도커를 통해 구축했는지 설명해 드리겠습니다.

## Docker를 통해 마인크래프트 이미지를 실행시켜보자!

아래 사진을 보시면 Docker Hub에 마인크래프트 서버를 실행시킬 수 있는 Docker Image가 공유된 것을 확인하실 수 있습니다.

![image](https://github.com/user-attachments/assets/c1c2b8e9-cfa8-4504-a87b-cf11f6c0bf05)

해당 Docker Hub에서 docker image를 구축하고자 하는 환경에 pull 받으시면 됩니다. 저희 에코노베이션은 AWS EC2를 사용하고 있습니다.

![image](https://github.com/user-attachments/assets/8f3da1b1-a35b-490b-9c94-e4b9d067e8e7)


위 사진을 보시면 `docker images` 커맨드를 통해 현재 EC2 인스턴스에 존재하는 마인크래프트 이미지를 출력해 보았습니다. 

이제 Docker Hub에서 pull 받은 image를 `docker run`커맨드를 통해 실행시킨 후 컨테이너화하면 정상적으로 마인크래프트를 이용하실 수 있습니다. 하지만 더욱 간단하게 실행시킬 수 있는 방법이 있습니다.

바로 docker compose입니다. docker compose는 Docker에서 제공하는 멀티 컨테이너 애플리케이션을 정의하고 실행하기 위한 도구입니다. docker compose를 통해 여러 이미지를 한 번에 실행할 수 있는 간편한 도구라고 보시면 됩니다. 미리 yml 형식의 파일을 선언하여 옵션값을 미리 설정한 후 이용하실 수 있고, 저희는 아래와 같이 docker-compose.yml을 선언하였습니다.

![image](https://github.com/user-attachments/assets/30a34cf0-924b-4d32-b876-cb9881c0dace)
위 선언된 매니페스트에 대해 간단히 소개해 드리겠습니다.

- **services: →** 여러 서비스를 정의할 수 있는 섹션입니다.
- **mc: →** 이 서비스는 `itzg/minecraft-server`라는 이미지를 사용하여 Minecraft 서버를 실행합니다. `mc`는 서비스의 이름이며, Docker Compose를 통해 이 서비스가 실행됩니다.
- **image: itzg/minecraft-server →** `itzg/minecraft-server`라는 Docker 이미지를 사용합니다.
- **tty: true →** 이 옵션은 컨테이너가 종료되지 않고 대기하도록 합니다.
- **stdin_open: true →** 표준 입력(입력 스트림)을 열어둡니다.
- **ports: →** Docker 컨테이너의 포트와 호스트 시스템의 포트를 매핑하는 설정입니다.
- **environment: →** 컨테이너에서 사용할 환경 변수를 정의합니다.
- **volumes: →** `./data:/data`: 호스트의 현재 디렉토리(`./`)에 있는 `data` 디렉토리를 컨테이너의 `/data` 디렉토리에 연결합니다. 이 설정을 통해 컨테이너가 종료되거나 재시작되어도 데이터가 지속됩니다.

위 설정을 통해 원하는 설정값을 미리 세팅하면 추후 이미지를 실행할 때 번거롭지 않게 `docker compose up` 커맨드로 쉽게 실행시킬 수 있습니다.

그렇다면 한번 Docker Hub에서 pull 받은 이미지가 정상적으로 작동하는지 확인해 볼까요?
`docker ps` 커맨드로 현재 실행 중인 컨테이너 목록을 출력할 수 있습니다

![image](https://github.com/user-attachments/assets/ad4f6ed2-cbd4-4002-ace3-7b26b73ea426)
사진을 자세히 보시면 25565 port를 사용하고 있는 마인크래프트 도커 컨테이너를 확인해 보실 수 있습니다. 이제 모든 준비를 마쳤으니 마인크래프트 서버에 접속해 볼까요?

EC2 Public IP로 접근하면 구축된 마인크래프트 서버에 접근이 가능합니다.

![image](https://github.com/user-attachments/assets/ac46f789-ab9f-4568-993f-ae25c5101d10)

(제가 만들고 있는 에코노 집입니다. 너무 커서 재료가 많이 들어가는 것이 단점이죠,,, )

이번 블로그는 가벼운 주제로 Docker에 대해 알아보았습니다. 최대한 가볍게 작성하려고 하다 보니 놓친 부분도 있는데 재밌게 읽어주셨다면 감사하겠습니다.

저희는 대부분 밤늦게까지 열심히 개발 후 새벽에 게임을 종종 하곤 합니다. 저희와 함께 코딩 후 보람찬 하루를 마인크래프트로 마무리하고 싶다면 28기 신입 모집으로 함께하시는건 어떤가요? 이 블로그가 올라갈 시기면 에코노베이션 28기 신입 모집을 진행하고 있을 것 같네요!

저는 여기서 물러가 보도록 하겠습니다. 다음에는 더 멋진 에코노베이션 회원분이 더 퀄리티 있는 블로그로 여러분을 찾아갈 예정입니다. 다음 블로그도 많이 기대해 주세요!!

### 레퍼런스 

docker<br>
[https://docs.docker.com/manuals/](https://docs.docker.com/manuals/)

docker compose overview<br>
[https://docs.docker.com/compose/](https://docs.docker.com/compose/)
