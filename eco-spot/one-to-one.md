[JPA] @OneToOne 에 지연 로딩을 적용했지만, 왜 지연 로딩이 안되고 즉시 로딩이 되는 걸까요?
=============================================================

혹시나 마크다운 문법이 보기 불편하시다면 원글에 해당하는 블로그 글을 [여기](https://medium.com/@gunhong951/jpa-onetoone-양방향-연관관계에서-나는-lazy-전략을-적용했다-그런데-왜-lazy로딩이-안되고-eagle-로딩이-되는-것인가-c9710dc82257)에서 확인하실 수 있습니다.

> 일대일 양방향 관계에서 Lazy loading이 적용되지 않는 문제가 발생. 원인을 분석한 결과 외래키를 관리하는 엔티티가 아닌, 반대 방향의 엔티티에서 조회 작업을 수행할 경우 연관관계 확인을 위해 불필요한 쿼리가 발생하는 것을 확인.
>
> 이를 해결하기 위해 불필요한 양방향 관계를 단방향 관계로 변경하여 필요 없는 쿼리 발생을 방지.

프로젝트를 진행하며 쿼리를 살펴보던 중 이상한 점을 발견한 적이 있습니다.
분명 A와 B 라는 서로 다른 엔티티의 연관관계 로딩전략을 Lazy 로 하였는데, 왜 A객체를 조회할 때 B객체도 함께 로딩하는 쿼리가 나가는 것일까요?

JPA 를 사용할 경우 ~~ToOne 관계에서 기본 로딩 전략.
===================================

JPA 를 사용할 경우 ~~ToOne 관계인 경우 로딩전략의 기본 값은 FetchType.EAGER 로 되어있습니다. 하지만 이런 전략은 N+1 문제와 같이 의도하지 않은 쿼리가 나가는 경우가 있어서 FetchType.LAZY 로 설정해서 사용하는 것을 권장합니다.
하지만 분명 FetchType.LAZY 로 설정하였음에도 의도하지 않은 쿼리가 발생했습니다. 도대체 왜 그런걸까요? 이 문제는 JPA가 지연로딩(LazyLoading) 을 위하여 **프록시** 객체를 사용하기 때문에 발생합니다.

이제 예제를 통해 살펴보자!
===============

먼저 설명을 위해 살펴볼 예제 DB 테이블은 아래와 같습니다.
MISSION 테이블에서 SPECIES 의 기본 키 값을 외래 키로 지니고 있습니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*KYTreSkmVShPaWhTjm3wTQ.png" width=70%>

JPA 엔티티는 아래와 같이 정의하였습니다.
Mission 엔티티에서 @JoinColumn 을 이용하여 species_id 값을 외래 키로 설정합니다. 그리고 서로 양방향 참조를 가지고 있습니다.

```java
@Entity
public class Species {
    @Id
    @Column(name = "species_id")
    private Long id;
    
    // FetchType.LAZY 적용.
    // Mission 엔티티의 species 필드를 mappedBy 해서 연관관계의 주인으로 적용.
    @OneToOne(fetch = FetchType.LAZY, mappedBy = "species")
    private Mission mission;
    private String speciesName;
}
```
```java
@Entity
public class Mission {
    @Id
    @Column(name = "mission_id")
    private Long id;
    
    @OneToOne(fetch = FetchType.LAZY)  // FetchType.LAZY 적용
    @JoinColumn(name = "species_id")
    private Species species;
    private String missionName;
}
```

글을 설명하기에 앞서 한가지 체크하고 넘어가야 할 부분…
===============================

바로 DB테이블과 Java 객체의 연관관계 페러다임 불일치 문제입니다.
아시는 분도 계시겠지만 혹시나 잘 모르실 수 있기 때문에 간략하게 설명을 추가하겠습니다.

**DB테이블**은 주 테이블이든 대상 테이블이든 어느 한쪽에라도 외래 키가 있으면 양쪽으로 조회할 수 있습니다. 이러한 관계를 **양방향 관계**라고 합니다.
아래 두개의 SQL 문은 모두 가능합니다.

```sql
select * from species s join mission m on s.species_id = m.species_id
select * from mission m join species s on m.species_id = s.species_id
```

하지만 **객체**는 참조를 사용해서 다른 객체와 연관관계를 가지고 참조에 접근해서 연관된 객체를 조회합니다. Species 객체에 Mission 객체의 참조가 존재하면 Mission 객체로 접근할 수 있지만, Mission 객체에 Species 객체에 해당하는 참조가 존재하지 않으면 Mission 객체는 Species 객체로 접근할 수 없습니다. 이러한 **단방향 관계**에서 양쪽 객체에 서로의 참조를 설정하고 한 객체에서 외래키를 관리한다면 두 객체는 데이터베이스와 같이 **양방향 관계로** 표현할 수 있습니다. (실제로는 단방향 관계 2개입니다.)

JPA 와 같은 ORM 기술은 이러한 테이블과 객체간의 연관관계 페러다임 불일치와 같은 여러 문제들을 해결하기 위해 나온 기술입니다.

다시 예제로 돌아가자.
============

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*LOnMZvGT3fv3igH-2IB7Jw.png" width=70%>

위의 그림을 살펴보면, 양쪽 객체는 서로 참조하는 필드를 가지고 있고 테이블은 일대일 관계를 맺고 있습니다. Mission 객체가 연관관계의 주인으로서 외래 키를 관리하고 MISSION 테이블과 매핑되어 있습니다.

객체는 객체 그래프(서로 다른 객체간의 참조를 통해 연결된 체인이 만드는 네트워크)로 연관된 객체들을 탐색할 수 있습니다. 그런데 객체가 데이터베이스에 저장되어 있으므로 연관된 객체를 마음껏 탐색하기는 어렵습니다.

그렇다면 어떻게 해야 객체가 연관된 객체를 마음껏 탐색할 수 있을까?
======================================

JPA는 이 문제를 해결하기 위해서 연관관계에 있는 객체들을 한 번에 DB에서 조회하여 서로 탐색이 가능하게 최적화를 합니다. 하지만 이러한 방법은 예기치 못한 쿼리가 발생할 수 있으며, 때로는 조회하는 데 비용이 클 수 있습니다. JPA 는 이 문제를 해결하기 위해서 **프록시**라는 기술을 사용합니다. 그리고 연관된 객체를 처음부터 데이터베이스에서 조회하는 것이 아니라 실제 사용하는 시점에 데이터베이스에서 조회할 수 있습니다. 이러한 방식을 **지연로딩(Lazy Loading)** 이라 합니다. 지연 로딩 기능을 사용하려면 실제 엔티티 객체 대신에 DB 조회를 지연할 수 있는 가짜 객체가 필요한데 이것을 **프록시 객체**라 합니다. 프록시 객체는 DB 접근을 위임 받았으며 실제 사용될 때 DB를 조회해서 **실제 엔티티 객체**를 생성합니다.

코드를 통해서 살펴보자.
=============

본 예제에서 조회되는 객체가 실제 엔티티인지 프록시인지 테스트하기 위하여 아래와 같은 코드를 작성 후 실행해 보았습니다.
먼저, 연관관계의 주인인 Mission 엔티티를 DB에서 조회하고 Species 엔티티는 참조(reference)를 통해 조회해 보았습니다.

```java
Long missionId = 1001L;
Long speciesId = 9001L;

Mission newMission = new Mission(missionId,"미션1"); // id, missionName
Species newSpecies = new Species(speciesId, "종1");  // id, speciesName
newMission.setSpecies(newSpecies);

speciesRepository.save(newSpecies); // species 을 먼저 저장해야 한다. (외래키 제약조건)
missionRepository.save(newMission);

entityManager.clear();

Mission mission = missionRepository.findById(missionId).get();

String missionClassName = mission.getClass().getName();
String speciesClassName = mission.getSpecies().getClass().getName();

System.out.println("missionClassName = " + missionClassName);
System.out.println("speciesClassName = " + speciesClassName);
```

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*iD6GJaNvOPjoUXE3ulUh0Q.jpeg" width=70%>

Species 엔티티는 **프록시 객체**가 생성된 것을 확인할 수 있습니다.
이번에는 반대로, 연관관계의 주인이 아닌 Species 엔티티를 DB에서 조회하고 Mission 엔티티를 참조를 통해 조회해 보겠습니다.

```java
Species species = speciesRepository.findById(species.getId()).get();

String speciesClassName = species.getClass().getName();
String missionClassName = species.getMission().getClass().getName();

System.out.println("speciesClassName = " + speciesClassName);
System.out.println("missionClassName = " + missionClassName);
```
<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*fgD9hO848cC5JjzQV9sdBA.jpeg" width=70%>

두 실행 코드는 서로 다른 엔티티를 DB에서 조회한다는 점을 제외하고 구조가 완전히 동일합니다. 각 엔티티는 OneToOne 연관관계를 매핑할 때 FetchType.LAZY 을 적용하였지만, Species 엔티티를 DB에서 조회할 때 Mission 엔티티의 조회 쿼리도 함께 발생하였고 Mission 객체는 프록시가 아닌 **실제 엔티티**가 생성되었습니다.

왜 FetchType.LAZY 을 설정하였는데 무시당하고 FetchType.EAGER 이 적용될까요?
========================================================

프록시를 사용하여 지연로딩을 적용할 때, 외래 키를 직접 관리하지 않는 객체의 일대일 관계는 지연 로딩으로 설정해도 즉시 로딩합니다.
이 문제는 **프록시의 한계** 때문에 발생하는 문제입니다.

**진짜 엔티티 객체**의 **가짜 객체 프록시**를 생성할 때는 **진짜 엔티티**에 대한 정보를 가지고 있어야 합니다. 가짜 객체가 진짜 객체인 척 하기위해 프록시를 사용하는데, 진짜 객체가 존재하지 않는다면 가짜 객체의 존재 의미는 사라지게 됩니다. 따라서 엔티티를 프록시로 조회할 때 진짜 엔티티의 식별자 값(PK)을 파라미터로 전달하는데 프록시 객체는 **이 식별자 값(PK)**을 정보로 가지고 있습니다.

실제로

```java
Mission mission = missionRepository.findById(missionId).get()
```

코드를 이용하여 조회한 Mission 엔티티를 디버거를 통해 살펴보면 위에서 speciesId 값(PK)으로 정의한 9001L 을 Species 프록시 객체가 가지고 있는 것을 확인할 수 있습니다. 이 정보를 이용하여 실제 사용될 때 Species 엔티티를 초기화합니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*VM-LC-UCWJHfcC6PtLs_1w.jpeg" width=70%>

이제 위에서 살펴봤던 객체와 테이블 이미지를 다시한번 살펴보겠습니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*LOnMZvGT3fv3igH-2IB7Jw.png" width=70%>

Mission 엔티티는 실제 사용할 때 DB 에서 MISSION 테이블을 조회합니다.
MISSION 테이블에는 외래 키를 관리하는 컬럼이 존재해서 테이블 조회 만으로 SPECIES 테이블과 연관관계가 있는 것을 알 수 있습니다. 이를 통해 species_id 값을 얻거나 연관된 데이터가 없음(null) 을 알 수 있습니다.
(참고로 @OneToOne에서 기본 optional 값은 true 입니다.
만약 false 로 설정하면 반드시 연관관계가 존재해야함 -> 객체의 필드값에 null 이 안됨.)

하지만 Species 엔티티는 실제 사용될 때 SPECIES 테이블을 조회하는 것으로 MISSION 테이블과 연관관계가 있는지 알 수 없습니다. FK 컬럼과 같이 다른 테이블과의 연관 관계 존재 여부를 확인할 컬럼이 없기 때문에 null 인지조차 알 수 없습니다.
따라서 MISSION 테이블과의 연관관계를 확인하기 위해서 MISSION 테이블에 SELECT 쿼리를 보내야 합니다.

여기서 우리가 적용한 FetchType.LAZY 가 무시당하게 됩니다.
---------------------------------------

JPA 는 species_id 와 관계를 맺고 있는 MISSION 테이블의 데이터를 찾습니다. 하지만 SELECT 쿼리로 데이터 존재 여부만 확인하는 것은 비효율적이란 생각이 들 것입니다. JPA 는 이를 효율적으로 처리하기 위하여 SELECT 쿼리와 함께 데이터도 가져와서 프록시 대신 실제 엔티티를 생성합니다.
이렇게 결과적으로 FetchType.EAGER 가 적용되었습니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gioOfduIfhjUpvc3gReECQ.jpeg" width=70%>

그리고 SELECT 쿼리를 날렸는데 연관된 데이터가 없으면 아래와 같이 null 이 됩니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Rr3YLTVVFyR7TjFrkpTyeQ.jpeg" width=70%>

그렇다면 @oneToMany 에서는 이러한 현상이 발생하지 않는가요?
======================================

하는 의문이 들 수 있습니다. 일(1)대다(N) 인 경우 항상 **다(N)** 쪽이 외래키를 가집니다. 이 경우 일(1)에 해당하는 객체는 연관관계의 주인이 아니며 매핑된 테이블을 조회했을 때, 앞선 경우와 마찬가지로 연관관계 여부를 확인할 컬럼이 존재하지 않습니다. 즉 이 경우도 SELECT 쿼리를 다(N) 에 해당하는 테이블로 보내야 하지 않나? 생각할 수 있습니다.

@oneToMany 인 경우는 Many 에 해당하는 필드가 컬렉션이라 괜찮다.
===========================================

일(1)에 해당하는 엔티티의 다(N) 에 해당하는 연관 관계 필드는 컬렉션 타입입니다. 지연 로딩을 적용할 때 엔티티에 컬렉션이 있으면 컬렉션을 추적하고 관리할 목적으로 하이버네이트는 **컬렉션 래퍼**라는 것을 제공합니다. 이것이 컬렉션에 대한 프록시 역할을 합니다. 따라서 SELECT 쿼리를 이용하여 존재 여부를 확인할 필요 없이 지연 로딩이 정상적으로 작동하게 됩니다.

그렇다면 마지막으로 OneToOne 양방향 연관관계에서 지연 로딩이 안되는 문제를 어떻게 해결할까요?
========================================================

사실 해결 방법은 크게 없습니다. 그냥 즉시 로딩이 되는 것을 견디며 사용하거나 불필요하게 양방향 연관 관계를 설정해야 하는 것이 아니라면 아래와 같이 단방향 연관 관계로 설정해도 됩니다.
또는 프록시 대신에 바이트코드를 조작하는 라이브러리를 사용하는 방법도 존재하는 것으로 알고 있습니다.

그리고 실제로 단방향 매핑만으로도 테이블과 객체의 연관관계 매핑은 완료되었습니다. 양방향 매핑은 단방향 매핑과 비교해서 연관관계의 주인도 정해야 하고, 두 개의 단방향 연관관계를 양방향으로 만들기 위한 로직도 잘 관리해야 할 만큼 복잡합니다. 양방향의 장점은 연관관계의 주인이 아닌 방향에서 연관관계의 주인 방향으로 객체 그래프 탐색 기능이 추가된 것뿐입니다.

따라서 반대방향으로 탐색 기능이 필요한 것이 아니라면 단방향을 우선적으로 사용하는 것을 권장합니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*RFzpJVyd-fi-07_US2tUUA.png" width=70%>
