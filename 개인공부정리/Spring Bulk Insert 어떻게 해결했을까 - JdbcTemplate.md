# Spring Batch Insert 어떻게 해결했을까 - JdbcTemplate

대용량의 데이터를 DB에 넣어야 하는 일이 발생했다. 

한 파일당 약 3만줄의 데이터를 파싱해서 20개의 테이블에 나눠 저장을 하였다. 파일은 모두 90개였던 것으로 기억한다. (전체 저장 데이터가 5G정도 나왔었다.)

다 저장해보니 용량이 어마무시했던 기억이 있는데, 지금은 사용하지 않아 없어졌다. 

하지만 Spring Batch Insert를 도전했던 좋은 추억이라 (^^) 기록을 해보려 한다. 



## JdbcTemplate Bulk Insert 활용기

### 1. 기본 사용 방법

bulk insert를 위해 batchUpdate() 함수를 사용하였다. 기본 사용 방법은 아래와 같다. 

```JAVA
@RequiredArgsConstructor
public class SampleJdbcTemplateRepository {

  private final JdbcTemplate jdbcTemplate;

  public void batchInsert(List<SampleDto> temp) {

    jdbcTemplate.batchUpdate("INSERT INTO sample_table ("
            + "a,\n"
            + "b )"
            + "VALUES (?,?)",
        new BatchPreparedStatementSetter() {
          @Override
          public void setValues(PreparedStatement ps, int i) throws SQLException {

            ps.setObject(1, temp.get(i).getA(), Types.VARCHAR);
            ps.setObject(2, temp.get(i).getB(), Types.VARCHAR);

          }

          @Override
          public int getBatchSize() {
            return temp.size();
          }
        });
  }

}
```



### 2. 어떻게 활용했나

참고 : https://homoefficio.github.io/2020/01/25/Spring-Data%EC%97%90%EC%84%9C-Batch-Insert-%EC%B5%9C%EC%A0%81%ED%99%94/

위의 블로그를 참고하여 코드를 작성하였다. 

서론에서 언급했지만 테이블이 약 20개에, 모두 연관관계로 매핑되어있었고, 데이터의 양도 많았기 때문에.. 

> 1. 저장할 전체 리스트를 지정한 batchSize만큼 자른다
> 2. 자른 리스트를 batchInsert 시킨다.
> 3. 마지막 남은 리스트도 마저 insert 시킨다. 

이 로직을 그대로 이용하였다. 

이 로직 중, 저장할 전체 리스트를 자르고 batchInsert함수를 호출하는 부분은 모든 테이블에 반복 적용되니 추상 클래스를 만들어 필요한 부분에 상속시켜주었다.



### 1. BathInsert 추상 클래스 (SampleJdbcTemplate.java)

```java
@RequiredArgsConstructor
public abstract class SampleJdbcTemplate<T> { // T에는 batchInsert를 하고싶은 도메인 

  private final int BATCH_SIZE = 1000;

  @Transactional
  public void save(List<T> dataList) {

    int batchCount = 0;// insert 총 횟수 확인을 위한 변수 

    List<T> temp = new ArrayList<>();

    for (int i = 0; i < dataList.size(); i++) {

      temp.add(dataList.get(i));

      if ((i + 1) % BATCH_SIZE == 0) {
        batchCount = vepBatchInsert(batchCount, temp);
      }
    }

    if (!temp.isEmpty()) {
      batchCount = vepBatchInsert(batchCount, temp); //for문을 벗어나고 남은 데이터가 존재하면 나머지 insert
    }

    log.info("batchCount : " + batchCount); 

  }

  protected abstract int batchInsert(int batchCount, List<T> temp); // 각 쿼리 실행을 위한 abstract 함수 

}
```



### 2. SampleJdbcTemplate구현 클래스 (SampleDtoJdbc.java)

```java
@Repository
@RequiredArgsConstructor
public class SampleDtoJdbc extends SampleJdbcTemplate<SampleDto> {

  private final JdbcTemplate jdbcTemplate;;

  @Override
  protected int batchInsert(int batchCount, List<SampleDto> temp) {

    jdbcTemplate.batchUpdate("INSERT INTO sample_table ("
            + "a,\n"
            + "b )"
            + "VALUES (?,?)",
        new BatchPreparedStatementSetter() {
          @Override
          public void setValues(PreparedStatement ps, int i) throws SQLException {

            ps.setObject(1, temp.get(i).getA(), Types.VARCHAR);
            ps.setObject(2, temp.get(i).getB(), Types.VARCHAR);

          }

          @Override
          public int getBatchSize() {
            return temp.size();
          }
        });

    temp.clear(); // temp list를 비워준다. 
    batchCount++; //insert가 완료되면, 총 횟수에 1을 더해준다. 

    return batchCount;
  }


}
```

해당 함수는 insert가 필요한 모든 테이블에 대해 생성해 주었다. 



### 그래서 Bulk Insert? Batch Insert가 뭐야? 

말 그대로 많은 양의 데이터를 Insert하는 것이다. 

JPA의 saveAll을 이용하면 데이터 리스트의 저장가능하지만, 이런 방식으로 insert를 했다간.. 

```sql
INSERT INTO test_table (~~~) VALUES (~~~)
```

이런 쿼리가 수백, 수천만건 반복될 것이다 -> 시간이 오래걸린다.

내가 저장했던 데이터를 JPA SaveAll을 이용하여 저장 했을 시 <u>2시간</u> 이상 걸렸던 것으로 기억한다..



그래서 spring에서 batch-insert를 지원한다고 하여 사용해 봤었지만.. 내가 원하는 기능은 아니었다. 

(참고 : https://jaehun2841.github.io/2020/11/22/2020-11-22-spring-data-jpa-batch-insert/#hibernateorder_inserts-hibernateorder_updates)

내가 지정한 id의 생성 방식도 <u>strategy = GenerationType.IDENTITY</u> 이렇게 지정했었을 뿐더러

batch-insert도 property에서 지정한 batch-size에 맞게 insert를 묶어서 보내주는 방식일 뿐 아래와 같은 형식은 아니었다. 

```sql
INSERT INTO TABLE (~~~~) VALUES 
(~~~~~),
(~~~~~~),
(~~~~~~);
```

batch-size를 지정하여 저장하였을 때는 약 <u>1시간</u> 정도의 시간이 소요되었다. 



**그럼 JdbcTamplate를 이용했을때는?**

https://wave1994.tistory.com/160

이 블로그를 참고해보면 Hibernate에서는 insert가 단 건으로 처리되지만, DB driver에서 해당 쿼리들을 모아 batch insert를 시켜준다고 한다.

JdbcTamplate을 사용했을 때는 <u>20분~30분</u>의 시간이 소요되었다. ~~정말 눈에 띄게 시간이 단축되어 성공했을때 울뻔했다..~~





## 후기

사실 작년에 작성했던 코드라 잘 기억도 안나고.. 기록도 거의 안남아서 힘들었다는 기억밖에 없었는데, 

글로 한번 정리하면서 관련 글도 읽고하니 새로운 사실도 알게 되어서 정리하길 잘했다는 생각이 든다. 

유전 데이터 분석 기능 구현부터 분석 결과 저장까지 약 한 달 정도의 시간이 걸렸던 것 같은데 이렇게 좋은 경험을 쌓을 수 있어서 나름 좋은 것 같다. 

관련 기능이 없어져서 관련한 모든 

~~*하지만 함께해서 별로였고 다신 내가 쓰는 데이터로 만나지 말자 VEP.. ^^..*~~ 
