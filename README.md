# JPA 스터디 내용정리
자바 ORM 표준 JPA 프로그래밍 - 김영한 님  

## 1장

### 패러다임의 불일치

객체지향 프로그래밍은 추상화, 캡슐화, 정보은닉, 상속, 다형성 등 시스템의 복잡성을 제어할 수 있는 다양한 장치들을 제공한다. 그래서 현대의 복잡한 애플리케이션은 대부분 객체지향 언어로 개발한다.

하지만 객체와 관계형 데이터베이스는 서로 지향하는 목적이 다르므로 둘의 기능과 표현방법이 다르다. 그렇기 때문에 이 패러다임 불일치 때문에 개발자가 중간에서 문제를 해결해야 한다. 이것이 리소스가 많이 든다.

예를들어, 상속 같은 경우 객체는 상속이라는 개념을 가지고 있지만 테이블은 없다. (일부 데이터 베이스는 상속이 있지만 객체의 상속과 다르다) 

객체의 상속구조를 저장하려면 객체를 분해해서 따로 저장해야 한다. 조회할 때도 마찬가지이다. 하나를 조회하고 다른 것을 조인을 해서 가져와야 한다. 

```java
list.add(album);
```

객체처럼 컬렉션으로 그냥 저장한다면 얼마나 좋을까?

그래서 JPA는 상속과 관련된 패러다임을 개발자 대신 해결해준다.

### 연관관계

객체는 참조를 사용해서 다른 객체와 연관관계를 표현한다. 테이블은 외래 키를 사용해서 연관관계를 표현한다. 객체는 참조가 있는 방향으로만 조회가능하지만 테이블은 외래키 하나로 양쪽에서 다 조회가능하다. 

이문제를 해결하려고 데이터베이스 테이블처럼 클래스를 이렇게 만들면,

```java
class Member {
	String id;
	Long teamId; // fk
...
}
```

```java
member.getTeam();
```

객체에서 Team을 가져올 수 없다. 

이 문제또한 해결하려면 개발자가 중간에서 연관관계를 다시 만들어 주어야 한다. JPA는 이걸 대신해준다.

### 그래프 탐색

객체는 그래프 탐색을 하면서 연관된 객체를 찾아나갈 수 있다. 하지만 SQL을 직접다루면 객체그래프를 어디까지 탐색해야하는지 정하기 애매하다. 다른말로 처음에 정해야한다. (어떤 메서드는 member만 조회 , 어떤 메서드는 member, team 까지 조회 등등)

JPA는 연관된 객체를 사용하는 시점에 쿼리를 날려서 데이터를 가져온다. 그렇기 때문에 메서드를 일일이 다 만들필요가 없다.

### 비교

SQL로 member를 가져오고 나서 추후에 한 번 더 가져온다고 생각해보자. 그 둘은 동일성(==) 비교에서 false가 나온다. 같은 데이터베이스 row에서 조회했지만 객체 측면에서 볼 때 둘은 다른 인스턴스다. 이건 우리가 원하는 결과가 아니다.

JPA는 같은 트랜잭션일 때 같은 객체가 조회되는 것을 보장한다. 

### JPA 사용이유

- 생산성: 반복적인 CRUD 코드를 만들 필요가 없음
- 유지보수: 필드를 추가나 삭제해도 수정해야할 코드가 줄어든다.
- 패러다임 불일치 해결
- 성능: 같은 트랜잭션 내에서 같은 member를 두 번 조회할 때 SQL을 사용했다면 디비 접근을 2번을 했어야 하지만 JPA를 썼으면 한번만 접근하고 두 번째는 이미 조회되었던 member를 이용하기 때문에 성능상 이점을 가질 수 있다.
- 데이터 접근 추상화와 벤더 독립성: JPA를 사용하면 로컬은 H2를 쓰고 개발환경은 MySQL 이나 Oracle을 쓸 수 있다.

## 2장

예제실습

- 엔티티 매니저 팩토리 생성

persistence.xml 에서 jpabook 이라는 영속성 유닛을 읽어서 엔티티 매니저 팩토리를 생성한다.

구현체에 따라서 커넥션 풀도 생성하므로 팩토리를 생성하는 비용이 아주크다. 그렇기 때문에 애플리케이션 전체에서 엔티티 매니저 팩토리를 딱 한 번만 생성하고 공유해서 사용해야 한다.

