# JPA OneToOne 관계인데 2개가 저장됨

구독플랫폼 프로젝트 회원가입/수정 부분을 진행하는 중이었다.

 회원가입을 먼저 진행하여 User를 생성하였고, 그 후에 Customer에 필요한 정보를 넣어 생성하거나 수정하는 로직을 만들었다.

간단한 insert니까 빨리 끝낼 수 있을 줄 알았다. 나의 경기도 오산이었다...



OneToOne관계니까 당연히 1개만 저장될 줄 알았는데 혹시 몰라 한번 더 눌러보니 2개가 저장되는 일이 발생하였다. 

뭐가 잘못이지? CaseCade, Fetch, Transactional 모두 수정해 보았지만 2개보다 더 많이 저장되는 경우도 발생해버렸다 ^^..



혹시나 하는 마음에 생성된 테이블을 보니.. Customer 테이블에 user_id FK에 유니크 설정이 되어있지 않았다!!!

그러니 2개고 3개고 막 들어가는 일이 발생했던 것이다!!!



이런 경우에는 @JoinColumn 어노테이션에 `unique = true` 옵션을 주면 된다고 한다. 기본 설정은 당연히 false로 되어있었다...^^

#### Customer

```java
@OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
@JoinColumn(name = "user_id", unique = true)
    private User user;
```



이후로 다시 실행하니 깔끔하게 1개만 저장되었다..!!!



<hr/>

##### 그런데 말입니다..!!

사실 OneToOne관계든 뭐든 save할 때, Customer 객체를 구분해서 업데이트를 하거나 새로 생성해주면 user_id가 중복될 일이 없었을 것 같다..

그치, 원래 이렇게 하는게 맞는거지... 

~~역시 사람이 머리가 나쁘면 몸이 고생하나보다..^^..~~ 좋은 공부가 되었다 ^^ 
