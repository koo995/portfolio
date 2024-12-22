API의 성능을 개선하기 위해선 어떻게 접근해야할까? (쿼리만 생각하면 될까?)
============================================

본 포스팅은 개인 프로젝트[GitHub](https://github.com/f-lab-edu/nutri-diary)에서 발생한 내용이며 원본 글은 [여기](https://medium.com/@gunhong951/api의-성능을-개선하기-위해선-어떻게-접근해야할까-쿼리만-생각하면-될까-7a2033cacdaf)에서 확인하실 수 있습니다.

---
> 이전 글: [[MySQL] Left Join에서 Subquery로 변경 후 쿼리 성능 30배 향상하기.](https://github.com/koo995/portfolio/blob/main/src/nutri-diary/query/README.md)
----------------------------------------------------

앞선 글에서 전문 검색(FullText Search)을 사용하는 쿼리의 구조를 변경하여 쿼리의 수행시간을 30배 단축했습니다.

하지만 리뷰 개수와 사용자들이 많이 선택한 상위 태그 등 추가 정보를 보여줘야 한다는 요구사항을 반영하기 위한 쿼리를 작성했을 때, 수행시간이 조금 증가 하는 단점이 있었습니다.

그리고 무엇보다 이러한 비즈니스적 요구사항이 쿼리 안에 녹아져 있어 DB접근 기술을 변경하거나 비즈니스 로직을 수정할 필요가 생겼을 때, 쿼리를 직접 수정해줘야 하는 만큼 유지보수 관점에서 유연성이 떨어진다는 문제점이 존재합니다.

> 하지만 그전에 해결해야 할 문제를 명확히 정의할 필요가 있었습니다.

어떤 문제를 해결하고 싶은가?
================

개선해야할 목표는 단순히 쿼리의 수행시간을 단축시키는 부분이 아닙니다. 쿼리의 수행시간 개선은 API의 성능을 향상시키는 도구 중 하나일 뿐입니다. 조금 더 크게 보아서 문제를 정의하면, 해당 쿼리를 포함하고 있는 API의 성능이 나오지 않는 것을 문제로 정의할 수 있습니다. 그리고 지금부터 목표는 API의 성능을 향상시키는 것입니다.

먼저 비즈니스 요구사항을 정의합니다.
====================

1.  /product?search=”xxx” 검색을 요청합니다. 또한 page와 size 파라미터로 페이징을 적용합니다.
2.  각 product에 해당하는 review의 갯수를 표시합니다.
3.  각 product에 대한 상위 3개 태그를 표시합니다. 사용자들이 리뷰 작성 시 자유롭게 선택한 태그들 중 가장 많이 선택된 3개를 보여줍니다.

어떻게 API의 성능을 비교할까?
==================

위의 요구사항을 만족하는 API를 만들기 위해 기존에는 DataGrip을 사용하여 쿼리의 수행시간을 분석했습니다. 하지만 프로세스 구현 방식이나 테이블 구조가 변경될 경우 데이터 구조도 함께 달라지기 때문에, 단순히 몇백만 개의 데이터로 쿼리 수행시간을 비교하는 것은 정확한 기준이 되기 어렵습니다.

따라서 더 정확한 성능 측정을 위한 기준이 필요해졌습니다.

API 성능을 제대로 테스트하기 위해서는 Postman으로 단일 응답 시간을 측정하는 것만으로는 부족했고, 더 많은 요청이 필요했습니다. 또한 검색 기능을 테스트할 때 동일한 키워드만으로는 한계가 있었습니다. MySQL InnoDB의 캐시 기능을 고려하면 다양한 데이터에 대한 접근이 필요했기 때문입니다. 이러한 이유로 nGrinder를 사용해서 트래픽을 발생시키고, Prometheus와 Grafana를 활용한 모니터링을 통해 API 성능을 평가하기로 결정했습니다.

테스트 구성
======

1. DB 의 스키마 구성.
----------------

앞선 쿼리 구조 변경 글에서 가져온 DDL을 그대로 사용합니다.

```sql
# 130만개의 데이터
# n-gram 알고리즘을 적용한 전문 검색 인덱스 사용
CREATE TABLE product (
    product_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(255) NOT NULL,
    product_corp VARCHAR(255),
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    FULLTEXT INDEX fulltext_idx (product_name, product_corp) WITH PARSER ngram
);
```

```sql
# 1000만개의 데이터
CREATE TABLE review (
    review_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT NOT NULL,
    content VARCHAR(255),
    rating TINYINT NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
);
CREATE INDEX idx_product_id ON review (product_id);
```

```sql
# 2000만개의 데이터
CREATE TABLE product_diet_tag (
    product_diet_tag_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    product_id BIGINT,
    diet_tag_id BIGINT,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL
);
CREATE INDEX idx_product_id ON product_diet_tag (product_id);
```

```sql
# 10개의 데이터
CREATE TABLE diet_tag (
    diet_tag_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    diet_tag_name VARCHAR(255) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL
);
```
![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*9ttty12Dwj73SNCmhCPc6A.png)

현재 MySQL DB에는

*   product 테이블 130만 rows
*   review 테이블 1000만 rows
*   product_diet_tag 테이블 2000만 rows
*   diet_tag 테이블 10개의 rows

저장되어 있습니다.

2. 모니터링과 nGrinder을 위한 클라우드 구축.
-------------------------------

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*k17uYzbmiL3FGP6vnMS5mw.png)

최적의 리소스를 할당하기 위해 서버를 각각의 인스턴스에 띄웠습니다.
(Prometheus와 Grafana만 제외하고…)

Target 애플리케이션 서버의 인스턴스 스펙: vCPU 2EA, Memory 8GB

Target MySQL 인스턴스 스펙: vCPU 2EA, Memory 8GB

모니터링 인스턴스 스펙: vCPU 2EA, Memory 8GB

nGrinder Controller 인스턴스 스펙: vCPU 2EA, Memory 8GB

nGrinder Agent 인스턴스 스펙: vCPU 2EA, Memory 8GB

3. 테스트 시나리오.
-------------

부하 테스트는 5분동안 진행하였고 가상 유저는 최대 300명으로 아래의 이미지처럼 30초마다 점진적으로 증가시켰습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*chqRPEBYoybFKStwhMaJNg.png)

