# 구건홍 | 포트폴리오

# Project.

## Nutri-Diary(개인 프로젝트) GitHub: [https://github.com/f-lab-edu/nutri-diary](https://github.com/f-lab-edu/nutri-diary)

## Stack: Spring Boot, Spring Data JDBC, JdbcTemplate, MySQL, NCP(Naver Cloud Platform), Jenkins

## 2024.07 ~ 진행중
현재 개인적으로 진행 중이며, 백엔드 부분을 전담하고 있습니다. 2025년 1월부터는 팀원들과 함께 기획, 디자인, 클라이언트 부분을 재구성해 나갈 예정입니다.

## 프로젝트 소개
오늘 하루 섭취한 음식을 다이어리에 기록하고, 유용한 다이어트 식품에 대한 정보를 공유하는 서비스입니다.

사용자는 날짜별로 섭취한 식품을 기록하여 하루 전체 또는 각 식사별 영양 정보를 체크할 수 있습니다. 사용자들은 새로운 식품 정보를 등록하고 기존 식품에 대한 후기를 댓글로 공유하여 다이어터들의 정보 불균형을 해소할 수 있습니다. 또한 자신이 선호하는 식단에 해당하는 식품들을 탐색할 수 있습니다.

## 성과
프로젝트 시작 전에는 객체지향 프로그래밍에 대한 실질적인 이해가 부족했습니다. 하지만, 프로젝트를 진행하면서 코드 리뷰를 통해 이전에는 알지 못했던 문제점들을 발견할 수 있었습니다. 코드를 이해하기 어렵다는 평가를 받았고, 요구사항이 변경되는 경우 유연하게 수정하기 어렵다는 점 등이 있었습니다. 이를 통해 코드가 단순히 기능 구현이나 자신만의 기록이 아니라, 다른 개발자들과 함께 코드를 쉽게 이해할 수 있도록 작성하는 방법에 대해 깊이 고민하게 되었습니다.

## 서비스 아키텍처
<a href="link"><img src="https://github.com/user-attachments/assets/4f8a65e4-8c32-40e6-9735-8d8345152b6f" width="70%"></a>

---

## About

프로젝트 초기 단계에서 Figma를 활용해 화면 정의서를 작성하고 서비스를 구상했습니다. 이를 바탕으로 필요한 요구사항들을 정의해 나갔습니다. 그다음, 비즈니스 흐름에 따라 필요한 정보들을 저장할 DB 스키마를 구현했으며, 중복을 최소화하는 방향으로 설계를 진행했습니다.

HTTP 통신의 보안 취약점을 해소하기 위해 ALB(Application Load Balancer)에 SSL 인증서를 적용하여 HTTPS 프로토콜을 통한 안전한 통신을 적용했습니다.

리소스별 외부 접근을 제한하기 위해 Public Subnet과 Private Subnet을 분리했습니다. DB 서버와 애플리케이션 서버는 보안상 중요한 자원이므로 Private Subnet에 배치했습니다. 이후 필요한 경우에만 Bastion 서버를 통해 애플리케이션 서버에 접근할 수 있도록 설정했습니다.

가용성 확보를 위해 트래픽 증가에 대응하는 ALB와 Auto Scaling을 적용했습니다. 애플리케이션 서버가 상태를 저장하지 않는 특성을 고려해 라운드 로빈 방식의 ALB를 선택했습니다. 메모리나 CPU 사용량이 60% 이상을 5분 이상 유지하면 Scale-out을, 그 미만으로 내려가면 Scale-in을 적용하도록 설정했습니다.

CI/CD와 무중단 배포 인프라 구성. Jenkins 서버를 별도로 구축하여 개발 과정의 CI/CD를 수행하고, NCP의 SourceDeploy(AWS의 CodeDeploy와 유사)를 활용해 Auto Scaling Group에 Blue/Green 방식의 무중단 배포 인프라를 구성했습니다. 

