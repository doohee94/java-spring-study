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



그래서 우선, 공통된 부분을 묶어 관리하기로 하였다. 

```java

public class CommonDto<T> {

	private Object response;
	private Class<T> tClass;
	private ObjectMapper mapper;

	public AnalysisCommonDto(Mono<ClientResponse> data, Class<T> classType, ObjectMapper mapper) {
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





