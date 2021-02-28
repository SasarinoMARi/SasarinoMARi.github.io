---
layout: post
title:  "Node로 컴퓨터 관리 서버 만들기 - 1"
date:   2021-02-24 20:44:00 +0900
categories: [nodejs, server]
background: '/assets/images/bg-post.jpg'
---

<br>

# 여는 글

[지난 포스트]({% post_url 2021-02-11-Fiddler_With_AndroidEmulator %})에서 Iptime 공유기의 Wake On Lan 패킷을 확인해 보았습니다.

WOL 서버를 만들기 전에, 오늘은 REST API로 간단하게 컴퓨터 조작을 할 수 있게 해주는 서버를 만들려고 합니다.

<br>

***

# 1. express 프레임워크로 웹서버 열기

node 개발 환경 설정은 모두 되어있다고 전제하겠습니다.
index.js에 다음과 같이 라우터 코드를 작성합니다.

```js
const app = require('express')();

app.get('/establishment', function (req, res, next) {
    var result = { error : 0, message : ""}
    var key = req.query.key
    if(key === undefined) {
        res.statusCode = 403
        res.message = "인증 정보가 잘못되었습니다."
        res.json();
        return;
    }
    res.json(result);
});

app.get('/lookup', function (req, res, next) {
    var result = { error : 0, message : ""}
    result.status = "standby"
    res.json(result);
});

var port = 80
var server = app.listen(port, function () {
    console.log(`Server has started on port ${port}`);
});
```

서버를 실행하고 `http://localhost/lookup`으로 접속하면 아래와 같은 json을 반환합니다.

![lookup 화면](/assets/images/20210224/001.jpg)*문제 없이 json result를 반환하네요*

<br>

***

# 2. nodejs로 컴퓨터 조작하기

이번엔 실제로 컴퓨터를 제어하는 코드를 짜보겠습니다.
child_process 모듈을 사용하면 node 상에서 간단하게 콘솔 명령어를 수행할 수 있습니다.

index.js 중간에 아래 내용을 추가합니다.

```js
function run(command) {
    const { exec } = require("child_process");
    exec(command, (error, stdout, stderr) => {
        if (error) {
            console.log(`error: ${error.message}`);
            return;
        }
        if (stderr) {
            console.log(`stderr: ${stderr}`);
            return;
        }
        console.log(`stdout: ${stdout}`);
    });
}

app.get('/sleep', function (req, res, next) {
    var result = { error : 0, message : ""}
    run("rundll32.exe powrprof.dll SetSuspendState");
    res.json(result);
});

app.get('/reboot', function (req, res, next) {
    var result = { error : 0, message : ""}
    run("shutdown /r /f");
    res.json(result);
});

app.get('/shutdown', function (req, res, next) {
    var result = { error : 0, message : ""}
    run("shutdown /s /f");
    res.json(result);
});

```

shutdown.exe로 재부팅과 컴퓨터 종료를 수행합니다.

rundll32.exe는 동적 라이브러리(*.dll)를 로드하고, 그 안의 함수를 실행하는 프로그램입니다.
powerprof.dll 파일의 SetSuspendState() 함수를 호출하면 컴퓨터를 절전 모드로 전환할 수 있습니다.

![shutdown 화면](/assets/images/20210224/002.jpg)*잘 종료된다!*

명령 프롬프트를 열고 `shutdown -a`를 입력하면 진행중인 시스템 종료를 취소할 수 있으니 참고.

<br>

***

# 3. 간단한 사용자 인증 수행하기

현재 사용자를 인증하는 기능이 없기 때문에 누구나 API만 알면 우리집 컴퓨터 끌 수 있다... 
간단하게 비밀번호를 사용해 로그인 기능을 구현해 보겠습니다.

로그인 과정은 아래와 같이 하겠습니다.

1. SHA256 알고리즘으로 비밀번호를 암호화한다.
2. 이를 파라미터로 establishment API를 호출
3. 서버에서 유효성 검사 후 세션 초기화

nodejs의 crypto 모듈을 사용하면 한줄로 암호화를 할 수 있는데, 이상하게도 이렇게 암호화된 값이 다른 곳에서 암호화한 값과 다릅니다. 
이유는 모르겠어요... 별로 알고 싶지도 않네요. 시간이 아까우므로 sha256 암호화 함수가 담긴 자바스크립트 파일을 구해왔습니다.

[sha256.js](/assets/images/20210224/sha256.js)

위 파일을 index.js와 같은 폴더에 복사하고 index.js 파일을 수정합니다.

```js
const app = require('express')();
const sha256 = require('./sha256').SHA256
const session = require('express-session');

app.use(session({
    secret: '임의의 비밀 키',
    resave: false,
    saveUninitialized: true
}));

/**
 * 저장된 비밀번호를 불러오는 함수.
 */
function getKey() {
    var k = `password`;
    return sha256(k);
}

/**
 * 403 Http Response를 설정하는 함수.
 */
function unauthorized(res) {
    res.statusCode = 403
    res.message = "인증 정보가 잘못되었습니다."
    res.json();
}

/**
 * 세션을 확인하고 로그인이 되어있는지 확인하는 함수.
 * 로그인 되어있으면 true를 반환, 아니라면 Response를 403으로 설정하고 false 반환.
 */
function checkLoggedIn(req, res) {
    if(req.session.loggedIn != true) {
        unauthorized(res);
        return false;
    }
    else return true;
}

app.get('/establishment', function (req, res, next) {
    var result = { error : 0, message : ""}
    var key = req.query.key
    if(key === undefined || key != getKey(key)) {
        unauthorized(res);
        return;
    }
    // 로그인 성공했음을 세션에 기록
    req.session.loggedIn = true;
    res.json(result);
});


app.get('/lookup', function (req, res, next) {
    // 가장 먼저 세션 확인
    if(!checkLoggedIn(req, res)) return;

    var result = { error : 0, message : ""}
    res.json(result);
});
```

shutdown 등 다른 api에도 checkLoggedIn() 함수를 써서 제일 먼저 로그인 여부를 검사하도록 합니다!

![403 화면](/assets/images/20210224/003.jpg)*이제 인증 없이 다른 API를 호출하면 권한 업음 오류를 반환한다.*

<br>

***

# 4. 사용자 인증 테스트

위에서 구현한 사용자 인증이 잘 동작하는지 크롬 개발자 콘솔에서 테스트 해보겠습니다.

먼저 콘솔에 sha256.js 파일 내용을 붙여넣습니다.
맨 끝에 module.exports 코드는 빼고 넣어주세요.

![크롬 콘솔 화면](/assets/images/20210224/004.jpg)

SHA256() 함수를 써서 비밀번호를 인코드하고 복사하고, 다음과 같이 establishment API를 호출합니다.

> http://localhost/establishment?key=[암호화된 키값]`

403이 뜨지 않고 연결에 성공하면 이어서 shutdown API를 호출해 컴퓨터를 끌 수 있습니다.

<br>

***

# 마치며

있으면 편할 것 같은데 만들기 너무 귀찮아서 몇 달 동안 미루고 미루던! 컴퓨터 관리 서버를 만들기 시작했습니다.

사실 컴퓨터가 꺼져있으면 관리 서버도 실행되지 않기 때문에 일종의 대시보드 서버에 사용자가 접속하고, 이 서버에서 컴퓨터 관리 서버에 명령을 전달하는 식으로 수정해야 할 것 같습니다.

웹 서비스는 거의 처음 만들어보는거라 생소한 것들 투성이지만 너무 즐겁네요. 다음에는 대시보드 서버를 만들어 보겠습니다.