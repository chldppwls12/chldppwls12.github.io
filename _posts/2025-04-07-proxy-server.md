---
title: Proxy Server(프록시 서버)란?
date: 2025-04-07 10:43:00 +/-TTTT
categories: [Network]
tags: [Proxy Server]
---
## 프록시 서버란?
Proxy는 '대리인', '대리자'를 의미합니다. 프록시 서버는 인터넷 상의 여러 네트워크들에 접속할 때 중계 역할을 해주는 프로그램 또는 컴퓨터를 의미합니다. <br/>


## 프록시 서버 사용 시 이점
### 1. 보안 강화
프록시 서버는 클라이언트와 서버 사이의 중간 계층에서 작동하여 양측을 보호합니다. <br/>
클라이언트의 IP 주소를 숨기고, 악성 요청을 필터링하여 보안을 강화할 수 있습니다. <br/>

### 2. 캐싱을 통한 성능 향상
자주 요청되는 콘텐츠를 프록시 서버에 캐싱하여 동일 요청이 올 경우, 서버에 접속하지 않고 저장된 데이터를 제공합니다.
이를 통해 응답 시간을 단축하고 대상 서버의 부하를 줄일 수 있습니다. <br/>

### 3. 접근 제어 및 콘텐츠 필터링
특정 웹 사이트나 콘텐츠에 대한 접근을 제한하거나 허용할 수 있어, 기업 등에서 인터넷 사용 정책을 시행하는데 활용됩니다. URL 필터링, 콘텐츠 차단, 시간대별 접근 제한 등 다양한 정책을 적용할 수 있습니다.

## 프록시 서버 종류
### Foward Proxy
<img src="/assets/img/proxy-server/forward-proxy-diagram.svg" width="600" alt="Forward Proxy">
- 클라이언트 바로 뒤에 위치합니다.
- 클라이언트가 서버에 접근할 때, 클라이언트는 타겟 서버의 주소를 포워드 프록시에 전달하며, 포워드 프록시가 인터넷으로 요청된 내용을 가져오는 방식입니다.
- 클라이언트의 요청을 프록시 서버가 대신 전달하여 클라이언트의 정보(IP 등)를 서버로부터 숨기거나, 특정 콘텐츠 필터링에 사용됩니다.
- 자주 접근하는 리소스를 로컬 디스크에 저장하여 성능을 향상시킵니다

### Reverse Proxy
<img src="/assets/img/proxy-server/reverse-proxy-diagram.svg" width="600" alt="Reverse Proxy">
- 웹서버/WAS 앞에 위치합니다.
- 내부망에서 프록시 서버와 내부망 서버가 통신하여 인터넷을 통해 요청이 들어오면 프록시 서버가 받아서 응답합니다.
- 여러 서버로 트래픽을 분산시켜 부하를 분산합니다(로드 밸런싱)
- 본래 서버의 IP 주소를 노출시키지 않을 수 있어서, DDos 등의 공격을 막는데 유용합니다.
- 정적 콘텐츠를 캐싱하여 서버 부하를 줄이고 응답 시간을 개선합니다
- 기업의 네트워크 환경에서 DMZ(Demilitarized Zone)라는 내부와 외부 네트워크 사이의 구간에 위치하여 보안을 강화합니다

<details>
<summary>DMZ와 Reverse Proxy</summary>

<img src="/assets/img/proxy-server/dmz-diagram.svg" width="600" alt="Reverse Proxy">
<br/>

보통 기업의 네트워크 환경에서는 DMZ라고 부르는 내부 네트워크/외부 네트워크 사이에 존재하는 구간이 존재합니다.<br/>
일반적으로 메일 서버, 웹 서버, FTP 서버 등 외부에 노출되어야하는 서버가 위치합니다. <br/>
WAS가 DB와 직접 연결되어 있어 WAS가 해킹 당하면 DB까지 해킹 당할 수 있어서, WAS를 DMZ에 직접 두지 않습니다.
대신 리버스 프록시 서버를 DMZ에 두고 WAS는 내부망에 위치시킨 후 서비스하는 것이 일반적입니다.
</details>