```java
/**
 * Interface used to interact with the entity manager factory
 * for the persistence unit.
 *
 * <p>When the application has finished using the entity manager
 * factory, and/or at application shutdown, the application should
 * close the entity manager factory.  Once an
 * <code>EntityManagerFactory</code> has been closed, all its entity managers
 * are considered to be in the closed state.
 *
 * @since Java Persistence 1.0
 */
public interface EntityManagerFactory {
```

주석 설명처럼 엔티티 매니저 팩토리가 닫혔다면 모든 엔티티 메니저 팩토리가 closed state 인걸로 간주한다.

- 엔티티 매니저 생성

JPA 기능의 대부분은 엔티티 매니저가 제공한다. 엔티티 매니저를 사용해서 엔티티를 데이터베이스에 CRUD할 수 있다. 엔티티 매니저는 내부의 데이터 소스 (db 커넥션)를 유지하면서 데이터베이스와 통신한다. 개발자는 엔티티 매니저를 가상의 데이터베이스로 생각하고 개발할 수 있다. 

엔티티 매니저는 데이터베이스 커넥션과 밀접한 관계가 있어서 스레드간 공유하거나 재사용하면 안된다.

- 종료

사용이 끝난 엔티티 매니저는 반드시 종료해야 한다. 그리고 엔티티 매니저 팩토리도 종료해야 한다.

- 트랜잭션 관리

JPA를 사용하려면 항상 트랜잭션 안에서 데이터를 변경해야 한다. 그렇기 때문에 엔티티 매니저에서 트랜잭션 api를 받아와야 한다.

- 수정

```java
//수정
member.setAge(20);

//한 건 조회
Member findMember = em.find(Member.class, id);
```

엔티티 수정시 JPA는 어떤 엔티티가 변경되었는지 추적한다. 그래서 따로 update()를 부를 필요가 없다.

- JPQL

애플리케이션이 필요한 데이터만 데이터베이스에서 불러오려면 결국에는 검색조건이 포함된 SQL을 사용해야 한다. 이것이 JPQL (Java Persistence Query Language)이다.

```java
List<Member> members = em.createQuery("select m from Member m", Member.class).getResultList();
```

이 조회 쿼리에서 Member는 테이블이 아니라 **엔티티 객체**를 의미한다. **JPQL은 데이터베이스 테이블을 전혀 알지 못한다**.

## 3장

### 엔티티 매니저 팩토리와 엔티티 매니저

데이터베이스를 하나만 사용하는 애플리케이션은 일반적으로 EntityManagerFactory를 하나만 생성한다. 이 팩토리는 여러 스레드가 동시에 접근해도 안전하지만 (다른 스레드 공유가능) 엔티티 매니저는 동시성 문제가 발생하므로 스레드간에 절대 공유하면 안된다.

엔티티 매니저는 데이터베이스 연결이 꼭 필요한 시점까지 커넥션을 얻지 않는다. 보통 트랜잭션을 시작할 때 커넥션을 획득한다. 하이버네이트는 엔티티 매니저 팩토리를 생성할 때 커넥션풀을 만드는데 J2EE(스프링 포함)환경에서는 해당 컨테이너가 제공ㅎ나느 데이터소스를 사용한다.

### 영속성 컨텍스트

엔티티를 영구저장하는 환경

> A persistence context is a set of entity instances in which for any persistent entity identity there is a unique entity instance

(여러 엔티티 매니저가 같은 영속성 컨텍스트에 접근할 수 있지만 일단은 엔티티 매니저 하나당 하나의 영속성 컨텍스트에 대응된다고 생각하자 - 추후에 나옴)

### 엔티티 생명주기

- 비영속(new/transient - 일시적인): 영속성 컨텍스트와 전혀 관계가 없는 상태

그냥 인스턴스만 new해서 생성한 상태

- 영속(managed): 영속성 컨텍스트에 저장된 상태

```java
entityManager.persist(엔티티);
```

- 준영속(detached): 영속성 컨텍스트에 저장되었다가 분리된 상태

```java
entityManager.detach(엔티티);
```

영속성 컨텍스트가 관리하지 않는 상태

- 삭제(removed): 삭제된 상태

### 영속성 컨텍스트 특징

- 영속성 컨텍스트는 엔티티를 식별자 값 (@Id)으로 식별하기 때문에 영속 상태는 반드시 식별자 값이 있어야 한다.
- flush 될 때 데이터베이스에 저장된다.
- 장점은 1차 캐시, 동일성 보장, 트랜잭션을 지원하는 쓰기 지연, 변경 감지, 지연 로딩

