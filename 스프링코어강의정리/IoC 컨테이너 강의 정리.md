# 스프링 IoC 컨테이너와 빈

### Inversion of Control

**IoC**, 의존관계 주입, 어떤 객체가 사용하는 의존객체를 직접 만들어 사용하는게 아니라 주입받아 사용하는 방법

```java
public class BookService{

	//BookRepository bookRepository = new BookRepository(); // 이렇게 쓰는게 아니라 

	BookRepository bookRepository; //이렇게 선언해서 

	public BookService(BookRepository bookRepository) {
		this.bookRepository = bookRepository; //생성자로 주입 
	}

}
```



### IoC 컨테이너란? 

IoC 기능을 하는 Bean들을 담고 있는 컨테이너 (ex) service, repository...

### Bean

스프링 IoC 컨테이너가 관리하는 객체 

#### 장점

1. 주입으로 의존성관리를 할 수 있다.
2. 싱글톤 타입으로 만들면 컨테이너에 하나만 생성되어서 성능, 메모리면에서 유리하다. 
3. PostConstruct 등을 이용하여 부가적 기능을 넣을 수 있다. 



### @Autowire

필요한 의존객체의 타입에 해당하는 빈을 찾아 주입 

#### 생성 가능 위치 

생성자, 세터, 필드



### 경우의 수

1. 해당 타입의 빈이 없는 경우 -> 안하면 됨
2. 해당 타입의 빈이 한개인 경우 -> 그 한개에 @Autowire 해주면 됨
3. 해당 타입의 빈이 여러개인 경우 
   
   - 해당 빈 이름으로 시도, 같은 이름으로 찾으면 해당 빈 사용, 못찾으면 실패
   
      

### 같은 타입의 빈이 여러개인 경우?

```java
public interface BookRepository{}

@Repository
public class MyBookRepository implements BookRepository{}

@Repository
public class DoheeBookRepository implements BookRepository{}

@Service
public class BookService{
	@Autowire
	BookRepository bookRepository; //이렇게 하면 어떤 구현체를 찾는지 몰라서 에러가 남
}

```

### 해결 방법

1. **@Primary annotation 사용 -> 추천 방법 type safe 한 방식**

```java

public interface BookRepository{}

@Repository
public class MyBookRepository implements BookRepository{}

@Repository @Primary
public class DoheeBookRepository implements BookRepository{}

public class BookService{

	@Autowire
	BookRepository bookRepository; // DoheeBookRepository를 bean으로 주입받음
}

```

2. **@Qualifier annotaion 사용** 

```java

public interface BookRepository{}

@Repository
public class MyBookRepository implements BookRepository{}

@Repository 
public class DoheeBookRepository implements BookRepository{}

public class BookService{

	@Autowire @Qualifier(DoheeBookRepository)
	BookRepository bookRepository; // DoheeBookRepository를 bean으로 주입받음
}

```

3. **해당 타입 모두 주입 받기**

```java

public interface BookRepository{}

@Repository
public class MyBookRepository implements BookRepository{}

@Repository 
public class DoheeBookRepository implements BookRepository{}

@Service
public class BookService{

	@Autowire
	List<BookRepository> bookRepository; // BookRepository의 모든 구현체 bean들을 가져옴
}

```

4. **구현체 이름으로 받아오기 -> 추천 X**
```java

public interface BookRepository{}

@Repository
public class MyBookRepository implements BookRepository{}

@Repository 
public class DoheeBookRepository implements BookRepository{}

@Service
public class BookService{

	@Autowire
	BookRepository doheeBookRepository; // DoheeBookRepository를 bean으로 주입받음
}

```

### 동작 원리

~~아직 제대로 이해한건 아니지만.. 다시 공부하기 ^^~~

@Autowire 어노테이션이 붙은 객체를 빈으로 주입받은 후 빈 생성(?) 

BeanPostProcessor
AutowiredAnnotationBeanPostProcessor 



### Component와 컴포넌트 스캔

#### @ComponentScan  

어노테이션은 스프링 3.0 부터 나옴

같은 패키지 안에 있는 Bean들만 탐색 

만약 다른 패키지에 있는데 Autowired를 할 경우, 컴포넌트 스캔이 되지 않고, 빈 등록이 되지 않으니 주입할 수 없음



#### @ComponentScan의 Filter

