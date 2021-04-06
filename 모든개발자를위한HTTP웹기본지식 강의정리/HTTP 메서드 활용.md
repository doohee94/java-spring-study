# 클라이언트에서 서버로 데이터 전송



### 데이터 전달 방식은 크게 2가지

1. 쿼리 파라미터를 통한 데이터 전송
   - GET
   - 주로 정렬 필터, 검색어 ex) /search?q=hello -> hello를 검색
2. 메시지 바디를 통한 데이터 전송
   - POST, PUT, PATCH
   - 회원가입, 상품주분, 리소스 등록, 리소스 변경.. 



### 클라이언트에서 서버로 데이터를 전송하는 상황 4가지

#### 1. 정적 데이터 조회 

```text
GET /static/star.jpg HTTP/1.1
Host: localhost:8080
```

클라이언트에서 star.jpg 정적 이미지 조회를 GET으로 요청하면 서버에서 해당 데이터를 찾아서 응답

- 이미지, 정적 텍스트 문서

- 조회는 GET 사용

- 정적 데이터는 쿼리파라미터 없이 리소스 경로로 단순하게 조회 가능

  

#### 2. 동적 데이터 조회

```text
GET /search?q=hello&hl=ko HTTP/1.1
Host: www.google.com
```

- 주로 검색, 게시판 목록 정렬 필터에 사용
- 조회는 GET 사용, 쿼리파라미터를 사용해 데이터 서버에 전달



#### 3. HTML FORM을 통한 데이터 전송

```html
<form action="/save" method="post">
    <input type="text" name="username"/>
    <input type="text" name="age"/>
    <button type="submit">전송</button>
</form>
```

이렇게 HTML Form 에 담고 POST로 전송하면 웹브라우저에서 자동으로

```text
POST /save HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

username=kim&age=20
```

이렇게 생성하여 서버로 요청을 보낸다. (바디에 변환해서..)



- 만약 FORM에서 method를 GET으로 보낸다면 쿼리파라미터로 생성해서 보냄

```text
GET /save?username=kim&age=20 HTTP/1.1
Host: localhost:8080
```



- HTML FORM에서는 multipart를 이용하여 바이너리 형태의 모든 형식의 파일을 전송할 수 있다. ex) 이미지, 영상 등등등

- HTML FORM은 GET과 POST만 지원



#### 4. HTTP API를 통한 데이터 전송

```text
POST /members HTTP/1.1
Content-Type: application/json

{
	"username":"lee"
	"age":20
}
```

이렇게 json 형식으로 데이터를 보냄



- 서버 to 서버 의 백엔드 시스템 통신
- 아이폰, 안드로이드의 앱 클라이언트 통신
- AJAX 와 같이 Form 전송 대신 웹 클라이언트 통신 혹은 리액트, 뷰와 함께 웹 클라이언트 API 통신
- POST, PUT, PATCH -> 메시지 바디를 통해 데이터 전송 가능
- GET -> 쿼리 파라미터로 데이터 전송
- Content-Type: application/json 을 주로 사용 (HTML, XML 등등 많이 쓰지만 JSON이 거의 표준처럼 사용됨)



# HTTP API 설계 예시

#### 참고하면 좋은 URI 설계 개념

> - 문서
>
>   - 단일 개념(파일 하나, 객체 인스턴스, 데이터베이스)
>
>   - ex) /members/100, /files/star.jpg
>
>     
>
> - 컬렉션
>
>   - 서버가 관리하는 리소스 디렉터리
>
>   - 서버가 리소스의 URI를 생성하고 관리 
>
>   - ex) 회원가입을 하면 서버에서 회원을 생성하고 ID를 부여하는 느낌..
>
>   - 대부분은 컬렉션을 사용한다
>
>     
>
> - 스토어
>
>   - 클라이언트가 관리하는 자원 저장소
>   - 클라이언트가 리소스의 URI를 알고 관리
>   - ex) 파일의 이름을 클라이언트가 알고 넣어줌.. (사실 잘 모르겠음)
>
> 
>
> - 컨트롤러 , 컨트롤 URI
>
>   - 문서 , 컬렉션, 스토어로 해결하기 어려운 추가 프로세스 실행
>
>   - 동사를 사용 
>
>   - ex) html form 은post랑 get만 사용 가능 -> delete를 위해서 /members/1/delete 이런식으로 사용
>
>     ​		이 경우에는 delete가 컨트롤 URI
>
>     

#### 회원관리 시스템으로 알아보는 API 설계(컬렉션)

```text
회원 목록 /members -> GET
회원 등록 /members -> POST
회원 조회 /members/{id} -> GET
회원 수정 /members/{id} -> PATCH, PUT, POST
회원 삭제 /members/{id} -> DELETE
```

#### 파일 관리 시스템으로 알아보는 API 설계(스토어)

```text
파일 목록 /files -> GET
파일 등록 /files/{filename} -> PUT
파일 조회 /files/{filename} -> GET
파일 삭제 /files/{filename} -> DELETE
파일 대량 등록 /files -> POST
```



### POST의 특징

- 클라이언트는 등록될 리소스의 URI를 모른다 (ex 회원 등록 /members -> POST)
- <u>**서버가 새로 등록된 리소스 URI를 생성해 준다.**</u> 



### PUT 의 특징

- <u>**클라이언트가 리소스의 URI를 알고 있어야 한다.**</u> (ex 파일 등록 /files/{filename} -> PUT)
- 클라이언트가 직접 리소스의 URI를 지정한다 -> 클라이언트가 리소스를 알고 직접 관리



### HTML FORM 사용시 컨트롤 URI 사용

- HTML FORM은 GET과 POST만 사용할 수 있어 사용에 제약이 있다

- 이런 제약을 해결하기 위해 동사로 된 리소스 경로 사용 

- ex) /new, /edit, /delete 

- HTTP 메서드로 해결하기 애매한 경우에도 사용한다. ex) /delivery

  