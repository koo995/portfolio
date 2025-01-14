Spring Boot Tomcat threads.max와 HikariCP maximum-pool-size를 기본으로 설정하면 나타나는 현상?
==============================================================================

[이전 글: MySQL의 전문 검색(Full-Text Search)은 100만개가 넘는 데이터에서 문제가 없을까?](https://github.com/koo995/portfolio/blob/main/nutri-diary/elasticsearch.md)

앞선 글에서 Elasticsearch를 도입하여 검색 API의 성능을 크게 향상시켰습니다.

1초 정도의 응답 시간이 충분하다고 생각했으나, 성능 테스트를 하며 트래픽이 몰리는 상황에서 요청을 하나 직접 날려보았습니다.

결과는 역시 1초 정도 걸렸지만 전혀 빠르다고 느껴지지 않았습니다.
특히, POST 요청을 날렸을 때(1200ms)는 조금 느리네? 생각마저 들었습니다.

이 결과가 만족스럽지 않아 Grafana 대시보드의 여러가지 지표들을 살펴보았습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*MHOW7M5DM4_2168wylWYqA.png)
![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*V0FwBps466aSHqtQUcD9ng.png)

이때 몇 가지 눈에 띄는 지표들이 보였습니다.
바로 JVM의 Thread 상태와 HikariCP의 커넥션 상태였습니다.

timed-waiting 상태의 스레드가 197개, HikariCP의 커넥션을 기다리는 스레드가 189개로 나타났습니다.

Tomcat 스레드 풀의 최대 사이즈 기본값은 200개.
HikariCP 커넥션 풀의 최대 사이즈 기본값은 10개입니다.

그리고 이 값들을 모두 기본 값으로 세팅하여 사용했을 때, 위의 대시보드와 같은 결과가 나타났습니다.

각 스레드는 실제로 어떤 메서드를 실행하고 있을까요?
=============================

Connections의 Pending 값을 보면 대략적으로 **timed-waiting** 상태에 있는 스레드들은 커넥션 풀을 얻기 위해 대기하고 있는 것으로 추정됩니다

하지만 100퍼센트 확신하기가 어려웠습니다.

왜냐하면, 테스트 중인 API에서는 Elasticsearch도 사용하고 있었습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*jhlYWkDN0uSq2vcA17uA3g.png)

Elasticsearch client의 설정 코드를 살펴보면

`DEFAULT_MAX_CONN_PER_ROUTE`: 단일 노드에 허용되는 최대 동시 연결 10개.

`DEFAULT_MAX_CONN_TOTAL`: 모든 노드에 허용되는 총 동시 연결 30개.

위의 값을 기본으로 사용하고 있습니다.

커넥션을 얻기 위해 Pending하고 있는 스레드는 189개인데, timed-waiting 스레드의 개수는 197개입니다. 8개 정도 차이가 납니다.

그리고 waiting 상태에 있는 스레드도 9개 정도 있습니다. 이 스레드들은 뭐하는 것인지 알기가 어려웠습니다.

따라서 Thread dump을 이용하여 각 스레드들의 상태를 조금 더 자세하게 살펴볼 필요가 있었습니다.

Thread dump를 이용한 분석
===================

Thread dump는 jcmd 명령어로 획득했고, 분석은 [fastthread.io](https://fastthread.io/) 사이트를 이용했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*FkmYznHC8xRWC41D93F8ew.png)

**runnable** 상태의 스레드 호출 스택
--------------------------

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*RX-WIIgPxnTbioJXQck3jA.png)![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*ZnLilR8OLiniJPp5qOa9qQ.png)

3개의 스레드 덤프를 획득하여 비교해보니 **runnable** 상태의 스레드는 대부분 JVM 내부의 작업과 GC를 처리하는 스레드로 구성되어 있고 실제로 커넥션에서 데이터를 읽고 있는 스레드는 2개정도 있었습니다.

MySQL를 읽는 스레드 1개, Elasticsearch를 읽는 스레드 1개 또는 MySQL를 읽는 스레드 2개, Elasticsearch를 읽는 스레드 0개인 경우가 있었습니다.

waiting 상태의 스레드 호출 스택
---------------------

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*_at5gnszLg_EQ7QhdBH0Ig.png)

waiting에 해당하는 스레드는 총 11개 이며 그 중에서 8개의 스레드가 Elastisearch의 데이터를 waiting 상태에서 기다리고 있었습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*s8mxHIl5wQsS_oSUOZzBIg.png)

소스 코드를 살펴보면 Elasticsearch는 요청을 처리할 때 비동기로 호출하고 결과를 기다린다고 적혀있습니다.

