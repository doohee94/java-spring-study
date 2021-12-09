# Spring에서 외부 API 호출하기 1 (RestTemplate)

프로젝트에서 외부 api를 이용하여 로직을 짜는 부분을 맡게 되었다. 

Spring에서 외부 api를 호출하기 위해서는 RestTemplate이나 Webclient를 사용한다고 하여 관련 자료를 찾아보고 적용해 보았다. 

사실 처음에는 webClient만을 사용하다가, 안되는 기능이 있어 restTemplate도 같이 사용하게 되었다... 

프로젝트를 진행하면서 새롭게 알게 된 사실이나 정리할 부분이 많아 이를 한번 정리해 보았다. 

## RestTemplate

restTemplate은 Spring 3부터 지원된 api로, api를 호출한 후 응답을 받을 때 까지 기다리는 **동기 방식**이다. 

spring5 버전부터는 restTemplate보다 webClient를 사용하라고 권고 하고 있다. 

> **NOTE:** As of 5.0 this class is in maintenance mode, with only minor requests for changes and bugs to be accepted going forward. Please, consider using the `org.springframework.web.reactive.client.WebClient` which has a more modern API and supports sync, async, and streaming scenarios.
>
> 참고 :  5.0 현재 이 클래스는 유지관리 모드에 있으며, 앞으로는 변경 및 버그에 대한 사소한 요청만 수락 될 예정입니다.  동기, 비동기 스트리밍 시나리오를 지원한 최신 API WebClient를 사용할 것을 고려하십시오



#### 사용 방법

##### 1. gradle

spring boot를 사용한다면 따로  설정해주지 않아도 된다. 만약, boot가 아니라면 `spring-webmvc` 를 의존성에 추가해야 한다. 



##### 2. config

RestTemplate를 사용할 때마다 객체를 생성해서 사용할 수 도 있지만, bean으로 등록하여 사용하는 방법을 택하였다. 

```java
@Configuration
@Slf4j
@RequiredArgsConstructor
public class RestTemplateConfig {

	private final VCFConfiguration vcfConfiguration;

	@Bean
	public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
		return restTemplateBuilder
			.requestFactory(() -> new BufferingClientHttpRequestFactory(new SimpleClientHttpRequestFactory()))
			.setConnectTimeout(Duration.ofMillis(5000)) // connection-timeout
			.setReadTimeout(Duration.ofMillis(5000)) // read-timeout
			.additionalMessageConverters(new StringHttpMessageConverter(Charset.forName("UTF-8")))
			.rootUri(String.format("http://address:port/api"))
			.errorHandler(new RestTemplateResponseErrorHandler())
			.build();
	}

}

```

rootUri부분에 호출 할 api의 주소를 넣어주면 된다. api 주소의 공통 부분까지 적어주면 편하다. 



에러가 발생할 경우, 에러 응답이 body로 넘어오게 되는데,  어떤 에러를 받을 것 인지, 받고 난 후 어떻게 처리할 것 인지를 따로 정의할 수 있다.

이를 errorHandler에 넣어준다.  (참고에 있는 블로그 참고)

내가 진행하고 있는 프로젝트에서는 에러 응답이 정해져 있어, 정해진 에러 코드만을 받아서 처리했다. 

```java
public class RestTemplateResponseErrorHandler implements ResponseErrorHandler {

	@Override
	public boolean hasError(ClientHttpResponse response) throws IOException {
		return response.getStatusCode() == HttpStatus.BAD_REQUEST
			|| response.getStatusCode() == HttpStatus.INTERNAL_SERVER_ERROR;
	}

	@Override
	public void handleError(ClientHttpResponse response) throws IOException {
		
	}
}

```



##### 3. api 호출

```java
@Service
@RequiredArgsConstructor
public class Service{
    private final RestTemplate restTemplate;
    
    public SampleDto useResTemplate(){
        
        HttpHeaders headers = new HttpHeaders();

		MultiValueMap<String, Object> map = new LinkedMultiValueMap<>();
		map.add("data1", "test");
		map.add("file", file);

		HttpEntity<MultiValueMap<String, Object>> request = new HttpEntity<>(map, headers);
        
        ResponseEntity<SampleDto> response = restTemplate.exchange(
        	"/sample",
            HttpMethod.POST,
            request,
            SampleDto.class
        );
        
        SampleDto sampleDto = response.getBody();
        
        return sampleDto;
    }
    
}


```

위의 코드는 POST요청을 위한 코드로 GET도 비슷하게 진행된다. 

*(GET과의 차이점은 get에는 request를 넣지 않는다. http 규격상 get요청에는 request body를 넣지 않기 때문..)*

 

`MultiValueMap<String, Object>` 을 이용하면 , formData를 넣는 것처럼 다양한 객체를 request body에 담을 수 있다. 이를 `HttpEntity<MultiValueMap<String, Object>>`에 담은 후, request로 보내면 된다. 



요청의 마지막`SampleDto.class` 를 이용해 response body의 형식을 정할 수 있다.

 responseBody로 가져오는 형식과 내가 만든 객체의 property가 다르다면 `@Jsonproperty`를 이용해 매핑 시킬 수 있다.

```java
@Data
public class SampleDto{
    
  @JsonProperty("test_value")
  private Integer testValue;
  @JsonProperty("readValue")
  private Integer readValue;
  @JsonProperty("number")
  private Integer strNumber;
   
}
```

 이후 `  SampleDto sampleDto = response.getBody()` 과 같이 객체로 변환 시킬 수 있다.

이때, 주의점은 매핑시킨 객체에 `setter`와  `기본생성자`가 있어야 한다는 것이다. 보통 response body는 json 형식으로 넘어오게 되는데, 이를 객체로 매핑 시킬 때 setter와 기본 생성자를 사용하여 매핑을 시켜주기 때문이다. 



##### 4. 에러처리

나 같은 경우 비즈니스 로직 상에서 에러를 잡아 처리할 일이 있어 핸들러에서 따로 에러를 처리하지 않았었다. 

RestTemplateResponseErrorHandler의 hasError 에서 에러를 잡아낸 후, 로직 상에서 다시 상태 코드를 받아 필요한 작업을 처리하였다. 

```java
.
.
 ResponseEntity<SampleDto> response = restTemplate.exchange(
        	"/sample",
            HttpMethod.POST,
            request,
            SampleDto.class
        );

if (response.getStatusCode() == HttpStatus.BAD_REQUEST
			|| response.getStatusCode() == HttpStatus.INTERNAL_SERVER_ERROR) {

		//필요한 로직 처리 
		}

.
.
.
```

이런 특수한(?) 경우가 아니라면 핸들러에서 Exception을 발생 시키는게 낫다고 생각한다..



### 참고

RestTemplate 설명 : https://advenoh.tistory.com/46

RestTemplate error Handler 관련 : https://dev-kimse9450.tistory.com/19
