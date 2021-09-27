# DOCKER에서 ELK 설치 및 사용하기

### 유용한 참고자료들

Elastic 가이드북(한글) : https://esbook.kimjmin.net/

centos에 docker 설치(docker-docs): https://docs.docker.com/engine/install/centos/

linux에 docker-compse 설치 : https://docs.docker.com/compose/install/#install-compose-on-linux-systems

docker-conpose 명령어 정리 : https://nirsa.tistory.com/81

dokcer-elk git : https://github.com/deviantony/docker-elk

elk config 수정 정리 : https://kkamagistory.tistory.com/771

logstash - jdbc 연결 참고 자료 : https://ohoroyoi.tistory.com/233

logstash - jdbc docs : https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html



## 1. Docker 설치

 https://docs.docker.com/engine/install/centos/ 

위의 docs를 참고하여 설치 

docker를 이미 설치했고 사용하고 있는 상태라면 삭제 X



## 2. docker-compose 설치

 https://docs.docker.com/compose/install/#install-compose-on-linux-systems

위의 docs를 참고하여 설치



## 3. docker-elk 설치

docker-elk 설치를 원하는 위치로 이동 후 아래 명령어 실행

```
$ git clone https://github.com/deviantony/docker-elk.git
$ cd docker-elk
```

https://judo0179.tistory.com/60

위의 블로그를 참고하여 elasticsearch, logstach, kibana config 수정(xpack 부분 주석 처리)



## 4. JDBC 연결 설정

DB 데이터를 elasticsearch에 저장하는 경우 사용



### 4-1. 원하는 JDBC driver파일 다운로드

postgres의 경우 https://jdbc.postgresql.org/  으로 이동하여 원하는 버전을 다운로드한다. 

다운로드 받은 후 원하는 위치에 저장 (ex /lib/.... )



### 4-2. Logstash Dockerfile 수정

```
$ cd /../docker-elk/logstash
$ vi Dockerfile
```

 아래 명령어 추가 (참고 : https://github.com/dimMaryanto93/docker-logstash-input-jdbc/blob/master/Dockerfile)

```
COPY /{jdbc dirve 저장경로}/{jdbc 파일 이름}.jar /usr/share/logstash/logstash-core/lib/jars/postgresql.jar
```

-> docker를 사용할 경우 이렇게 지정해 줘야함. 



### 4-3. Logstash pipline conf 수정

```
input {
  jdbc {
     jdbc_connection_string => "jdbc:postgresql://{IP}:{PORT}/{DATABASE}"
     jdbc_user => "{userName}"
     jdbc_password => "{password}"
     jdbc_driver_class => "org.postgresql.Driver"
     jdbc_driver_library => "/usr/share/logstash/logstash-core/lib/jars/postgresql.jar"
     statement => "select * from table"
     jdbc_paging_enabled => true
     jdbc_page_size => 10000
     jdbc_pool_timeout => 300
     schedule => "*/1 * * * *"

 }
}

# filter{}

output {
  elasticsearch {
    hosts => "elasticsearch:9200"
    index => "index_name"
    document_id => "%{id}"
    doc_as_upsert => true
 }
}

```



#### input 

참고 : https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html#plugins-inputs-jdbc-options

| options                | input example                                             | 설명                                                         |
| ---------------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| jdbc_connection_string | jdbc:postgresql://localhost:5432/db                       | database 연결 정보를 입력하는 부분.<br />연결할 DB 정보에 맞게 입력 |
| jdbc_user              | user                                                      | database user 입력                                           |
| jdbc_password          | password                                                  | database password 입력                                       |
| jdbc_driver_class      | org.postgresql.Driver                                     | driver class 입력                                            |
| jdbc_driver_library    | /usr/share/logstash/logstash-core/lib/jars/postgresql.jar | 4-2 에서 copy한 jar 경로 입력 <br />사용 할 driver jar 의 경로가 입력되야함. |
| statement              | select * from table                                       | 저장할 data  query 입력                                      |
| jdbc_paging_enabled    | true                                                      | 페이징을 사용하여 데이터를 저장할 경우 사용                  |
| jdbc_page_size         | 10000                                                     | 페이지 사이즈 지정                                           |
| jdbc_pool_timeout      | 300                                                       | jdbc timeout 지정                                            |
| schedule               | */1 * * * *                                               | 스케쥴 지정 linux cron형식으로 스케쥴 지정                   |

다른 추가적 옵션들은 참고 확인



#### output 

참고 : https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html

| options       | output example     | 설명                                                         |
| ------------- | ------------------ | ------------------------------------------------------------ |
| hosts         | elasticsearch:9200 | output 을 elasticsearch로 지정하여 hosts를 elasticsearch:9200로 지정<br />만약 다른 output일 경우 다르게 지정해주면 된다. |
| index         | index_name         | 원하는 index name 지정                                       |
| document_id   | %{id}              | document id 지정. <br />input의 statement 쿼리에서 나온 컬럼의 이름을 사용한다. |
| doc_as_upsert | true               | elasticsearch에 존재하지 않을 경우 새로 생성                 |



## 5. docker-compose 실행

설치한 docker-elk 디렉토리로 이동하여 실행시켜야 한다. 

```
$ cd /../docker-elk
```

- 실행 : docker-compose build && docker-compose up -d
- 이미지까지 삭제 후 down : docker-compose down --rmi all
- 실행 후 로그 확인 : docker-compose logs -f



## 6. 기타

### 6-1. Elasticsearch

- 기본 접속 정보 : {IP}:9200 

- index 정보 : {IP}:9200/index_name

  - index가 가지고 있는 컬럼의 정보를 확인할 수 있다. 

- data 확인 : {IP}:9200/index_name/_search

  - 따로 쿼리를 주지 않으면 전체 데이터 검색 

  - 검색시 걸린 시간과 데이터.. 등등.. 확인 가능 

    ```json
    {
    	"took": 1425, //1425ms_걸렸다. 
    	"timed_out": false,
    	"_shards": {
    		"total": 1,
    		"successful": 1,
    		"skipped": 0,
    		"failed": 0
    	},
    	"hits": {
    		"total": {
    			"value": 10000, //elasticsearch는_최대_만개_출력.._더_출력하고_싶으면_다른_기능_적용하여_사용.. 
    			"relation": "gte" //데이터가_value보다_많거나(gt)_같음(e)을_의미 
            },
    		"max_score": 1,
    		"hits": [{
    					"_index": "index_name", //설정한_index_name
    					"_type": "_doc",
    					"_id": "32856265", //설정한_id_값
    					"_score": 1,
    					"_source": {
                            			"id": 32856265,
                            			"value_1" : "value",
                            			"value_2" : 2
                            //.
                            //.
                            //.                    	
    ```

- value 검색 :  {IP}:9200/index_name/_search**?q={property}:{search}**

  - `?` 뒤에 q로 검색한다. 
  - property를 지정하면 해당 property에서 검색 `ex) ...?q=value_1:value`
  - property를 지정하지 않으면 전체 propery에 해당되는 내용 검색 `ex) ...?q=value`



### 6-2. Kibana

- 기본 접속정보 : {IP}:5601
- 7.15.0 버전 기준 inex 확인하기 
  1. 왼쪽 상단 메뉴 버튼 클릭
  2. 맨 하단 Management -> Stack Management
  3. Data -> Index Management 
  4. 생성한 index 확인 

