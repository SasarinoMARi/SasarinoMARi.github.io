---
layout: post
title:  "[NodeJS] 따라쟁이 트위터 계정 만들기"
date:   2021-03-29 21:59:00 +0900
categories: [nodejs, twitter]
background: '/assets/images/bg-post.jpg'
---

저는 토이 프로젝트로 `트윗지기`라는 트위터 관리 앱을 만들고 있습니다. <br>
이 앱에는 본인이 쓴 예전 글을 한번에 정리해주는 트윗 청소기 기능이 있는데, 아무래도 돌이킬 수 없는 민감한 기능이다 보니 테스트는데 어려움이 많았습니다. <br>


테스트용 계정을 하나 만들고, 제가 쓴 트윗을 따라서 트윗하는 따라쟁이 봇을 만들었습니다. <br>
라즈베리 서버에 세팅하기까지 약 1시간 30분 걸리는 간단한 프로그램입니다.

<br>

***

# 1. 스크립트 작성하기

프로젝트 폴더에 터미널을 열고 아래 명령을 입력합니다.

```sh
npm install twitter
```

해당 모듈에 관한 자세한 내용은 [Github](https://github.com/desmondmorris/node-twitter)에서 확인할 수 있습니다.

<br>

***

index.js 파일을 만들고 코드를 입력합니다.

```js
const fs = require('fs');
const Twitter = require('twitter');

/**
 * 트위터 클라이언트입니다.
 * 인증부를 구현하는건 번거롭기 때문에 타 클라이언트에서 미리 토큰을 발급받아서 사용합니다.
 */
var client = new Twitter({
    consumer_key: '71g4xTdMwP81t4********',
    consumer_secret: 'ZWOBhN21QrVPWV6B7iD8TUvPGgIBhBoNXH****************',
    access_token_key: '13764968833********-tgxu2o4UU2wV7t****************',
    access_token_secret: '7lxhYF8F6dhqZxoYlcEjkun6IzpAN****************'
});

/**
 * 트윗 작성 코드
 */
function postTweet(content) {
    client.post('statuses/update', { status: content }, function (error, tweet, response) {
        if (!error) {
            console.log("tweet success: " + content);
        } else {
            console.log(error);
        }
    });
}

const ltFilename = 'lastest_tweets';        // 최근 트윗을 저장할 파일명
const mention_regexp = /@[A-Za-z0-9_]*/g;   // 멘션을 걸러낼 정규식

/**
 * 최근 트윗을 불러오고 따라 트윗하는 코드
 */
function copyLaststTweets() {
    /**
     * 유저 타임라인 불러오기
     */
    client.get('statuses/user_timeline', 
               { screen_name: 'SasarinoMARi', count: 20 }, 
               function (error, tweets, response) {
        if (error) {
            console.log(error);
            return;
        }

        /**
         * 이전 루프에서 이미 따라한 트윗 중복 트윗하지 않게 처리
         */
        var lastest_tweets = [];
        if(fs.existsSync(ltFilename)) 
            lastest_tweets = fs.readFileSync(ltFilename, 'utf8').split(',');

        var new_tweets = []
        tweets.forEach(tweet => {
            let id = String(tweet.id)
            new_tweets.push(id);
            if(lastest_tweets.includes(id)) return;

            /**
             * 멘션이 포함된 트윗을 따라하면 다른 유저분들께 민폐이므로
             * @로 시작하는 아이디를 제거하고 트윗할 수 있도록 한다.
             */
            let t = tweet.text.replace(mention_regexp, '').trim();
            postTweet(t);
        });

        /**
         * 이번 루프에서 따라한 트윗을 파일로 저장
         */
	      fs.writeFileSync(ltFilename, new_tweets.join(','), 'utf8');
    });
}

copyLaststTweets();
```

setInterval() 함수를 이요해 루프를 처리해도 되지만 <br>
라즈베리파이 위에서 24시간 돌릴 거기 때문에 crontab으로 주기적으로 호출하도록 하겠습니다.

<br>

*** 

# 2. crontab 설정하기

루트 권한으로 crontab 에디터를 엽니다.

```
sudo crontab -e
```

그리고 가장 아랫줄에 아래와 같이 추가했습니다.

```
0,20,40 * * * * node /home/pi/Curiosity/index.js
```

이러면 20분 주기로 스크립트가 동작하는걸 확인할 수 있습니다.

<br>

***

# 마치며

역시 스크립트 언어가 생산성이 좋긴 하네요! <br>
생각한걸 바로바로 만들 수 있다는 점이 정말 매력적인 것 같아요.<br>

개발 도중 정규식을 잘못 짠 상태로 프로그램을 실행하는 바람에 일부 팔로워들에게 테스트 계정으로 멘션이 가버려서 너무 죄송스러웠습니다 ㅠㅠ... <br>
항상 테스트 하기 전에 위험한 부분은 없는지 충분한 검토를 거친 뒤에 진행해야 하는데 매번 실수하고 나서 아차하는 것 같아 마음이 아픕니다.