timed-waiting 상태의 스레드 호출 스택
---------------------------

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*XvXVn9PKKFFG1UTWPkrvrA.png)![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*qkzxYP0X1t2GGeYjcxhoIA.png)

총 합 190개의 스레드가 HikariCP의 커넥션을 얻기 위해 **timed-waiting** 상태에 있습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*u_U5QzEySbVZxtHLRbPVjg.png)

예를 들어 Thread1 ~ Thread9+ 이 동시에 `HikariDataSource.getConnection()` 메서드를 호출했다면 Thread1 ~ Thread5만 커넥션을 획득하고 나머지는 **timed-waiting** 상태에서 지정된 시간(`hikari.connection_timeout default 30s`)동안 반환된 커넥션을 얻기 위해 대기합니다.

이제 무엇이 문제일까요?
=============

현재 테스트를 위해 사용하고 있는 서버의 스펙은
vCPU 2개, Memory 8GB 단일 인스턴스입니다.

1.  서버 인스턴스의 CPU 개수에 비해서 Tomcat 스레드의 갯수가 너무 많습니다. 기본적으로 애플리케이션 서버가 IO Bound 작업이 대부분이지만 2개의 CPU 갯수에 비해서 200개의 스레드는 너무 많다고 생각됩니다.
2.  요청을 처리하는 스레드는 200개인데 반해 HikariCP의 커넥션은 10개가 최대입니다. 커넥션을 얻지 못한 수많은 나머지 스레드는 대기하게 되고 이는 결국 요청과 응답 사이의 처리 시간이 증가하는 문제로 이어집니다.

Tomcat의 max-thread와 HikariCP의 maximum-pool-size를 변경한 테스트
========================================================

적절한 Thread 수와 Connection 수를 어떻게 정할까 찾았지만 정해진 규칙은 없는 것 같았습니다.

IO Bound 작업은 일반적으로 CPU 사용량보다 네트워크 IO와 같은 대기 시간이 많은 작업에 의존합니다. 따라서 CPU 코어 개수보다 많은 스레드를 생성해도 컨텍스트 스위칭 비용이 상대적으로 적습니다.

그렇다면 커넥션 대기 시간을 줄이기 위해 스레드와 커넥션 풀의 크기 차이를 조절하고 여러번의 테스트를 통해 비교하겠습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*z8syub-Vr8FXFluO_WVi1A.png)![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*OAWiRxb9tCLkZfhwFHJ1rw.png)

결과를 확인하면

Tomcat의 max-thread: 60개
HikariCP의 max-connection-pool: 40개

위의 설정값을 적용했을 때 응답 시간이 약 200ms으로 가장 빠른 응답 시간이 나왔습니다.

이미지에는 추가하지 않았지만

Tomcat의 max-thread: 30개
HikariCP의 max-connection-pool: 20개

의 경우도 테스트를 진행해 보았습니다. 하지만 다른 문제점이 있어 아래에서 따로 살펴보겠습니다.

커넥션 수를 늘렸는데 DB 서버는 괜찮을까요?
=========================

![captionless image](https://miro.medium.com/v2/resize:fit:1002/format:webp/1*cfyZ59swt5TdvLBZhBcS7A.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1000/format:webp/1*X1aSMsVP94XO1YwmJTwAgQ.png)
--- | ---

보시는 것과 같이 크게 유의미한 차이는 없었습니다.
대부분의 running에 해당하는 커넥션은 1~2개 정도만 필요했으며 DB 인스턴스의 CPU 사용량이 약간 증가했습니다.

다른 희생된 자원은 없을까요?
================

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*uyfOjVrWHgkxSHanSYv4jw.png)|![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*-d_4AnQ3_0nSO_GjmK9o7g.png)![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*ymdF_IozL-fOTi2QoUo1eQ.png)

응답 시간은 감소했으나, runnable 상태의 스레드 수와 애플리케이션 서버의 CPU 부하가 조금 증가했습니다.

POST 요청은 500ms로 충분할까요?
======================

지금까지 실제 상황을 가정하여 요청 비율은 GET 90%, POST 10% 로 설정했습니다. 그리고 product을 저장할 때 MySQL과 Elasticsearch의 데이터 동기화는 애플리케이션에서 직접 관리하는 가장 단순한 방식을 선택했습니다.

코드 레벨에서는 하나의 스레드가 Elasticsearch에 데이터를 저장하고 뒤이어 MySQL에 데이터를 저장합니다.

