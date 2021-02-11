---
layout: post
title:  "Fiddler로 안드로이드 에뮬레이터 패킷 캡쳐하기"
date:   2021-02-11 16:22:00 +0900
categories: [network, android]
---


# 여는 글

요즘 밖에서 집에 있는 데스크톱에 접속해 작업을 할 일이 많네요. 

컴퓨터가 켜져있으면 접속하는거야 문제 없지만 항상 켜두고다니자니 전기세가 아까워 WOL(Wake On Lan) 기능을 통해 원격으로 컴퓨터를 켜서 사용하고 끄는 식으로 활용하고 있습니다.

지금은 직접 공유기 관리 페이지에 접속해서 켜거나, 플레이스토어에서 내려받은 ipTime WOL 앱을 쓰거나 하고 있지만 좀 더 편리하게 사용하기 위해 직접 관리 서버를 만들고 싶습니다.

부팅 뿐 아니라 모니터링이나 원격 종료, 간단한 명령 수행 지원 정도를 할 수 있으면 정말 유용할 것 같네요!

오늘은 그 첫 단계를 위해 피들러와 안드로이드 에뮬레이터를 사용해 ipTime WOL 앱이 어떤 패킷을 보내 컴퓨터를 켜고 있는지 알아보고자 합니다. 
그런 다음에 통신 내용을 모방해 앱에서 켜는 것처럼 Http Request를 보내면 해결될 것 같네요!


----------------


# 1. 프로그램 설치와 환경 설정


### 피들러(Fiddler) 설치

[피들러 공식 홈페이지](https://www.telerik.com/fiddler)에서 설치 마법사를 내려받고 설치합니다.

설치는 그냥 쭉 진행하면 되는데 프로그램을 사용하려면 계정을 만들고 로그인을 해야하는게 조금 귀찮네요.


### 안드로이드 에뮬레이터 설치

에뮬레이터 쪽은 미뮤(Memu) 앱플레이어를 사용했습니다. 마찬가지로 [미뮤 공식 홈페이지](https://www.memuplay.com/)에서 무료로 설치할 수 있습니다.

에뮬레이터는 어떤걸 사용하든 상관 없습니다만 Https 통신의 경우 피들러의 루트 인증서가 설치되지 않으면 정상적으로 동작하지 않기 때문에 설치를 위해 루트 권한이 필요합니다. 
저는 안할거라 잘 모르겠네요... 필요하다면 루팅 기능을 지원하는 앱플레이어를 사용하세요.


### 에뮬레이터 프록시 설정

피들러 설정에서 Connections 탭을 보면 프록시 서버 설정이 있습니다.
Allow remote computers to connect를 체크하고 저장해주세요.

![프록시 서버 설정 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20210211001.png)

설정을 저장하고 피들러를 한번 껏다 켜주시는게 좋습니다.


에뮬레이터의 네트워크 설정에 들어가서 다음과 같이 네트워크에 프록시 설정을 해 주세요.

![네트워크 설정 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20210211002.png)

프록시 호스트 이름은 LAN에서 할당받은 ip 주소를 입력해주시면 됩니다.
포트는 위에서 설정한 것과 동일하게 맞춰주세요.


# 2. 패킷 분석해 보기

설정을 잘 마쳤다면 피들러의 Live Traffic 패널에 에뮬레이터에서 이루어진 네트워크 통신이 캡쳐됩니다. 
Process 필터에 Contains Memu와 같이 걸어주어 원하는  프로세스만 모아서 볼 수 있어요.

![트래픽 일람 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20210211003.png)

ipTime WOL 앱으로 공유기에 접속하고 매직 패킷을 보내는 것까지 무사히 캡쳐되었네요.

![패킷 자세히보기 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20210211004.png)

아..아니나 다를까 공유기 인증 정보를 평문으로 주고받고 있네요. 이 정신나간 놈들아!!!


# 마치며

상상이상으로 간단하게 스니핑을 할 수 있어서 정말 좋았어요!

프록시 서버도 그렇고, https 패킷을 볼 때 필요한 루트 인증서도 피들러에서 지원하기 때문에 버튼만 누르면 설치된다는 모양입니다. 사용자가 필요로 할만한 것들을 전부 간편하게 쓸 수 있게 해둔 점이 정말 마음에 드는 프로그램이네요. 

다음에는 오늘 얻은 패킷들을 바탕으로 매직 패킷을 보내는 node서버를 만들어보고자 합니다. 읽어주셔서 감사합니다.