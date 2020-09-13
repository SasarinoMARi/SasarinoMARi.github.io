---
layout: post
title:  "FreeProxy 프로그램을 이용해 간단하게 프록시 서버를 만들고 TeamViewer에 연결하기"
date:   2020-09-13 20:02:00 +0900
categories: [windows, server, security, remote, network]
---

# 0. 서론

추가예정.

----------------

# 1. FreeProxy 다운로드

FreeProxy는 프리웨어입니다. 제작사인 [Handcrafted Software 다운로드 페이지](https://www.handcraftedsoftware.org/index.php?page=download&op=getFile&id=2&title=FreeProxy-Internet-Suite)에서 무료로 받을 수 있습니다.

설치 과정에서 별도로 신경쓸 점은 없습니다. 하지만 실행할 때는 반드시 `관리자 권한`으로 실행해 주세요.

![FreePorxy가 실행된 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913001.png)

위 사진에서 빨간 네모 표시 되어있는 부분을 더블 클릭해 프록시 설정 창을 엽니다.

![Proxy 설정 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913002.png)

프로토콜과 포트는 기본 설정대로 `Http Proxy`, `8080번 포트`를 사용했습니다.
Local Binding과 Remote Binding 설정은 각각 어떤 네트워크 어댑터를 사용해서 인바운드/아웃바운드를 처리할 것인지를 지정하는 필드입니다.
여기서는 양쪽 모두 `인터넷에 연결된 어댑터를 선택`해 패킷이 해당 포트를 거쳐가도록 만들었습니다.

완료 버튼을 눌러 설정을 저장하고, 상단 메뉴의 `Start/Stop`을 클릭하면 나오는 팝업에서 Start 버튼을 눌러 프록시를 실행합니다.

![Proxy 실행 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913003.png)

중간에 설정을 저장하라는 alert가 나오면 저장 후 진행하시면 됩니다.
위 사진의 오른쪽에서 프록시가 실행되어 Service가 Running 상태인 것을 확인할 수 있습니다.

**이로서 현재 컴퓨터의 8080번 포트로 요청을 보내 프록시 서버로서 사용할 수 있게 되었습니다!**

추가적으로, 공유기를 사용 중이시라면 당연하게도 포트 포워딩 설정을 해주셔야 합니다.
저의 경우엔 사내망 라우터에서 80번과 443번 포트 이외 대부분의 포트를 차단하고 있어 443→8080으로 포트포워딩을 구성했습니다.

----------------

# 2. 팀뷰어에서 프록시 설정하기

TeamViewer 창을 열고, 메뉴에서 `기타 > 옵션 > 일반 > 네트워크 설정 > 프록시 설정` 순서대로 접근해 프록시 설정 메뉴를 엽니다.
TeamViewer 15.9.4 버전을 기준으로 작성되었습니다. 버전이 달라도 프록시 설정을 찾아서 열어 주세요.

![프록시 설정화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913004.png)

위와 같은 프록시 설정 화면에서 `원격지 PC의 IP 주소`와 `FreeProxy에서 설정한 포트 번호`를 입력해 주세요.
포트포워딩 설정을 한 경우 공유기의 외부 IP와 포트포워딩 설정한 포트 번호를 써주세요.
OK 버튼을 누른 후 서버에 연결되기까지 잠시 기다리면 설정 완료입니다.

![프록시 설정 전후 비교](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913005.png)

위는 프록시 설정 전후의 패킷 흐름 비교입니다.
왼쪽은 Proxy가 비활성화된 일반적인 상태입니다. TeamViewer 서버로 추정되는 어디론가에 연결되어 있네요.
오른쪽은 Proxy가 활성화된 상태입니다. 저희 집 공유기의 외부 IP주소로 연결이 되고 있는 모습이네요. (IP는 가렸습니다.)