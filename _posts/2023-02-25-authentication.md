---
layout: single
title : 인증 기능은 어떻게 구현할 수 있을까?
excerpt : ""
---

![img](/assets/images/authentication%20vs%20%20authorization.png)

## 인증(Authentication)

- 사용자가 누구인지 확인하는 절차이다.
- 아이디와 패스워드를 통해 로그인하는 것

## 인가(Authorization)

- 인증 이후의 프로세스
- 인증을 받은 사용자가 다른 기능을 이용할 수 있도록 허가해준다. (인증된 사용자의 자원 접근 권한 확인 및 허가)

## HTTP의 stateless

웹 사이트의 HTTP 통신 위에서 동작하고 모든 요청과 응답은 stateless한 특성을 가진다. 즉 서버에서 클라이언트의 이전 상태를 기억하고 있지 않는다. 로그인을 통해 인증을 거쳐도 이후 요청에서는 이전의 인증된 상태를 유지하지 않게 된다.

## 인증을 어떻게 구현해야 할까?

## 1. Cookie

- Cookie는 서버에서 사용자 브라우저로 전송하는 작은 데이터로 key : value 형식의 문자열이다.
- 클라이언트가 웹사이트를 방문할 경우, 해당 웹사이트는 서버는 응답 헤더의 `Set-Cookie`에 담아서 Cookie를 클라이언트의 브라우저로 보낸다. 클라이언트의 브라우저는 Cookie 값을 저장한다.
- 이후 클라이언트는 요청을 보낼 때마다, 저장된 Cookie를 요청 헤더에 담아서 보낸다.
- 서버는 Cookie에 담긴 정보를 바탕으로 클라이언트가 누구인지 식별할 수 있다.

- 장점
  - 기존 로그인을 위한 정보만을 사용하므로 인증을 위한 추가적인 데이터 저장이 필요 없다.
  
- 단점
  - 보안에 취약하다. 요청 시 Cookie의 값을 그대로 보내기 때문에 유출 및 조작 당할 위험이 존재한다.
  - Cookie는 용량 제한이 있어 많은 정보를 담을 수 없다.
  - Cookie의 사이즈가 커질수록 네트워크에 부하가 심해진다.
  - 브라우저간 Cookie 공유가 불가능하다.

- 과정
  1. 로그인 요청을 한다.

  2. 로그인 처리를 하고 서버는 로그인 요청에 대한 응답을 작성할 때 Cookie를 담아서 보낸다.
![img2](/assets/images/Cookie1.png)

  3. 클라이언트는 요청을 보낼 때마다 저장된 Cookie를 요청 헤더에 담아 보냄으로써 인증받도록 할 수 있다.
![img3](/assets/images/Cookie2.png)

## 2. Cookie & Session

- Session은 클라이언트의 인증 정보를 Cookie가 아닌 서버에 저장하고 관리한다.
- Session은 사용자의 주요 정보가 아닌, 단지 사용자를 식별할 수 있는 값을 생성해 Cookie로 주고 받는다.

![img4](/assets/images/Cookie3.png)

- 과정
  1. 서버는 클라이언트의 로그인 요청에 대한 응답을 작성할 때, 해당 인증 정보는 서버의 Session 저장소에 저장하고 클라이언트에게는 저장된 Session 정보의 식별자인 JSESSIONID 쿠키를 발급한다.
  2. 클라이언트는 요청을 보낼 때마다 JSESSIONID 쿠키를 함께 보낸다.
  3. 서버는 JSESSIONID 쿠키의 유효성을 판별해 클라이언트를 식별한다.
 
- 장점
  - 클라이언트의 인증 정보가 외부에 노출되지 않는다.
  - 사용자마다 고유한 Session id가 발급되기 때문에 요청이 들어올 때마다 회원정보를 확인할 필요가 없다.

- 단점
  - 해커가 JSESSIONID 쿠키를 사용해 위장할 수 있다.
  - 서버에서 Session을 따로 관리하기 때문에 서버가 여러 개일 경우 로그인 상태 유지에 어려움이 있다.
  - 다중 서버 환경에서 각 서버가 개별적인 Session 저장소를 가질 경우 Session id가 저장되지 않은 곳으로 요청이 가면 클라이언트 식별이 불가능하다.

## 3. JWT(JSON Web Token)

- 인증에 필요한 정보를 암호화시킨 토큰을 의미한다.
- [JWT TIL](https://github.com/dilmah0203/TIL/blob/main/JWT%20Token.md)

<br>

참고

[https://dev.gmarket.com/45](https://dev.gmarket.com/45)