리팩터링 과정에서 생산성 향상과 애플리케이션의 구조의 안정성을 위해 테스트 코드를 작성했습니다. JaCoCo 기준으로 테스트 커버리지 80% 이상을 유지했으며, 테스트 코드 작성이 어려운 경우에는 기존 애플리케이션 코드 구조의 문제점을 파악하고 리팩터링을 진행했습니다.

향후 JPA로 변경을 고려하여 영속성 계층의 의존성을 역전시켜 구현했습니다. 서비스 레이어가 영속성 문제에 독립적으로 동작하도록 인터페이스 기반의 간접 계층을 추가했습니다. 이를 통해 영속성 계층의 리팩터링이 서비스 레이어에 영향을 주지 않도록 설계했습니다.

H2와 MySQL의 DB 엔진 차이로 인한 여러 이슈를 해결하기 위해 TestContainers를 도입하여 서버 환경과 동일한 테스트 환경을 구축했습니다. 특히 MySQL의 전문 검색 기능을 사용하는 테스트 코드는 H2에서 실행이 불가능했기 때문에 이러한 선택이 필요했습니다.

---

## Issues

- 조회 쿼리의 전체 수행 시간 30배 향상
  > [자세히 보기: [https://github.com/koo995/resume/blob/main/src/nutri-diary/query/README.md](https://github.com/koo995/resume/blob/main/src/nutri-diary/query/README.md)]
  
  기존 쿼리 수행 시 약 3~4초가 소요. 실행 계획을 분석하여 문제점을 파악한 후, 쿼리 구조를 변경.
  그 결과, 쿼리의 수행 시간이 약 0.08초로 줄어들어 30배 이상의 성능 향상을 달성.
    
- Spring Data JDBC(또는 JPA)에서 DB 엔진에 따른 데이터 타입의 변환 차이
  >[자세히 보기: [https://github.com/koo995/resume/blob/main/src/nutri-diary/converter/README.md](https://github.com/koo995/resume/blob/main/src/nutri-diary/converter/README.md)]
    
  테스트 코드 실행 시 JSON 타입 컬럼값을 객체로 변환하지 못하는 문제가 발생. 상황을 구체적으로 파악하기 위해 여러 케이스를 테스트해 본 결과, JPA에서도 동일한 문제가 발생.
  이는 H2가 JSON 타입의 컬럼을 MySQL과 다르게 처리해서 발생하는 문제로 판단. 여러 해결 방안을 고려한 끝에, JSON 타입 대신 TEXT 타입으로 컬럼을 설정하기로 결정. 
    
- 단순한 비즈니스 로직의 과도한 복잡성을 개선하여 간결한 구조로 리팩터링
  > [자세히 보기: [https://github.com/f-lab-edu/nutri-diary/wiki/영양성분-계산-로직-리팩터링](https://github.com/f-lab-edu/nutri-diary/wiki/%EC%98%81%EC%96%91%EC%84%B1%EB%B6%84-%EA%B3%84%EC%82%B0-%EB%A1%9C%EC%A7%81-%EB%A6%AC%ED%8C%A9%ED%84%B0%EB%A7%81)]
  
  비즈니스 로직을 전략 패턴과 팩토리 패턴을 적용하여 구현. 하지만 코드 리뷰를 통해 이 접근 방식이 비즈니스 로직을 이해하기 어렵게 만들고, 요구사항에 비해 복잡하다는 피드백을 받음.
  총 3번의 리팩터링을 거쳐, 공통 부분을 추상화하고 Value Object를 생성. 이를 통해 11개의 클래스와 340줄의 코드를 5개의 클래스와 235줄로 줄여 더 간결하고 이해하기 쉬운 구조로 개선.
    

## Eco-Spot(팀 프로젝트) GitHub: [https://github.com/koo995/24_Solution_Challenge](https://github.com/koo995/24_Solution_Challenge)

## Stack: Spring Boot, Spring Data JPA, QueryDSL, MySQL, GCP(Google Cloud Platform), Firebase

## 2024.01 ~ 2024.02

본 프로젝트는 4명의 팀원이 함께 진행했습니다. 저는 백엔드를 전담했으며, 클라이언트는 안드로이드 앱으로 구현되었습니다.

## 프로젝트 소개

AI 서비스 Gemini를 활용해 생태 지도를 만들어가는 애플리케이션 서비스입니다.

사용자가 특정 생물을 촬영하면 Gemini가 해당 생물을 식별하여 명칭을 알려줍니다. 사용자는 촬영한 생물들로 개인 도감을 만들고, 다른 사용자들이 저장한 생물들의 위치를 지도에서 추적할 수 있습니다. 또한 특정 생물 촬영 미션을 만들고 서로의 미션을 달성하면서 즐겁게 생태지도를 더욱 풍성하게 만들어갈 수 있습니다.

## 성과

프로젝트를 진행하며 여러 문제를 해결하는 과정에서 특히 기억에 남는 것은 스프링의 트랜잭션 처리 방법과 DB의 트랜잭션 처리 방법에 대한 이해가 깊어진 점입니다. 프로젝트 초기에는 기본기 부족으로 제대로 이해하지 못하고 잘못된 방식으로 문제를 해결했습니다. 그러나 몇 개월 후 다시 문제를 해결하는 과정에서 더 근본적인 문제점들을 발견하게 되었고, 이를 통해 실질적인 성장을 경험했습니다.

## Architecture
<img width="500" alt="arch" src="https://github.com/user-attachments/assets/57a740c1-78c3-4492-b314-ab67c9c0063f">
---

## About

매주 오프라인 미팅을 통해 4명의 팀원이 프로젝트 진행 상황과 의견을 공유했습니다. 초기에는 온라인으로 미팅을 진행했으나 의사소통에 어려움이 많아 오프라인 미팅으로 전환했습니다. 그러나 팀원들 간 구현 역량과 프로젝트 목표에 차이가 있어 각자의 현재 상황을 파악하고 공유할 필요성을 느꼈습니다. 이에 팀원들과 솔직한 대화를 나누어 각자의 강점과 약점을 공유했고, 이를 바탕으로 팀원들의 역할을 효과적으로 분담했습니다. 저는 이 과정에서 백엔드를 전담하게 되었습니다.

GCP를 활용해 간단한 인프라를 구성했습니다. 팀원들마다 선호하는 기술 스택과 아키텍처가 달랐지만, 시간적 제약으로 인해 제가 인프라 구성을 전담하게 되었습니다. 요구사항을 효율적으로 충족할 수 있는 심플한 아키텍처를 설계했고, 이를 통해 의견 충돌을 최소화하며 프로젝트를 신속하게 진행할 수 있었습니다.

---

## Issues

- JPA에서 엔티티 간의 연관관계 확인을 위한 의도치 않은 쿼리 발생
  > [자세히 보기: [https://github.com/koo995/resume/blob/main/src/eco-spot/one-to-one/README.md](https://github.com/koo995/resume/blob/main/src/eco-spot/one-to-one/README.md)]
  
  일대일 양방향 관계에서 Lazy loading이 적용되지 않는 문제가 발생. 원인을 분석한 결과, 외래키를 관리하는 엔티티가 아닌 반대 방향에서 조회 시 연관관계 확인을 위해 불필요한 쿼리가 발생하는 것을 확인.
  이를 해결하기 위해 불필요한 양방향 관계를 단방향 관계로 변경하여 필요 없는 쿼리 발생을 방지.
    
- Firebase을 이용한 인증 과정에서 동시성 문제
  > [자세히 보기: [https://github.com/koo995/resume/blob/main/src/eco-spot/transaction/README.md](https://github.com/koo995/resume/blob/main/src/eco-spot/transaction/README.md)]
  
  인증 과정에서 중복 회원가입으로 인한 API 실행 에러 발생. 문제 해결을 위해 인증 과정을 원자화해야 한다고 판단했지만 문제 해결 실패.
  상황을 더 분석한 결과, Spring AOP 트랜잭션 처리와 애플리케이션/DB 원자성 구분 미흡이 원인. 해결책으로 유니크 제약 조건을 적용하고 재시도 로직을 구현.