filter 옵션을 이용하여 원하지 않는 클래스들을 스캔 제외 할 수 있음

```java
@ComponentScan(excludeFilters = {
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFIlter.class)
})
```



#### 정리하자면

컴포넌트 스캔의 범위가 어떻게 되느냐와 어떤 컴포넌트를 제외하고 스캔할 것이냐 이런 옵션이 있음

@Repository 나 @Service, @Controller, @Configuration  어노테이션도 @Component 를 포함하고 있음

**단점** : 등록해야 하는 빈이 많은 경우에는 구동시간이 오래 걸린다



##### 만약 등록해야하는 빈이 많아서 구동시간이 긴게 싫다면..?

Spring 5에 나온 펑션을 사용하여 빈 등록 

어떤 조건에 따라 빈 등록이 가능하다..

컴포넌트 스캔 말고 따로 빈을 등록하는 경우에만 사용하는게 좋을거 같다. 



### Bean의 스코프

Bean을 등록할 때 아무런 설정도 하지 않으면 싱글톤으로 설정 됨



#### 싱글톤 스코프란?

어플리케이션 전반에 걸쳐 해당 빈의 인스턴스가 1개 뿐인 스코프

```java
@Component
public class Single {
    
    @Autowired
    private Proto proto;

    public Proto getProto() {
        return proto;
    }
}


@Component
public class Proto {
}

@Component
public class AppRunner implements ApplicationRunner {

    @Autowired
    Single single;

    @Autowired
    Proto proto;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println(proto);
        System.out.println(single.getProto());
    }
}

```

AppRunner의 proto 와 Single에서 가져온 proto 둘 다 같은 객체임을 알 수 있다. 



#### 프로토타입 스코프?

매번 새로운 인스턴스를 만들어서 사용하는 스코프 

 ex) Request, Session, WebSocket ...

```java
@Component @Scope("prototype")
public class Proto {
}

@Component
public class AppRunner implements ApplicationRunner {

    @Autowired
    ApplicationContext ctx;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println(ctx.getBean(Proto.class));//다른 객체로 가져옴
        System.out.println(ctx.getBean(Proto.class));
        System.out.println(ctx.getBean(Proto.class));
        System.out.println("///");
        System.out.println(ctx.getBean(Single.class)); //같은 객체로 가져옴
        System.out.println(ctx.getBean(Single.class));
        System.out.println(ctx.getBean(Single.class));
    }
}
```

이런 식으로 스코프에 설정을 해주면, 빈을 받아 올때 다른 객체들로 가져온다



##### 만약, 싱글톤 스코프의 Bean이 Prototype의 빈을 참조하는 경우라면.. ? 

다른 객체로 가져올 것 같지만 같은 proto 객체가 출력됨

```java
@Component
public class AppRunner implements ApplicationRunner {

    @Autowired
    ApplicationContext ctx;

    @Override
    public void run(ApplicationArguments args) throws Exception {
		System.out.println(ctx.getBean(Single.class).getProto()); //같은 객체로 가져옴
        System.out.println(ctx.getBean(Single.class).getProto());
        System.out.println(ctx.getBean(Single.class).getProto());
    }
}
```



이 상황을 해결하기 위해서 proxyMode 설정을 해준다. 이렇게 하면 위의 코드에서 다른 객체가 나옴

```java
@Component @Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class Proto {
}
```



###### proxyMode = ScopedProxyMode.TARGET_CLASS 

클래스기반의 프록시로 해당 bean을 감싸라. 

싱글톤 스코프에서 프로토타입을 사용할 경우 계속 바뀌어야 하는데 그럴 수가 없으니 프록시로 해당 빈을 감싸라.. 

그러면 빈을 감싸고 있는 프록시가 빈으로 등록됨 그 프록시 빈을 single에서 주입하는 것. 



또다른 해결방법은?  이 방법보다는 위의 방법을... 더 추천(?)

```java
@Component
public class Single {
    
    @Autowired
    private ObjectProvicer<Proto> proto;

    public Proto getProto() {
        return proto.getIfAvailable();
    }
}
```



##### 정리하자면.. 

보통의 경우에는 싱글톤을 사용함

프로퍼티가 공유되기 때문에 스레드 safe하게 개발해야함. 

초기 구동시 인스턴스가 생성되기 때문에 구동 시간이 걸릴 수 있음. 