만약 Elasticsearch 데이터 저장 시 즉각적인 응답이 필요하지 않다면, 이 작업을 비동기 처리를 통해 응답 시간을 단축할 수 있을 것으로 보입니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*Ma1o_lQIn1kvuXua0SizXQ.png)

결과를 확인하면 GET 요청은 2번의 테스트에서 모두 200ms 정도를 유지하였고 POST 요청은 약 500ms에서 125ms까지 줄어들었습니다.

최종 결과
=====

GET 요청: 1초 -> 200ms (5배 개선)
POST 요청: 1200ms -> 125ms (10배 개선)

정리
==

처음 API를 개발할 때는 단순히 리뷰 개수를 product와 review 테이블을 JOIN해서 원하는 결과만 가져오면 충분하다고 생각했습니다.

하지만 전문 검색 인덱스를 사용하면서 쿼리 수행 시간이 크게 늘어나는 문제가 발생했습니다. 실행 계획을 분석하고 쿼리 구조를 개선하여 수행 시간을 단축했지만, 여전히 문제가 남아있었습니다.

API에서 제공해야 할 요구사항이 늘어날수록 이를 억지로 쿼리에 추가하다 보니 쿼리가 복잡해지는 문제였습니다.

또한, product 테이블을 탐색하는 것은 전문 검색을 사용하는 경우뿐만 아니라 다양한 경우에도 사용될 수 있었습니다.

이때마다 복잡한 서브 쿼리나 조인을 통해서 review의 갯수와 tag를 가져오는 쿼리를 작성하는 것은 비효율적이라 판단했고, 이에 따라 프로세스와 테이블 구조를 더 유연하게 재사용할 수 있도록 개선할 필요성을 느꼈습니다.

하지만 테이블 구조를 변경한다면 새로운 데이터를 넣어야 하고, 이전 테이블 구조와 다른 데이터를 가진 상태에서 쿼리 시간을 비교하는 것은 부적절하다고 판단했습니다. 더 정확한 기준을 잡기 위해 고민하던 중, 처음부터 해결하고자 했던 문제가 무엇이었는지 돌이켜 보았고 해당 쿼리를 포함한 API의 성능을 개선하는 것이 원하던 목표였음을 깨닳았습니다.

처음에는 쿼리 튜닝으로만 문제를 해결하려 했습니다. 그러나 쿼리 튜닝은 문제를 해결하기 위한 여러 도구 중 하나의 방법이였고 문제 그 자체가 아니였습니다. 이후 더 넓게 문제를 정의하니 고려할 수 있는 선택지가 많아졌고 기존 API 개발 방식의 문제점도 파악할 수 있었습니다.

그리고 결과적으로 사용 가능한 수준의 API를 개발할 수 있게 되었습니다

여기서부터는 추가로 개선하거나 알아볼 내용들입니다.
============================

테스트를 수십 번 반복하면서 여러 가지 의문점들과 개선해야할 문제들이 더 있었습니다.

첫 번째
----

[Grafana 대시보드 누락(단절?) 현상.](https://medium.com/@gunhong951/grafana-대시보드-누락-단절-현상-057841a00bc6?source=post_page-----3567f9bca2df--------------------------------)

윗글의 원인을 파악하였더니 성능 테스트 과정에서 모든 스레드가 요청을 처리하는 데 사용되면서 스레드 반환에도 시간이 증가했습니다.

이로 인해 Prometheus의 메트릭을 가져오는 요청에 대한 스레드 획득 시간이 증가하고 결국 응답 시간도 증가하는 현상이 나타났습니다.

이러한 현상은 Prometheus의 요청뿐만 아니라 다른 API의 요청에도 나타날 것이라 생각하여 테스트를 진행해 봤습니다.

기존에는 Elasticsearch 관련 자원인 product에 대한 GET과 POST 요청만 처리했으나, 이번에는 이와 무관한 다른 요청들을 시나리오에 추가했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*y7yTeL1Hd6GR8gLHSxFftw.png)

결과는 모든 요청의 응답 시간이 증가하여 800ms 이상 소요되었습니다.
(스레드 풀과 HikariCP 커넥션 풀을 조절하기 이전의 테스트 결과입니다.)

빨간색으로 박스친 `GET /store` `GET /store/{store_id}`요청은 단독으로 진행한 테스트에서 100~200ms 정도의 평균 응답 시간을 보이는 매우 가벼운 요청입니다.

하지만 상대적으로 무거운 작업인 `GET /product` 요청과 `POST /product/new` 요청과 함께 테스트를 진행했더니 결국 요청의 응답 시간이 늘어나는 문제로 이어졌습니다.