### 1차 캐시

영속성 컨텍스트는 내부에 캐시를 가지고 있는데 이것을 1차 캐시라 한다. 내부에 키가 @Id인 Map으로 관리한다. 이 식별자는 데이터베이스 기본 키와 매핑되어있다. 

```java
/**
 * Find by primary key.
 * Search for an entity of the specified class and primary key.
 * If the entity instance is contained in the persistence context,
 * it is returned from there.
 * @param entityClass  entity class
 * @param primaryKey  primary key
 * @return the found entity instance or null if the entity does
 *         not exist
 * @throws IllegalArgumentException if the first argument does
 *         not denote an entity type or the second argument is
 *         is not a valid type for that entitys primary key or
 *         is null
 */
public <T> T find(Class<T> entityClass, Object primaryKey);
```

primaryKey로 찾는 모습

em.find()를 호출하면 우선 1차 캐시에서 식별자 값으로 엔티티를 찾는다. 있다면 디비조회를 하지않고 메모리에 있는 1차 캐시에서 엔티티를 조회한다. 1차 캐시에 없다면 디비 조회 후 1차 캐시에 저장하고 반환한다.

⇒ 성능상 이점과 엔티티의 동일성을 보장한다. REPEATABLE READ 등급의 트랜잭션 격리 수준을 데이터베이스가 아닌 애플리케이션 차원에서 제공한다는 장점. 

엔티티 매니저는 트랜잭션을 커밋하기 직전까지 내부 쿼리 저장소에 sql을 모아둔다. 그리고 트랜잭션을 커밋할 때 모아둔 쿼리를 데이터베이스에 보낸다. (쓰기 지연)

데이터를 저장하는 즉시 등록 쿼리를 디비에 보내거나 모아서 한번에 보내거나 트랜잭션 커밋전까진 반영되지 않는다. 그렇기 때문에 마지막에 모아서 한방에 보내는게 더 효율적이다.

### SQL 수정 쿼리의 위험성

SQL을 사용해서 수정쿼리를 만들면 여러 개 만들다가 빼먹을 수 있다. 그리고 비지니스 로직을 분석하기 위해 쿼리를 계속 확인해야 하는 문제가 있다.

JPA는 엔티티의 수정을 자동으로 감지(Dirty Checking)한다. 엔티티를 영속성 컨텍스트에 보관할 때 최초상태의 스냅샷을 만들어 놓는다. 그리고 flush 시점에 스냅샷과 엔티티를 비교해서 변경된 엔티티를 찾는다. 

참고로 수정된 필드만 반영되는 것이 아니라 엔티티의 모든 필드를 업데이트한다.

디비 전송량이 증가하는 단점이 있지만 장점도 있다.

- 수정 쿼리가 일정하다. 애플리케이션 로딩 시점에 수정 쿼리를 미리 만들고 재사용할 수 있다.
- 디비에 동일한 쿼리를 보내면 데비는 이전에 한번 파싱된 쿼리를 재사용 할 수 있다.

### 다이나믹 업데이트

일반적으로 컬럼이 30개 이상이 되면 @DynamicUpdate를 이용한다. 이걸 쓰면 변경된 컬럼만 업데이트된다. (하이버네이트 확장기능) 근데 컬럼이 30개 된다고 하면 테이블 설계상 책임 분리가 안되었을 가능성이 높다 (함정..ㅋㅋ)

### 플러시

플러시는 영속성 컨텍스트의 변경 내용을 데이터베이스에 반영한다. 

- 엔티티 매니저 flush 직접 호출 (거의 안함)
- 트랜잭션 커밋 시 자동 호출 (JPA가 해줌)
- JPQL 쿼리 실행 시

JPQL 쿼리 실행 시 SQL로 변환되어 디비에서 엔티티를 조회할 것이다. 그런데 영속화 되어있는 (아직 플러시 하지않은) 인스턴스들이 디비에 반영이 안되어있다. 그렇기 때문에 쿼리 실행 직전에 플러시를 해서 변경된 내용을 디비에 반영한다. JPA는 이런 문제들을 방지하기 위해 JPQL 실행 시 플러시를 자동호출한다.

식별자 기준으로 찾는 find() 메서드는 호출 시 플러시 하지 않는다.

플러시 모드 (AUTO, COMMIT)이 있는데 추후 정리.

### 준영속

준영속 상태로 만들면 1차 캐시부터 쓰기 지연 저장소까지 다 제거한다. 

