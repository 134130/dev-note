# Get vs Find
.NET 및 C#을 사용하며 Microsoft에서 기본적으로 제공하는 API들은 대부분 get/find prefix 컨벤션을 강하게 지키고 있다.\
때문에 나에게는 해당 네이밍 컨벤션이 너무나도 당연한 것이었으나, Java/Kotlin 생태계로 넘어오며 생각보다 모르는 사람들이 많다는걸 느꼈다.\
get/find prefix를 사용하지 않고 `throwIfNull`, `getOrNull` 등의 suffix를 사용하는 사람들에게 이 글을 보여주면 좋겠다.

## Summary
- get: 데이터를 가져온다. (무조건 데이터를 가져오며, 데이터가 없는 경우 오류가 발생한다.)
- find: 데이터를 찾는다. (데이터를 찾아보며, 데이터가 없을 수 도 있다.)

> find는 '찾아보다' 라는 뜻과 같이 조회 시간이 오래걸릴 수 도 있음을 내포하고 있으나, 나는 동작상 오류의 발생 유무를 더 중요하게 생각한다.

## Description
실제 API들이 제공하고 있는 형태를 살펴보자.

### Java Spring
- `springframework.data.jpa`의 `JpaRepository`에서는 다음과 같이 정의하고 있다 ([Spring Docs](https://docs.spring.io/spring-data/jpa/docs/current/api/org/springframework/data/jpa/repository/JpaRepository.html))
  ```java
  public interface JpaRepository {
      T getOne(ID id);
      List findAll();
  }
  ```
  > `getOne(ID id)`는 `T`를 리턴하고 있다.<br>
  > 해당 메서드의 JavaDoc @throws 를 따라가면, 엔티티가 없는 경우 오류를 던지고 있음을 알 수 있다. ([jakarta Docs](https://jakarta.ee/specifications/persistence/3.1/apidocs/jakarta.persistence/jakarta/persistence/entitymanager#getReference(java.lang.Class,java.lang.Object)))
  > - `IllegalArgumentException` - if the first argument does not denote an entity type or the second argument is not a valid type for that entity's primary key or is null
  > - **`EntityNotFoundException` - if the entity state cannot be accessed**

- `springframework.data`의 `CrudRepository`에서는 다음과 같이 정의하고 있다 ([Spring Docs](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html))
  ```java
  public interface CrudRepository<T, ID> extends Repository<T, ID> {
      Optional<T> findById(ID primaryKey); 
      Iterable<T> findAll();
  }
  ```
  > `findById(ID primaryKey)`는 `Optional<T>`을 리턴하고 있다.<br>
  > 해당 메서드의 JavaDoc @throws 에는 해당 오류가 없다.
  > - `IllegalArgumentException` – if id is null.


### C#
C#은 많은 Collection API를 언어적 레벨에서 제공하고 있으며, 이러한 컨벤션을 잘 지키고 있다.

- `Find`는 `T?`을 리턴한다. ([List<T>.find()](https://learn.microsoft.com/ko-kr/dotnet/api/system.collections.generic.list-1.find?view=net-7.0))\
  ```csharp
  public T? Find (Predicate<T> match);
  ```
  > 주어진 match 조건을 통해 탐색하며 검색하기 때문에 조회시간이 오래걸릴 수 도 있으며, 값이 없을 수 도 있다.
- `GetValueAtIndex(int index)`는 `TValue`를 리턴한다. ([SortedList<TKey, TValue>.GetValueAtIndex(Int32)](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.sortedlist-2.getvalueatindex?view=net-7.0))\
  ```csharp
  public TValue GetValueAtIndex (int index);
  ```
  > C#에서 Exception을 내부적으로 처리하고 싶은 경우에는 Try-pattern을 사용한다. ([SortedList<TKey,TValue>.TryGetValue(TKey, TValue)](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.sortedlist-2.trygetvalue?view=net-7.0))
  > ```csharp
  > public bool TryGetValue (TKey key, out TValue value);
  > ```
  > 위의 경우 true인 경우에만 value를 사용해야하며, false인 경우는 value를 사용해서는 안된다. 때문에 `TValue`는 non-null 이다.
- `GetValueOrDefault`는 `TValue?`를 리턴한다. ([CollectionExtensions.GetValueOrDefault](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.collectionextensions.getvalueordefault?view=net-7.0))\
  ```csharp
  public static TValue? GetValueOrDefault<TKey,TValue> (
      this System.Collections.Generic.IReadOnlyDictionary<TKey,TValue> dictionary,
      TKey key
  );
  ```
  > 데이터를 조회하는 시간이 오래걸리지 않기 때문에 `Get`을 사용하며, `OrDefault`를 통해 값이 없을 수 도 있음을 알려주고 있다.

## Conclusion
get / find 컨벤션을 잘 지킨다면, `getElementOrThrowIfNotExists`, `getElementOrNull` 등의 메서드 명이 불필요하게 길어지지 않고도 동작을 충분히 설명할 수 있다.\
물론 탐색성 성격을 나타내는 find와 즉발성 성격을 나타내는 get의 특징도 잘 활용할 수 있으면 좋겠지만, 팀원들간의 합의를 통해 오류를 동작에 대한 컨벤션을 먼저 적용해보면 어떨까?

> JpaRepository의 `findAll()` 메서드는 `List?`가 아닌 `List`를 리턴하고 있다.
> null-List와 empty-List의 차이는 뭘까?
> 
> 메서드명을 잘 작성해 두면 이후 코드를 작성하는 생산성이 크게 올라갈 수 있다.
> API를 사용할 때 왜 이런 메서드명이 되었는지 다시 한 번 고민해보고, 나의 메서드를 작성할 때 적용할 수 있는 사람이 되자.

## Related
- https://tuhrig.de/find-vs-get/
- [좋은 함수 만들기 - Null 을 다루는 방법](https://jojoldu.tistory.com/721)https://jojoldu.tistory.com/721
- [Number와 boolean 은 최대한 Not Null로 선언하기](https://jojoldu.tistory.com/718)