이러한 현상이 나타난 이유는 서블릿 기반의 톰캣 방식(Spring MVC + Tomcat)이 블로킹(Blocking) 방식으로 동작하기 때문입니다.

요청을 처리할 때 톰캣 스레드가 서블릿 컨테이너에 들어오는 요청을 하나씩 맡아 처리하고, 해당 로직이 끝날 때까지 스레드를 점유하고 있습니다. 따라서 해당 스레드는 Elasticsearch와 DB의 IO작업이 끝날 때까지 다른 요청 처리에 사용되지 못하고 대기하게 됩니다.

이 때문에 최악의 경우 특정 API의 요청이 오랜 시간 IO 작업에 묶여 있으면 톰캣의 스레드 풀이 고갈되고, 다른 요청까지 대기해야 하는 상황이 벌어집니다.

이를 해결하기 위해서는 크게 논블로킹(Non-Blocking)방식으로 API호출(Spring WebFlux), 스레드 풀 분리 또는 서버를 분리하는 방법들을 고려할 수 있습니다.

두 번째
----

Tomcat의 max-thread: 60개
HikariCP의 max-connection-pool: 40개

지금까지 위의 설정값을 적용했을 때 응답 시간이 약 200ms으로 가장 빠른 응답 시간이 나왔다고 글을 작성했습니다.

하지만 사실

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*jEyuaGWW_uNEOSR1dBUiww.png)

Tomcat의 max-thread: 30개
HikariCP의 max-connection-pool: 20개

설정값을 적용했을 때 응답시간이 조금 더 빨랐습니다.

그렇지만 이상한 점이 하나 있었습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dj7ECcJnCcrBkTO0z9kQ_A.png)

max-thread 30개 HikariCP의 max-connection-pool이 20개인 경우 초당 처리량(RPS)은 오히려 더 낮게 나타났습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*b0b9lymPtKLCaRKTyvqIcw.png)

간단히 설명하면 쓰레드 수를 줄이면 컨텍스트 스위칭이나 커넥션 경합 같은 오버헤드가 줄어듭니다. 따라서 CPU의 사용량도 줄어들고 적은 수의 요청만이 서버 내부 자원을 넉넉하게 사용하기 때문에 단건 응답 시간은 빠르게 나올 수 있었습니다.

하지만 동시에 받을 수 있는 요청이 제한적이므로, 많은 요청이 들어오면 스레드 풀이 꽉 차서 새 요청을 빨리 처리하지 못합니다. 그 결과 RPS가 낮은 결과가 나타났습니다. 트레이드 오프 현상입니다.

세 번째
----

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*jnmdOowlxS7haWVJJGRp5Q.png)

Tomcat의 max-thread: 200개
HikariCP의 max-connection-pool: 180개

이 경우는 RPS의 차이는 거의 없었지만 응답 시간이 상당히 길게 나왔습니다.

그리고 그 이유를 위의 트레이드 오프 상황과 유사하게 스레드가 증가하여 CPU 등 리소스 경합이 더 빈번해지고, OS 레벨에서의 컨텍스트 스위칭 비용이 증가하여 응답 시간이 길게 나왔다고 생각했습니다.

하지만 조금 더 살펴보니 기존의 문제였던 timed-waiting 상태의 스레드는 줄었지만 오히려 waiting 상태의 스레드는 증가했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*UpvvEVPATsiuVQNzSA3i2A.png)

스레드 덤프를 분석해 보았을 때 180개의 waiting 스레드 중에서 140개의 스레드가 Elasticsearch의 비동기 응답을 받기 위해 이 상태에 있었습니다.
Elasticsearch의 총 동시 연결은 30개로 위에서 소스 코드를 보여드렸습니다. 140개의 스레드가 30개의 연결을 차지하기 위해 경합이 발생한 것으로 보입니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*rqNzOPvZV_s7NN8Ycrcc_A.png)![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*5mxrVgHz85fRwndHzTaMLw.png)

또한 Connection Usage Time이 매우 높게 나왔습니다. 커넥션 풀의 개수가 여유로워 Connection Acquire Time은 낮게 나왔는데 그 뒤 사용 시간이 크게 나왔습니다.

runnable 스레드를 살펴보면 MySQL에서 데이터를 읽어 오고 있는 스레드는 다른 경우와 유사하게 최대 4개 정도 나타났습니다. 커넥션을 얻은 이후로 무슨 이유로 빨리 처리하지 못하는 것인지 waiting 스레드를 포함하여 관계를 조금 더 살펴볼 필요가 있습니다.