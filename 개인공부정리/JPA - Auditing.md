# JPA - Auditing

엔티티 변경 시점에 언제, 누가 변경했는지에 대한 정보를 기록하는 기능

1. Auditing 정보를 담은 추상 클래스 생성

   ```java
   @MappedSuperclass 
   @EntityListeners(AuditingEntityListener.class)
   public abstract class BaseTimeEntity {
   
       @CreatedDate
       private LocalDateTime createdDate;
   
       @LastModifiedDate
       private LocalDateTime lastModitiedDate;
   
   }
   ```

   | 어노테이션                                     | 설명                                                         |
   | ---------------------------------------------- | ------------------------------------------------------------ |
   | @MappedSuperclass                              | 엔티티 클래스들이 해당 어노테이션이 달린 클래스를 상속할 경우 CreatedDate 등의 어노테이션을 컬럼으로 인식 |
   | @EntityListeners(AuditingEntityListener.class) | 해당 클래스에 Auditing 기능을 포함                           |
   | @CreatedDate                                   | 엔티티가 생성되어 DB에저장될 때 시간 자동저장                |
   | @LastModifiedDate                              | 엔티티가 수정될 시 시간 자동 저장                            |
   | @CreatedBy                                     | 엔티티를 생성한 user 저장                                    |
   | @LastModifiedBy                                | 엔티티를 수정한 user 저장                                    |

2. 원하는 엔티티에 Auditing 클래스 상속 및 Application main에 @EnableJpaAuditing 설정

   ```java
   @Entity
   @Table(name = "user")
   @Getter
   @NoArgsConstructor(access = AccessLevel.PROTECTED)
   public class User extends BaseTimeEntity {
       @Id
       @Column(name = "user_id")
       @GeneratedValue(strategy = GenerationType.IDENTITY)
       long id;
   
       //...생략
   }
   
   ```

   ```java
   @EnableJpaAuditing
   @SpringBootApplication
   public class PlatformApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(PlatformApplication.class, args);
       }
   
   }
   ```

   

3. DB에 저장 시 createdDate와 lastModitiedDate가 같이 저장된 것을 볼 수 있다.

