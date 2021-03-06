---
layout: post
title: 마우스와 로봇팔로 주호민 작가님 그리기
subtitle: 마우스와 Canvas를 이용해 G-Code를 생성하고, 로봇팔에 전송하여 주호민 작가님 그림과 사인받기(+주작가님 방문?!)
background: '/img/posts/08.jpg'
comments: true

date:   2020-04-07 12:00:00 
lastmod : 2020-05-25 12:00:00
sitemap :
  changefreq : weekly
  priority : 1.0
---

# 마우스와 로봇팔로 주호민 작가님 그리기

저번 [온라인 사인회 포스팅](https://ductility.github.io/2020/03/21/%EC%98%A8%EB%9D%BC%EC%9D%B8%EC%82%AC%EC%9D%B8%ED%9A%8C/)에 이어 이번에는 **주호민 작가**님 그림과 사인을 받아보기로 했다.   
지난번 포스팅에서는 [LaserGRBL](http://lasergrbl.com/)을 이용해서 이미지를 g-code로 변환했다. 하지만 이 방법에는 큰 단점이 있었는데, 아무리 얇은 선이라도, 흰색 -> 검정색 -> 흰색으로 바뀌는 것이기 때문에 g-code를 생성할 때 두 줄을 생성했다. 그 결과로 지저분해 보이는 그림이 탄생했다.   

![_config.yml]({{ site.baseurl }}/img/posts/200321/이말년 두줄.jpg){: width="100%"}

이런 문제점을 어떻게 고쳐야 할지 고민하고 있던 중, 전자 서명을 쓸 일이 생겨 우연히 [온라인 필기 만들기 서명, 전자 서명 패드](https://www.signaturemaker.in/ko/draw-a-signature-online/) 사이트를 방문했다. 전자 서명을 만들 때 마우스 클릭을 한 채로 드래그를 하면 글씨(그림)가 써진다. 이 때 마우스를 클릭해서 누르는 것은 로봇팔의 펜을 종이에 닿게 하는 것(M3 명령), 마우스를 떼는 것은 펜을 종이에서 떼는 것(M5 명령)으로 대응시킬 수 있을 것이다. 또, 전자서명은 마우스 포인터의 위치를 바탕으로 만드는 것이기 때문에 정확한 좌표를 구해 로봇에 넘겨줄 수 있겠다고 생각했다. 그래서 이 아이디어를 구체화 시키는것에 도전해 보았다.

<br>

## Canvas를 이용해 마우스로 그림 그리기

검색을 통해 [마우스 드래깅으로 그림 그리기](https://kkk-kkk.tistory.com/entry/%EC%98%88%EC%A0%9C-11-11-%EB%A7%88%EC%9A%B0%EC%8A%A4-%EB%93%9C%EB%9E%98%EA%B9%85%EC%9C%BC%EB%A1%9C-%EC%BA%94%EB%B2%84%EC%8A%A4%EC%97%90-%EA%B7%B8%EB%A6%BC-%EA%B7%B8%EB%A6%AC%EA%B8%B0) 에 대해 설명 해 주고 예제를 제공한 
[Hell..o World..!! (너래쟁이)](https://kkk-kkk.tistory.com/)님의 티스토리 페이지를 찾았다. 여기서 제공한 예제를 이용하여 *main.html*라는 파일을 만들어 Canvas객체와 Mouse 이벤트 동작 테스트를 해 보았다. 다음 그림처럼 그림을 그릴 수 있었다.

![_config.yml]({{ site.baseurl }}/img/posts/200407/좌표확인.jpg){: width="100%"}

그림을 그릴 때 Mouse 이벤트의 좌표를 consol.log로 찍어 봤는데 x,y 좌표가 잘 찍히는 것을 확인할 수 있다.   
하지만 이것만으로는 로봇팔을 작동시킬 수 없다. Canvas와 Mouse 이벤트를 이용해 받은 좌표데이터를 이용해 로봇팔에 코드를 전송 해 주어야 하는데, 브라우저에서 직접 시리얼 통신을 하여 g-code를 전송하는것은 복잡하기도 하고 컨트롤러의 역할을 하는 기기와 로봇팔이 꼭 연결되어 있어야 하므로 모바일 기기로의 확장성도 떨어진다. 그래서 [Node.JS](https://nodejs.org/ko/)를 이용해 간단한 웹 서버를 만들고, PC와 모바일에서 웹 페이지에 접근하여 이용할 수 있는 웹 어플리케이션을 만들기로 했다.

<br>

## Node.JS로 간단한 서버 만들기

NodeJs의 내장 모듈인 http모듈을 이용해 앞서 만들었던 *main.html*을 **localhost:8080** 에 접속하면 볼 수 있도록 만들었다. 이 프로젝트에는 단 하나의 페이지만이 사용되기 때문에 굳이 복잡하게 다른 모듈이나 프레임워크를 사용하지 않았다. 이 때 서버를 실행시키는 파일 *server.js*의 실행방법은 매우 쉽다.

```bash
$ node server.js
```
서버가 연결되면 ```Listen on port 8080```이라는 응답을 하도록 만들어 두었고, 접속해 봤더니 잘 작동 했다.   
*main.js*에서 Canvas와 Mouse 이벤트를 이용, 적절한 g-code를 생성하고 "그리기" 버튼을 누르면 서버에서 데이터를 받아 g-code 파일을 생성하게 했다. 이 때는 NodeJs내장 모듈인 fs모듈을 이용했다.

<br>

## gcode-cli로 만들어진 파일 로봇에 전송하기

처음에는 g-code 데이터를 한 줄씩 읽어 시리얼 통신으로 보내려고 했다. 아두이노 IDE를 이용해 한 줄씩 g-code를 보내보니 로봇팔이 정상 작동을 했고, 그래서 NodeJS의 serialport 모듈을 이용하여 g-code 데이터를 전송하려고 생각했다. 하지만 g-code를 시리얼 통신으로 보내고 나면 기기에서 호스트 장비에게 "ok"등의 피드백을 주는데, 내 부족한 실력으로는 그 피드백을 캐치하는것이 힘들었다. 그래서 검색을 통해 [gcode-cli](https://github.com/hzeller/gcode-cli/)라는 CLI환경에서 쉽게 g-code를 전송해 주는 소프트웨어를 찾았다. gcode-cli를 이용해 앞선 단계에서 생성한 파일을 로봇팔에 보내보니 정상적으로 작동했다!

그렇다면 이제 그림과 싸인을 받아보자.

<br>

## 마우스+로봇팔로 그림/싸인 받기
[![_config.yml]({{ site.baseurl }}/img/posts/200407/유튜브썸네일.jpg){: width="100%"}](https://www.youtube.com/watch?v=fTB9tCB1Yvo?t=0s){: target="_blank"}   
*이미지를 누르면 해당 동영상 링크가 새 창에서 열립니다.*

놀랍게도 주호민 작가님이 직접(?!) 방문해서 싸인을 해 주셨다!

![_config.yml]({{ site.baseurl }}/img/posts/200407/주호민_무릎.jpg){: width="100%"}   
사실 내 무릎이다...

...아무튼 정상적으로 잘 작동하는 것을 볼 수 있다.   
그런데 한 가지 아쉬운 점이 생겼다. 마우스로 그리면 아무리 신경을 써도 개떡같은 그림이 나올수 밖에 없었다. 그래서 가지고 있는 아이패드를 활용해 그림을 그려보기로 했다.
[2편_아이패드와_로봇팔로_그림_그리기]()