```java
em.detach();
em.clear();
```

detach는 특정 엔티티 하나를 준영속 상태로 만드는 것이고 clear는 영속성 컨텍스트 내의 모든 엔티티를 준영속 상태로 만든다. 

참고로 영속 상태의 엔티티는 주로 영속성 컨텍스트가 종료되면서 준영속 상태가 된다. 개발자가 직접 준영속 상태로 만드는 것은 드물다.

준영속 상태는 영속성 컨텍스트가 제공하는 어떤 기능도 동작하지 않는다. 하지만 식별자 값는 가지고 있다. 그리고 지연로딩시 문제가 발생한다. (관리되지 않는 엔티티이므로)

다시 영속상태로 변경하려면 병합(merge)을 사용해야 한다.

## 4장

### 기본내용

- Entity

@Entity 적용 시 기본 생성자 (파라미터가 없는 protected, public) 필수

final, enum, interace, inner 클래스에는 사용할 수 없다.

저장할 필드에 final을 사용하면 안된다.

ddl auto 전략

- create, create-drop, update, validate, none

운영에서는 스키마를 수정할 수 있는 옵션을 쓰면 안된다.

@Column 제약조건은 DDL 자동생성시에만 사용되고 JPA 실행로직과는 상관없다. 직접 DDL 만드는 경우는 사용할 필요는 없지만 이 기능을 사용하면 개발자가 엔티티만 보고도 제약조건을 파악할 수 있는 장점이 있다.

### 기본 키 매핑

- 직접 할당: 기본 키를 애플리케이션에서 직접 할당하는 방법.
- 자동생성

IDENTITY 전략

기본 키 생성을 데이터베이스에 위임하는 전략. 주로 MySQL, PostgreSQL 등에서 사용. MySQL에서는 auto_increment를 사용한다. 디비에 저장할 때 저 값을 비워주면 디비가 순서대로 값을 채워준다. 그렇기 때문에 데이터베이스에 값을 저장하고 나서야 기본 키 값을 구할 수 있다. ⇒ 기본 키 값을 얻어오기 위해 디비를 추가조회한다. 

참고) 이 전략은 insert 한 후에야 기본 키를 조회할 수 있는데 JDBC3에 추가된 Statement.getGeneratedKeys()라는걸 사용하면 저장하면서 동시에 키 값을 얻어올 수 있어서 디비와 통신을 한 번만 할 수 있다.

em.persist()를 호출하는 즉시 insert sql이 디비에 전달된다. 그래서 트랜잭션을 지원하는 쓰기 지연이 동작하지 않는다. 

### 식별자 선택 전략

자연 키 (비지니스에 의미가 있는 키: 주민번호 이메일 등)

대리 키 (비지니스와 관련없는 임의로 만들어진 키)

자연키 보다는 대리키를 권장한다. 왜냐하면 자연키, 예를들어 전화번호는 전화번호가 없을수도 있고 변경될 수 있다. 추가로 주민번호는 키로 그럴싸해보이지만 여러가지 이유로 변경될 수 있기 때문에 적합하지 않다.

### 필드와 컬럼 매핑

@Column 

컬럼 매핑

nullable: null 값 허용여부 설정

unique: 한 컬럼에 유니크 조건 걸 떄 사용. 두 컬럼 이상 시 클래스 레벨에서 @Table.uniqueConstraints를 사용해야 한다.

length: 문자 길이 제약 조건인데 String 타입에만 사용한다.

precision, scale: BigDecimal 타입에서 사용한다. double, float에서는 적용안되고 아주 큰 숫자나 정밀한 소수를 다룰 때 사용한다.

참고로 자바 원시형 타입은 null 값을 가질 수 없으므로 컬럼을 원시형으로 쓴다면 nullable=false 로 쓰는것이 안전하다. 

@Enumerated

자바의 enum 타입을 매핑할 때 사용한다.

Ordinal은 enum 순서를 매핑하는데 중간에 enum이 삽입되면 순서가 밀리게 된다. 기존 디비에 저장된 값은 그대로기 때문에 문제가 발생한다. 그래서 EnumType.String을 권장한다.

@Temporal

TemporalType.DATE: 디비의 date 타입 (2013-10-11)

TemporalType.TIME: 디비의 time (11:11:11)

TemproalType.TIMESTAMP: 디비의 timestamp (2013-10-11 11:11:11)

@Lob

디비의 BLOB, CLOB 타입과 매핑하는데 매핑하는 타입이 문자면 CLOB으로 하고 나머지는 BLOB으로 사용한다. 