API 호출 시나리오에서는 검색 키워드를 랜덤한 숫자나 문자열로 다양하게 적용하고, size값을 10으로 고정했으며, page값은 1만이하의 랜덤값을 사용했습니다.

(JdbcTemplate의 batch insert를 활용하여 데이터를 입력한 결과, 데이터에 일정한 규칙성이 생겼고 특정 키워드 검색 시 페이지가 1만 개까지 생성되었습니다.)

첫 번째 테스트는 전문 검색(FullTest Search)만을 이용한 테스트입니다.
==============================================

요구사항을 모두 만족하는 API를 테스트하기 전에, 전문 검색 기능만 단독으로 사용했을 때 어느 정도 API 성능이 나올지 궁금해서 테스트를 진행해 보았습니다.

아래의 코드는 API 테스트에 사용된 레포지토리 클래스입니다.
한번의 API 호출에서 쿼리는 2가지가 사용되며, 이번 테스트에서 product 테이블을 제외한 다른 테이블은 건드리지 않았습니다.

```java
@Repository
@RequiredArgsConstructor
@Transactional
public class JdbcTemplateProductSearchRepository {
    private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;
    DataClassRowMapper<ProductSearchResponse> beanPropertyRowMapper = new DataClassRowMapper<>(ProductSearchResponse.class);
    public Page<ProductSearchResponse> findFullTextSearch(String keyword, Pageable pageable) {
        Integer total = getTotalCount(keyword);
        
        // OFFSET과 LIMIT을 적용했습니다.
        String sql = "SELECT " +
                "p.product_id, p.product_name, p.product_corp " +
                "FROM product p " +
                "WHERE MATCH (p.product_name, p.product_corp) AGAINST (:keyword)" +
                "LIMIT :offset, :limit";
        MapSqlParameterSource parameters = new MapSqlParameterSource()
                .addValue("keyword", keyword)
                .addValue("offset", pageable.getOffset())
                .addValue("limit", pageable.getPageSize());
        List<ProductSearchResponse> queried = namedParameterJdbcTemplate.query(sql, parameters, beanPropertyRowMapper);
        return new PageImpl<>(queried, pageable, total);
    }
    
    // 페이징 처리를 위해 전체 갯수를 조회하는 COUNT(*) 쿼리.
    private Integer getTotalCount(String keyword) {
        String sql = "SELECT COUNT(*) " +
                "FROM product p " +
                "WHERE MATCH (p.product_name, p.product_corp) AGAINST (:keyword)";
        MapSqlParameterSource parameters = new MapSqlParameterSource()
                .addValue("keyword", keyword);
        return namedParameterJdbcTemplate.queryForObject(sql, parameters, Integer.class);
    }
}
```

