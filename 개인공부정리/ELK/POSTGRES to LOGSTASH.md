# ELK_POSTGRES to LOGSTASH

postgres 데이터를 logstash를 이용해 elasticsearch에 저장하는 방법은 `개인공부정리/DOCKER에서 ELK 설치 및 사용하기.md` 이 파일에 있지만, 그 후에 더 추가해서 사용한 방법이 있어 따로 정리해보았다. 



우선, 아래와 같은 conf 파일을 logstash에서 기본적으로 사용한다고 가정한다. 

```text
# cd logstash/pipeline
# vi {이름}.conf
```

```text
input {
  jdbc {
     jdbc_connection_string => "jdbc:postgresql://{IP}:{PORT}/{DATABASE}"
     jdbc_user => "{USER}"
     jdbc_password => "{PASSWORD}"
     jdbc_driver_class => "org.postgresql.Driver"
     jdbc_driver_library => "/usr/share/logstash/logstash-core/lib/jars/postgresql.jar"

     statement => "select * from test.test test where id >= ? and id< ? + ?  order by id asc"

     use_column_value => true
     tracking_column => "id"
     tracking_column_type => "numeric"

     use_prepared_statements => true
     prepared_statement_bind_values => [":sql_last_value", ":sql_last_value", 10000]
     prepared_statement_name => "id"

     last_run_metadata_path => "/usr/share/logstash/last_run_metadata/last_value_1.yml"

     jdbc_pool_timeout => 300
     schedule => "0 * * * *"

 }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "my_index"
    document_id => "%{id}"
    doc_as_upsert => true
    template => "/usr/share/logstash/template/my_basic_template.json"
    user => "elastic"
    password => "changegame!"

 }
}
```



## 1. pipeline

보통 logstsh의 pipeline을 사용하는 경우, `pipeline.yml`파일을 생성해서 관리를 해준다고 하는데, docker-elk의 경우는 pipeline 폴더에 원하는 설정파일들을 생성해 준 후,  `docker-compose.yml`내 파일의 경로만 잘 설정해 준다면 문제 없이 작동하는 것 같다. 

> 하지만 이럴경우 훗날 설정파일이 많아졌을 때 관리하기가 힘들어 지는 일이 생기니.. yml파일을 만들어서 사용하는것이 좋을 것 같다 😅



## 2. sql_last_value 파일 

설정파일이 많아질 경우, last_value의 값도 많아지는 경우가 생긴다. 

이럴 경우 파일을 각각 생성해서 관리하는 것이 좋을 것 같다. 

나같은 경우 logstash 디렉토리에 `last_run_metadata` 디렉토리를 생성하여 그 안에서 파일을 따로 관리하였다. 

```text
last_run_metadata_path => "/usr/share/logstash/last_run_metadata/last_value_1.yml"
```

설정 파일 내부에 해당 파일을 적용시켜 줄 수 있다. 



## 3. index template 파일

postgres에서 아무런 설정 없이 index를 생성할 경우 컬럼이 자동매핑되어 데이터가 저장된다. 

이럴경우 string data type 저장 시, 문제가 발생하게 된다. 

기본적으로 string은 text type과 keyword type으로 나뉘는데, 자동매핑인 경우 이 두 타입을 모두 저장하게 된다. 

그러면 데이터의 크기도 커질 뿐만아니라 쿼리를 실행시킬 시 시간도 오래걸리고, 정확한 값을 찾을 경우에는 `name.keyword` 이런식으로 키워드를 검색해주어야 한다. 

그래서 logstash를 이용해 저장할 경우 기본 템플릿을 정하여 데이터를 매핑 시킬 수 있다. 



**template 파일 예시**

```json
{
    "index_patterns": [
        "my_index*"
    ],
    "mappings": {
        "properties": {
            "name": {
                "type": "keyword",
                "ignore_above": 2147483647

            },
            "title": {
                "type": "keyword",
                "ignore_above": 2147483647

            },
              "score": {
                "type": "float"
            },
              "age": {
                "type": "long"
            }

        }
    }
}
```

> - keyword 타입을 지정할 경우 `ignore_above`값을 지정해 주는 것이 좋다. 기본값은 256인데, 256자가 넘어가게 되면 짤려서 저장이 되기 때문이다. 최대 값은 2147483647이다. 
> - index_patterns는 해당 템플릿을 적용할 인덱스의 title을 지정할 수 있다. 정규표현식을 사용한다. 



해당 파일을 만들면, conf 파일에서 설정해준다. 

나같은 경우 logstash에 template 디렉토리만들어 그 안에 템플릿을 저장하였다. 

```text
template => "/usr/share/logstash/template/my_basic_template.json"
```



## 4. docker-compose.yml 파일 수정

위에서 템플릿 파일이나 logstash 설정파일, last_value 파일 등등 여러개 파일을 생성하였다. 

생성한 파일의 경로를 docker 경로에 맞게 설정해주어야 한다. 

또한, docker 이미지 생성 시, 생성한 파일들도 생성할 수 있도록 명령어를 작성해 넣어주어야 한다. 



**docker-compose.yml 예시**

```yaml
 logstash:
   ...
    command: /bin/sh -c "touch /usr/share/logstash/last_run_metadata/last_value_1.yml
                        && bin/logstash
                        && touch /usr/share/logstash/template/my_basic_template.json
                        && bin/logstash"
    volumes:
     ...
      - ./logstash/last_run_metadata/:/usr/share/logstash/last_run_metadata/
      - ./logstash/template/:/usr/share/logstash/template/
...

```

1. command 부분의 명령어를 통해 생성한 last_value 파일과 템플릿 파일을 docker 경로에 설정해준다. 

2. 새로 생성한 디렉토리가 있다면,  volumes 부분에 경로를 설정해준다. 

   `ex) {내가 만든 경로}:{docker에 설정할 경로}`



## 5.Permission denied

last_value 파일이나 템플릿 파일을 생성해서 사용할 경우, Permission denied 문제가 발생한다. 

```text
chown 1000:1000 -R 해당 폴더
```

상단의 명령어를 통해 권한 문제를 해결할 수 있다. (참고 블로그 확인)



## 참고

docker-elk git : https://github.com/deviantony/docker-elk

permission denied : https://epicarts.tistory.com/66