@Transient

이 필드는 매핑하지 않는다.

@Access

JPA가 엔티티에 접근하는 방식을 지정한다.

필드접근: AccessType.FIELD

필드의 접근권한이 private이어도 직접 접근할 수 있따.

프로퍼티 접근: AccessType.PROPERTY

접근자(getter)를 사용한다.

## 5장 연관관계 매핑 기초

- 양방향: 양쪽 모두 서로 참조 하는 것
- 단방향: 한쪽만 참조하는 것
- 연관관계의 주인: 객체를 양방향 연관관계로 만들면 연관관계의 주인을 정해야 한다.

참조를 통한 연관관계는 언제나 단방향이다. 양방향으로 만들고 싶으면 각자 단방향 한 개씩 총 2개를 만들어야 한다.

### 객체 연관관계 vs 테이블 연관관계

- 객체 연관관계: 객체는 참조로 연관관계를 맺는다.
- 테이블 연관관계: 테이블은 외래 키로 연관관계를 맺는다.

### 객체 관계 매핑

@JoinColumn: 조인 컬럼은 외래키를 매핑할 때 사용한다. 

이거를 생략을 하면 기본전략을 사용한다.

기본전략: 필드명 + (언더바) + 참조하는 테이블의 컬럼명

```java
@ManyToOne
private Team team;

team_TEAM_ID
```

### 조회

- 객체 그래프 탐색

```java
member.getTeam();
```

member와 연관된 team 엔티티를 조회할 수 있다.

- 객체 지향 쿼리 사용

```java
String jpql = "select m from Member m join m.team t where t.name=:teamName
```

실제 수행되는 SQL 문보다 간결하게 쓸 수 있다.

### 양방향 연관관계

Member.class

```java
@ManyToOne
@JoinColumn(name="TEAM_ID")
private Team team;
```

Team.class

```java
@OneToMany(mappedBy="team")
private List<Member> members = new ArrayList<>();
```

팀과 회원은 일대다 관계이므로 컬렉션을 추가했다. 

### 연관관계의 주인

양방향은 서로 다른 단방향 두개 이므로 엄밀히 보면 양방향 연관관계는 없다. 하지만 앞서 언급한 것 처럼 디비 테이블은 외래 키 하나로 양방향 연관관계를 표현할 수 있다.

객체 단방향 매핑에서는 연관관계를 관리하는 부분이 한 곳이기 때문에 이 참조로 외래키를 관리하면 되었지만 양방향에서는 참조가 두 곳이므로 차이가 발생한다. 그렇기 때문에 한 곳을 정해서 외래키를 관리하게 해야하는데 이것이 연관관계의 주인이라고 한다.

연관관계의 주인만이 디비 연관관계에 매핑되고 외래 키를 관리(등록 수정 삭제)할 수 있다. → 주인이 아닌 쪽에 mappedBy 설정.

Member 가 외래 키를 관리하면 Member 테이블 (자기 테이블) 에 있는 외래 키를 관리하면 된다. 하지만 Team이 외래 키를 관리하면 물리적으로 다른 테이블에 있는 외래 키를 관리해야 한다. 주인이 아닌 쪽은 읽기만 가능하고 외래 키 변경은 할 수 없다.

참고로 항상 다 쪽이 외래키를 가진다. 그렇기 때문에 @ManyToOne에는 mappedBy 속성이 없다.

### 양방향의 주의점

```java
team.getMembers().add(member1);
```

주인이 아닌 쪽에 값을 입력하면 무시된다. 

흔히 하는 실수는 연관관계의 주인에는 값을 입력하지 않고 주인이 아닌쪽에 값을 입력하는 것이다. 

```java
Memeber member = new Member("...");

Team team = new Team("...");
team.getMembers().add(member);
```

디비조회를 해보면 member의 team은 null이다. 연관관계의 주인이 아닌쪽에서만 값을 지정했기 때문이다.

### 실제 쓰임새

객체 관점에서는 양쪽 방향에 값을 다 입력해주어야 안전하다. JPA를 사용하지 않고 코드 단에서만 테스트할 때 문제가 발생한다. 그렇기 때문에 코드를 작성할 때는 양쪽 다 해주자.

### 연관관계 편의 메서드

양쪽 다 설정을 해주어야 하는 상황에서 실수로 빼먹을 수 있다. 그렇기 때문에 setter 메서드에서 편의메서드를 작성한다.

