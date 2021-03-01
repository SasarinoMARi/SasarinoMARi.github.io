---
layout: post
title:  "Node로 컴퓨터 관리 서버 만들기 - 2"
date:   2021-02-24 20:44:00 +0900
categories: [nodejs, server]
background: '/assets/images/bg-post.jpg'
---

<br>

# 여는 글

[지난 포스트]({% post_url 2021-02-24-SPCMConsole1 %})에서 내 PC를 관리할 수 있는 간단한 서버를 만들어 보았습니다. 
이 서버의 문제점은 PC가 꺼져있을 경우 서버도 오프라인이기 때문에, 컴퓨터를 켜는 WOL 기능을 서버에 구현할 수 없다는 거였죠.

이를 해결하기 위해 서버를 하나 더 만들어 보겠습니다! 
먼저 기존 서버를 원격 서버(Remote Server), 오늘 만들 서버를 콘솔(Console Server)라 하겠습니다.
사용자가 콘솔 서버에 접속하면, 콘솔 서버가 리모트 서버로 요청을 전송해 실제 명령이 전달되는 식으로 만들 거예요.

콘솔 서버는 [heroku](https://www.heroku.com) 무료 서버에 호스팅 할 예정입니다.

<br>

***

# 1. 인증 방식을 토큰으로 변경

지난 시간에 사용자 인증을 세션 방식으로 구현했는데요. 
조금 더 찾아보니 REST API에선 세션을 여는 방식보다는 토큰을 발급하고 이를 사용해 통신하는 방법이 조금 더 올바른 방법이라고 합니다. 
(세션을 여는 것이 REST API의 무상태성 원칙을 깨기 때문이라네요.)

토큰은 원격 서버와 콘솔 서버에서 모두 사용할 거기 때문에 클래스화 시켜 모듈로 만들어 보았습니다.

```js
expirationPeriod = 1000 * 60 * 30; // 30 minute
tokenLength = 256;

class token {
    constructor(){
        // Make random token string
        var t                = '';
        var characters       = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
        var charactersLength = characters.length;
        for ( var i = 0; i < tokenLength; i++ ) {
            t += characters.charAt(Math.floor(Math.random() * charactersLength));
        }
        this.token = t;

        // Set expire date
        this.expireDate = new Date(new Date().getTime() + expirationPeriod)
    }
}

class tokenManager {
    storage = [];

    constructor() {
        this.__removeExpiredToken();
    }

    /**
     * 유효기한 지난 토큰을 리스트에서 삭제
     */
    __removeExpiredToken() {
        setTimeout(() => {
            let d = new Date();
            this.storage = this.storage.filter(function(t){
                return t.expireDate - d > 0
            });
            this.__removeExpiredToken();
          }, 1000 * 60 * 10);
    }

    /**
     * 새 토큰을 발급
     */
    new() {
        let t = new token();
        this.storage.push(t);
        return t.token;
    }

    /**
     * 토큰이 유효한지 확인
     * @param {string} token 확인할 토큰 값
     */
    contains(token) {
        for(var i= 0; i< this.storage.length; i++) {
            let t = this.storage[i];
            if(t.token === token) {
                if(t.expireDate - new Date() > 0) {
                    return true;
                }
            }
        }
        return false;
    }
};

module.exports = new tokenManager();
```

외부에서 new() 메서드를 통해 새 토큰을 발급하고 contains() 메서드를 통해 유효한 토큰인지 확인할 수 있습니다.

먼저 라우터의 establishment api에 해당 코드를 적용합니다.

```js
app.get('/establishment', function (req, res, next) {
    const ip = getIp(req);

    var result = { error : 0, message : ""}
    var key = req.headers.key // Request Header에서 가져오기
    if(key === undefined || !checkKey(key)) {
        console.log(`${ip} : establishment failed`);
        unauthorized(res);
    }
    else {
        console.log(`${ip} : establishment successed`);
        result.token = tokenManager.new(); // 새 토큰 발급!
        res.json(result);
    }
});
```

그리고 또, 저번 포스트에선 키값을 쿼리 파라미터로 전송했었는데요. 
이 방식은 Url에 키 값이 노출되기 때문에 하는 김에 보안을 생각해서 Request Header에 담아 전송하도록 수정했습니다.

만약 establishment가 성공하면 새 토큰을 발급하고 이를 json result에 포함시켜 반환합니다.
연속해서 API를 호출할 때 이 토큰 값을 Request header에 넣어 사용하도록 합시다.


```js
function checkLoggedIn(req, res) {
    let token = req.headers.token; // Request Header에서 가져오기
    console.log("token: " + token);

    if(!tokenManager.contains(token)) { 
        unauthorized(res);
        return false;
    }
    else return true;
}
```

다른 API에서는 위와 같이 토큰이 유효한지 판단합니다.
토큰 값이 없거나 유효하지 않으면 403 오류를 반환하고 있습니다.

<br>

***

# 2. 콘솔 서버에서 원격 서버 조작하기

nodejs에서 다른 서버에 리퀘스트를 보내기 위해서 request 모듈을 사용했습니다.

그런데 이 모듈... 다 만들고서야 안 거지만 얼마 전에 deprecated 되어 이젠 사용하지 않을 거라 하네요.
우선은 잘 돌아가니 놔두고 천천히 고쳐야겠어요.

콘솔 서버에서 원격 서버로 명령을 전달하기 위해 remoteServer 모듈을 만들었습니다.

```js

const request = require('request');
baseUrl = "원격 서버 주소";

class remoteServer {
    establishment(callback) {
        console.log("api : establishment()");

        const key = require("../common/sha256").SHA256(require("../common/secret").key)
        const options = {
            uri: baseUrl + "establishment",
            headers: {
              'key': key
            }
        };

        request.get(options, function (error, response, body) {
            if (error) {
                console.log("error : " + error);
            }
            else {
                const statusCode = response && response.statusCode;
                console.log("statusCode : " + statusCode);

                if (statusCode == 200) {
                    let json = JSON.parse(body);
                    callback.success(json.token);
                }
                else {
                    callback.error();
                }
            }
        });
    };

    lookup(callback) {
        console.log("api : lookup()");

        const options = {
            uri: baseUrl + "lookup"
        };

        request.get(options, function (error, response, body) {
            if (error) {
                console.log("error : " + error);
            }
            else {
                const statusCode = response && response.statusCode;
                console.log("statusCode : " + statusCode);

                if (statusCode == 200) {
                    let json = JSON.parse(body);
                    console.log("json result : " + json);
                    callback.success(json);
                }
                else {
                    callback.error();
                }
            }
        });
    }

    __generalCall(path, callback) {
        this.establishment({success : function(token) {
            console.log("api : " + path);

            const options = {
                uri: baseUrl + path,
                headers: {
                  'token': token
                }
            };
    
            request.get(options, function (error, response, body) {
                if (error) {
                    console.log("error : " + error);
                }
                else {
                    const statusCode = response && response.statusCode;
                    console.log("statusCode : " + statusCode);
    
                    if (statusCode == 200) {
                        let json = JSON.parse(body);
                        callback.success(json.token);
                    }
                    else {
                        callback.error();
                    }
                }
            });
        }, failed: function () {
            callback.error();
        }});
    }

    sleep(callback) {
        this.__generalCall("sleep", callback);
    }
    reboot(callback) {
        this.__generalCall("reboot", callback);
    }
    shutdown(callback) {
        this.__generalCall("shutdown", callback);
    }
    do(callback) {
        this.__generalCall("do", callback);
    }
    logs(callback) {
        this.__generalCall("logs", callback);
    }
};

module.exports = new remoteServer();
```

그리고 라우터쪽 api 코드는 아래와 같이 작성합니다.

```js
app.get('/lookup', function (req, res, next) {
    remoteServer.lookup({
        success : function () {

        },
        error : function () {

        }
    });
    res.json({ error : 0, message : ""});
});
```

remoteServer 모듈로 lookup을 하면 success callback이 호출되기 전에 response를 반환하므로, 여기서는 await를 써서 기다려줬다 진행해야 할 것 같습니다.

먼저 await-request.js 파일에 아래 코드를 작성합니다.

```js
const request = require('request')

module.exports = async (value) => 
    new Promise((resolve, reject) => {
        request.get(value, (error, response, data) => {
            if(error) reject(error)
            else resolve(data)
        })
    })
```

remoteServer.js의 lookup API를 다음과 같이 수정했습니다.

```js
async lookup(callback) {
    logger.d("api : lookup()");

    const options = {
        uri: baseUrl + "lookup",
        timeout: 1000 * 5
    };

    let request = require("./await-request")
    
    var result = null;
    try {
        result = await request(options);
        console.log(result);
    }
    catch (err) {
        console.error(err);
    }

    return result;
```

이러면 서버가 살아있을 경우 result를, 죽어있을 경우 null을 반환하게 됩니다.

마지막으로 라우터의 lookup 부분을 아래와 같이 수정했습니다.

```js
app.get('/lookup', async function(req, res, next) {
    var json = { error : 0, message : ""};

    var result = await remoteServer.lookup();
    if(result === null) json.error = 1
    res.json(json);
});
```

만약 json result에서 error가 1이라면 서버가 오프라인인 것으로 판단할 수 있겠네요!

<br>

***

# 3. Wake On Lan 기능 구현하기

[지난 포스트]({% post_url 2021-02-11-Fiddler_With_AndroidEmulator %})에서 Iptime 공유기의 Wake On Lan 패킷을 스니핑 했습니다.
드디어 이때 알아낸 콜 스택을 활용할 때가 왔네요.

Fiddler가 캡쳐한 패킷에 따르면 checkup > version > hostinfo > login_handler > info > wol_apply 순으로 api가 호출됩니다.
이중에 checkup, version, hostinfo, info는 경우에 따라 생략할 수 있을 것 같지만 일단은 흐름을 그대로 따라가 보겠습니다.

```js
const host = "공유기 관리 페이지 주소";
const iptime_username = '공유기 관리 페이지 아이디'
const iptime_username = '공유기 관리 페이지 비밀번호'

const request = require('request');
const baseUrl = `http://${host}/`;

function checkup() {
    console.log("api : checkup");

    const options = {
        uri: baseUrl + "checkup",
        headers: {
            'Host' : host,
            'Connection': 'Keep-Alive',
            'User-Agent': 'Apache-HttpClient/UNAVAILABLE (java 1.4)'
        }
    };

    request.get(options, function (error, response, body) {
        if (error) {
            console.log("error : " + error);
        }
        else {
            const statusCode = response && response.statusCode;
            if (statusCode == 200) {
                version(body);
            }
            else {
                console.log("statusCode : " + statusCode);
            }
        }
    });
}

function version() {
    console.log("api : version");

    const options = {
        uri: baseUrl + "version",
        headers: {
            'Referer': baseUrl,
            'Host' : host,
            'Connection': 'Keep-Alive',
            'User-Agent': 'Apache-HttpClient/UNAVAILABLE (java 1.4)'
        }
    };

    request.get(options, function (error, response, body) {
        if (error) {
            console.log("error : " + error);
        }
        else {
            const statusCode = response && response.statusCode;
            if (statusCode == 200) {
                hostinfo(body);
            }
            else {
                console.log("statusCode : " + statusCode);
            }
        }
    });
}

function hostinfo() {
    console.log("api : hostinfo");

    const options = {
        uri: baseUrl + "login/hostinfo.cgi",
        headers: {
            'Referer': baseUrl,
            'Host' : host,
            'Connection': 'Keep-Alive',
            'User-Agent': 'Apache-HttpClient/UNAVAILABLE (java 1.4)'
        },
        qs: {
            'act':'auth'
        }
    };

    request.get(options, function (error, response, body) {
        if (error) {
            console.log("error : " + error);
        }
        else {
            const statusCode = response && response.statusCode;
            if (statusCode == 200) {
                login_handler(body);
            }
            else {
                console.log("statusCode : " + statusCode);
            }
        }
    });
}

function login_handler() {
    console.log("api : login_handler");

    const postData = {
        'username': iptime_username,
        'passwd': iptime_password,
        'act': 'session_id'
    };

    const options = {
        uri: baseUrl + "sess-bin/login_handler.cgi",
        headers: {
            'Referer': baseUrl,
            'Host' : host,
            'Connection': 'Keep-Alive',
            'User-Agent': 'Apache-HttpClient/UNAVAILABLE (java 1.4)'
        },
        form: postData,
        json: false
    };

    request.post(options, function (error, response, body) {
        if (error) {
            console.log("error : " + error);
        }
        else {
            const statusCode = response && response.statusCode;
            if (statusCode == 200) {
                info(body);
            }
            else {
                console.log("statusCode : " + statusCode);
            }
        }
    });
}

function info(session) {
    console.log("api : info");

    const options = {
        uri: baseUrl + "sess-bin/info.cgi",
        headers: {
            'Cookie': `efm_session_id=${session}`,
            'Referer': baseUrl,
            'Host' : host,
            'Connection': 'Keep-Alive',
            'User-Agent': 'Apache-HttpClient/UNAVAILABLE (java 1.4)'
        },
        qs: {
            'act':'wol_list'
        }
    };

    request.get(options, function (error, response, body) {
        if (error) {
            console.log("error : " + error);
        }
        else {
            const statusCode = response && response.statusCode;
            if (statusCode == 200) {
                mac = body.split(";")[0];
                wol_apply(session, mac);
            }
            else {
                console.log("statusCode : " + statusCode);
            }           
        }
    });
}

function wol_apply(session, mac) {
    console.log("api : wol_apply : " + mac);

    const options = {
        uri: baseUrl + "sess-bin/wol_apply.cgi",
        headers: {
            'Cookie': `efm_session_id=${session}`,
            'Referer': baseUrl,
            'Host' : host,
            'Connection': 'Keep-Alive',
            'User-Agent': 'Apache-HttpClient/UNAVAILABLE (java 1.4)'
        },
        qs: {
            'act':'wakeup',
            'mac':mac
        }
    };

    request.get(options, function (error, response, body) {
        if (error) {
            console.log("error : " + error);
        }
        else {
            const statusCode = response && response.statusCode;
            if (statusCode == 200) {
                if(body==="Ok") done(body);
                else console.log("body: " + body);
            }
            else {
                console.log("statusCode : " + statusCode);
            }           
        }
    });
}

function done() {
    console.log('done!!');
}

module.exports = {
    wakeup : function () {
        checkup();
    }
}
```

엄청 기네요...

모듈을 임포트하고 wakeup 함수를 호출하면 checkup부터 순차적으로 호출합니다.

마지막으로 라우터에 연결만 시켜주면 끝입니다.

``` js
app.get('/wakeup', function (req, res, next) {
    if(!checkLoggedIn(req, res)) return;

    var result = { error : 0, message : ""}
    require("./iptime-wol").wakeup(); // Wake On Lan!
    res.json(result);
});
```

<br>

***

# 마치며

드디어 완성했네요! 아직 보완할 점이 많지만 굵직한 뼈대는 다 만들었다고 생각하니 후련합니다.
우선 안드로이드 앱과 콘솔 서버에 관리할 수 있는 웹 페이지를 추가해서 사용하고자 합니다.
Wake On Lan이 워낙 연식이 있는 기술이다 보니 중간에 뭐가 어그러지면 원인 찾기가 굉장히 힘드네요 ㅜㅜ 