![captionless image](https://miro.medium.com/v2/resize:fit:474/format:webp/1*sL1B-jcQ7G6v-CepOqPUJQ.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1528/format:webp/1*8-4unalVeI1Iza9dBqsMaQ.png)
--- | ---

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*i2YpvBsr5ciQJttT9DhZcw.png)

테스트의 결과는 매우 심각했습니다.

평균 TPS(Transaction Per Second)가 2.1 나왔고, 응답 시간은 점점 늦어져 30초가 나왔습니다. 또한 에러율이 51.3%가 나왔습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*8dQKK3sSJ1qoTVRA_8l6_g.png)

에러의 원인은 애플리케이션 로그를 확인해보니 connection timeout 에러가 발생했습니다. 이는 HikariCP 커넥션 풀의 모든 커넥션이 사용 중인 상황에서, 새로운 커넥션을 얻기 위한 대기 시간이 HikariCP의 기본 설정값 30초를 초과했기 때문입니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1000/format:webp/1*ReIpQZ46eLlUPTijBDapnA.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1000/format:webp/1*J0pegVVmhg94vCZC0DJfqg.png)
--- | ---

NCP에서 제공하는 DB 인스턴스 모니터링을 통해 문제를 분석한 결과, CPU 사용량이 최대치에 도달했으며 이것이 성능 병목지점임을 확인했습니다.

번외 테스트
------

API의 실제 성능을 측정하기 위해 HikariCP의 connection_timeout을 300초로 변경한 후, 커넥션 획득 타임 아웃 에러가 발생하지 않는 테스트를 진행했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:476/format:webp/1*AL3fEPpR-40MEGEgCKXizg.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1526/format:webp/1*qYcYVxchiUsholskUrgUJA.png)
--- | ---

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gVPBnfwIYzBSBzqzCJv3zQ.png)

API의 응답은 모두 성공하였지만, 평균 TPS(Transaction Per Second)가 1.9 나왔고, 응답 시간은 점점 늦어져 최대 1.79분이 나왔습니다.

그리고 다음 번외 테스트로 DB 인스턴스의 스펙을 단계적으로 Scale Up 해보았습니다.

스펙 변경 내역:
기존: vCPU 2EA, Memory 8GB
첫 번째 Scale Up: vCPU 8EA, Memory 32GB
두 번째 Scale Up: vCPU 32EA, Memory 128GB

