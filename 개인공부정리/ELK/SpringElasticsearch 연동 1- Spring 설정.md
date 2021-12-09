# SpringElasticsearch 연동 1- Spring 설정

#### 유용한 참고자료

- **elasticsearch java builder 모음**

  https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-java-builders.html

- **spring-data-elasitcsearch VS rest-high-level-client**

  https://elsboo.tistory.com/6





## 기본 설정 

### 의존성 추가

```text
implementation 'org.springframework.boot:spring-boot-starter-data-elasticsearch'
compile 'org.elasticsearch.client:elasticsearch-rest-high-level-client:7.15.0'
compile 'org.elasticsearch.client:elasticsearch-rest-client:7.15.0'
compile 'org.elasticsearch:elasticsearch:7.15.0'
```

elasticsearch버전에 맞춰 의존성을 추가해 준다. 

spring 에서 elasticsearch를 사용할 때, JPA와 같이 spring-data를 사용하느냐, client를 두고 쿼리를 직접 작성하느냐로 갈린다. 결론부터 말하자면 나는 high-leve-client를 이용하였다. 



간편하게 하용하려면 **spring-data**를 사용하여 처리하면 될 것같지만, 버전에 따라 변동이 심하다 하기도 하고.. 내가 수행하려는 쿼리가 간편하진 않아서 사용하진 않았다. 쿼리 결과가 직접 객체에 매핑되는 부분이 아쉽긴 했다.. 

client를 둔다면, low-level이냐, high-level이냐로 갈리게 된다. 

**low-level**의 경우 직접 요청을 만들어서 호출하는 방식인데, 사실 이건 사용해본적이 없어서 잘 모르겠다. 쿼리를 한줄한줄 직접 타이핑하는 방식으로 보인다. 

**high-level**의 경우 쿼리빌더를 생성하여 쿼리를 작성할 수 있다. elasticsearch에서 다양한 쿼리빌더(거의 모두..?)를 제공해주고 있으며, 상황에 맞게 끼워 넣기만하여 편하게 쿼리를 작성할 수 있었다. 

엘라스틱에서는 high-level-client를 밀어주고 있다고 한다. 



### application-elk.yml

```yaml
elasticsearch:
  host: localhost
  port: 9200
  api_key: key

spring:
  profiles:
    active: elk
```

### ElasticsearchProperty.java

```java
@Configuration
@ConfigurationProperties(prefix = "elasticsearch")
@Getter
@Setter
@ToString
public class ElasticsearchProperty {

	private String host;
	private int port;
    private String apiKey;
}

```

편한 설정을 위하여 yml 파일을 만들고 프로퍼티를 생성해주었다. 

host와 port는 elasticsearch가 구동되고 있는 서버IP와 포트번드롤 입력해주면 되고, 

apiKey와 관련한 것은 elk-security 관련 자료를 찾아보면 생성할 수 있다. (`개인공부정리/ELK_Security_설정.md`)



### ElasticsearchConfig.java

```java
@Configuration
@Slf4j
@RequiredArgsConstructor
public class ElasticsearchConfig {

	private final ElasticsearchProperty property;

	@Bean
	public RestHighLevelClient getRestClient() {

		String host = property.getHost();
		int port = property.getPort();
		String apiKey = property.getApiKey();

		HttpHost httpHost = new HttpHost(host, port, "http");

		Header[] headers = new Header[]{new BasicHeader("Authorization", "ApiKey " + apiKey)};


		RestClientBuilder builder = RestClient.builder(httpHost)
			.setRequestConfigCallback(
				requestConfigBuilder -> requestConfigBuilder
					.setConnectTimeout(30000)
					.setSocketTimeout(300000))
			.setHttpClientConfigCallback(
				httpClientBuilder -> httpClientBuilder
					.setConnectionReuseStrategy((response, context) -> true)
					.setKeepAliveStrategy(((response, context) -> 300000))
					.setDefaultHeaders(Arrays.asList(headers))
					.setDefaultIOReactorConfig(IOReactorConfig.custom()
						.setIoThreadCount(4)
						.build())

			);
       
		return new RestHighLevelClient(builder);
	}
}

```

ElasticsearchConfig 파일을 생성하여 RestHighLevelClient Bean을 생성해주어야한다. 

이 설정에서 응답의 timeout이나, security관련 헤더 설정, 쓰레드 설정 등을 해줄 수 있다. 

