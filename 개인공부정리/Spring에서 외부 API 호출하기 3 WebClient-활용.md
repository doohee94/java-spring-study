# Spring에서 외부 API 호출하기 3 WebClient-활용

그.래.서

대-충 webClient를 사용하는 방법은 알았고, 그 후엔 어떻게 하면 효율적으로 사용할 수 있는지 계속 고민을 하였었다.

*<u>~~(효율적이라 쓰고 귀찮아서 어떻게 하면 한번에 처리할 수 있을까 머리 굴린거라고 읽는.. )~~</u>*



대략적인 상황은 이랬었다. 

- 약 20개의 각기 다른 api를 호출.
- 각 Response는 당연히 다르다. (같은 것도 있긴 했다.)
- 외부 api에서 서버에러가 발생하는 경우(http status가 40x, 50x로 오는 경우)  response가 정해진 형식으로 도착한다. 
- 그 외 예상하지 못한 오류가 발생하면 그냥 에러로 넘어온다. 
- 어떤 api는 상태를 기다렸다가 다시 재 조회를 하여 로직을 처리해야한다. 



처음에는 무작정 20개의 api들을 다 따로 처리하였었다.

(원래 프로젝트에서 api 호출 service와 객체를 처리하는 service를 나누기는 했었다.)

```java
@Service
@RequiredArgsConstructor
public class Service{
    
    private final WebClient webClient;
    
    public SampleDto api1(){
         SampleDto sampleDto = webClient.get()
        ...
        .block();
        
        return sampleDto;
    }
    
    public SampleDto api2(){
         SampleDto sampleDto = webClient.get()
        ...
        .block();
        
        return sampleDto;
    }
   
    .
    .
    .
    
}
```



이렇게 되니 반복되는 부분이 눈에 보였고, 뭔가 비효율 적이라는 생각이 들었다. ~~*<u>초보개발자의 한계란.. 꼭 해봐야 비효율적인게 눈에 보인다...ㅠㅠ..</u>*~~ 



그래서 우선, 공통된 부분을 묶어 관리 할 수 있도록  Service에서 api를 호출하고, exchange까지 완료한 후, 공통클래스에 넘겨버렸다.... 

```java
@Service
@RequiredArgsConstructor
public class Service{
    
    private final WebClient webClient;
    private final ObjectMapper mapper;
    
    public CommonDto<SampleDto> getSampleCommonDto() {

    Mono<ClientResponse> exchange = webClient.get()
         .uri(uriBuilder -> uriBuilder
            .path(String.format("/sample/sample2/sample3"))
            .queryParam("x", 2)
            .queryParam("y", 3)
            .build())
        .exchange();
        
    return new CommonDto<>(exchange, SampleDto.class, mapper);

  }
   
    .
    .
    .
    
}
```

CommDto가 응답을 공통적으로 처리하는 클래스이다. 

`CommonDto` 생성자의 첫번째 인자로는 api 호출 부분을, 두번째 인자로는 Response를 받는 Dto class를, 세번째 인자로는 mapper 클래스를 넣어주었다.



```java
public class CommonDto<T> {

	private Object response;
	private Class<T> tClass;
	private ObjectMapper mapper;

	public CommonDto(Mono<ClientResponse> data, Class<T> classType, ObjectMapper mapper) {
		this.tClass = classType;
		this.mapper = mapper;
		this.response = data
			.flatMap(t -> {
				if (t.statusCode().value() == HttpStatus.BAD_REQUEST_400 
					|| t.statusCode().value() == HttpStatus.INTERNAL_SERVER_ERROR_500) { //1

					if (isErrorResponse()) { //2. .. 관련 코드는 따로 없습니다. 로직상 필요해서 넣은 부분 
						return t.bodyToMono(ErrorResponseDto.class);
					}
					return Mono.error(new DataCallFailException()); //3
				}

				return t.bodyToMono(String.class);//4
			})
			.block();
	}

	public T convertStringToObject() throws IOException {
		return this.mapper.readValue((String)response, tClass);
	}

	public boolean isErrorType() {
		return this.response instanceof ErrorResponse.ErrorResponseContainer;
	}
}

```

CommonDto를 생성해 보았다. 

일단 Clascc와 mapper을 초기화 해주고, 응답은 원하는 로직에 맞게 짜주었다. 

내가 진행했던 프로젝트에서는 외부 api에서 응답코드를 주고, 특정 에러인 경우 정해진 형식으로 넘어왔다. 

그래서 특정 api에 특정 에러가 발생한다면  `ErrorResponseDto` 형식으로 바꿔 response를 초기화 해주고, 그렇지 않다면 Exception을 발생시켰다. 



