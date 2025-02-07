## 문제 상황
**JPQL** 혹은 **JPA Query Method**를 통해 DB의 데이터를 수정 및 삭제하는 경우, 아래와 같은 예외가 발생하였습니다.


> org.springframework.dao.InvalidDataAccessApiUsageException: No EntityManager with actual transaction available for current thread - cannot reliably process 'remove' call
> 

```java
// 예외 발생 지점
public void deleteByMemberIdAndId(LoginMember loginMember, long id) {
    reservationRepository.deleteByMemberIdAndId(loginMember.getId(), id);
}
```

### 예외 메세지

```java
org.springframework.dao.InvalidDataAccessApiUsageException: No EntityManager with actual transaction available for current thread - cannot reliably process 'remove' call ...
```

>**이 오류 메시지는 현재 스레드에 실제 트랜잭션이 활성화된 EntityManager가 없어서, 
<br> JPA의 remove() 호출을 신뢰성 있게 처리할 수 없다는 것을 의미합니다.**

---


`Data JPA`를 사용하면서 주어지는 `JPA` 메서드만을 가지고는 원하는 결과를 도출해내기 어려운 상황이 많습니다.

- ex) `BulkDelete`와 같이 `JPQL Query`를 저희가 직접 작성해야하는 경우

그런 상황에서 선택할 수 있는 방안은 크게 2가지가 존재합니다.

- **JPQL**을 사용하여 DB 데이터를 수정 및 삭제

- **JPA Query Method**를 활용하여 DB 데이터를 수정 및 삭제

해당 글에서는 위와 같은 예외가 일어나지 않으면서, **JPQL** 혹은 **JPA Query Method**를 통해 DB의 데이터를 수정 및 삭제하려면 어떻게 구성해야하는지 알아보도록 하겠습니다.

## JPQL을 사용한다면?

### 가정

> Service - `@Transactional 사용 X` <br>
>Repository - `@Modyfing 사용 X`
> 

아래와 같이 JPQL을 통해 디비에 쿼리를 날리는 경우에도 정상적으로 삭제가 되지 않는 문제가 발생합니다.

```java
@Query("DELETE FROM MemoryMember mm WHERE mm.memory.id = :memoryId")
void deleteAllByMemoryIdInBatch(@Param("memoryId") Long memoryId);
```

`Spring Data JPA`에서 JPQL로 작성된 **수정(UPDATE)** 또는 **삭제(DELETE)** 쿼리는 기본적으로 조회 쿼리로 간주되기 때문에 이 경우 **JPA는 변경 작업임을 인식하지 못하고 실행을 거부**합니다.

조회쿼리로 구성되기 때문에 `JPA Entity Manager`는 스냅샷은 찍지만 `flush`와 같은 데이터 반영 작업이 이루어지지 않기에, 더티체킹이 이루어지지 않게 됩니다.

따라서 해당 경우에는 `Service` 딴에 `@Transactional`을 붙여주어야하고, `Repository` 딴에도 `@Modyfing`을 붙여주어야 합니다. 

- `@Modifying`은 JPA에게 이 쿼리가 **변경 작업**임을 알려주며, **조회 쿼리로 처리되지 않도록 합니다**.

여기서 **중요한 부분**은 `@Modifying`은 **데이터 변경 작업**을 위한 애노테이션이며, 트랜잭션을 관리하지 않는다는 점입니다. 따라서, 데이터베이스 변경 작업이 올바르게 커밋되려면 **트랜잭션이 활성화**되어 있어야 합니다.

---

### 번외) JPQL은 항상 flush()를 유발할까?

- `JPQL`이 적용된 엔티티가 영속성 컨텍스트 내부에 존재하는 경우에만 특정 엔티티에 대해서 `flush()`가 일어남.
- `JPQL`이 적용된 엔티티가 아닌 다른 엔티티가 영속성 컨텍스트 내부에 존재하는 경우에는 다른 엔티티를 대상으로 `flsuh()`가 일어나지 않음.
- `@Modyfing`의 속성을 `flushAutomatically = true` 로 설정하게 되면 다른 엔티티가 영속성 컨텍스트 내부에 존재하더라도 모든 엔티티를 대상으로 `flush()`가 일어나게 됩니다.

---

### 결론

**@Transactional과 @Modyfing을 붙여주어야 합니다.**

```java
// Service
@Transactional
public void deleteMemory(long memoryId, Member member) {
...
}
```

```java
// Repository
@Modifying
@Query("DELETE FROM Moment m WHERE m.memory.id = :memoryId")
void deleteAllByMemoryIdInBatch(@Param("memoryId") Long memoryId);
```

## JPA Query Method를 사용한다면?

### 가정

> Service - `@Transactional 사용 X`
> 

아래와 같이 `JPA Query Method`를 사용하는 상황에서는 위의 `JPQL`를 사용하는 상황과 다르게 `JPA` 메서드 명명 규칙에 따라 자동으로 생성된 `deleteBy...()` 메서드는 `Spring Data JPA`가 **자동으로 데이터베이스 변경 작업으로 인식**하기 때문에 `@Modyfing`을 사용하지 않아도 됩니다.

```java
deleteByMemberIdAndId(loginMember.getId(), id);
```

하지만 JpaRepoistory 의 구현체의 delete 관련 메서드를 호출한 것이 아니라 우리가 직접 작성한 delete 관련 메서드를 호출한 것이기에 아래와는 다르게 @Transactional이 자동으로 붙어있지 않은 상태입니다. 

일반적으로 JPA에서 **트랜잭션이 생성되지 않으면 영속성 컨텍스트는 생성되지 않습니다**. JPA의 `EntityManager`는 트랜잭션과 밀접하게 연결되어 있으며, 대부분의 경우 **트랜잭션이 시작될 때 영속성 컨텍스트가 생성**됩니다. 하지만 트랜잭션이 생성되지 않았기에 영속성 컨텍스트가 생성되지 않았고, 그로인해 `JPA Entity Manager`가 디비에 쓰기작업을 할 때 트랜잭션이 활성화 되어 있지 않다는 예외가 발생하게 되었습니다.

리포지토리 메서드가 트랜잭션 내에서 실행되지 않으면 추가로 데이터의 일관성 문제가 생기거나 예외가 발생할 수 있기에 **@Transactional**을 붙여주어야 합니다.

> 아래와 같이 JPA 리포지토리 구현체에 자동으로 트랜잭션이 설정됩니다.
> 
![image](/img/blog/0204-1.png)

### 결론

**@Transactional을 붙여주어야 함, @Modyfing은 붙이지 않아도 됩니다.**
