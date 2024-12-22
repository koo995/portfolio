[MySQL] Left Join에서 Subquery로 변경 후 쿼리 성능 30배 향상하기.
====================================================

본 포스팅은 개인 프로젝트[[GitHub](https://github.com/f-lab-edu/nutri-diary)]에서 발생한 내용이며 [원글](https://medium.com/@gunhong951/%EC%A0%84%EB%AC%B8%EA%B2%80%EC%83%89-%EA%B3%BC%EC%97%B0-%EB%8D%94-%EB%B9%A0%EB%A5%B8%EA%B0%80-3db2e3fa0c89)은 여기서 확인하실 수 있습니다.
> 기존 쿼리 수행 시 약 3~4초가 소요. 실행 계획을 분석하여 문제점을 파악한 후, 쿼리 구조를 변경.
>
> 그 결과, 쿼리의 수행 시간이 약 0.08초로 줄어들어 30배 이상의 성능 향상을 달성.

먼저, 각 테이블의 DDL은 다음과 같습니다.

외래키는 사용하지 않았고 대신 인덱스는 별도로 설정했습니다.

```sql
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

요구사항으로는 product 테이블을 전문 검색(FULLTEXT SEARCH)할 때, 검색결과 product와 함께 해당 product에 해당하는 review의 갯수도 함께 보여줘야 합니다.

현재 product의 데이터는 100만개, review의 데이터는 1000만개가 삽입되어있습니다.

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

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*r-qEWXWA9AwjQCjHTINC4w.png" width=70%>

위의 요구사항을 위해 작성한 쿼리는 위와 같고, 예시를 위해 “크리스피"라는 단어로 전문 검색을 실행했습니다.

하지만 위의 쿼리를 적용하고 테스트를 해봤을 때 3913(ms)초의 시간이 걸리는 만큼 심각하게 느린 속도를 보입니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*8Zs2U9qbPE92nNJi2Uaw9w.png" width=70%>
<img src="https://miro.medium.com/v2/resize:fit:2000/format:webp/1*86YKH_qfed7f2WTFFmwwiQ.png" width=90%>

실행 순서는 다음 기준으로 읽으면 됩니다.

*   들여쓰기가 같은 레벨에서는 상단에 위치한 라인이 먼저 실행.
*   들여쓰기가 다른 레벨에서는 가장 안쪽에 위치한 라인이 먼저 실행.

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
이때, 테이블에서 읽은 rows가 1000만개(10e+6)이고 마지막 row를 가져오는데 걸린 시간이 무려 3220(ms)나 걸렸습니다. GROUP BY product_id와 count()을 활용하여 review의 갯수를 집계할 때, 앞선 작업에서 모든 인덱스를 탐색하느라 시간이 많이 소모되었습니다.

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

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*VCbLuss8sfQzfCKTaS7l1Q.png" width=70%>
<img src="https://miro.medium.com/v2/resize:fit:2000/format:webp/1*oQai6VZIHHRQMyWe4xTR-w.png" width=90%>

다시 실행계획을 분석해 보았을 때, 예상한 대로 전문 검색으로 탐색된 product의 레코드에 대해서만 review 테이블을 탐색하는 것을 볼 수 있습니다.

하지만 문제가 있습니다.

전문검색에서 탐색되어 읽어온 product의 데이터가 90911개 입니다.

product 한개당 읽어온 review 레코드는 평균 10개로 Nested loop left join 을 할 때 대략 90만개의 데이터를 처리하게 되고, 마지막 레코드를 읽는데 3065(ms)이 걸리는 만큼 여전히 성능 저하가 나타납니다.

아무래도 Group By 와 같은 집계 연산이 전문 검색 뒤에 있어서 모든 레코드를 읽고 마지막에 집계 연산을 위한 임시 테이블 생성 후 limit을 정리하는 것 같습니다.

이 부분을 GPT를 이용해서 찾아보았고 답변은 아래와 같습니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*xCM50xog0IYEdLU49Zqyjg.png" width=50%>

변경 전 쿼리는 limit 최적화가 이루어져서 필요한 레코드의 갯수만 읽은 것으로 추정됩니다.

limit Optimization 관련해서는 아래의 링크를 참고해 주세요!

> [https://www.percona.com/blog/mysql-explain-limits-and-errors/](https://www.percona.com/blog/mysql-explain-limits-and-errors/)
>
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

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*c1Myglfsdb9k7QfT-SHWWA.png" width=70%>
<img src="https://miro.medium.com/v2/resize:fit:2000/format:webp/1*sEdC_aDLT5z7Dc7hSOG_Vw.png" width=90%>

제일 먼저 product 테이블에 기본 키(PRIMARY)를 활용한 인덱스 스캔이 수행되었습니다.
product_name에는 인덱스가 걸려있지 않습니다.
(인덱스를 설정하더라도 와일드카드(%)가 앞에 있으면 인덱스를 활용할 수 없음)
따라서 기본 키를 활용하여 레코드를 읽으며 “크리스피"가 들어가있는 레코드를 필터링합니다.

여기서 MySQL은 순차적으로 레코드를 스캔하면서 limit 20에 약간의 여유를 두고 22개의 레코드를 찾는 즉시 탐색을 종료합니다. 그 과정에서 총 240개의 레코드를 탐색했습니다. 즉, limit optimization이 이루어졌고 별도의 임시테이블 생성 없이 Group By 집계연산이 수행되었습니다.

전문검색을 활용할 때는 MATCH AGAINST 특성상 이러한 최적화가 이루어지지 못한 것 같습니다.

하지만 그렇다고 이런 방식의 검색을 활용하자는 것은 아닙니다. 만약 테이블의 레코드가 매우 많고 찾고자 하는 데이터가 매우 뒤에 있다면 Index full scan을 할 때 매우 많은 시간이 걸릴 수 있습니다.
(더미데이터를 삽입할 때, JdbcTemplate의 batch insert을 이용하여 넣다보니 규칙성이 들어갔고, 따라서 limit 갯수를 빠르게 충족한 듯 합니다.)

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

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*DbM_4E-Fvny12o9xiBYfCA.png" width=70%>
<img src="https://miro.medium.com/v2/resize:fit:2000/format:webp/1*fm7gj2G8nJNl1lJAMAMyew.png" width=90%>

실행계획을 살펴봤을 때, 전문검색의 결과에서 limit 20에 근접하는 레코드 갯수를 읽어왔습니다.
그리고 여기서 총 쿼리의 실행시간은 84(ms)정도 나왔습니다.

결과적으로 인덱스를 수정하기 보단, 쿼리의 구조를 조인에서 서브쿼리로 변경하는 것으로 실행시간이 3000~4000(ms)에서 100(ms)미만으로 30배 이상 향상되었습니다.

그러나 요구사항 추가
=======

사실 요구사항이 하나 더 존재했습니다.

product을 전문검색으로 탐색했을 때, 해당 product의 review 갯수와 함께 product에 연관된 product_diet_tag중에서 많이 선택된 상위 3개의 태그를 함께 보여줘야합니다. product_diet_tag는 사용자가 특정 product에 대한 review를 저장할 때 여러개가 선택될 수 있습니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*OGQNdTKE2fAh8AyGLtcNuw.png" width=70%>
<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*wqAMGHOtgMFZe6t2A7ydjg.png" width=60%>

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

DML은 위와 같이 작성했으나 요구사항을 만족하는 쿼리를 작성하는 도중 너무 복잡하여 diet_tag 테이블에 있던 diet_tag_name 컬럼을 product_diet_tag테이블에 역정규화를 진행했습니다.

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gbyq3dP_rYJb3BHqrZeLfw.png" width=60%>

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

<img src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*b5YmhCJM97gtz20FWORTEQ.png" width=70%>
<img src="https://miro.medium.com/v2/resize:fit:2000/format:webp/1*OmaEboWkXDHaxPWZ-vdAgg.png" width=90%>

쿼리의 수행시간은 평균적으로 150\~250(ms)가 나왔습니다.
작업 하나 하나가 큰 비용이 들지는 않지만, 여러 과정이 누적되니 대략 2\~3배정도 시간이 늘어났습니다.

여기서 쿼리를 더욱 최적화를 할 방법을 찾아볼 수 있겠지만,
과연 이러한 한방 쿼리가 적절할까요?

여기서 문제점은 수행시간이 증가한 부분도 있지만, 서비스의 비즈니스 로직들이 쿼리안에 녹아 들어있습니다.

만약, 비즈니스 로직에 변경이 생긴다면 쿼리도 유지보수의 대상이 되며 이러한 부분은 도메인 모델이나 비즈니스 모델에서 풀어나가는 방법이 유지보수 관점에서 조금 더 유연하게 받아들이기 쉬워 보입니다.

또한, 역정규화를 적용한 만큼 데이터의 정합성을 맞추기 위해 애플리케이션 레벨에서 별도의 관리가 필요하고 Group By을 이용한 집계 정보를 실시간으로 보여줄 필요가 있는지도 따져볼 필요가 있습니다.

다음 편에서는 쿼리에 녹아있는 비즈니스 로직을 도메인 영역과 비즈니스 영역으로 옮기며 쿼리를 개선해 보겠습니다.

다음 글: [API의 성능을 개선하기 위해선 어떻게 접근해야할까? (쿼리만 생각하면 될까?)](https://github.com/koo995/portfolio/blob/main/src/nutri-diary/es/README.md)
