# 스프링 IoC 컨테이너와 빈

## Inversion of Control
### IoC, 의존관계 주입, 어떤 객체가 사용하는 의존객체를 직접 만들어 사용하는게 아니라 주입받아 사용하는 방법

```java
public class BookService{

	//BookRepository bookRepository = new BookRepository(); // 이렇게 쓰는게 아니라 

	BookRepository bookRepository; //이렇게 선언해서 

	public BookService(BookRepository bookRepository) {
		this.bookRepository = bookRepository; //생성자로 주입 
	}

}
```

## IoC 컨테이너란? 
### IoC 기능을 하는 Bean들을 담고 있는 컨테이너 (ex) service, repository...


## Bean
### 스프링 IoC 컨테이너가 관리하는 객체 
### 장점
1. 주입으로 의존성관리를 할 수 있다.
2. 싱글톤 타입으로 만들면 컨테이너에 하나만 생성되어서 성능, 메모리면에서 유리하다. 
3. PostConstruct 등을 이용하여 부가적 기능을 넣을 수 있다. 


<hr/>

# @Autowire
- 필요한 의존객체의 타입에 해당하는 빈을 찾아 주입 

## 생성 가능 위치 
- 생성자, 세터, 필드

## 경우의 수
1. 해당 타입의 빈이 없는 경우 -> 안하면 됨
2. 해당 타입의 빈이 한개인 경우 -> 그 한개에 @Autowire 해주면 됨
3. 해당 타입의 빈이 여러개인 경우 
   - 해당 빈 이름으로 시도, 같은 이름으로 찾으면 해당 빈 사용, 못찾으면 실패 


## 같은 타입의 빈이 여러개인 경우?

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

1. @Primary annotation 사용 -> 추천 방법 type safe 한 방식

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

2. @Qualifier annotaion 사용 

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

3. 해당 타입 모두 주입 받기

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

4. 구현체 이름으로 받아오기 -> 추천 X
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

## 동작 원리

```text

	~~아직 제대로 이해한건 아니지만.. 다시 공부하기 ^^ ~~

 	@Autowire 어노테이션이 붙은 객체를 빈으로 주입받은 후 빈 생성(?) 

 	BeanPostProcessor
 	AutowiredAnnotationBeanPostProcessor 

```