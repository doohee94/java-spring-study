## ELK - SNV 정리

- java heap memory 늘리기

  ```text
  $vi docker-elk/docker-compose.yml
  ```

  ```text
  service:
  	elasticsearch:
  		...
  		environment:
  			EX_JAVA_OPTS: "-Xmx8G -Xms4G"
  ```

  최대 8G/최소4G로 수정

- elasticsearch 기본 최대 조회 개수 1만개  -> 현재 10만개로 변경

  (참고 : https://wedul.site/518)

  ```json
  PUT your_index_name/_settings
  { 
    "max_result_window" : 500000 
  }
  ```

  

- kibana에서 실행시 timeout이 걸려 postman에서 실행하는게 좋음 (설정에서 timeout을 변경할 수 있지만 어느정도 걸리는지 확인이 되지 않아.. )


- 기본 주소 `localhost:9200/index_name/_search`

  body - row에 원하는 쿼리 생성해서 send



### search 방법 정리

- 간단한 키워드를 검색하는 쿼리는 GET 방식으로도 충분히 동작하지만, 조건이 많이 들어가는 쿼리는 POST를 이용하여 쿼리를 생성해 보내야함

- `GET index_name/_search` 를 사용해서 응답을 받으면 total.value 값이 최대 1만개까지 밖에 표시가 안됨 

  ```json
   "hits" : {
      "total" : {
        "value" : 10000,
        "relation" : "gte" //1만보다_크거나_같음을_의미 
      }
    }
  ```

- `GET index_name/_search?scroll=1m` 을 통해 검색할시 total.value의 정확한 값이 검색됨 이를 활용하여 paging에 활용하면 좋을것 같음. 

  ```json
  "hits" : {
      "total" : {
        "value" : 17862948,
        "relation" : "eq"
      }
  }
  ```

- elasticsearch의 paging은 from/size로 실행 -> 조건이 위에서 설정한 max_result_window보다 넘을 경우 에러 발생

  ```json
  POST index_name/_search
  {
  	"size": 10, //size_0도_지정_가능_->_통계응답만_필요한_쿼리에서는_본래_응답_쿼리가_필요_없으므로_size_0_지정_가능
      "from": 0
      //...
  }
  ```

  ```json
  POST index_name/_search
  {
  	"size": 10,  // from_조건에_10만이_들어가서_100010조건을_검색할_수_없어_에러_발생
      "from": 100000
  }
  
  ```

  ```json
  POST index_name/_search?scroll=1m
  {
  	"size": 10 // scroll_조건이_붙으면_from조건_넣으면_안됨,__size도_0을_넣으면_에러발생
  }
  ```

- 통계쿼리의 경우 aggs 조건에서 paging 처리 -> 이 경우 전체 결과값이 필요 없으니 전체 size는 0으로 생성 / default size = 10

  ```json
  "aggs": {
  		"first": {
  			"terms": {
  				"field": "id"
  				, "size": 2000
  			},
  			"aggs": {
  			  	"paging": { //first_내부_aggs를_생성하여_paging_이름_생성_후_bucket_sort_조건_생성
  					"bucket_sort": {
  						"from": 0,
  						"size": 100
  						}
  				},
  				"second":{
  					"terms": {
  						"field": "name",
  						"size": 10
  					}
  				}
  			}
  		}
  		
  }
  ```

  **size의 크기에 따라 쿼리 속도 차이가 심함...** 

  

## 1. 같은 필드를 조건으로 넣기

```json
POST index_name / _search 
{
  "size": 0,
  "query": {
    "bool": {

      "must": [{
        "bool": {
          "should": [{
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "aaa"
                }
              }]
            }
          }, {
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "bbb"
                }
              }]
            }
          }, {
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "ccc"
                }
              }]
            }
          }]
        }
      }]
    }
  },
  "_source": false
}
```



## 2. 필터 조건 넣기

query에 있는 조건을 검색 후 필터를 통해 다시 걸러내는 것 같음.. 

```json
POST index_name / _search 
{
  "size": 0,
  "query": {
    "bool": {

      "must": [{
        "bool": {
          "should": [{
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "aaa"
                }
              }]
            }
          }, {
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "bbb"
                }
              }]
            }
          }, {
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "ccc"
                }
              }]
            }
          }]
        }
      }],
      "filter": [{
        "terms": {
          "age": [
            18,80,20
          ]
        }
      }, {
        "term": {
          "school.keyword": "highSchool"
        }
      }]
    }
  },
  "_source": false
}
```



## 3. 통계

agg를 통해 통계 가능

agg 안에 agg를 계속 넣어서 계속 값을 구할 수있다. 

```json
POST index_name / _search 
{
  "size": 0,
  "query": {
    "bool": {

      "must": [{
        "bool": {
          "should": [{
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "aaa"
                }
              }]
            }
          }, {
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "bbb"
                }
              }]
            }
          }, {
            "bool": {
              "must": [{
                "term": {
                  "name.keyword": "ccc"
                }
              }]
            }
          }]
        }
      }],
      "filter": [{
        "terms": {
          "age": [
            18,80,20
          ]
        }
      }, {
        "term": {
          "school.keyword": "highSchool"
        }
      }]
    }
  },
  "aggs": {
    "school": {
      "terms": {
        "field": "school.keyword",
        "size": 10
      },
      "aggs": {
        "age": {
          "terms": {
            "field": "age.keyword",
            "size": 10
          },
          "aggs": {
            "name_count": {
              "cardinality": {
                "field": "name.keyword"
              }
            }
          }
        }
      }
    }

  },
  "_source": false
}
```



## 4. 응답

2번의 variant 필터를 넣은 쿼리를 실행할 경우의 응답 

```json
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 4,
      "relation": "eq"
    },
    "max_score": null,
    "hits": []
  },
  "aggregations": {
    "school": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [{
        "key": "highScool",
        "doc_count": 4,
        "age": {
          "doc_count_error_upper_bound": 0,
          "sum_other_doc_count": 0,
          "buckets": [{
            "key": 18,
            "doc_count": 4,
            "name_count": {
              "value": 185 
            }
          }]
        }
      }]
    }
  }
}
```



## 5. agg 페이징

```json
POST index_name / _search 
{
    "size": 0,

    "query": {
      "bool": {
        "must": [{
              "bool": {
                "should": [
                  //구하고싶은_모든_조건 
                ]
              }
            ,
            "aggs": {
              "school": {
                "terms": {
                  "field": "school.keyword",
                  "size": 20000
                },
                "aggs": {
                  "paging": {
                    "bucket_sort": {
                      "from": 0,
                      "size": 100
                    }
                  },
                  "age": {
                    "terms": {
                      "field": "age",
                      "size": 20
                    },
                    "aggs": {
                      "name_count": {
                        "cardinality": {
                          "field": "name.keyword"
                        }
                      }
                    }
                  }
                }
              }

            },
            "_source": false
```



데이터가 너무 많을경우 buckets_size와 관련해서 에러가 발생할 수있다. `too_many_buckets_exception`

https://shinwusub.tistory.com/m/129 를 참고하여 해결.. 

