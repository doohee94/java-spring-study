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

WebClient역시 사용시에 생성해서 사용하는 방법도 있지만, bean으로 등록하는 방법을 사용하였다. 

```java
@Configuration
public class WebClientConfig {

	@Bean
	public ReactorResourceFactory resourceFactory() {
		ReactorResourceFactory factory = new ReactorResourceFactory();
		factory.setUseGlobalResources(false);
		return factory;
	}

	@Bean
	public WebClient webClient(){
		return WebClient.builder()
			.baseUrl("http://localhost:8080/api/")
			.build();
	}

}
```

`baseUrl`부분에 필요한 주소를 넣어준다. 이 외에도 connection time 공통적인 content-type,   logging 설정 등을 추가할 수 있다. (참고자료 두번째 링크)



##### 3. api 호출

```java
@Service
@RequiredArgsConstructor
public class Service{
    
     private final WebClient webClient;
    
    public SampleDto useWebClient(){
        
         SampleDto sampleDto = webClient.get()
        .uri(uriBuilder -> uriBuilder
            .path(String.format("/sample/sample2/sample3"))
            .queryParam("x", 2)
            .queryParam("y", 3)
            .build())
    	.retrieve() 
        .bodyToMono(SampleDto.class)
        .block();
        
        
        return sampleDto;
    }
    
}
```

GET 방식의 예제이다. 이렇게 하면 `http://localhost:8080/api/sample/sample2/sample3?x=2&y=2` 이런 형식의 api를 호출할 것이고, 

그에 따른 응답이 SampleDto에 담기게 된다. 



uri 설정 이후에도 Content-Type이나 header 등등을 설정할 수 있다. 

모두 설정한 후에 Http 응답결과를 가져오기 위해서는 `retrieve` 나 `exchange` 를 쓰게 된다. 

이 둘의 차이점은 바로 Response를 처리하냐, 아님 다른 동작을 더 하느냐 인데, retrieve를 사용하게 되면 `bodyToMono(SampleDto.class)` 와 같이 바로 reponse 를 처리할 수 있다. exchange같은 경우 응답을 받아 다른 가공을 한 후 객체로 처리할 수 있다.

exchange같은 경우 memory leak 가능성 때문에 retrieve 사용을 권고 하고 있다는데.. 나같은 경우 중간에 처리할 작업들이 있어 exchange를 사용하였다. 



POST는 아래와 같은 방식으로 사용한다. 

```java
@Service
@RequiredArgsConstructor
public class Service{
    
    private final WebClient webClient;
    
    public SampleDto useWebClient(){
        
         SampleDto sampleDto = webClient.post()  //post로 변경 
        .uri("/sample") //query parameter이 없으면 단순 String으로 넣어도 무방하다. 
        .body(BodyInserters.fromFormData("id", "admin"))
        .retrieve() 
        .bodyToMono(SampleDto.class)
        .block();
        
        return sampleDto;
    }
    
}
```



Post는 request body에 필요한 정보를 담아가기 때문에 body 부분에 데이터를 넣어주게된다. 

단순 String data를 form-data로 보내는 것이라면 간단히 `BodyInserters.fromFormData().with()...` 를 반복해서 넣어주기만 해도 request가 잘 가게된다. 

Spring 버전이 좀더 높으면 `bodyValue` 도 사용할 수 있는데, 현재 프로젝트에서는 버전이 낮아 사용하지 못하였다...



> 이번 프로젝트에서는 파일 전송 부분을 RestTemplete로 구현을 하였는데, 무슨 이유인지는 모르겠지만 파일을 전송하는 경우 attribute를 찾을 수 없다며 계속 오류가 발생하였다. body에 파일을 보낼 때 key부분에 맞는 값을 잘 넣어줬는데도 불구하고.. 계속...
>
>  개인적으로 spring끼리 전송 테스트를 했을 때는 파일을 잘 주고 받아서 문제 없겠다 싶었는데, webClient와 외부 api 통신의 무엇인가가 잘 안맞았던 것 같다
>
> 개인적으로 이부분이 좀 아쉬웠다... 



##### 4. 에러처리

```java
@Service
@RequiredArgsConstructor
public class Service{
    
    private final WebClient webClient;
    
    public SampleDto useWebClient(){
        
         SampleDto sampleDto = webClient.get()
        .uri(uriBuilder -> uriBuilder
            .path(String.format("/sample/%s/%s", sample2, sample3))
            .queryParam("x", pcaRequest.getScreenWidth())
            .queryParam("y", pcaRequest.getScreenHeight())
            .build())
        .retrieve() 
        .bodyToMono(SampleDto.class)
        .onStatus(status -> status.is4xxClientError() 
                          || status.is5xxServerError()
             , clientResponse ->
                           clientResponse.bodyToMono(String.class)
                           .map(body -> new RuntimeException(body)))
        .block();
        
      
        return sampleDto;
    }
    
}
```

webClient는 각 요청마다 에러처리를 해야한다. `retrieve`라면 위의 예제 코드와 같이 `onStatus` 에 필요한 로직을 넣어준다. (참고자료 맨 밑 링크 참고 )



*exchange의 예외처리는 Spring에서 외부 API 호출하기 3 WebClient-활용편에서 다루겠습니다.* 



## 참고

https://www.baeldung.com/spring-5-webclient

https://www.baeldung.com/spring-log-webclient-calls

Reactcive programing  : https://brunch.co.kr/@springboot/152 (시리즈 다 읽는것을 추천!)

webFlux 참고 : https://devuna.tistory.com/108

Flux와 Mono 참고 : https://devuna.tistory.com/120

webClient 좋은 참고 : https://medium.com/@odysseymoon/spring-webclient-%EC%82%AC%EC%9A%A9%EB%B2%95-5f92d295edc0