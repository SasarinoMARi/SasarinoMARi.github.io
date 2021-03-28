---
layout: post
title:  "[라즈베리파이] init.d 시작프로그램 등록하기"
date:   2021-03-28 11:07:00 +0900
categories: [linux, raspberrypi]
background: '/assets/images/bg-post.jpg'
---

<br>

# 여는 글

윈도우에선 시작프로그램을 `시작 메뉴의 Startup 폴더`나 `작업 스케줄러`로 간단히 등록할 수 있지만 <br>
리눅스의 경우 시스템 환경이나 프로그램이 실행되어야 타이밍에 따라 몇 가지 방법을 사용해 시작 프로그램을 등록합니다. <br>

부팅시 프로그램이 실행되어야 하는 경우 init.d를 사용하는 방법이 일반적입니다. <br>
그 밖에 주기적으로 실행되어야 하는 프로그램은 crontab을 사용할 수 있습니다.

<br>

***

# 1. init.d 스크립트 작성하기

init.d 폴더는 /etc/init.d 경로에 위치해 있습니다. <br>
이곳에 스크립트를 만들어 넣으면 커널이 초기화되고 우선순위에 따라 스크립트가 실행됩니다.

![init.d 폴더 내부](/assets/images/20210328/001.jpg)*/etc/init.d 폴더 내부의 모습입니다.*

***

기본적으로 쉘 스크립트와 동일하지만 몇 가지 주석이 필요합니다. <br>
아래는 제가 사용중인 스크립트입니다.

```sh
#!/bin/sh
#
### BEGIN INIT INFO
# Provides:          spcm
# Required-Start:    $remote_fs $syslog
# Required-Stop:     $remote_fs $syslog
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: spc management daemon
# Description:       spc management daemon
### END INIT INFO

export PATH=$PATH:/spcm

case "$1" in
  start)
    echo "starting spcmd"
    (cd /spcm && sudo sh startup.sh)
    ;;
  stop)
    echo "Stopping spcmd"
    ;;
  *)
    echo "uasge: spcm start / stop"
    ;;
esac

exit 0
```

맨 윗줄은 어떤 쉘로 이 스크립트를 실행할지 나타냅니다.<br>
그 아래 INIT INFO 블록은 스크립트가 어떻게 실행되어야 할 지를 설정하는 구간입니다.

* `Providers`는 고유한 이름입니다.
* `Required-Start`와 `Required-Stop`은 스크립트가 시작되기 전에 어떤 기능들이 활성화 되어야하는지 나타내는 코드입니다. 
  자세한 목록은 [여기](https://wiki.debian.org/LSBInitScripts)에서 확인할 수 있습니다.
* `Default-Start`와 `Default-Stop`은 스크립트가 어떤 런레벨에서 실행되어야하는지 나타내는 코드입니다.

*** 

밑으로는 쉘 스크립트를 작성하면 됩니다.

`export PATH=...`는 하위 프로세스가 어느 경로에서 시작할지 정의하는 코드입니다. <br>
첫 번째 인자가 start일 경우 startup.sh를 실행하고, stop일 경우 아무 처리를 하고 있지 않은 코드를 확인할 수 있습니다.

<br>

***

# 2. 작성한 스크립트를 rc.d에 등록하기

우선 실행 권한이 필요하기 때문에 `chmod 755` 명령어로 실행 권한을 부여합니다. <br>
그런 다음 `update-rc.d` 명령어로 rd.d에 스크립트를 등록합니다.<br>

```
chmod 스크립트명
update-rc.d 스크립트명 defaults
```

<br>

***

# 마치며

정리하고 보니 내용은 별거 없는데 알기 쉽게 설명해놓은 글이 없어서 헛고생을 한 것 같네요 ㅜㅜ <br>
제가 쓴 글도 잘 설명한게 맞는지 자신은 없습니다... <br>

아무튼! 이번에 파이를 장만해서 이것저것 만들며 놀고있어요. <br>
리눅스를 제대로 써보는건 처음이라서 정말 재미있습니다. <br>