![captionless image](https://miro.medium.com/v2/resize:fit:470/format:webp/1*b9egePIDCAFs2MHLyk7wUA.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1532/format:webp/1*VyizEEd0CJ5QhlONzW5X1Q.png)
--- | ---

![captionless image](https://miro.medium.com/v2/resize:fit:726/format:webp/1*xYMAfZqzcksO-mLKNECojQ.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1276/format:webp/1*2l3j3lRFaOJMZPSmG-RodA.png)
--- | ---

테스트 결과, 두 번째 Scale Up 스펙(vCPU 32EA, Memory 128GB)으로 테스트를 진행해도 평균 TPS 10, 응답 시간이 평균 15초가 걸렸습니다.
다만 CPU 사용량은 30%까지 감소했습니다.

(Grafana 대시보드에서 중간에 값이 비어있는 현상이 발견되었습니다. DB 인스턴스의 스펙을 최대 사양으로 Scale Up하여 CPU 사용량을 30%까지 낮췄음에도 이 현상이 계속되는 것으로 보아, Prometheus가 메트릭 수집에 실패한 것으로 추정됩니다. 모든 Grafana 대시보드의 값이 일시적으로 누락된 점을 고려할 때 Actuator가 제대로 동작하지 않은 것으로 의심되며, 이 문제는 추후 살펴보도록 하겠습니다.)

두 번째 테스트는 이전 글에서 사용했던 쿼리를 적용하여 모든 요구사항을 만족한 테스트입니다.
===================================================

```sql
# 사용된 쿼리
SELECT p.product_id,
       p.product_name,
       p.product_corp,
       (SELECT COUNT(*) FROM review r WHERE r.product_id = p.product_id)  AS review_count,
       (SELECT GROUP_CONCAT(diet_tag_name)
        FROM (SELECT diet_tag_name
              FROM product_diet_tag pdt
              WHERE pdt.product_id = p.product_id
              GROUP BY diet_tag_name
              ORDER BY COUNT(diet_tag_name) DESC
              LIMIT 3) AS top3_diet_tag)  AS top3_diet_tag_names
FROM product p
WHERE MATCH(p.product_name, p.product_corp) AGAINST(:keyword)
LIMIT :offset, :limit;
```

![captionless image](https://miro.medium.com/v2/resize:fit:444/format:webp/1*8qJFkn2icTeYEAmMs1Mcyw.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1558/format:webp/1*YbjrgWC4xIajTE9NmRIssw.png)
--- | ---

예상했던 대로 테스트 결과는 처참했으며, 전문 검색을 단독으로 사용했을 때와 비교해봐도 크게 달라진 점이 없습니다.

왜 이렇게 낮은 성능이 나오는 것일까요?
======================

이는 전문 검색 인덱스의 특징을 고려해야 합니다.

전문 검색에는 기존의 MySQL InnoDB 스토리지 엔진에서 제공하는 일반적인 용도의 B-tree 인덱스를 사용할 수 없습니다.

전문 검색(Full Text search) 인덱스는 문서 전체를 분석하고 검색하기 위한 인덱싱 알고리즘입니다. 이는 일반화된 기능의 명칭이며, 인덱싱 기법에 따라 크게 “**단어의 어근 분석**”과 “**n-gram 분석**” 알고리즘으로 나눌 수 있습니다.

“**단어의 어근 분석**”은 영어와 같이 단어의 변형이 있는 경우 그 단어의 뿌리인 명사 또는 어근을 찾아 인덱싱 합니다.

하지만 한국어, 중국어, 일본어는 이러한 단어 변형이 거의 없으므로, 본문을 일정 길이로 잘라서 토큰으로 인덱싱하는 “**n-gram 분석**”을 활용하면 정확성과 효율성을 향상시킬 수 있었습니다.

하지만 이러한 방식에도 단점이 있습니다.

MySQL 서버는 전문 검색 쿼리가 오면 인덱싱할 때와 동일하게 검색어를 토큰 사이즈에 맞게 잘라냅니다. 그리고 잘려진 토큰들에 대해 일치하는 단어의 갯수, 빈도 등을 확인해서 일치율을 계산합니다. 그리고 각 토큰들의 결과에 대해 동등 비교 연산이 수행되며, 이 과정에서 가중치 계산과 정렬이 이루어집니다. 따라서 검색어가 길수록 더 많은 토큰이 생성되어 CPU에 가해지는 부하가 더 커지기도 합니다.

실행계획을 분석해보면 OFFSET이 2000, LIMIT이 10인 경우 2010개의 row를 읽은 것으로 나타납니다.

```sql
SELECT p.product_id, p.product_name, p.product_corp FROM product p WHERE MATCH (p.product_name, p.product_corp) AGAINST ('닭가슴살') LIMIT 2000, 10;
```

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*VH03J_JJXqka-LymM-uNzQ.png)

하지만 실제 내부적으로 계산되는 행의 수는 단순히 OFFSET과 LIMIT으로 지정한 개수만큼이 아닙니다.

MySQL은 전문 검색으로 매칭되는 모든 rows에 대해 가중치를 계산하고 정렬한 뒤, 정렬 결과에서 상위 2000번째부터 2010번째까지 반환합니다.

따라서 페이징을 최적화하기 위한 방법들인 No Offset, 커버링 인덱스, Total Count 최적화 같은 기법들을 사용하기에도 어려움이 있었습니다.

실제로 OFFSET을 0으로 설정한 경우와 비교해보았으나 성능상 유의미한 차이는 없었고, 페이징을 위한 COUNT(*) 쿼리를 제거했을 때도 미미한 성능 개선만 있었습니다.
(단순히 하나의 트랜잭션에서 2개의 쿼리가 나가던 것을 1개의 쿼리만 나가도록 바꾸니 성능이 대략 2배 정도 좋아지는 매우 당연한 현상…)

![captionless image](https://miro.medium.com/v2/resize:fit:464/format:webp/1*i2GNyY5MwBDBoOq6jV4Quw.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1538/format:webp/1*Cy1jzImjlqR23S9msNbr5g.png)
--- | ---

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*57nSqZfrQM7fvlfuY05B9g.png)

TPS: 2.1 -> 3.7

에러율: 51.3% -> 28.6%

Elasticsearch 엔진을 도입.
=====================

기존에는 Elasticsearch을 도입하기에는 비용 부담이 크다는 단점 때문에 MySQL에서 제공하는 전문 검색을 선택했습니다.

하지만 전문 검색을 활용한 경우 100만개가 넘는 데이터에서 성능이 제대로 나오지 않는 치명적인 단점이 발견되었습니다.

또한 애플리케이션에서 사용자가 호출할 API패턴을 고려해 보았을 때, 식품 검색 API는 사용자가 매우 자주 호출하는 API로 서비스에서 핵심적인 역할을 한다는 판단이 들었습니다. 따라서 비용을 투자해서라도 성능을 끌어올릴 필요가 생겼습니다.

이전 테스트에서 DB 엔진을 Scale UP했지만 여전히 TPS는 평균 10 정도, 응답시간은 15초에 가까웠습니다. 제한된 비용 내에서는 Scale UP보다 Elasticsearch 도입이 더 효과적일 것으로 판단했습니다.

초기에는 외부 저장소를 활용한 캐시도 고려했지만, 캐시는 검색 조건이 너무 다양하여 적용이 어려웠고 데이터의 특성과 사용 패턴 분석이 선행되어야 효과적인 부분 캐싱이 가능할 것으로 판단해서 선택하지 않았습니다.

Elasticsearch 엔진은 GCP에서 제공하는 Elastic Cloud 서비스를 이용.
---------------------------------------------------

![captionless image](https://miro.medium.com/v2/resize:fit:1740/format:webp/1*bVUm5Jl0ntMIY38EzURt3A.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:262/format:webp/1*lIFNAznaRll3gEEo0VcXWQ.png)
--- | ---

인스턴스 스펙: vCPU 5개, Memory 4GB
KIBANA를 위한 1GB 메모리 무료 제공
총 비용: 1시간 0.3321$ -> 0.3321 * 24 * 30 = 239$ = 345,594원(환율 1446원)

현재 프로젝트의 클라우드 서버를 구성한 NCP도 Search Engine Service라는 서비스를 제공하지만, 이상하게 클러스터 접속이 안 되는 문제가 계속 발생했습니다.

NCP는 Search Engine Service 클러스터를 구성하는데 최소 4대의 인스턴스가 필요하며 vCPU 2EA, Memory 8GB 인스턴스 4개를 사용한다면 가격은 GCP의 Elastic Cloud와 비슷하지만 성능은 더 뛰어날 것으로 예상됩니다.
(다만 UI와 클러스터 구성의 편의성은 Elastic Cloud가 압도적으로 편합니다.
또한, 새롭게 가입하면 40만원의 크레딧도 지급받을 수 있습니다.)

직접 인스턴스에 Elasticsearch을 구축하는 방법도 고려해보았지만 처음 사용해보는 기술에 대해서 아무것도 모르는 채로 파이프라인을 구성하는데 시간과 어려움이 많이 들것 같다는 생각이 들었습니다.

따라서 우선적으로 Elasticsearch 엔진을 사용하면 얼만큼 API 성능이 나오는지 확인하는데 중점을 두어, 빠르게 구성할 수 있는 클라우드 서비스를 선택했습니다.

Elasticsearch 엔진을 이용한 3번째 테스트.
==============================

![captionless image](https://miro.medium.com/v2/resize:fit:502/format:webp/1*iw3BTCPEiFVAHw20W02gvg.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1500/format:webp/1*h23MsP2eqHoDmZhZE4KjKw.png)
--- | ---

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*KFkJOmtwjf-y4_50-LlJSA.png)

위의 테스트 결과는 리뷰 개수와 상위 태그 표시와 같은 요구사항이 적용되지 않은, product에 대한 단일 조회만을 테스트한 결과입니다.

테스트 결과를 보면 MySQL의 전문 검색을 이용한 product 단일 조회(connection timeout 에러 제거한 경우)에 비해서

평균 TPS: 1.9 -> 192.5 (약 100배 이상 향상)
응답 시간: 대략 1분 -> 1초 미만 (1/60 이상 줄어듬)

와 같은 결과가 나타났습니다.

테스트 시나리오는 이전 테스트들과 거의 동일하게 진행했으며, 가상 사용자 수가 점진적으로 늘어남에 따라 TPS도 함께 증가했습니다.

쿼리에 있던 요구사항을 비즈니스 레이어로 이동.
==========================

[이전 글](https://github.com/koo995/portfolio/blob/main/src/nutri-diary/query/README.md#%EA%B7%B8%EB%9F%AC%EB%82%98-%EC%9A%94%EA%B5%AC%EC%82%AC%ED%95%AD-%EC%B6%94%EA%B0%80)에서는 쿼리 안에 모든 요구사항이 들어 있었습니다.

요구사항들은 다음과 같습니다.

1.  각 product에 해당하는 review의 갯수를 표시합니다.
2.  각 product에 대한 상위 3개 태그를 표시합니다. 사용자들이 리뷰 작성 시 자유롭게 선택한 태그들 중 가장 많이 선택된 3개를 보여줍니다.

기존의 테이블 구성은 아래 이미지와 같습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*QL7bAGSi6n7ot-vkY-kneA.png)

product와 diet_tag는 다대다 관계를 가지며, 중간에 product_diet_tag 테이블을 두어 각 product에 연관된 diet_tag를 저장합니다.

사용자는 product에 대한 review를 작성할 때, 원하는 diet_tag를 여러개 선택해서 저장합니다. 따라서 하나의 review가 작성될 때 product_diet_tag의 데이터는 여러개가 저장됩니다.

예를 들어, product가 100만 개가 있고 각 product마다 100개의 review가 있다고 가정해 보겠습니다. 그리고 사용자들이 각 review마다 3개의 diet_tag를 골고루 지정한다면, product_diet_tag의 데이터는 대략 **3억 개**가 저장됩니다.

이러한 설계는 문제점이 있다고 판단하여 테이블 구조와 저장 프로세스의 개선이 필요하다고 생각했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Ew9khtq6ZiAucZdNOw40Eg.png)

사용자들이 review를 작성할때 product_diet_tag에 tag_count 필드를 만들어 해당 값을 증가시키면 데이터 용량을 훨씬 줄일 수 있습니다.

다시 예를 들어, product가 100만 개 있고 diet_tag가 10개라고 가정해보겠습니다. 이 경우 product_diet_tag의 데이터는 **최대 1000만 개**로 줄어듭니다. product_diet_tag의 수는 product 수와 diet_tag 수에만 의존하며, review 수에는 영향을 받지 않습니다. 사용자가 review를 엄청 많이 등록하더라도 tag_count 값만 증가시키면 됩니다.

이제 조회 코드를 살펴보겠습니다.

기존의 코드는 아래와 같이 JdbcTemplate을 이용하여 Repository에서 모든 로직을 처리하고 있었습니다.

```java
@Repository
@RequiredArgsConstructor
@Transactional
public class JdbcTemplateProductSearchRepository {
    private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;
    DataClassRowMapper<ProductSearchResponse> beanPropertyRowMapper = new DataClassRowMapper<>(ProductSearchResponse.class);
    public Page<ProductSearchResponse> findFullTextSearch(String keyword, Pageable pageable) {
        Integer total = getTotalCount(keyword);
        String sql = "SELECT " +
                "p.product_id, p.product_name, p.product_corp, " +
                "(SELECT COUNT(*) FROM review r WHERE r.product_id = p.product_id) AS review_count, " +
                "(SELECT AVG(r.rating) FROM review r WHERE r.product_id = p.product_id) AS review_avg_rating, " +
                "(SELECT GROUP_CONCAT(diet_tag_name) FROM (SELECT dt.diet_tag_name FROM product_diet_tag pdt JOIN diet_tag dt ON pdt.diet_tag_id = dt.diet_tag_id where pdt.product_id = p.product_id ORDER BY pdt.tag_count DESC LIMIT 3) AS top3_diet_tag) AS top3_diet_tag_names " +
                "FROM product p " +
                "WHERE MATCH (p.product_name, p.product_corp) AGAINST (:keyword)" +
                "LIMIT :offset, :limit";
        MapSqlParameterSource parameters = new MapSqlParameterSource()
                .addValue("keyword", keyword)
                .addValue("offset", pageable.getOffset())
                .addValue("limit", pageable.getPageSize());
        List<ProductSearchResponse> queried = namedParameterJdbcTemplate.query(sql, parameters, beanPropertyRowMapper);
        return new PageImpl<>(queried, pageable, total);
    }
    private Integer getTotalCount(String keyword) {
        String sql = "SELECT COUNT(*) " +
                "FROM product p " +
                "WHERE MATCH (p.product_name, p.product_corp) AGAINST (:keyword)";
        MapSqlParameterSource parameters = new MapSqlParameterSource()
                .addValue("keyword", keyword);
        return namedParameterJdbcTemplate.queryForObject(sql, parameters, Integer.class);
    }
}
```

위의 코드를 리팩터링하여 아래의 코드로 바꿨습니다.

```java
// 쿼리에 IN절을 사용해서 각 productId에 해당하는 리뷰들의 갯수를 한번에 조회합니다.
public interface ReviewRepository extends CrudRepository<Review, Long> {
    @Query("SELECT product_id, COUNT(*) AS review_count FROM review WHERE product_id IN (:productIds) GROUP BY product_id")
    List<ProductReviewCount> countReviewsByProductIds(@Param("productIds") List<Long> productIds);
}
```

```java
// 쿼리에 IN절을 사용해서 각 productId에 해당하는 product_diet_tag를 모두 조회합니다.
// diet_tag 테이블과 조인하여 diet_tag_name을 가져옵니다.
public interface ProductDietTagRepository extends CrudRepository<ProductDietTag, Long> {
    @Query("SELECT pdt.product_id, dt.diet_tag_name, pdt.tag_count " +
            "FROM product_diet_tag pdt " +
            "LEFT JOIN diet_tag dt on pdt.diet_tag_id = dt.diet_tag_id " +
            "WHERE product_id IN (:productIds)"
    )
    List<ProductDietTagDto> findByProductIds(@Param("productIds") List<Long> productIds);
}
```

```java
@RequiredArgsConstructor
@Transactional(readOnly = true)
@Service
public class ProductSearchService {
    // spring data elasticsearch 리포지토리
    private final ProductDocumentRepository productDocumentRepository;
    private final ReviewRepository reviewRepository;
    private final ProductDietTagRepository productDietTagRepository;
    private final ProductSearchResponseMapper productSearchResponseMapper;
    public Page<ProductSearchResponse> search(String keyword, Pageable pageable) {
        // ProductDocument는 Elasticsearch에 저장된 각 데이터를 매핑합니다.
        // Elasticsearch을 이용해서 keyword에 해당하는 ProductDocument를 검색합니다.
        Page<ProductDocument> productDocuments = productDocumentRepository.findByProductName(keyword, pageable);
        
        // 검색 결과가 없으면 예외를 반환합니다.
        if (productDocuments.isEmpty()) {
            throw new BusinessException(PRODUCT_NOT_FOUND);
        }
        
        // Product의 id값을 추출합니다.
        List<Long> productIds = productDocuments.stream()
                .map(ProductDocument::getId)
                .toList();
        
        List<ProductDietTagDto> tags = productDietTagRepository.findByProductIds(productIds);
        List<ProductReviewCount> productReviewCounts = reviewRepository.countReviewsByProductIds(productIds);
        
        // productSearchResponseMapper클래스가
        // 각 productId에 맞게 tags를 count값 순으로 내림차순으로 정렬한 후 3개를 선택.
        // 또한 각 productId에 맞게 review 갯수도 매핑하여 결과 반환.
        return productSearchResponseMapper.toResponseList(productDocuments, tags, productReviewCounts);
    }
}
```

결과적으로 한방 쿼리로 해결하던 것을 3번의 DB 접근으로 나눴지만, 더이상 복잡한 쿼리가 필요 없어졌고 비즈니스 로직과 데이터 접근 로직이 강하게 결합되어 있던 것을 분리하여 코드 가독성이 향상되었습니다.

리팩터링된 코드는 각 역할(제품 조회, 리뷰 조회, 태그 조회)을 개별적으로 처리하고, 서비스 계층에서 결과를 조합하는 방식으로 재사용성 또한 증가했습니다.

마지막 테스트 진행.
===========

이전 테스트와의 차이점은 테스트 시간을 15분으로 늘렸다는 점입니다. 앞선 테스트에서 요구사항이 없는 Elasticsearch 단독 테스트에서 이미 어느 정도 성능이 나오는 것을 확인했기에 조금 더 긴 시간 동안 테스트를 진행했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*B-FyxCFjfhCdVRSvYVpOFA.png)

사용자는 점진적으로 증가하다 4분 정도 지난 후 300명을 유지했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:494/format:webp/1*5-u1ENZnO_NVKbSTmQjUyQ.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1508/format:webp/1*MZkdHTIozb5Tw3Pgfqsh1w.png)
--- | ---

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*2rHS7a4UvAVFWmQIBlw_CA.png)![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*wbpnBXwrK6MdI0-lCz7gdQ.png)

