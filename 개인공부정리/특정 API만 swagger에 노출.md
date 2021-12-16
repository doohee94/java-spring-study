# 특정 API만 Swagger에 노출

현재 프로젝트에 개발 서버가 2대가 있는데, 이 중 한 서버에서는 특정 API 만 swagger에 노출 해야 하는 문제가 발생하였다. 

swagger를 설정 할 때, basepackage를 지정해서 사용하는 방법도 있지만, 그럼 컨트롤러에 있는 모든 API가 노출되기도 하고...별로인 부분이 있어  그보다 좀더 커스텀하게..? 사용 할 수 있는 방법을 찾아 적용해 보았다. 



## 1. Profile 설정

아래와 같이 profile을 만들고 각 서버에 맞는 profile에 include 시켜주었다. 

(파일을 만들지 않고 바로 넣는것도 가능한...!)

특정 API만 노출해야하는 서버라면 true,  전체 노출 하는 서버라면 false로 지정해준다. 

##### application-dev-open.yml

```yaml
swagger:
  is-open: true 
```

##### application-dev2-open.yml

``` yaml
swagger:
  is-open: false
```

만약 서버를 1대만 운영한다면, 굳이 파일을 만들지 않아도 된다. 



profile을 다 만들었다면,  아래와 같이 config 클래스를 만들어 준다. 

```java
@Configuration
@ConfigurationProperties(prefix = "swagger")
@Getter
@Setter
@ToString
public class SwaggerProfileConfig {

	private Boolean isOpen;

}
```



## 2. Annotation 생성

open 할 API를 구분해 주기 위하여 어노테이션을 생성해주었다. 

특별한 기능은 없고, 그냥 구분용이다.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface OpenApi {
}
```



## 3. Swagger Config 수정

swagger에서 api 출력을 설정하는 부분은 `Docket` 이다. 

SwaggerConfig에 1 부분에서 생성한 SwaggerProfileConfig를 선언해 준후, Docket 부분을 수정한다. 

Docket에 아래와 같이 apis 부분에 핸들러만 설정해 주면 된다. 

```java
	Docket docket = new Docket(DocumentationType.SWAGGER_2)
			...기존 설정
			.apis(RequestHandlerSelectors.basePackage("내가 설정한 basepackage"))
			.apis(handler -> {
				if (Boolean.TRUE.equals(profileConfig.getIsOpen())) { 
					return handler.isAnnotatedWith(OpenApi.class);
				}
				return true;
			})
        	...나머지 설정
			;
```

간략히 설명하자면, 

basepackage내부의 모든 API를 출력할건데, isOpen이라면, OpenAPi 어노테이션이 붙은 API만 출력한다. 

만약 아니라면 전체 다 출력한다. 



## 4. 컨트롤러 부분

```java
@RestController
@RequiredArgsConstructor
public class SampleController {

	private final SampleService sampleService;

	@OpenApi
	@GetMapping("/sample")
	@ResponseStatus(HttpStatus.OK)
	public SampleDto findSample() {
		return sampleService.findSample();
	}
    
	@GetMapping("/sample2")
	@ResponseStatus(HttpStatus.OK)
	public SampleDto findSample2() {
		return sampleService.findSample();
	}
}
```

위와 같이 컨트롤러가 구성되어있다고 했을 때,  isOpen이 ture라면, `/sample` API 만 swagger에 노출이 되고, 

isOpen이 false라면 두 API 모두 출력되는 것을 확인할 수 있다. 