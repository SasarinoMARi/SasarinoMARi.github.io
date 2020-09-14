---
layout: post
title:  "FreeProxy 프로그램을 이용해 간단하게 프록시 서버를 만들고 TeamViewer에 연결하기"
date:   2020-09-13 20:02:00 +0900
categories: [windows, server, security, remote, network]
---

# 여는 글

일부 사내망 등에서 보안을 이유로 `TeamViewer`가 차단된 경우가 있습니다.

이번 포스트에서는 `FreeProxy`를 이용해 원격지 컴퓨터에 역방향 프록시를 구성하고 팀뷰어에 연결하는 방법을 다룹니다.
쉽게 생각하면 방화벽이 팀뷰어 서버를 차단하고 있으니 우리는 프록시 서버에 연결하고, 프록시 서버에선 팀뷰어 서버에 연결을 하는거죠.

`FreeProxy`는 팀뷰어 뿐 아니라 프록시를 지원하는 다른 모든 프로그램에 사용할 수 있습니다.

----------------

# 1. FreeProxy 다운로드

FreeProxy는 프리웨어입니다. 제작사인 [Handcrafted Software 다운로드 페이지](https://www.handcraftedsoftware.org/index.php?page=download&op=getFile&id=2&title=FreeProxy-Internet-Suite)에서 무료로 받을 수 있습니다.

설치 과정에서 별도로 신경쓸 점은 없습니다. 하지만 실행할 때는 반드시 `관리자 권한`으로 실행해 주세요.

![FreePorxy가 실행된 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913001.jpg)

위 사진에서 빨간 네모 표시 되어있는 부분을 더블 클릭해 프록시 설정 창을 엽니다.

![Proxy 설정 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913002.jpg)

프로토콜과 포트는 기본 설정대로 `Http Proxy`, `8080번 포트`를 사용했습니다.
Local Binding과 Remote Binding 설정은 각각 어떤 네트워크 어댑터를 사용해서 인바운드/아웃바운드를 처리할 것인지를 지정하는 필드입니다.
여기서는 양쪽 모두 `인터넷에 연결된 어댑터를 선택`해 패킷이 해당 포트를 거쳐가도록 만들었습니다.

완료 버튼을 눌러 설정을 저장하고, 상단 메뉴의 `Start/Stop`을 클릭하면 나오는 팝업에서 Start 버튼을 눌러 프록시를 실행합니다.

![Proxy 실행 화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913003.jpg)

중간에 설정을 저장하라는 alert가 나오면 저장 후 진행하시면 됩니다.
위 사진의 오른쪽에서 프록시가 실행되어 Service가 Running 상태인 것을 확인할 수 있습니다.

**이로서 현재 컴퓨터의 8080번 포트로 요청을 보내 프록시 서버로서 사용할 수 있게 되었습니다!**

아래는 기타 참고할만한 내용들입니다.
 - 한 번 서비스를 가동하면 컴퓨터를 재부팅해도 서비스가 유지됩니다.
 - 당연하지만 공유기를 사용 중이시라면 포트 포워딩 설정을 해주셔야 합니다.
 제 경우엔 사내망 라우터에서 80번과 443번 포트 이외 대부분의 포트를 차단하고 있어 443→8080으로 포트포워딩을 구성했습니다.

----------------

# 2. 팀뷰어에서 프록시 설정하기

TeamViewer 창을 열고, 메뉴에서 `기타 > 옵션 > 일반 > 네트워크 설정 > 프록시 설정` 순서대로 접근해 프록시 설정 메뉴를 엽니다.
TeamViewer 15.9.4 버전을 기준으로 작성되었습니다. 버전이 달라도 프록시 설정을 찾아서 열어 주세요.

![프록시 설정화면](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913004.jpg)

위와 같은 프록시 설정 화면에서 `원격지 PC의 IP 주소`와 `FreeProxy에서 설정한 포트 번호`를 입력해주세요.
포트포워딩 설정을 한 경우 공유기의 외부 IP와 포트포워딩 설정한 포트 번호를 쓰면 됩니다..

OK 버튼을 누른 후 서버에 연결되기까지 잠시 기다리면 설정 완료입니다.

![프록시 설정 전후 비교](https://raw.githubusercontent.com/sasarinomari/sasarinomari.github.io/master/static/img/_posts/20200913005.jpg)

위는 프록시 설정 전후의 패킷 흐름 비교입니다.

Proxy가 비활성화된 상태에서는 TeamViewer 서버로 추정되는 어딘가에 연결되어 있네요.
오른쪽은 Proxy가 활성화된 상태입니다. 저희집 공유기의 외부 IP주소로 연결이 되고 있는 모습입니다. (IP는 가렸습니다)

----------------

# 닫는 글

이로서 TeamViewer가 차단된 환경에서도 원격 데스크탑 접속을 할 수 있게 되었습니다.

제 경우엔 해당하지 않지만, 방화벽에서 차단하는것 뿐 아니라 해당 PC에서 실행중인 프로세스 등을 관리 프로그램에서 감시하고 있는 경우도 있을 테니 주의하시기 바랍니다.

또, 프록시 설정 전에 TeamViewer를 켤 경우 팀뷰어 서버로 연결을 시도하기 때문에 로그가 남는 것이 꺼려진다면 네트워크를 비활성화 하고 작업하거나, PC의 방화벽 아웃바운드 설정에서 통신을 잠시 차단하시고 작업하시길 권장드립니다.