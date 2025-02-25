[MySQL] Left Join에서 Subquery로 변경 후 쿼리 성능 30배 향상하기.
====================================================

본 포스팅은 개인 프로젝트[[GitHub](https://github.com/f-lab-edu/nutri-diary)]에서 발생한 내용이며, 혹시나 마크다운 문법이 보기 불편하시다면 원글에 해당하는 블로그 글을 [여기](https://medium.com/@gunhong951/%EC%A0%84%EB%AC%B8%EA%B2%80%EC%83%89-%EA%B3%BC%EC%97%B0-%EB%8D%94-%EB%B9%A0%EB%A5%B8%EA%B0%80-3db2e3fa0c89)서 확인하실 수 있습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*ZsbhcEkWAtxB6zs8pv-Jlw.png)

문제 상황
=====

MySQL의 전문 검색(FullText Search)을 활용한 검색 API를 구현하고 Postman으로 테스트를 진행했습니다.

그런데 테스트 결과 응답 시간이 무려 6.85초나 소요되었습니다. 즉시 전문 검색 인덱스 적용 여부를 확인했으나, 전문 검색 인덱스는 정상적으로 생성되어 있었습니다.

> 인덱스는 생성되었지만 인덱스를 제대로 타지 않았나?

하는 의문이 들었고, 이제부터 그 원인을 추적해보고자 합니다.

API의 **요구사항과 DDL**
==================

API의 요구 사항으로는 product을 검색할 때, 검색 결과와 함께 각 product와 연관된 review들의 총 갯수도 보여줘야 합니다.

현재 product 테이블의 데이터는 100만개, review 테이블의 데이터는 1000만개 저장되어있습니다.

각 테이블의 DDL은 아래와 같습니다.
외래키는 사용하지 않았고 대신 인덱스는 별도로 설정했습니다.

```sql
# ngram 분석 알고리즘을 활용하는 전문 검색 인덱스 생성
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

문제의 쿼리
======

```sql
SELECT
    p.product_id,
    p.product_name,
    p.product_corp,
    IFNULL(review_count, 0) AS review_count
FROM product p
LEFT JOIN (
    SELECT product_id, COUNT(*) AS review_count
    FROM review
    GROUP BY product_id
) r ON p.product_id = r.product_id
WHERE MATCH (p.product_name, p.product_corp) AGAINST ('크리스피')
LIMIT 1, 20;
```

![쿼리 수행 결과](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*r-qEWXWA9AwjQCjHTINC4w.png)

위의 요구사항을 위해 작성한 쿼리는 위와 같고, 예시를 위해 “크리스피"라는 단어로 전문 검색을 실행했습니다.

하지만 위의 쿼리를 적용하고 테스트를 해봤을 때 3913(ms)초의 시간이 걸리는 만큼 심각하게 느린 속도를 보입니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*8Zs2U9qbPE92nNJi2Uaw9w.png)

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*suhlBU0TRlUyDlx9M43vXg.png)

실행 순서는 빨간 번호 순이며, 다음 기준으로 읽으면 됩니다.

*   들여쓰기( -> 모양의 화살표)가 같은 들여쓰기 레벨에서는 상단에 위치한 라인이 먼저 실행.
    (2번과 6번 중에서 2번이 먼저 실행)
*   들여쓰기가 다른 레벨에서는 가장 오른쪽에 위치한 라인이 먼저 실행.
    (2번과 1번 중에서 1번이 먼저 실행)

제일 첫 실행 작업인 “Full-text index search on p using fulltext_idx” 기준으로 실행계획의 용어들을 설명하면

*   cost : 총 쿼리 비용에서 이 쿼리의 기여도.
*   rows: 예상 결과의 레코드 수를 말한다.
*   actual time=n..m: n은 각 loop에서 첫번째 레코드를 읽기까지의 평균 시간, m은 마지막 레코드를 읽기까지의 평균 시간.
*   actual rows: 한번의 loop에서 product_name=”크리스피”을 이용하여 실제 읽은 평균 레코드 수.
*   actual loops: product_name=”크리스피"을 이용하여 product 테이블에서 레코드를 찾는 작업을 반복한 횟수.

이제, 실행계획을 따라가 보겠습니다.

먼저, product 테이블에 전문 검색을 수행합니다.

이 부분에서는 26개의 레코드를 가져오는데 평균 22(ms)가 걸리는 만큼 성능 저하에 영향을 미치지 않습니다.

그리고 review 테이블에서 Covering index scan을 수행하는데
이때, 테이블에서 읽은 rows가 1000만개(10e+6)이고 마지막 row를 가져오는데 걸린 시간이 무려 3220(ms)나 걸렸습니다.

GROUP BY product_id와 count()을 활용하여 연관된 review의 갯수를 집계할 때, 앞선 작업에서 모든 review의 product_id 인덱스를 탐색하느라 시간이 많이 소모되었습니다.

이후 리뷰 갯수에 대한 집계연산을 수행하고 LEFT JOIN을 처리하는데 대략 4418(ms)의 시간이 걸렸습니다.
(EXPLAIN ANALYZE을 사용해서 실제 쿼리보다 시간이 조금 더 걸렸습니다.)

첫번째 쿼리 수정
=========

문제점을 확인해 보았을 때 review 테이블에서 index full scan이 처리됩니다.
이 문제를 해결하기 위해서는 전문 검색으로 탐색된 product에 연관된 review들만 선택한다면 많은 rows의 값을 줄일 수 있을 것입니다.

```sql
SELECT p.product_id, p.product_name, p.product_corp, COUNT(*) AS review_count
FROM product p
LEFT JOIN review r ON p.product_id = r.product_id
WHERE MATCH(p.product_name, p.product_corp) AGAINST('크리스피')
GROUP BY p.product_id
LIMIT 1, 20;
```

LEFT JOIN 안에 있던 서브쿼리를 바깥으로 빼내고 review 테이블을 조인하도록 변경했습니다. 그리고 GROUP BY를 전문 검색 이후에 하도록 변경을 했습니다.

하지만 여전히 쿼리가 3642(ms)정도로 매우 낮은 성능을 보였습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*VCbLuss8sfQzfCKTaS7l1Q.png)

![변경 후 쿼리의 실행계획](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*KCGvitbTyykblQGIPmAbsQ.png)

다시 실행계획을 분석해 보았을 때, 예상한 대로 전문 검색으로 탐색된 product의 레코드에 대해서만 review 테이블을 탐색하는 것을 볼 수 있습니다.

하지만 문제가 있습니다.

전문검색에서 탐색되어 읽어온 product의 데이터가 90911개 입니다.

product 한개당 읽어온 review 레코드는 평균 10개로 Nested loop left join 을 할 때 대략 90만개의 데이터를 처리하게 되고, 마지막 레코드를 읽는데 3065(ms)이 걸리는 만큼 여전히 성능 저하가 나타납니다.

실행 계획을 조금 더 살펴보면 908796개의 레코드를 조인 결과로 얻고 Group By와 같은 집계 연산을 위해 임시 테이블을 만든 후 limit 작업을 수행합니다.

변경 전 쿼리는 limit Optimization가 이루어져서 전문 검색을 수행하며 필요한 레코드의 개수만 읽었으나 이번에는 이 작업이 이루어지지 않았습니다.

limit Optimization 관련해서는 아래의 링크를 참고해 주세요!

> [https://www.percona.com/blog/mysql-explain-limits-and-errors/](https://www.percona.com/blog/mysql-explain-limits-and-errors/)
> [https://dev.mysql.com/doc/refman/8.4/en/limit-optimization.html](https://dev.mysql.com/doc/refman/8.4/en/limit-optimization.html)

```sql
SELECT p.product_id, p.product_name, p.product_corp, COUNT(*) AS review_count
FROM product p
LEFT JOIN review r ON p.product_id = r.product_id
WHERE p.product_name LIKE "%크리스피%"
GROUP BY p.product_id
LIMIT 1, 20;
```

추가로 한가지 궁금증이 생겨 똑같은 구조의 쿼리에서 전문 검색 대신 WHERE LIKE 문을 사용해 봤습니다. 그런데 놀랍게도 35(ms)로 매우 빠른 성능이 나왔습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*c1Myglfsdb9k7QfT-SHWWA.png)

![WHERE LIKE 문을 이용한 실행계획](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*sEdC_aDLT5z7Dc7hSOG_Vw.png)

제일 먼저 product 테이블에 기본 키(PRIMARY)를 활용한 인덱스 스캔이 수행되었습니다.
product_name에는 인덱스가 걸려있지 않습니다.
(인덱스를 설정하더라도 와일드카드(%)가 앞에 있으면 인덱스를 활용할 수 없음)
따라서 기본 키를 활용하여 레코드를 읽으며 “크리스피"가 들어가있는 레코드를 필터링합니다.

여기서 MySQL은 순차적으로 레코드를 스캔하면서 limit 20에 약간의 여유를 두고 22개의 레코드를 찾는 즉시 탐색을 종료합니다. 그 과정에서 총 240개의 레코드를 탐색했습니다. 즉, limit optimization이 이루어졌고 별도의 임시테이블 생성 없이 Group By 집계연산이 수행되었습니다.

전문검색을 활용할 때는 MATCH AGAINST 특성상 이러한 최적화가 이루어지지 못한 것 같습니다.

하지만 그렇다고 이런 방식의 검색을 활용하자는 것은 아닙니다. 만약 테이블의 레코드가 매우 많고 찾고자 하는 데이터가 매우 뒤에 있다면 Index full scan을 할 때 매우 많은 시간이 걸릴 수 있습니다.

사실 이러한 현상은 더미데이터를 삽입할 때 JdbcTemplate의 batch insert를 사용하면서 데이터에 규칙성이 생겼고, 이로 인해 limit 개수가 빠르게 충족되었습니다.

또한 전문 검색 시에는 이러한 규칙성으로 인해 하나의 전문 검색 인덱스에 연관된 데이터가 과도하게 많이 저장되어(예시의 경우 90911개) 스캔 대상이 증가하는 문제가 발생했습니다.

두번째 쿼리 수정
=========

```sql
SELECT p.product_id,
       p.product_name,
       p.product_corp,
       (SELECT COUNT(*) FROM review r WHERE r.product_id = p.product_id)  AS review_count
FROM product p
WHERE MATCH(p.product_name, p.product_corp) AGAINST('크리스피')
LIMIT 1, 20;
```

여기서는 JOIN을 사용하지 않고, Subquery를 사용했습니다.
먼저, product 테이블에 전문검색 결과를 탐색하고 그 결과에 해당하는 review만 갯수를 구했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*DbM_4E-Fvny12o9xiBYfCA.png)

![subquery을 이용한 경우 실행 계획](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*fm7gj2G8nJNl1lJAMAMyew.png)

실행계획을 살펴봤을 때, 전문검색의 결과에서 limit 20에 근접하는 레코드 갯수를 읽어왔습니다.
그리고 여기서 총 쿼리의 실행시간은 84(ms)정도 나왔습니다.

결과적으로 인덱스를 수정하기 보단, 쿼리의 구조를 조인에서 서브쿼리로 변경하는 것으로 실행시간이 3000~4000(ms)에서 100(ms)미만으로 30배 이상 향상되었습니다.

요구사항 추가
=======

사실 요구사항이 하나 더 존재했습니다.

각 product에 대한 상위 3개 태그를 표시합니다. 사용자들이 리뷰 작성 시 자유롭게 선택한 태그들 중 가장 많이 선택된 3개를 보여줍니다.

![쿼리 실행 결과](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*OGQNdTKE2fAh8AyGLtcNuw.png)![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*wqAMGHOtgMFZe6t2A7ydjg.png)

```sql
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
CREATE TABLE diet_tag (
    diet_tag_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    diet_tag_name VARCHAR(255) NOT NULL,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL
);
```

diet_tag테이블에는 미리 정해진 10개의 데이터가 존재하고 product_diet_tag 테이블에는 2000만개의 데이터가 저장되어 있습니다.

DML은 위와 같이 작성했으나 요구사항을 만족하는 쿼리를 작성하는 도중 너무 복잡하여 diet_tag 테이블에 있던 diet_tag_name 컬럼을 product_diet_tag테이블에 역정규화를 진행했습니다.

![역정규화를 진행한 테이블](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gbyq3dP_rYJb3BHqrZeLfw.png)

```sql
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
WHERE MATCH(p.product_name, p.product_corp) AGAINST('크리스피')
LIMIT 1, 20;
```

앞선 쿼리와 유사하게 여기서도 Subquery로 풀어나갔습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*b5YmhCJM97gtz20FWORTEQ.png)

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*OmaEboWkXDHaxPWZ-vdAgg.png)

쿼리의 수행시간은 평균적으로 150~250(ms)가 나왔습니다.
작업 하나 하나가 큰 비용이 들지는 않지만, 여러 과정이 누적되니 대략 2~3배정도 시간이 늘어났습니다.

더 나은 방법은 없을까요?
==============

여기서 쿼리를 더욱 최적화를 할 방법을 찾아볼 수 있겠지만, 과연 이러한 한방 쿼리가 적절한지 생각해볼 필요가 있습니다.

여기서 문제점은 수행시간이 증가한 부분도 있지만, 서비스의 비즈니스 로직들이 쿼리안에 녹아 들어있습니다.

만약, 비즈니스 로직에 변경이 생긴다면 쿼리도 유지보수의 대상이 되며 이러한 부분은 도메인 모델이나 비즈니스 모델에서 풀어나가는 방법이 유지보수 관점에서 조금 더 유연하게 받아들이기 쉬워 보입니다.

또한, 테스트 코드를 작성하기도 어렵습니다. 레포지토리의 테스트 코드를 작성한다면 하나의 테스트에 검색 여부, 리뷰 개수, tag정보 이 모든 걸 반영한 테스트를 작성해야 하고 요구 사항이 늘어날수록 레포지토리의 테스트 코드도 계속 수정해 나가야 합니다.

다음 편에서는 쿼리에 녹아있는 비즈니스 로직을 도메인 영역과 비즈니스 영역으로 옮기며 쿼리를 개선해 보겠습니다.

[다음 편: MySQL의 전문 검색(Full-Text Search)은 100만개가 넘는 데이터에서 문제가 없을까?](https://github.com/koo995/portfolio/blob/main/nutri-diary/elasticsearch.md)