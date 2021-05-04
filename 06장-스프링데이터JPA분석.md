## 스프링 데이터 JPA 구현체 분석



실제로 JpaRepository에서 사용하는 대부분의 메서드는 `SimpleJpaRepository` 에 있다.

##### @Repository가 있는 효과

- JPA, JDBC 등 영속성 계층에서 터지는 Exception을 스프링이 추상화한 Exception으로 변경시켜줌

- 아무 기술을 바꿔도, 동일한 Exception을 얻을 수 있음. 기존의 코드를 유지해도 된다는 유지보수성에 장점이 있음.



##### 클래스단에 `@Transactional(readOnly=true)`

- 이건 flush() 작업을 하지 않아서 조회시 약간의 성능 향상을 얻을 수 있다.

- 그런데, 저장,수정,삭제에는 `@Transaction` 이 따로 걸려있음. 이는 서비스 계층에서 Transaction이 있다면 이를 이어 받아서 트랜잭션이 진행된다.
- 그런데 만약 Service 단에서 트랜잭션 없이 영속시킬 때(SimpleJpaRepository의 트랜잭션만 활용하면) save, update, delete 후 영속성 컨텍스트는 없을 것이다.

`save()` 에서 사용하는 `persist` 와 `merge` 의 차이

- 새로운 엔티티면 persist, 준영속 상태면 merge
- merge의 단점 : DB에 Select Query가 한 번 나간다는 점. 가급적이면 merge를 쓰지 마라.
- 데이터 변경은 `변경감지` 를 사용하라. merge는 준영속 상태를 다시 영속상태로 만들 때 쓰는 것입니다.





## 새로운 엔티티를 구분하는 방법

먼저, save() 함수를 확인해보자.

```java
@Transactional
@Override
public <S extends T> S save(S entity) {

  Assert.notNull(entity, "Entity must not be null.");

  if (entityInformation.isNew(entity)) {
    em.persist(entity);
    return entity;
  } else {
    return em.merge(entity);
  }
}
```

> isNew의 판단 요소는 무엇일까?

##### 새로운 엔티티를 판단하는 기본 전략 (isNewStratege)

- 식별자가 객체일 때 `null` 로 판단

```java
@Transient // DATAJPA-622
@Override
public boolean isNew() {
  return null == getId();
}
```

- 식별자가 자바 기본 타입일 때 `0` 으로 판단

```java
@Override
public boolean isNew(Object entity) {

  Assert.notNull(entity, "Entity must not be null!");

  if (!(entity instanceof Persistable)) {
    throw new IllegalArgumentException(
      String.format("Given object of type %s does not implement %s!", entity.getClass(), Persistable.class));
  }

  return ((Persistable<?>) entity).isNew();
}
```

`PersistentEntityIsNewStrategy` 클래스를 보면 다음과 같이 isNew 전략이 정의된 것을 확인할 수 있다.

```java
@Override
public boolean isNew(Object entity) {

  Object value = valueLookup.apply(entity);

  if (value == null) {
    return true;
  }

  if (valueType != null && !valueType.isPrimitive()) {
    return false;
  }

  if (value instanceof Number) {
    return ((Number) value).longValue() == 0;
  }

  throw new IllegalArgumentException(
    String.format("Could not determine whether %s is new! Unsupported identifier or version property!", entity));
}
```

```java
if (entityInformation.isNew(entity)) { // id가 null
			em.persist(entity); // id가 null
			return entity; // persist이후 id에 값이 채워짐
  ...
```

그런데 만약 다음과 같은 상황이 있다고 가정하자.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item {
    @Id
    private String id;

    public Item(String id) {
        this.id = id;
    }
}
```

그리고 테스트는 다음과 같이 한다.

```java
@Test
void save(){
  Item item = new Item("A");
  itemRepository.save(item);
}
```

PK가 Primitive 이미 Id가 정해져 있기 때문에 isNew가 아니라고 판단한다.

merge를 호출하면 select 이후 insert 쿼리가 나간다.

```sql
 select
        item0_.id as id1_0_0_ 
    from
        item item0_ 
    where
        item0_.id=?
        
 insert 
    into
        item
        (id) 
    values
        (?)
```

- `Persistable`인터페이스를 구현해서 판단 로직을 변경할 수 있다.

  > 참고: JPA 식별자 생성 전략이 @GenerateValue 면 save() 호출 시점에 식별자가 없으므로 새로운 엔티티로 인식해서 정상 동작한다. 그런데 JPA 식별자 생성 전략이 @Id 만 사용해서 직접 할당이면 이미 식별자 값이 있는 상태로 save() 를 호출한다. 따라서 이 경우 merge() 가 호출된다. merge() 는 우선 DB를 호출해서 값을 확인하고, DB에 값이 없으면 새로운 엔티티로 인지하므로 매우 비효율 적이다. 따라서 Persistable 를 사용해서 새로운 엔티티 확인 여부를 직접 구현하게는 효과적이다.
  >
  > 등록시간( @CreatedDate )을 조합해서 사용하면 이 필드로 새로운 엔티티 여부를 편리하게 확인할 수 있다.
  >
  >  (@CreatedDate에 값이 없으면 새로운 엔티티로 판단)

이를 해결하기 위해 Item을 다음과 같이 수정한다.

```java
@Entity
@Getter
@EntityListeners(AuditingEntityListener.class)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item implements Persistable<String> {
    @Id
    private String id;

    @CreatedDate // Persist 될 때 값이 채워지는 것
    private LocalDateTime createdDate;

    public Item(String id) {
        this.id = id;
    }

    @Override
    public boolean isNew() {
        return createdDate == null;
    }
}
```

그리고 다시 다음 테스트를 반복하면 isNew = true로 인식된다.

```java
@Test
void save(){
  Item item = new Item("A");
  itemRepository.save(item);
}
```

