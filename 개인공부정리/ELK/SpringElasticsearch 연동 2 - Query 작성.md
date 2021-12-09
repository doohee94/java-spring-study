# Spring Elasticsearch 연동 2 - Query Builder

#### 유용한 참고자료

- **Java High Level Rest Client 사용 정리 guide**

  https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html



## 1. 기본 사용법

Spring Elasticsearch 연동 1과 같이 설정을 완료하였다면, 바로 호출해서 사용가능하다. 

```java
@Service
@RequiredArgsConstructor
public class ElasticsearchService {

	private final RestHighLevelClient client;

	private static final String INDEX = "my_index";
	
    public SearchResponse sampleQuery() throws IOException {
		
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder()
            .query(QueryBuilders.matchAllQuery())
        //  .aggregation() // 필요할 경우 사용
            .size(0); 
      
		SearchRequest searchRequest = new SearchRequest(INDEX)
            .source(searchSourceBuilder);
		
        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
        
		return searchResponse;
	}
    
}

```

위의 코드는 하단의 쿼리를 수행한 것과 같다. 

```json
GET /_search
{	"size":0,
    "query": {
        "match_all": {}
    }
}
```



이런식으로 query, aggregtion, size를 만들어 원하는 쿼리를 작성 한 후, `SearchRequest`를 생성한다. 

그 후, `client.search()`를 통해 쿼리를 보내고, 그 응답을 `SearchResponse`로 받게된다. 



## 2. Query 만들기

```json
POST my_index / _search 
{
	"query": {
		"bool": {
			"filter": [{
				"terms": {
					"name": [
						"ABC",
						"BCD"
					]
				}
			}, {
				"exists": {
					"field": "a_field"
				
				}
			}, {
				"exists": {
					"field": "b_field"
				}
			}, {
				"terms": {
					"c_field": [
						"AAAA"
					]
				}
			}]
		}
	},
	"aggregations": {
		"agg_a_field": {
			"terms": {
				"field": "a_field",
				"size": 10000,
				"order": [{
					"name_count.value": "desc"
				}]
			},
			"aggregations": {
				"name_count": {
					"cardinality": {
						"field": "name"
					}
				},
				"paging": {
					"bucket_sort": {
						"sort": [],
						"from": 0,
						"size": 20
					}
				}
			}
		}
	},
	"size": 0
}
```

위의 elasticsearch 쿼리를 SearchSourceBuilder로 변환하면, 아래의 코드와 같아진다. 

```java
@Service
@RequiredArgsConstructor
public class ElasticsearchService {

	private final RestHighLevelClient client;

	private static final String INDEX = "my_index";
	
    //전체 쿼리 처리 
    public SearchResponse sampleQuery(List<String>names, List<String>cFields) throws IOException {
		
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder()
            .query(myQuery(names, cFields))
            .aggregation(myAggregation()) 
            .size(0); 
      
		SearchRequest searchRequest = new SearchRequest(INDEX)
            .source(searchSourceBuilder);
		
        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
        
		return searchResponse;
	}
    
    //query 부분
    private QueryBuilder myQuery(List<String>names, List<String>cFields){
			return QueryBuilders.boolQuery()
                .filter(QueryBuilders.termsQuery("name", names))  //names = ["ABC","BCD"]
				.filter(QueryBuilders.existsQuery("a_field"))
				.filter(QueryBuilders.existsQuery("b_field"))
                .filter(QueryBuilders.termsQuery("c_field", cFields)) //cFields = ["AAAA"]
				;
    }
    
    //aggregation 부분
    private TermsAggregationBuilder myAggregation(){

			CardinalityAggregationBuilder nameCardinalityAggs = AggregationBuilders
				.cardinality("name_count")
				.field(name);
        
			BucketSortPipelineAggregationBuilder paging = PipelineAggregatorBuilders
				.bucketSort("paging", new ArrayList<>())
				.from(0)
				.size(20);
        
			this.aggregationBuilder = AggregationBuilders
				.terms("agg_a_field")
				.field("a_field")
				.order(BucketOrder.aggregation("name_count.value", false)) //name_count의 value의 내림차순으로 정렬
				.size(10000)
				.subAggregation(nameCardinalityAggs)
				.subAggregation(paging);
    }
    
}

```





## 3. 응답 처리 

위의 쿼리를 실행시키면 아래와 같은 응답이 출력된다. (필요 부분만 표기)

```json
{
  "took" : 197,
  "timed_out" : false,
  "_shards" : {...},
  "hits" : {...},
  "aggregations" : {
    "agg_a_field" : {
      "buckets" : [
        {
          "key" : 1,
          "doc_count" : 400,
          "name_count" : {
            "value" : 300
          }
        },
        {
          "key" : 2,
          "doc_count" : 400,
          "name_count" : {
            "value" : 200
          }
        },
        {
          "key" : 3,
          "doc_count" : 3,
          "name_count" : {
            "value" : 100
          }
        }
        //후략
      ]
    }
  }
}

```

> 간단히 출력을 설명하자면, query를 수행한 후, a_field를 가지고 있는 name의 수가 많은 순으로 출력한 것이다. 
>
> select a_field, count(distinct name) from my_index where {query 조건} group by a_field order by count(distinct name) desc;
>
> a_filed의 데이터 타입이 number이기 때문에 nubmer형식으로 표기되었다. 



 `SearchResponse`로 받게 되면, 객체로 넘어오게 되는데 필요한 부분을 사용하기 위해 파싱이 필요하였다.

내가 원하는 정보는 buckets 배열에 있는 key(a_field의 값)과 name_count의 value 부분이다. 

```java
@Getter
public class ElasticResponse {
	private Map<Long, Long> data; // a_filed, name_count 값을 map으로 저장 

	public ElasticResponse(SearchResponse searchResponse) {
		this.data = new HashMap<>();

		Terms terms = searchResponse.getAggregations().get("agg_a_field");

		for (Terms.Bucket bucket : terms.getBuckets()) {
			Long a_field = (Long)bucket.getKey();
			Cardinality nameCount = bucket.getAggregations().get("name_count");

			this.data.put(a_field, nameCount.getValue());
		}
	}

}
```



사실 파싱하는 로직을 짜면서 이게 왜 이렇게 해야하나 시행착오가 많았는데, 지금 보니 조금 알 것같기도 하다... 

1. 실행 쿼리의 aggregation 부분에 `agg_a_field` 가 terms Query로 구성되어 있어, searchResponse의 `aggregations `필드에서도 `Terms`객체로 `agg_a_field`를 받는다. 
2. 각 terms 안에 name_count를 찾는 부분도 Cardinality로 되어있으니 name_count도 마찬가지로 `Cardinality`로 객체를 받는다. 
3. 받아서 원하는 데이터를 찾아 가공한다. 



내가 이해한게 맞는건지는 잘 모르겠지만.. 틀린다면 추후 글을 수정해야겠다 :(