응답이 성공적으로 온다면, 우선 `String ` 으로 변환시켰다. 그 이유는 프로젝트에서 그렇게 필요했기 때문에.. 

그래서 Class와 ObjectMapper를 받아 `convertStringToObject` 메소드를 통해 원하는 객체 형식으로 변환하여 사용하였다. 



```java
@Service
@RequiredArgsConstructor
public class RealUseService{
    
    private final Service service;
    
    public SampleDto getSampleDto() {

    	CommonDto<SampleDto> commonDto =  service.getSampleCommonDto();
        SampleDto sampleDto = commonDto.convertStringToObject();
        
    return sampleDto;

  }
   
    .
    .
    .
    
}
```

이런식으로 api를 호출하는 서비스와 사용하는 서비스를 나눠 사용하였다. 그냥 단순한 호출이 아닌 필요한 로직도 넣고.. 

단순 호출 api의 경우에는 이렇게 처리 했다.



------



대부분이 단순 호출인 경우였지만, api호출 시 상태가 업데이트 되면 그에 따라 필요한 결과를 호출해야하는 경우도 있었다. 

다행히 api에서 예상 대기시간을 주었다. 

예상 대기시간을 이용하여 그 시간동안 기다린 후 상태를 체크하고, 상태가 `완료`라면 로직 수행, 아니라면 다시 기다리는 로직을 짰다. 



어떻게 코드를 짤까 고민하다가, webFlux라면 뭔가 관련된 코드가 있을것이라 생각되어 찾아보았고, 비슷한 코드가 있다는 것을 발견했다. 

*~~WebFlux에 관한 지식이 부족하여 관련 문건을 얼마나 찾아봤는지 모른다..ㅠㅠ...~~* 

거듭된 실패 끝에.. StackOverFlow에서 나와 비슷한 고민을 해결한 사례가 있어 참고하여 코드를 수정하였다.(참고 확인)

```java
  public SampleDto getStatus(long estimateTime) {

    Flux<SampleDto> sampleFlux =  webClient.get()
        .uri("/sample")
        .retrieve()
        .bodyToMono(SampleDto.class)
        .doOnError(t -> {
          throw new DataCallFailException();
        })
        .repeatWhen(longFlux -> Flux.interval(Duration.ofSeconds(estimateTime)))
        .skipUntil(t -> t.getStatus.equal("실패")
            || t.getStatus.equal("에러")
            || t.getStatus.equal("완료"))
        ;
      
      
     return sampleFlux.blockFirst();
  }
```

위에서 응답을 바로 받는 코드들은 Mono를 사용하였는데, 이 로직은 Flux를 사용하였다. 일정 시간동안 여러 응답이 들어오기 때문!

`retirieve`를 통해 `bodyToMono`로 결과 값을 받는건 앞과 동일하다. 

그 이후 `repeatWhen`을 통해 일정 시간동안 요청을 반복한다. 언제까지? `skipUntil`의 조건을 충족할 때 까지.

`skipUntil`은 안에 조건들이 충족될 때까지 모든 요청을 skip한다. 나같은 경우 실패, 에러, 완료의 조건이 충족되기 전 까지는 응답을 받지 않았다. 

조건이 충족됐다면 `blockFirst`를 통해 response를 SampleDto 형식으로 받는 그런 구조였다. 

skipUntil 말고 `take`조건도 있으니 참고자료를 통해 확인하면 좋을 것 같다. 





## 참고

https://stackoverflow.com/questions/54591964/how-to-delay-repeated-webclient-get-request

https://javacan.tistory.com/entry/Reactor-Start-4-tbasic-ransformation





# WebClient 활용 후기



처음 webClient를 사용하려 했을때는 뭐 그냥 쓰면 되겠지..^^ 하고 겁도 없이 덤빈것 같다 ^^^

사용하고 나니 너무 만만히 본 것 같은 느낌..^^^ 특히 WebFlux에 대해 공부하면 공부할수록 더 모르는 사실이 많이 발견되었다. 

사실 뭐.. block를 써도 되는것인지 subscribe로 하면 될거같은데 난 왜 하면 안되는 것인지.. 

내 로직은 왜 이렇게 구린것인지....

공부를 더 많이 했다면 지금보다 훨씬 더 좋은 코드를 짤 수 있었을 텐데.. 그 부분이 조금 아쉽다.

앞으로 공부를 더 해나가서 나중에 프로젝트에 코드 리팩토링을 전체적으로 싹 하는게 목표다... 언제가 될지는 모르겠지만.. 

WebFlux가 점점 대세가 된다고 하니 바짝 공부하는 것도 좋을 것 같다!!!

공부해야지!!!!!