TPS: 185.6
평균 응답 시간: 968ms(Grafana 기준)~1399ms(nGrinder 기준)
(단독 조회 테스트에 비해 모두 300ms 정도 증가)

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*bJpJkMvxGWdLUzly4Hdfaw.png)

애플리케이션 테스트 서버의 CPU 사용량은 대략 30~40%정도로 증가했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gSWWbSRK5eYI65UYUCnQrA.png)

전문 검색만을 이용할 때 가볍게 100%을 찍던 MySQL 서버의 CPU 사용량도 15% 내외로 줄어들었습니다.

최종 결과
평균 TPS: 1.9 -> 185.6 (약 100배 이상 향상)
응답 시간: 대략 1분 -> 1초 (Grafana 대시보드 기준 1/60 이상 줄어듬)

정리.
===

처음 API를 개발할 때는 단순히 리뷰 개수를 product와 review 테이블을 JOIN해서 원하는 결과만 가져오면 충분하다고 생각했습니다.

하지만 전문 검색 인덱스를 사용하면서 쿼리 수행 시간이 크게 늘어나는 문제가 발생했습니다. 실행 계획을 분석하고 쿼리 구조를 개선하여 수행 시간을 단축했지만, 여전히 문제가 남아있었습니다.

API에서 제공해야 할 요구사항이 늘어날수록 이를 억지로 쿼리에 추가하다 보니 쿼리가 너무 복잡해지는 문제였습니다.

