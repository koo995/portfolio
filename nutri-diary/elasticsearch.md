MySQL의 전문 검색(Full-Text Search)은 100만개가 넘는 데이터에서 문제가 없을까?
============================================

본 포스팅은 개인 프로젝트[GitHub](https://github.com/f-lab-edu/nutri-diary)에서 발생한 내용이며, 혹시나 마크다운 문법이 보기 불편하시다면 원글에 해당하는 블로그 글을 [여기](https://medium.com/@gunhong951/api의-성능을-개선하기-위해선-어떻게-접근해야할까-쿼리만-생각하면-될까-7a2033cacdaf)에서 확인하실 수 있습니다.

---
> 이전 글: [[MySQL] Left Join에서 Subquery로 변경 후 쿼리 성능 30배 향상하기.](https://github.com/koo995/portfolio/blob/main/nutri-diary/fulltext-search.md)
----------------------------------------------------

앞선 글에서 전문 검색(FullText Search)을 사용하는 쿼리의 구조를 변경하여 쿼리의 수행시간을 30배 단축했습니다.

하지만 리뷰 개수와 사용자들이 많이 선택한 상위 태그 등 추가 정보를 보여줘야 한다는 요구사항을 반영하기 위한 쿼리를 작성했을 때, 수행시간이 조금 증가 하는 단점이 있었습니다.

그리고 무엇보다 이러한 비즈니스적 요구사항이 쿼리 안에 녹아져 있어 DB접근 기술을 변경하거나 비즈니스 로직을 수정할 필요가 생겼을 때, 쿼리를 직접 수정해줘야 하는 만큼 유지보수 관점에서 유연성이 떨어진다는 문제점이 존재합니다.

> 하지만 그전에 해결해야 할 문제를 명확히 정의할 필요가 있었습니다.

어떤 문제를 해결하고 싶은가?
================

개선해야할 사항은 단순히 쿼리의 수행시간을 단축시키는 부분이 아닙니다. 쿼리의 수행시간 개선은 API의 성능을 향상시키는 도구 중 하나일 뿐입니다.

조금 더 크게 보아서 문제를 정의하면, 기존 API는 쿼리의 수행시간으로 인해 Postman으로 요청 시 응답까지 대략 6.85초가 걸렸지만, 결국 API의 성능이 나오지 않는 것을 문제로 정의할 수 있습니다.

그리고 지금부터 목표는 API의 성능이 나오지 않는 것을 해결하는 것입니다.

먼저 비즈니스 요구사항을 정의합니다
===================

1.  /product?search=”xxx” 검색을 요청합니다. 또한 page와 size 파라미터로 페이징을 적용합니다.
2.  각 product에 해당하는 review의 갯수를 표시합니다.
3.  각 product에 대한 상위 3개 태그를 표시합니다. 사용자들이 리뷰 작성 시 자유롭게 선택한 태그들 중 가장 많이 선택된 3개를 보여줍니다.

어떻게 API의 성능을 비교할까?
==================

위의 요구사항을 만족하는 API를 만들기 위해 기존에는 DataGrip을 사용하여 쿼리의 수행시간을 분석했습니다. 하지만 프로세스 구현 방식이나 테이블 구조가 변경될 경우 데이터 구조도 함께 달라지기 때문에, 단순히 몇백만 개의 데이터로 쿼리 수행시간을 비교하는 것은 정확한 기준이 되기 어렵습니다.

따라서 더 정확한 성능 측정을 위한 기준이 필요해졌습니다.

API 성능을 제대로 테스트하기 위해서는 Postman으로 단일 응답 시간을 측정하는 것만으로는 부족했고, 더 많은 요청이 필요했습니다. 또한 검색 기능을 테스트할 때 동일한 키워드만으로는 한계가 있었습니다. MySQL InnoDB의 캐시 기능을 고려하면 다양한 데이터에 대한 접근이 필요했기 때문입니다. 이러한 이유로 nGrinder를 사용해서 다양한 데이터에 접근하는 트래픽을 발생시키고, Prometheus와 Grafana를 활용한 모니터링을 통해 API 성능을 평가하기로 결정했습니다.

테스트 구성
======

1. DB 의 스키마 구성
---------------

앞선 쿼리 구조 변경 글에서 가져온 DDL을 그대로 사용합니다.

```sql
# 150만개의 데이터
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

*   product 테이블 150만 rows
*   review 테이블 1000만 rows
*   product_diet_tag 테이블 2000만 rows
*   diet_tag 테이블 10개의 rows

저장되어 있습니다.

2. 모니터링과 nGrinder을 위한 클라우드 구축
------------------------------

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*k17uYzbmiL3FGP6vnMS5mw.png)

최적의 리소스를 할당하기 위해 서버를 각각의 인스턴스에 띄웠습니다.
(Prometheus와 Grafana만 제외하고…)

Target 애플리케이션 서버의 인스턴스 스펙: vCPU 2EA, Memory 8GB
Target MySQL 인스턴스 스펙: vCPU 2EA, Memory 8GB

모니터링 인스턴스 스펙: vCPU 2EA, Memory 8GB
nGrinder Controller 인스턴스 스펙: vCPU 2EA, Memory 8GB
nGrinder Agent 인스턴스 스펙: vCPU 2EA, Memory 8GB

3. 테스트 시나리오
------------

부하 테스트는 5분동안 진행하였고 가상 유저는 최대 300명으로 아래의 이미지처럼 30초마다 점진적으로 증가시켰습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*chqRPEBYoybFKStwhMaJNg.png)

API 호출 시나리오에서는 검색 키워드, size, page 값을 랜덤한 문자열이나 숫자로 다양하게 적용했습니다.

첫 번째 테스트는 전문 검색(FullText Search)만을 이용한 테스트입니다
=============================================

요구사항을 모두 만족하는 API를 테스트하기 전에, 전문 검색 기능만 단독으로 사용했을 때 어느 정도 API 성능이 나올지 궁금해서 테스트를 진행해 보았습니다.

아래의 코드는 API 테스트에 사용된 레포지토리 클래스입니다.
한번의 API 호출에서 페이징을 위해 쿼리는 2가지가 사용되며, 이번 테스트에서 product 테이블을 제외한 다른 테이블은 건드리지 않았습니다.

```java
@Repository
@RequiredArgsConstructor
@Transactional
public class JdbcTemplateProductSearchRepository {

    private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;
    private final DataClassRowMapper<ProductSearchResponse> beanPropertyRowMapper = new DataClassRowMapper<>(ProductSearchResponse.class);

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

![captionless image](https://miro.medium.com/v2/resize:fit:456/format:webp/1*sL1B-jcQ7G6v-CepOqPUJQ.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1546/format:webp/1*Ut3cJXl-yIHtE6epRuIPNA.png)
--- | ---

테스트의 결과는 매우 심각했습니다.

평균 TPS(Transaction Per Second)가 2.1 나왔고, 응답 시간은 점차 증가하여 30초가 지속되었습니다. 또한 에러율이 51.3%(958개 중에 491개 실패)가 나왔습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*8dQKK3sSJ1qoTVRA_8l6_g.png)

에러의 원인은 애플리케이션 로그를 확인해보니 connection timeout 에러가 발생했습니다. 이는 HikariCP 커넥션 풀의 모든 커넥션이 사용 중인 상황에서, 새로운 커넥션을 얻기 위한 대기 시간이 HikariCP의 기본 설정값 30초를 초과했기 때문입니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1000/format:webp/1*P7JKFoLJ3toVDYQgzcZOOw.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1000/format:webp/1*J0pegVVmhg94vCZC0DJfqg.png)
--- | ---

NCP에서 제공하는 DB 인스턴스 모니터링을 통해 문제를 분석한 결과, CPU 사용량이 최대치에 도달했으며 이것이 성능 병목 지점임을 확인했습니다.

번외 테스트
------

API의 실제 성능을 측정하기 위해 HikariCP의 connection_timeout을 여유롭게 300초로 변경한 후, 커넥션 획득 타임 아웃 에러가 발생하지 않는 테스트를 진행했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:460/format:webp/1*Vf2NMzY0c-jlmR_pTxMJ0w.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1542/format:webp/1*Gk6EePOfoRhdOEvBlzT9lQ.png)
--- | ---

API의 응답은 모두 성공하였지만, 평균 TPS(Transaction Per Second)가 1.9 나왔고, 평균 응답 시간은 대략 1.67분을 유지, 최대 1.79분이 나왔습니다.

그리고 다음 번외 테스트로 DB 인스턴스의 스펙을 단계적으로 Scale Up 해보았습니다.

스펙 변경 내역:
기존: vCPU 2EA, Memory 8GB
첫 번째 Scale Up: vCPU 8EA, Memory 32GB
두 번째 Scale Up: vCPU 32EA, Memory 128GB

![captionless image](https://miro.medium.com/v2/resize:fit:306/format:webp/1*O4qYDWQe0Fwimvs0gjtgEg.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1082/format:webp/1*msyuzbxee7ns6j1z8hqDIg.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:614/format:webp/1*GJE_PWm4x9eF-_XRvUJW3w.png)
--- | --- | ---

테스트 결과, 두 번째 Scale Up 스펙(vCPU 32EA, Memory 128GB)으로 테스트를 진행해도 평균 TPS 10.5, 평균 응답 시간은 대략 15~20초가 걸렸습니다.
다만 CPU 사용량은 30%까지 감소했습니다.

(Grafana 대시보드에서 중간에 값이 비어있는 현상이 계속 발견되었습니다. 이에 대한 글은 아래에 링크로 남겨두었습니다. )

[Grafana 대시보드 누락(단절?) 현상.](https://medium.com/@gunhong951/grafana-대시보드-누락-단절-현상-057841a00bc6?source=post_page-----7a2033cacdaf--------------------------------)

두 번째 테스트는 이전 글에서 사용했던 쿼리를 적용하여 모든 요구사항을 만족한 테스트입니다
==================================================

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

![captionless image](https://miro.medium.com/v2/resize:fit:444/format:webp/1*8qJFkn2icTeYEAmMs1Mcyw.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1558/format:webp/1*xhXJg2RwqN3K7i6BmDNvcw.png)
--- | ---

예상했던 대로 테스트 결과는 처참했으며, 전문 검색을 단독으로 사용했을 때와 비교해봐도 크게 달라진 점이 없습니다.

왜 이렇게 낮은 성능이 나오는 것일까요?
======================

이는 전문 검색 인덱스의 특징을 고려해야 합니다.

전문 검색에는 기존의 MySQL InnoDB 스토리지 엔진에서 제공하는 일반적인 용도의 B-tree 인덱스를 사용할 수 없습니다.

전문 검색(Full Text search) 인덱스는 문서 전체를 분석하고 검색하기 위한 역색인 구조의 인덱싱 알고리즘입니다.

그리고 인덱싱 기법에 따라 크게 “**단어의 어근 분석**”과 “**n-gram 분석**” 알고리즘으로 나눌 수 있습니다.

“**단어의 어근 분석**”은 영어와 같이 단어의 변형이 있는 경우 그 단어의 뿌리인 명사 또는 어근을 찾아 인덱싱 합니다.

하지만 한국어, 중국어, 일본어는 이러한 단어 변형이 거의 없으므로, 본문을 일정 길이로 잘라서 토큰으로 인덱싱하는 “**n-gram 분석**”을 활용하면 정확성과 효율성을 향상시킬 수 있습니다.

그리고 product테이블에는 n-gram 분석을 활용한 전문 검색 인덱스가 생성되어 있습니다.

하지만 이러한 방식에도 단점들이 있습니다.

MySQL 서버는 전문 검색 쿼리가 오면 인덱싱할 때와 동일하게 검색어를 토큰 사이즈에 맞게 잘라냅니다. 그리고 잘려진 토큰들에 대해 일치하는 단어의 갯수, 빈도 등을 확인해서 일치율을 계산합니다.

그리고 각 토큰들의 결과에 대해 비교 연산이 수행되며, 이 과정에서 가중치 계산과 정렬이 이루어집니다.

따라서 검색어가 길수록 더 많은 토큰이 생성되고 각 토큰의 결과를 모아 비교 연산을 수행하니 CPU에 가해지는 부하가 더 커지기도 합니다.

또한, 실행계획을 분석해보면 OFFSET이 2000, LIMIT이 10인 경우 2010개의 row를 읽은 것으로 나타납니다.

```sql
SELECT p.product_id, p.product_name, p.product_corp FROM product p WHERE MATCH (p.product_name, p.product_corp) AGAINST ('닭가슴살') LIMIT 2000, 10;
```

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*VH03J_JJXqka-LymM-uNzQ.png)

하지만 실제 내부적으로 계산되는 행의 수는 단순히 OFFSET과 LIMIT으로 지정한 개수만큼이 아닙니다.

MySQL은 전문 검색으로 매칭되는 모든 rows에 대해 가중치를 계산하고 정렬한 뒤, 정렬 결과에서 상위 2000번째부터 2010번째까지 반환합니다.

따라서 페이징을 최적화하기 위한 방법들인 No Offset, 커버링 인덱스, Total Count 최적화 같은 기법들을 사용하기에도 어려움이 있었습니다.

실제로 OFFSET을 0으로 설정한 경우와 비교해보았으나 성능상 유의미한 차이는 없었고, 페이징을 위한 COUNT(*) 쿼리를 제거했을 때도 미미한 성능 개선만 있었습니다.
(단순히 하나의 트랜잭션에서 2개의 쿼리가 나가던 것을 1개의 쿼리만 나가도록 바꾸니 성능이 대략 2배 정도 좋아지는 매우 당연한 현상…)

TPS: 2.1 -> 3.7
에러율: 51.3% -> 28.6%(1372개 중에 393개 실패)

Elasticsearch 엔진을 도입
====================

기존에는 Elasticsearch도입을 비용 부담이 크다는 단점 때문에 선택하지 않았고 MySQL에서 제공하는 전문 검색을 선택했습니다.

하지만 전문 검색을 활용한 경우 100만개가 넘는 데이터에서 성능이 제대로 나오지 않는 치명적인 단점이 발견되었습니다.

또한 애플리케이션에서 사용자가 호출할 API패턴을 고려해 보았을 때, 식품 검색 API는 사용자가 매우 자주 호출하는 API로 서비스에서 핵심적인 역할을 한다는 판단이 들었습니다. 따라서 비용을 투자해서라도 성능을 끌어올릴 필요가 생겼습니다.

이전 테스트에서 DB 엔진을 Scale UP했지만 여전히 TPS는 평균 10 정도, 응답시간은 15초에 가까웠습니다. 제한된 비용 내에서는 Scale UP보다 Elasticsearch 도입이 더 효과적일 것으로 판단했습니다.

초기에는 외부 저장소를 활용한 캐시도 고려했지만, 캐시는 검색 조건이 너무 다양하여 적용이 어려웠고 데이터의 특성과 사용 패턴 분석이 선행되어야 효과적인 부분 캐싱이 가능할 것으로 판단해서 선택하지 않았습니다.

Elasticsearch 엔진은 GCP에서 제공하는 Elastic Cloud 서비스를 이용
--------------------------------------------------

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

Elasticsearch 엔진을 이용한 3번째 테스트
=============================

![captionless image](https://miro.medium.com/v2/resize:fit:484/format:webp/1*iw3BTCPEiFVAHw20W02gvg.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1518/format:webp/1*nS-D20sl8iKpiSSRnUVCXQ.jpeg)
--- | ---

위의 테스트 결과는 리뷰 개수와 상위 태그 표시와 같은 요구사항 없이 product에 대한 검색(GET 요청)만을 진행한 테스트 결과입니다.

테스트 결과를 보면 MySQL의 전문 검색을 이용한 경우에 비해서
(connection timeout 에러 제거한 경우)

평균 TPS: 1.9 -> 192.5 (약 100배 향상)
응답 시간: 대략 1.67분 -> 800ms (800ms/100200ms = 1/125 줄어듬)

와 같은 결과가 나타났습니다.

쿼리에 있던 요구사항을 비즈니스 레이어로 이동
=========================

[이전 글](https://github.com/koo995/portfolio/blob/main/nutri-diary/fulltext-search.md)에서는 쿼리 안에 모든 요구사항이 들어 있었습니다.

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

    private final DataClassRowMapper<ProductSearchResponse> beanPropertyRowMapper = new DataClassRowMapper<>(ProductSearchResponse.class);

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

위의 코드를 리팩터링하여 아래의 코드로 변경했습니다.

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

마지막 테스트 진행
==========

이전 테스트와의 차이점
------------

1.  **위의 요구사항들을 모두 반영.**
2.  **기존의 GET(검색)요청과 함께 POST(저장)요청을 추가했습니다.**
    실제 상황을 가정하여 요청 비율은 GET 90%, POST 10% 로 설정했습니다.
    데이터 저장 시 MySQL과 Elasticsearch의 데이터 동기화는 애플리케이션에서 직접 관리하는 가장 단순한 방식을 선택했습니다.
3.  **테스트 시간을 25분으로 증가.**
    앞선 테스트에서 요구사항이 없는 Elasticsearch 단독 테스트에서 이미 어느 정도 성능이 나오는 것을 확인했기에 조금 더 긴 시간 동안 테스트를 진행했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*5T-4MPEijt2pq6YKlgW47w.png)

사용자는 점진적으로 증가하다 4분 정도 지난 후 300명을 유지했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:478/format:webp/1*l1KZm8fVy1iVF9Jb4MlS0A.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1524/format:webp/1*cWpGpQ4cJIPNVbaLKlV2Ww.png)
--- | ---

요구사항이 없는 검색(GET) 요청 테스트와 비교했을 때, TPS가 192.5에서 179.6으로 조금 감소했습니다.

각 API 요청 별로 평균 응답 시간을 살펴보면 GET 요청은 876ms, POST 요청은 GET 요청보다 약 400ms 정도 더 소요된 1.21초(1210ms)의 결과가 나타났습니다. POST 요청은 Elasticsearch 엔진에 쓰기 작업(읽기에 비해서 상대적으로 느림)과 MySQL에 쓰기 작업을 동기적으로 처리하는데 이 부분에서 더 많은 시간이 소요된 것으로 판단됩니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1250/format:webp/1*hHDHxIidIsZbrqTkhwCNJw.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:752/format:webp/1*deRnReG2OtYLnUS98zcFFQ.png)
--- | ---

애플리케이션 서버의 CPU 사용량은 대략 20~30%정도로 나타났습니다.

그리고 전문 검색만을 이용할 때 가볍게 100%을 찍던 MySQL 서버의 CPU 사용량도 20~25% 정도로 줄어들었습니다.

최종 결과
-----

평균 TPS: 1.9 -> 179.6
(약 90배 향상)

평균 응답 시간: 대략 1.67분(100.2s) -> 876ms
(검색에 해당하는 GET 요청 기준 약 115배 향상)

개선해야 할 점
--------

Elasticsearch엔진을 사용하여 트래픽 상황 속에서도 1초 미만이라는 짧은 응답 시간이 나왔지만, 실제로 체감해보면 빠르다는 느낌을 받기가 어려웠습니다.

더욱이 POST 요청에 해당하는 작업은 400ms 정도가 더 걸려 살짝 느리다는 느낌까지 들었습니다.

결과가 만족스럽지 않습니다.

이때, Grafana대시보드를 살펴보다 JVM의 Thread 상태를 보았습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*MHOW7M5DM4_2168wylWYqA.png)

timed-waiting 스레드가 무려 197개에 해당하는 것으로 나타납니다.

무언가 이상하다는 느낌이 들었고, 이를 개선하면 더 나은 성능이 나올 것 같은 직감이 들었습니다.

[다음 편](https://github.com/koo995/portfolio/blob/main/nutri-diary/threadPool-hikariPool.md)