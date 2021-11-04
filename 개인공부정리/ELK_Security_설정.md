# ELK Security 설정

#### 유용한 참고자료

security 적용 기본(username + password) : https://ddoniblog.tistory.com/33?category=867457

security api key 적용 : https://ddoniblog.tistory.com/34?category=867457



<hr/>

elasticsearch 무료버전인 basic license에서 security 기능은 api-key 관리 기능까지 제공을 한다.

참고(https://www.elastic.co/kr/subscriptions)

이에 맞는 elk security 설정을 정리해 보았다. 



## 1. config 파일 설정



elk를 설치한 후에는 자동으로 `elastic`이라는 username이 생성되는 것 같다. 

`elasitc`은 **`superuser`** 권한으로 모든 인덱스 및 데이터를 포함하여 클러스터에 대한 전체 액세스 권한을 부여한다. (참고:https://www.elastic.co/guide/en/elasticsearch/reference/7.15/built-in-roles.html)

logstash나 kibana를 설정할 때, 다른 user를 만들어 설정할 수 있을 것 같지만 우선 `elastic`을 사용하여 설정하였다. 

### 1-1. docker-compose.yml 파일 설정

```text
$vi {PATH}/docker-elk/docker-compose.yml
```

```yaml
..
 elasticsearch:
   ...
    environment:
     ...
      ELASTIC_PASSWORD: password 
..
```

`ELASTIC_PASSWORD`에 기존에 있던 password를 지우고 원하는 password를 설정한다. 

이 password는 `elastic` username의 password이다. 



### 1-2. elasitcsearch.yml 파일 설정

```text
$ vi {PATH}/doceker-elk/elasticserch/config/elasitcsearch.yml 
```

기존 파일에 security 설정 추가

```yaml
#security
xpack.license.self_generated.type: basic
xpack.security.enabled: true
xpack.monitoring.collection.enabled: true
xpack.security.authc.api_key.enabled: true #apiKey 사용설정.
```



### 1-3. logstash.yml 파일 설정

```text
$ vi {PATH}/doceker-elk/logstash/config/logstash.yml 
```

기존 파일에 security 설정 추가

```yaml
#security
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.username: elastic
xpack.monitoring.elasticsearch.password: password
```



### 1-4. kibana.yml 파일 설정

```text
$ vi {PATH}/doceker-elk/kibana/config/kibana.yml 
```

기존 파일에 security 설정 추가

```yaml
#security
elasticsearch.username: elastic
elasticsearch.password: password
xpack.security.encryptionKey: "elasticsearch_security_key__char"
xpack.security.sessionTimeout: 600000
monitoring.ui.container.elasticsearch.enabled: true
```



### 1-5. logstash pipeline 수정

logstash에서 사용중인 conf 파일들에 elasticsearch를 사용하는 부분이 있다면  user와 password를 넣어주어야한다. 

나같은 경우, output 부분에서 사용하였는데,  아래와 같은 식으로 설정을 해 주었다. 

```text
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "my_index"
    document_id => "%{id}"
    doc_as_upsert => true
    user => "elastic"
    password => "password"
 }
}
```



##### 위 파일들의 설정이 끝나면 docker-compose를 재시작 시킨다. 



## 2. Security 설정 확인

security 설정이 완료된 후 9200 포트나, kibana로 접속하게 되면 username과 password를 입력하는 화면이 나온다. 

```text
username : elastic
password : 설정값
```

위에서 설정했던 값으로 입력해서 접속하여 원하는 상태가 나오는지 확인한다. 

user가 elastic이라면 최고권한이기 때문에 아무런 장애없이 출력되어야 정상이다. 



### 3. API key 발급

```text
Kibana 접속 -> 메뉴 -> Management -> Stack Management -> Security -> API keys

1. 오른쪽 상단에 Create API Key 클릭 후 User 확인
2. 원하는 name 입력
3. 권한, 만료시간, metadata 포함여부 체크
4. 생성
5. 생성된 후 화면에 보이는 api key 복사 (base64형식과 json 형식 모두 저장하는 것을 추천)
```



elasitcsearch에서 제공하는 기본 key 형식은 아래와 같다. 

```json
{"id":"idididididididi","name":"key-name","api_key":"keykeykeykeykey"}
```

이를 이용해 key를 사용하려면 base64로 변환하여야 하는데, id와 api_key사이에 `:`를 넣고 base64로 인코딩시키면 된다. 

키바나를 이용해서 발급받은 경우 화면에 두 형식 다 출력되기 때문에 이런 과정이 필요하지 않지만, elascticsearch에 직접 요청을 하는 경우 응답이 위와같은 json으로 나와 인코딩을 시켜 key를 사용하여야 한다. 



## 4. API Key 사용

elasticserch 에 쿼리를 요청할 때, 헤더에 아래와 같이 포함시켜준다. 

`Authorization: ApiKey {인코딩된 key}`

spring에서 Elasticsearch RestClient를 설정할 경우도 마찬가지로 헤더에 포함시켜 준다. 

`ApiKey  ` 를 쓰고 한칸 띄워주는 것을 잊지 말자.. 

```java
Header[] headers = new Header[]{new BasicHeader("Authorization", "ApiKey " + apiKey)};
RestClientBuilder builder = RestClient.builder(httpHost)
		.setHttpClientConfigCallback(
			httpClientBuilder -> httpClientBuilder
				.setDefaultHeaders(Arrays.asList(headers))
				)

		);

```





## 5. 기타 설정사항

### 1. api key를 사용하지 않고 username과 password만 사용할 경우

참고 :https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/_basic_authentication.html

```java
BasicCredentialsProvider provider = new BasicCredentialsProvider();
		provider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials("elastic","password"));

RestClientBuilder builder = RestClient.builder(httpHost)
		.setHttpClientConfigCallback(
			httpClientBuilder -> httpClientBuilder
				.setDefaultCredentialsProvider(provider)
			);
```

provider를 설정해서 restClient 설정을 수정한다. 



### 2. user를 변경하고 싶은 경우

1. kibana에 superuser 권한으로 접속
2. Kibana 접속 -> 메뉴 -> Management -> Stack Management -> Security -> Roles
3. Create Role 
   1. `role name`에 원하는 name 입력 (api key 생성을 위한 role이라 manage_api_key 등 추천)
   2. `Elasticsearch -> Cluster privileges` 에 `manage_api_key`, `manage_security` 추가
   3. `indices`에 `metrics-*` 추가 
   4. `privileges`에 `manage`추가
4. 유저 생성
   1. username과 password 입력
   2. Roles에 원하는 role 추가 
      - health 체크를 위한 `apm_system`
      - query 요청 및 응답을 위한 `editor`
      - api key 생성을 위한 `manage_api_key` (위에서 만든 role)
   3. create 
5. 로그아웃 후 생성한 user로 접속 -> api key 생성