또한, product 테이블을 탐색하는 것은 전문 검색을 사용하는 경우뿐만 아니라 다양한 경우에도 사용될 수 있었습니다. 이때마다 복잡한 서브쿼리나 조인을 통해서 review의 갯수나 tag를 가져오는 쿼리를 작성하는 것은 비효율적이라 판단했고, 이에 따라 프로세스와 테이블 구조를 더 유연하게 재사용할 수 있도록 개선할 필요성을 느꼈습니다.

하지만 테이블 구조를 변경한다면 새로운 데이터를 넣고 쿼리 시간을 측정해서 이전 테이블 구조의 결과와 비교하는 방식은 기준이 부적절하다 생각했습니다. 더 정확한 기준을 잡기 위해 고민하던 중, 실제로 해결하고자 했던 문제는 해당 쿼리를 포함한 API의 성능을 개선하는 것이 문제였음을 돌이켜 보았습니다.

처음에는 쿼리 튜닝으로만 문제를 해결하려 했으나, 더 넓게 문제를 정의하니 고려할 수 있는 선택지가 많아졌고 기존 API 개발 방식의 문제점도 파악할 수 있었습니다. 그리고 결과적으로 실제 사용 가능한 수준의 API를 개발할 수 있게 되었습니다.

Elasticsearch 도입의 단점?
---------------------

API 성능은 향상되었지만 그에 따른 트레이드오프가 있었습니다. 우선 추가 리소스가 필요한 비용적인 문제가 있고, MySQL 테이블과 Elasticsearch 클러스터 간의 데이터 일관성 문제가 발생할 수 있습니다. product 테이블의 모든 컬럼이 필요하지는 않지만, product id와 일부 필수 컬럼들의 동기화를 위한 추가 작업이 필요합니다.