### Nginx Reverse Proxy 설정
Nginx에서 Reverse Proxy를 설정해봅니다.
   
```nginx
server {
  listen 80;
  server_name example.com;

  location / {
    proxy_pass http://localhost:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Fowarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Fowarded-Proto $schema;
  }
}
```
{: file="nginx.conf"}

- `server_name`: 요청을 처리할 도메인
- `proxy_pass`: 요청을 전달할 백엔드 서버 주소
- `proxy_set_header`: 백엔드 서버에 전달할 HTTP 헤더 설정
  - `Host`: 원래 호스트 이름 전달
  - `X-Real-IP`: 클라이언트의 실제 IP 주소 전달
  - `X-Forwarded-For`: 프록시 체인을 통과한 클라이언트 IP 목록
  - `X-Forwarded-Proto`: 원래 요청에서 사용된 프로토콜

해당 설정을 하면, 아래와 같은 순서로 진행됩니다.

1. 사용자가 브라우저에서 `http://example.com`으로 접속합니다.
2. 해당 요청은 Nginx 서버(80 포트)로 전달됩니다.
3. Nginx는 이 요청을 `http://localhost:8080`으로 전달합니다.
4. 이 포트에서 실행 중인 WAS가 요청을 처리합니다.
5. WAS가 응답을 생성하면 Nginx가 클라이언트한테 전달합니다.

## 프록시 서버, VPN

프록시 서버와 VPN은 모두 네트워크 트래픽을 중간 서버를 통해 라우팅하는 기술이지만, 구조와 용도에 차이가 있습니다. <br/>
프록시 서버는 데이터 암호화 기능이 없고, 보안보다는 성능 및 접근 제어에 중점을 둡니다. 웹 요청을 중개하고 캐싱하여 성능을 향상시키며, 콘텐츠 필터링과 같은 접근 제어에 유용합니다. <br/>
VPN은 인터넷 트래픽을 암호화하여 사용자의 데이터를 보호합니다. 공공 네트워크에서 안전한 데이터 전송이 필요하거나, 지리적 제한이 있는 콘텐츠에 접근할 때 주로 사용됩니다. 고도의 암호화로 데이터를 보호하고, 모든 네트워크 트래픽을 안전하게 처리합니다.

| 구분          | 프록시 서버                            | VPN                                                   |
| ------------- | -------------------------------------- | ----------------------------------------------------- |
| 익명성        | 제한적(IP 숨김)                        | 강력(전체 트래픽 익명화)                              |
| 데이터 암호화 | 없음                                   | 있음                                                  |
| 속도          | 빠름(캐싱으로 속도 최적화)             | 상대적으로 느림(암호화로 인한 속도 감소)              |
| 보안 수준     | 낮음(기본 보안)                        | 높음(네트워크 및 데이터 보호)                         |
| 활용 사례     | 간단한 IP 숨기기, 웹 필터링, 접근 제어 | 보안이 필요한 상황, 글로벌 콘텐츠 접근, 개인정보 보호 |

---
참고 자료
- [MDN 프록시 서버](https://developer.mozilla.org/ko/docs/Glossary/Proxy_server)
- [[Network] 프록시 서버란? (feat. 필요한 이유) (What is a Proxy server?)](https://fomaios.tistory.com/entry/Network-%ED%94%84%EB%A1%9D%EC%8B%9C-%EC%84%9C%EB%B2%84%EB%9E%80-feat-%ED%95%84%EC%9A%94%ED%95%9C-%EC%9D%B4%EC%9C%A0-What-is-a-Proxy-server)
- [Proxy Server(프록시 서버)란?](https://velog.io/@jangwonyoon/Proxy-Server%ED%94%84%EB%A1%9D%EC%8B%9C-%EC%84%9C%EB%B2%84%EB%9E%80)
- [Reverse Proxy / Forward Proxy 정의 & 차이 정리](https://inpa.tistory.com/entry/NETWORK-%F0%9F%93%A1-Reverse-Proxy-Forward-Proxy-%EC%A0%95%EC%9D%98-%EC%B0%A8%EC%9D%B4-%EC%A0%95%EB%A6%AC)