```java
public void setTeam(Team team) {
	this.team = team;
	team.getMembers().add(this);
}
```

라고만 생각하면 오산이다. (버그가 있다)

```java
public void setTeam(Team team) {
	if (this.team != null) {
		this.team.getMembers().remove(this);
	}
	this.team = team;
	team.getMembers().add(this);
}
```

바로 위처럼 삭제되지 않은 연관관계를 제거하는것이 빠졌다. 기존의 관계가 있다면 그것을 먼저 끊어주고나서 새로운 관계를 맺어야한다.

그렇기 때문에 객체에서 양방향 관계를 사용하면 로직을 견고하게 작성해야 한다. 위의 예제에서 team은 연관관계의 주인이 아니기 때문에 디비에 반영이 되지는 않는다. 연관관계의 주인인 member가 team을 참조하게 바뀌었으므로 정상반영이 되기는 한다. 하지만 반영이후 영속성 컨텍스트가 살아있는 상태에서 team의 getMember를 하면 기존의 member가 반환된다. 그렇기 때문에 제거하는것이 좋다.

### 결론

양방향의 장점은 반대방향으로 객체 그래프 탐색이 가능하다는 것뿐이다. 단방향 매핑만으로도 테이블과 객체의 연관관계 매핑은 완료된다. 양방향을 사용하려면 객체 양쪽에서 관리가 필요하다.

