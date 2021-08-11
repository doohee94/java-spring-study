# Spring에서 외부 API 호출하기 2 (Webclient)

spring 5 이상에서는 webclient를 지향한다 하여 프로젝트에서는 webclient를 주로 사용하였다. 

~~*원래는 전체적으로 다 쓰고 싶었지만 안되는걸 어떡하나요*~~ 

webclient를 사용하면서 약간 힘든점이 있었다면 webclient는 <u>webFlux를 사용한다는 것</u>이었다. 

reactive programming은 너무 미지의 세계였고.. 공부할 것도 많고.. 여러모로 장벽이 느껴졌지만 천천히 공부해 가면서 프로젝트를 진행해보았다. *~~(천천히 해도 됐을지는 모르겠다..^^.. )~~*



## WebClient

WebClient에 관한 설명이다.

> Simply put, *WebClient* is an interface representing the main entry point for performing web requests.
>
> It was created as part of the Spring Web Reactive module, and will be replacing the classic *RestTemplate* in these scenarios. In addition, the new client is a reactive, non-blocking solution that works over the HTTP/1.1 protocol.
>
> It's important to note that even though it is, in fact, a non-blocking client and it belongs to the *spring-webflux* library, the solution offers support for both synchronous and asynchronous operations, making it suitable also for applications running on a Servlet Stack.
>
> This can be achieved by blocking the operation to obtain the result. Of course, this practice is not suggested if we're working on a Reactive Stack.

짧은 영어실력으로 간단히 정리하자면,

> webClient는 웹 요청을 수행하기 위한 진입점을 나타내는 인터페이스이다.
>
> spring web reactive 모듈의 일부로 생성되었으며, restTemplate를 대체한다.
>
> HTTP/1.1 프로토콜을 통해 작동하는 반응형(reactive), 논블로킹 솔루션이다
>
> 논블로킹, 반응형이지만 동기, 비동기를 모두 지원하여 서블릿 애플리케이션에도 적합하다
>
> 하지만 반응형에는 권장하지 않는다.

webClient는 

webFulx는 보통 반응형 프로젝트에서 사용하기 때문에 방문했던 블로그나 자료마다  'block'을 굉장히 지양하는 모습이 보였다. 'block'을 할 경우에는 reactive의 장점을 살리지 못하고, 성능에도 영향을 미친다고 한다. 

하지만 그건 reactive 프로젝트일 경우의 문제.. 현재 진행하고 있는 프로젝트는 반응형 프로젝트가 아닐 뿐더러, 외부 api를 가져와서 가공을 해야했기 때문에 'block'은 필수였다. 



개발에 앞서 webClient를 사용하기 위해서는 WebFlux에 대해 조금 공부할 필요가 있었다..

그래서 정말 필요한 개념만 읽고 넘어갔었다. 

### WebFlux

>  reactive 스타일의 어플리케이션 개발을 도와주는 모듈
>
>  reactive-stack web framework이며 non-blocking에 reactive stream을 지원

더 짧게 정리하자면, Spring에서 reactive-programming을 도와주는 모듈이다. 

*(MVC와의 차이를 보기 위해 참고자료를 보면 더 좋습니다. )*



#### 주요객체

Spring webFlux에서 사용하는 recative library는 Reactor라고 하고, Reactor은 Reactive Streams의 구현체이다. 

Reactor의 주요객체에는 Flux와 Mono가 있다. 

- Mono : 0~1개의 데이터 전달
- Flux : 0~N개의 데이터 전달 



처음 Flux와 Mono를 보았을때는 이게 뭔소리인가 싶었다. 데이터는 무조건 1개 아닌가..? N개는 무슨소리지..?

그래서 나는 뭘 써야 하지?

나같은 경우는 외부 API를 호출할 때 사용하는거라 대부분의 경우 Mono를 사용하였다.

Flux를 쓴 경우는 딱 한 경우, 일정 시간에 맞춰 반복해서 데이터를 호출해야하는 경우가 있었는데 그 때 Flux를 사용하였었다. (자세한 코드는 밑에..)



#### 동기? 비동기? blocking? non-blocking?

이 주제에 관해서는 따로 정리를 해야될것같다. 여기서는 간단히 만 정리하자면, 

- 동기 : 호출된 함수의 수행결과를 호출한 함수가 관리
- 비동기 : 호출된 함수의 수행결과를 본인만 관리
- blocking : 호출된 함수가 자신이 할 일을 마칠때 까지 제어권을 가지고 호출한 함수에게 안돌려줌
- non-blocking : 호출된 함수가 일을 안끝내고 제어권을 호출한 함수에게 넘겨줌



역시, 봐도봐도 모르겠다.

 동기 = 블로킹.. 이런식으로 설명한 블로그들도 있긴 하지만, 저 네개의 상태를 따로 봐야하는게 맞는 것 같다. 

이 블로그에서 설명했던 것이 제일 와닿았던 설명이었다. 

(https://musma.github.io/2019/04/17/blocking-and-synchronous.html)





*그래서, webClient는 언제쓰는건데,*

*외부 api 불러올때...*

*어떻게 쓰는건데..*

# WebClient 사용하기

##### 1. gradle

```java
  compile 'org.springframework.boot:spring-boot-starter-webflux'
```

gradle에 이 한 줄만 추가해주면 된다. 



##### 2. config







##### 3. api 호출

##### 4. 에러처리







## 참고

https://www.baeldung.com/spring-5-webclient

https://www.baeldung.com/spring-log-webclient-calls

Reactcive programing  : https://brunch.co.kr/@springboot/152 (시리즈 다 읽는것을 추천!)

webFlux 참고 : https://devuna.tistory.com/108

Flux와 Mono 참고 : https://devuna.tistory.com/120
