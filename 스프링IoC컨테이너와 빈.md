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

<hr/>

##Bean
### 스프링 IoC 컨테이너가 관리하는 객체 
### 장점
1. 주입으로 의존성관리를 할 수 있다.
2. 싱글톤 타입으로 만들면 컨테이너에 하나만 생성되어서 성능, 메모리면에서 유리하다. 
3. PostConstruct 등을 이용하여 부가적 기능을 넣을 수 있다. 
