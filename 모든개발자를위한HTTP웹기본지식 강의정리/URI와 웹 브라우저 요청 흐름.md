# URI

Uniform Resorce Identifier

리소스를 식별하는 통일된 방식

### URL, URN ?

- URL은 리소스가 이 위치에 있음을 지정
- URN은 리소스에 이름을 부여 ex) isbn:1234566 
- URN은 잘 사용되지 않아서 거의 URI = URL

### URL 전체 문법

```text
scheme://[userinfo@]host[:port][/path][?query][#fragment]
ex)
https://www.google.com:443/search?q=hello&hl=ko
```

> - https - 프로토콜
> - www.google.com - 호스트명
> - 443 - 포트번호
> - /search - path
> - q=hello&hl=ko - query string

#### 프로토콜(어떤 방식으로 자원에 접근할 것인가 하는 규칙)사용 ex) http, https, ftp 

- http는 80, https는 443.... 포트는 생략가능
- https는 http에 보안 추가 

#### userinfo

- 거의 사용하지 않음

#### host

- 호스트명
- 도메인명 또는 IP 주소를 직접 사용

#### port

- 접속 포트, 일반적으로 생략

#### path

- 리소스 경로, 계층적 구조 ex) /users/1 

#### query

-  key=value 형태 
- 문자형태로 넘어감
- ?로 시작 &로 추가

#### fragment

- html 내부 북마크 등에 사용
- 서버에 전송하는 정보는 아님 

# 웹 브라우저 요청 흐름

1. 클라이언트 웹 브라우저에서 DNS 조회를 통해 서버 IP를 가져오고, 

   서버쪽에 HTTP 요청 메세지를 생성

   ```text
   GET /search?q=hello&hl=ko HTTP/1.1
   HOST: www.google.com
   ```

2. SOCKET 라이브러리를 통해 전달

   - TCP/IP 연결 (IP, PORT 정보)

   - 데이터 전달

3.  HTTP 메세지 포함 하여 TCP/IP 패킷 생성 

4. 서버에 패킷이 도착하면 원하는 요청을 처리하고 응답메세지를 만듦

   ```text
   HTTP/1.1 200 ok
   Content-Type: text/html;charset=UTF-8
   Content-Length:3423
   
   <html>
   	<body>..</body>
   </html>
   ```

   content type 에는 응답 형식과 언어를 담고 length에는 응답 형식의 길이를 담음 

5. 응답 패킷을 클라이언트에 전송하여 웹 브라우저에서 html을 읽어서 렌더링

   