중요한 점은 비지니스 중요도를 생각해서 연관관계의 주인을 정하면 안된다. 단지 외래 키 관리자 정도의 의미를 부여해야 한다. 그리고 서로를 호출하지 않게 (무한루프) 조심해야 한다.
## 5장 연관관계 매핑 기초
                                                                                                     
 - 양방향: 양쪽 모두 서로 참조 하는 것
 - 단방향: 한쪽만 참조하는 것
 - 연관관계의 주인: 객체를 양방향 연관관계로 만들면 연관관계의 주인을 정해야 한다.
 
 참조를 통한 연관관계는 언제나 단방향이다. 양방향으로 만들고 싶으면 각자 단방향 한 개씩 총 2개를 만들어야 한다.
 
 ### 객체 연관관계 vs 테이블 연관관계
 
 - 객체 연관관계: 객체는 참조로 연관관계를 맺는다.
 - 테이블 연관관계: 테이블은 외래 키로 연관관계를 맺는다.
 
 ### 객체 관계 매핑
 
 @JoinColumn: 조인 컬럼은 외래키를 매핑할 때 사용한다. 
 
 이거를 생략을 하면 기본전략을 사용한다.
 
 기본전략: 필드명 + (언더바) + 참조하는 테이블의 컬럼명
 
 ```java
 @ManyToOne
 private Team team;
 
 team_TEAM_ID
 ```
 
 ### 조회
 
 - 객체 그래프 탐색
 
 ```java
 member.getTeam();
 ```
 
 member와 연관된 team 엔티티를 조회할 수 있다.
 
 - 객체 지향 쿼리 사용
 
 ```java
 String jpql = "select m from Member m join m.team t where t.name=:teamName
 ```
 
 실제 수행되는 SQL 문보다 간결하게 쓸 수 있다.
 
 ### 양방향 연관관계
 
 Member.class
 
 ```java
 @ManyToOne
 @JoinColumn(name="TEAM_ID")
 private Team team;
 ```
 
 Team.class
 
 ```java
 @OneToMany(mappedBy="team")
 private List<Member> members = new ArrayList<>();
 ```
 
 팀과 회원은 일대다 관계이므로 컬렉션을 추가했다. 
 
 ### 연관관계의 주인
 
 양방향은 서로 다른 단방향 두개 이므로 엄밀히 보면 양방향 연관관계는 없다. 하지만 앞서 언급한 것 처럼 디비 테이블은 외래 키 하나로 양방향 연관관계를 표현할 수 있다.
 
 객체 단방향 매핑에서는 연관관계를 관리하는 부분이 한 곳이기 때문에 이 참조로 외래키를 관리하면 되었지만 양방향에서는 참조가 두 곳이므로 차이가 발생한다. 그렇기 때문에 한 곳을 정해서 외래키를 관리하게 해야하는데 이것이 연관관계의 주인이라고 한다.
 
 연관관계의 주인만이 디비 연관관계에 매핑되고 외래 키를 관리(등록 수정 삭제)할 수 있다. → 주인이 아닌 쪽에 mappedBy 설정.
 
 Member 가 외래 키를 관리하면 Member 테이블 (자기 테이블) 에 있는 외래 키를 관리하면 된다. 하지만 Team이 외래 키를 관리하면 물리적으로 다른 테이블에 있는 외래 키를 관리해야 한다. 주인이 아닌 쪽은 읽기만 가능하고 외래 키 변경은 할 수 없다.
 
 참고로 항상 다 쪽이 외래키를 가진다. 그렇기 때문에 @ManyToOne에는 mappedBy 속성이 없다.
 
 ### 양방향의 주의점
 
 ```java
 team.getMembers().add(member1);
 ```
 
 주인이 아닌 쪽에 값을 입력하면 무시된다. 
 
 흔히 하는 실수는 연관관계의 주인에는 값을 입력하지 않고 주인이 아닌쪽에 값을 입력하는 것이다. 
 
 ```java
 Memeber member = new Member("...");
 
 Team team = new Team("...");
 team.getMembers().add(member);
 ```
 
 디비조회를 해보면 member의 team은 null이다. 연관관계의 주인이 아닌쪽에서만 값을 지정했기 때문이다.
 
 ### 실제 쓰임새
 
 객체 관점에서는 양쪽 방향에 값을 다 입력해주어야 안전하다. JPA를 사용하지 않고 코드 단에서만 테스트할 때 문제가 발생한다. 그렇기 때문에 코드를 작성할 때는 양쪽 다 해주자.
 
 ### 연관관계 편의 메서드
 
 양쪽 다 설정을 해주어야 하는 상황에서 실수로 빼먹을 수 있다. 그렇기 때문에 setter 메서드에서 편의메서드를 작성한다.
 
 ```java
 public void setTeam(Team team) {
    this.team = team;
    team.getMembers().add(this);
 }
 ```
 
 라고만 생각하면 오산이다. (버그가 있다)
 
 ```java
 public void setTeam(Team team) {
    if (this.team != null) {
        this.team.getMembers().remove(this);
    }
    this.team = team;
    team.getMembers().add(this);
 }
 ```
 
 바로 위처럼 삭제되지 않은 연관관계를 제거하는것이 빠졌다. 기존의 관계가 있다면 그것을 먼저 끊어주고나서 새로운 관계를 맺어야한다.
 
 그렇기 때문에 객체에서 양방향 관계를 사용하면 로직을 견고하게 작성해야 한다. 위의 예제에서 team은 연관관계의 주인이 아니기 때문에 디비에 반영이 되지는 않는다. 연관관계의 주인인 member가 team을 참조하게 바뀌었으므로 정상반영이 되기는 한다. 하지만 반영이후 영속성 컨텍스트가 살아있는 상태에서 team의 getMember를 하면 기존의 member가 반환된다. 그렇기 때문에 제거하는것이 좋다.
 
 ### 결론
 
 양방향의 장점은 반대방향으로 객체 그래프 탐색이 가능하다는 것뿐이다. 단방향 매핑만으로도 테이블과 객체의 연관관계 매핑은 완료된다. 양방향을 사용하려면 객체 양쪽에서 관리가 필요하다.
 
 중요한 점은 비지니스 중요도를 생각해서 연관관계의 주인을 정하면 안된다. 단지 외래 키 관리자 정도의 의미를 부여해야 한다. 그리고 서로를 호출하지 않게 (무한루프) 조심해야 한다.
 
 
 ## 6장 다양한 연관관계 매핑
 
 엔티티의 연관관계를 만들 때 고려해야할 3가지
 
 - 다중성
 - 단방향, 양방향
 - 연관관계의 주인
 
 ### 다중성
 
 @ManyToOne, @OneToMany, @OneToOne, @ManyToMany
 
 실무에서 다대다는 거의 사용하지 않는다.
 
 ### 일 대 다 단방향 매핑?
 
 일 대 다 단방향 매핑은 반대쪽 테이블에 있는 외래 키를 관리하게 된다. 
 
 ```java
 @OneToMany
 @JoinColumn(name="TEAM_ID")
 private List<Member> members =new ArrayList();
 ```
 
 일대다 단방향 매핑은 @JoinColumn을 명시하지 않으면 JPA가 연결테이블을 만들어버린다. (조인테이블 자동생성)
 
 단점으로는 외래 키가 본인 테이블에 있으면 엔티티의 저장과 연관관계 처리를 insert SQL 한방에 끝낼 수 있는데 다른 테이블에 외래 키가 있기 때문에 update SQL이 한 번 더 필요하다.
 
 → 이럴 바에 (관리하기 어려우므로) 다 대 일 양방향 매핑을 하는게 낫다.
 
 ### 일 대 일
 
 일대일 관계는 양쪽이 서로 하나의 관계만 가진다. 일대일 관계에서는 어느쪽이든 외래 키를 가질 수 있다.
 
 - 주 테이블에 외래 키
 
 객체지향 개발자들이 주로 사용. 주 테이블만 확인해도 대상 테이블과의 연관관계를 알 수 있다.
 
 단방향, 양방향 모두 가능하다.
 
 Member.class
 
 ```java
 @OneToOne
 @JoinColumn(name="LOCKER_ID")
 private Locker locker;
 ```
 
 Member쪽에만 설정하면 일대일 단방향이다. 
 
 추가로 Locker쪽에도 설정해주면 일대일 양방향이다.
 
 ```java
 @OneToOne(mappedBy="locker")
 privtae Member member;
 ```
 
 양방향이므로 주인을 설정하면 된다.
 
 갑자기 의문점?: 일대일이면 양방향으로 하는게 더 편리하지 않을까? 대상쪽에서 주쪽을 참조할일이 아예없는 비지니스로직이라면 단방향으로 하고 그러는건가?
 
 - 대상 테이블에 외래 키
 
 전통적인 디비 개발자들은 보통 대상 테이블에 외래 키를 두는 것을 선호한다. 장점은 테이블 관계를 일대일에서 일대다로 변경할 때 테이블 구조를 그대로 유지할 수 있다.
 
 일대일에서 단방향 대상 테이블매핑은 JPA가 지원하지 않는다. 
 
 양방향을 보면,
 
 Member.class
 
 ```java
 @OneToOne(mappedBy="member")
 private Locker locker;
 ```
 
 Locker.class
 
 ```java
 @OneToOne
 @JoinColumn(name="MEMBER_ID")
 private Member member;
 ```
 
 ### 다대다
 
 조인 테이블이 쉽게 만들어지지만 실무에서는 보통 조인 테이블에서 끝나는 것이 아니라 그 테이블에 필드가 더 필요한 경우가 많다. 이 상태에서 추가한 필드는 매핑할 수 없다. 그렇기 때문에 일대다, 다대일 관계로 풀어야한다.
 
 ### 복합 기본키
 
 회원과 상품이 있을 때 둘은 다대다 관계이다. 그래서 회원_상품이라는 연결을 담당하는 엔티티를 만든다.
 
 그 엔티티의 식별자에 관한 내용
 
 ```java
 @Entity
 @IdClass(MemberProductId.class)
 public class MemberProduct {
 	@Id
 	@ManyToOne
 	@JoinColumn(name="MEMBER_ID")
 	private Member member;
 
 	@Id
 	@ManyToOne
 	@JoinColumn(name="PRODUCT_ID")
 	private Product product;
 }
 ```
 
 MemberProductId.class
 
 ```java
 public class MemberProductId implements Serializable {
 	private String member;
 	private String product
 
 	// equals and hashcode 
 }
 ```
 
 회원 상품 엔티티를 보면 @IdClass를 이용해 복합 기본키를 매핑했다.
 
 복합 기본키는 JPA에서 사용하려면 별도의 식별자 클래스를 만들어야 한다. 그리고 Serializable 을 구현해야하고 equals and hashcode 메서드 구현, 기본 생성자 필요, 클래스 public 이라는 제약조건들이 있다. 그렇기 때문에 새로운 기본 키를 만드는것을 추천한다.
 
 MemberProduct 라는 이름 대신 Order 라는 이름으로 변경을 하고 그 만의 @Id 를 둔다.
 
 ```java
 public class Order {
 	@Id @GeneratedValue
 	@Column(name="ORDER_ID")
 	private Long id;
 
 	@ManyToOne
 	@JoinColumn(name="MEMBER_ID")
 	privtae Member member;
 
 	@ManyToOne
 	@JoinColumn(name="PRODUCT_ID")
 	private Product product;
 }
 ```
 
 대리키를 사용함으로써 훨씬 단순해졌다. 기존의 복합 기본키를 사용하는 것은 각 테이블의 주키를 가져와서 pk, fk역할을 하는 키가 두 개 생겼었다.→식별관계
 
 하지만 지금은 그 키들을 단순히 외래 키의 역할만하게 하고 주 키를 새로 만들었다. → 비 식별 관계