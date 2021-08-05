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



개발에 앞서 webClient를 사용하기 위해서는 WebFlux에 대해 조금 공부할 필요가 있었다.

### WebFlux



<수정중>







## 참고

https://www.baeldung.com/spring-5-webclient

https://www.baeldung.com/spring-log-webclient-calls

Reactcive programing  : https://brunch.co.kr/@springboot/152 (시리즈 다 읽는것을 추천!)
