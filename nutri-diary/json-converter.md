[H2 Database] JSON 컬럼에 대한 Converter 에러 상황.
============================================

혹시나 마크다운 문법이 보기 불편하시다면 원글에 해당하는 블로그 글을 [여기](https://medium.com/@gunhong951/h2-database-json-%EC%BB%AC%EB%9F%BC%EC%97%90-%EB%8C%80%ED%95%9C-converter-%EC%97%90%EB%9F%AC-%EC%83%81%ED%99%A9-bedaa2ac3938)서 확인할 수 있습니다.

---

프로젝트를 테스트하는 도중 아래의 그림과 같이 컨버터를 찾을 수 없다는 에러사항이 발생했습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*hbbOVZ9_vUVt8w7D540hlw.png)

저는 현재 Spring Data JDBC 을 사용하는 중이며, 혹시나 컨버터가 제대로 등록되지 않았는지 체크해 보았지만 컨버터는 제대로 등록되었습니다.

상황을 조금 더 구체적으로 파악하기 위해 Data JPA, Data JDBC, H2, MySQL 을 번갈아 가며 살펴봤더니, 이러한 에러는 MySQL 로 개발할 때는 나타나지 않았으며 H2와 연결된 테스트 코드 실행 시에만 나타났습니다.

본 포스팅의 예제는 [여기](https://github.com/koo995/jsonConverter)서 확인하실 수 있습니다.

Data JPA 를 사용하는 경우

> Error attempting to apply AttributeConverter
> …
> Caused by: java.lang.RuntimeException: com.fasterxml.jackson.databind.exc.MismatchedInputException

Data JDBC 를 사용하는 경우

> No converter found capable of converting from type [byte[]] to type ~~

와 같은 에러 메시지가 나타납니다.

예시 상황을 재구성해서 테스트하기 위해 아래와 같은 코드를 구성해 보았습니다.
===========================================

```java
public class Member {
    @Id
    @Column("MEMBER_ID")
    private Long id;

    private String username;

    private Address address;

    public Member(String username, Address address) {
        this.username = username;
        this.address = address;
    } 
}
```

```java
// Data JPA의 컨버터와 Data JDBC의 컨버터에서
// Jackson 라이브러리의 ObjectMapper.writeValueAsString()를 이용하여
// Address <-> String 컨버팅
public class Address {
    private String street;
    private String city;
    private String state;
    private String zip;

    public Address(String street, String city, String state, String zip) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.zip = zip;
    }
}
```

```sql
# Address 객체를 JSON 타입으로 저장.
CREATE TABLE MEMBER (
    MEMBER_ID BIGINT PRIMARY KEY AUTO_INCREMENT,
    USERNAME VARCHAR(255) NOT NULL,
    ADDRESS JSON
);
```

결론부터 이야기하면 이는 H2 데이터베이스에서 JSON 타입을 처리하는 방식이 MySQL 과 달라서 나타나는 에러였습니다.

먼저, H2에서 JSON 타입을 다루는 방법을 살펴보겠습니다.
==================================

```sql
INSERT INTO MEMBER(USERNAME, ADDRESS) VALUES('테스트이름', '{"city":"seoul", "street":"nowon"}');
INSERT INTO MEMBER(USERNAME, ADDRESS) VALUES('테스트이름', JSON '{"city":"seoul", "street":"nowon"}');
```

![captionless image](https://miro.medium.com/v2/resize:fit:546/format:webp/1*_htWGco4lMxUewwjzbyibg.png) | ![captionless image](https://miro.medium.com/v2/resize:fit:1456/format:webp/1*Ah4a1O9kcYaaN1TIbvZ_Hw.jpeg)
--- | ---

H2 는 위와 같이 **첫번째 쿼리**를 실행하는 것으로 MEMBER_ID 3번 row와 같이 ADDRESS 컬럼에 escaped string 모양의 JSON이 저장됩니다.

(지금부터는 MEMBER_ID 을 생략하고 편하게 1, 2, 3번이라 하겠습니다.)

그리고 그리고 **두번째 쿼리**와 같이 **JSON 이라는 포멧을 지정**해주면 1, 2 번 row 와 같이 깔끔?한 형식으로 저장이 됩니다.

3번과 같은 escaped string JSON 형식을 [H2 에서는 **JSON String** 이라고 부르는 것 같습니다.](https://github.com/h2database/h2database/issues/3417#issuecomment-1027681852)

```sql
INSERT INTO MEMBER(USERNAME, ADDRESS) VALUES(?, ?);
# 2번째 바인딩 변수에 FORMAT JSON 명시
INSERT INTO MEMBER(USERNAME, ADDRESS) VALUES(?, ? FORMAT JSON);
```

그리고 Data JPA, Data JDBC 를 사용하여 데이터 접근 메서드를 실행하면 **첫번째 쿼리**와 같이 바인딩 변수 형식으로 쿼리가 날라가는 것을 본 적 있으실 겁니다.

**첫번째 쿼리**는 **3번 row** 와 같이 escaped string 형태의 JSON 데이터(Json String)로 저장됩니다.

**두번째 쿼리**는 바인딩 변수 뒤에 **FORMAT JSON**을 명시했습니다.

이 경우 **1, 2 번 row** 처럼 깔끔?하게 저장되지만, PreparedStatement을 사용해야 하므로 Data JPA나 Data JDBC 를 사용할 때는 적용하기 어렵습니다.

추가로 H2 문서에는 아래와 같이 방법이 적혀있습니다.

>[To set a JSON value with java.lang.String in a PreparedStatement use a FORMAT JSON data format (INSERT INTO TEST(ID, DATA) VALUES (?, ? FORMAT JSON)) or use setObject(parameter, jsonText, H2Type.JSON) instead of setString().](https://h2database.com/html/datatypes.html#json_type)

요약하면, 문자열을 String형식으로 JSON값을 넣지 말자.
쿼리 안에서는 FORMAT JSON을 써주거나, 자바 코드에서는 setObject()와 H2Type.JSON을 사용하라고 이야기합니다.

이제 MySQL에서 JSON 타입을 다루는 방법을 살펴보겠습니다.
====================================

```sql
INSERT INTO MEMBER(USERNAME, ADDRESS) VALUES('테스트이름', '{"city":"seoul", "street":"nowon"}');
INSERT INTO MEMBER(USERNAME, ADDRESS) VALUES(?, ?, ?);
```

[MySQL 에서 JSON 타입을 다루는 방법](https://dev.mysql.com/doc/refman/8.4/en/json.html#json-values)은 많지만, 위의 쿼리들과 같이 특정한 포멧 형식을 지정해주지 않더라도 아래의 이미지처럼 Escaped String 형식이 아닌 깔끔한 JSON형태로 DB에 저장됩니다.

따라서 Spring Data 기술을 사용해도 문제가 없습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*wdZiHSpKyblsmhul2J-5zw.png)

지금쯤이면 아래의 H2 이미지에서 1, 2번 row 와 3번 row 의 차이가 무엇인지 궁금하실 겁니다.
==========================================================

![위의 이미지를 다시 가져왔습니다.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Ah4a1O9kcYaaN1TIbvZ_Hw.jpeg)

H2 에서는 아래와 같이 설명을 합니다.

>[Attempt to write a string without “FORMAT JSON” claues will cause implicit conversion of character string value to simple JSON with a string literal inside it ('text' -> JSON '"text"').](https://github.com/h2database/h2database/issues/3782#issuecomment-1517086930)
>
>[when you pass a string literal to a JSON column it is converted to a JSON String object. If you have a string literal with a JSON text, you need to mark it explicitly with the FORMAT JSON clause.](https://github.com/h2database/h2database/issues/2389#issuecomment-572945919)

번역하면, FORMAT JSON을 명시하지 않고 String을 저장하면 문자열 리터럴 안에 JSON 문자열이 들어있는 형식으로 암묵적으로 변환됩니다.
(‘text’ -> JSON ‘“text”’).

만약 JSON 텍스트를 포함한 String 리터럴을 전달하려면, 명시적으로 FORMAT JSON 을 표시해야 합니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*YnNrq6VCLaJJXHMK03cP6Q.png)

Data JPA 또는 Data JDBC와 같은 데이터 접근 기술을 사용하여 객체를 H2 DB에 저장하면 위의 이미지와 같은 쿼리 로그가 나타납니다.

그리고 이 데이터를 SELECT 하여 가져와보면 아래와 같이 escaped string 리터럴이 또 한번 감싸진 것을 볼 수 있습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*UrMXFpb7iFr3P14J-M7Olg.png)

여기서 의문이 또 생길 수 있습니다.
====================

1.  저렇게 escaped string 데이터는 JSON 타입이 아닌 varchar 타입인가?
2.  따옴표가 들어간 것과 컨버터가 작동하지 못하는 것이 무슨 연관이지?

먼저, 첫번째 의문.
-----------

escaped 처리된 데이터도 JSON 타입이 맞습니다. 실제로 JSON이 아닌 일반적인 String(varchar)은 유효한 JSON이 아니여서 JSON 타입으로 지정한 H2의 컬럼에 INSERT가 불가능합니다.

[H2에서는 JSON 을 byte[] 또는 String으로 다룹니다](https://h2database.com/html/grammar.html#json). 그리고 저장된 결과에도 차이가 나는데 byte[] 로 H2 JSON컬럼에 저장하면 JSON Object(escaped 되지 않은 깔끔한 형태)가 저장됩니다.

하지만, Address 객체와 같이 Jackson라이브러리의 `ObjectMapper.writeValueAsString()`을 이용해서 String으로 변형하여 JSON 텍스트 형식의 리터럴을 H2 JSON 컬럼에 저장한다면 String 리터럴에 escaped 처리가 되어 저장됩니다.

이제 두번째 의문입니다.
-------------

Data JDBC 를 사용할 때는, byte[] 에서 “Address" 으로 변환하는 컨버터를 찾을 수 없다는 아래와 같은 에러메시지가 나타났고

> No converter found capable of converting from type [byte[]] to type [Address]

Data JPA를 사용할 때는, Address 객체 생성을 위해 String 값을 받는데 이때 String 값에서 Address 객체로 deserialize 할 수 없다는 에러 메시지가 출력되었습니다.

> Error attempting to apply AttributeConverter
> …
> Cannot construct instance of `Address` (although at least one Creator exists): no String-argument constructor/factory method to deserialize from String value (‘{“street”:”1234",”city”:”Main”,”state”:”St”,”zip”:”12345"}’)

먼저, Data JDBC 부터 살펴보겠습니다.
=========================

Data JDBC 는 컨버터(DB source value -> Address)를 선택할 때, 각 DB Driver에서 제공하는 타입의 포멧을 가지고 컨버터를 선택합니다. 이게 무슨 말이냐면, [H2는 저장된 값과 Java Object 의 매핑 방법에서 JSON 타입은 byte[] 타입으로 매핑합니다.](https://h2database.com/html/datatypes.html#json_type)

그리고 Data JDBC는 쿼리 실행 후 얻은 결과(H2.ResultSet)에서 엔티티로 변환하고자 할 때 Converter가 필요하고 이때, 적절한 Converter 의 타입을 찾는 데 있어서 H2.ResultSet 에 저장된 값의 타입을 힌트로 얻습니다. H2 는 이 과정에서 JSON 값인 경우 byte[] 타입을 전달합니다.

![H2 Driver 내부에 있는 ValueToObjectConverter](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*cBfRN07A6fwLFg0qRQwXbA.png)

그리고 그림으로 대략 표현하면 아래와 같은 흐름입니다.
(실제보다 많이 단순화한 것이라 이해를 위해 간단히 참고바랍니다.)

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*K5GBMaYtUABA3tC5QuI-Bw.png)

그래서 아래와 같은 String -> Address Converter 를 등록했더라도, 그것을 찾을 수 없다는 에러가 나타납니다.

```java
// Data JDBC의 컨버터
@Slf4j
@RequiredArgsConstructor
@ReadingConverter
public class JsonToAddressConverter implements Converter<String, Address> {

    private final ObjectMapper objectMapper;

    @Override
    public Address convert(String source) {
        try {
            return objectMapper.readValue(source, Address.class);
        } catch (IOException e) {
            log.info("JSON 타입을 Address 객체로 변경에 실패했습니다.");
            throw new RuntimeException(e);
        }
    }
}
```

그렇다면 Converter 의 타입을 byte[] 로 하면 어떨까요?
----------------------------------------

H2 는 JSON 데이터를 byte[] 로 매핑하기 때문에 INSERT 시 String 타입이 아닌 byte[] 로 매핑하면 JSON 타입의 **읽기와 쓰기가 모두 정상**적으로 동작합니다.

하지만 이 경우 MySQL 에서는 INSERT 할 때 에러가 나타납니다. 그리고 H2 만을 위해서 JSON 타입으로 다루어질 객체를 byte[] 로 바꾸는 것은 적절한 해결책이 되지 못합니다.

그러면 Data JDBC에서 왜 MySQL은 String 으로 주고 받아도 예외가 발생하지 않는 걸까요?
----------------------------------------------------------

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*TGg8KMMdKfK_9voxU9Xi1g.png)

[MySQL은 이 과정에서 JSON 값인 경우 String 타입을 매핑합니다.](https://github.com/mysql/mysql-connector-j/blob/release/9.x/src/main/user-impl/java/com/mysql/cj/jdbc/result/ResultSetImpl.java#L1234) 따라서 예외가 발생하지 않습니다.
(추가로 INSERT 할때도 String 타입으로 넣어야합니다. 그래서 byte[] 타입은 에러가 발생.)

이제 Data JPA 를 살펴보겠습니다.
======================

JPA 는 컨버터를 선택할 때 Data JDBC 와 다른 방법으로 작동합니다.
Data JDBC 는 연결된 DB의 Driver 에게 타입을 묻는 반면, JPA 는 변환하고자 하는 타입을 미리 가지고 있습니다. 그리고 그 타입에 맞게 ResultSet 에서 값을 추출합니다.

그림으로 간단히 표현하면 아래와 같습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gxFe7-rMTefAuBzClvtX-Q.png)

이러한 이유로 Data JPA는 Data JDBC와 다르게 컨버터를 찾을 수 없다는 에러가 나타나지 않습니다. 하지만 DB로 부터 읽어온 값을 String 값으로 변환 후 아래와 같은 컨버터를 사용하게 되는데,

```java
// Data JPA의 컨버터
@Slf4j
@RequiredArgsConstructor
@Converter(autoApply = true)
public class AddressConverter implements AttributeConverter<Address, String> {

    private final ObjectMapper objectMapper;

    @Override
    public String convertToDatabaseColumn(Address address) {
        try {
            return objectMapper.writeValueAsString(address);
        } catch (Exception e) {
            log.info("Address 타입을 Json 으로 변환할 수 없습니다.");
            throw new RuntimeException(e);
        }
    }

    @Override
    public Address convertToEntityAttribute(String source) {
        try {
            return objectMapper.readValue(source, Address.class);
        } catch (Exception e) {
            log.info("Json 타입을 Address 타입으로 변환할 수 없습니다.");
            throw new RuntimeException(e);
        }
    }
}
```

이 과정에서 DB에서 읽어온 String 타입의 value는 아래와 같습니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*pAc4IiOz8Wt6-NGxPCo2RA.png)

Jackson 라이브러리의 ObjectMapper은 escaped 처리가 된 문자열을 한번 더 감싼 String 리터럴을 Address 객체로 변환하지 못합니다.

![captionless image](https://miro.medium.com/v2/resize:fit:2000/format:webp/1*S41u0PDWxkgGeCeMANctew.png)

따라서 Address객체를 생성하는 MismatchedInputException 에러 메시지가 출력되었습니다.

![위의 이미지를 다시 가져왔습니다.](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Ah4a1O9kcYaaN1TIbvZ_Hw.jpeg)

추가적으로, JPA는 1, 2번 row 와 같이 깔끔하게 저장되어 있는 JSON 데이터를 컨버터의 source 타입을 String 으로 하던 byte[]로 하던 모두 읽어올 수 있습니다. 왜냐하면 Java의 기본적인 입출력은 바이트 단위로 운반되는데 JPA 의 컨버터 타입을 byte[] 로 지정해 두었으면 byte[] 그대로 읽고, String을 지정해 두었으면 읽어온 byte[] 단위의 값을 String으로 변환합니다. 즉, 타입에 따라 적절한 변환 전략을 선택합니다.

하지만 Data Jdbc는 1, 2번 row와 같이 깔끔하게 저장된 JSON 데이터조차
Address <-> String 컨버팅 전략인 경우 byte[] 과의 컨버팅 전략이 아니란 이유로 컨버팅할 수 없습니다.

그렇다면 Data Jdbc에서 컨버팅이 안되는 문제를 어떻게 해결할까요?
========================================

아직 Data Jdbc에서 이 문제가 해결되진 못한 것 같습니다. 개인적으로 오픈소스에 기여하고 싶은 욕심이 생겨서 원인을 찾아보았지만,

1.  데이터를 INSERT 할 때 H2 는 JSON 포멧을 지정해줘야 함. 아니면 byte[] 타입으로 컨버팅하여 저장해야함. → MySQL 은 에러 발생
2.  H2, MySQL Driver에서 JSON 타입을 서로 다른 포멧으로 전달함 → 매칭 해야하는 컨버터의 타입이 달라짐. H2는 byte[]와 객체간 컨버팅, MySQL은 String과 객체간 컨버팅 전략을 설정해야함.

이러한 여러 DB 엔진의 타입 변환 차이를 범용적으로 처리하기 위해서는 단순히 소스 코드 몇 줄 수정한다고 해결될 것이 아니였습니다.

그래서 현재 선택할 수 있는 최선의 방법은 3가지가 있습니다.

먼저, Spring Boot Profile 을 분리하여 테스트 코드에서는 H2 전용 컨버터를 등록하는 방법을 적용합니다. 이 컨버터는 byte[] ←→ Address 을 변환합니다.

두번째는, H2와 MySQL의 JSON 타입으로 설정한 필드를 Varchar 또는 Text 타입으로 변경합니다. Text 타입으로 지정한 컬럼에 JSON 형태의 문자열을 저장하는 것으로 위의 이슈를 해결할 수 있습니다.

세번째로는 Docker 컨테이너 위에서 테스트코드를 수행하는 방법도 있습니다. TestConatiners를 이용하면 Docker 컨테이너에서 MySQL DB서버를 띄우고 그 위에서 테스트 코드가 수행됩니다. 이 방법은 운영환경과 테스트 환경을 일치시킬 수 있다는 점에서 장점이 되지만, 테스트 코드를 한번 수행하는 데 있어서 시간이 오래걸린다는 단점이 있습니다.

저의 경우는 두번째 방법을 선택했는데, 첫번째 방법인 Profile을 분리하여 H2용의 컨버터를 생성하는 방법은 추후 관리가 복잡해질 우려가 있다고 판단했습니다. 만약 DB에 JSON으로 변환해서 관리해야 할 클래스가 많아질수록 그 클래스들에 대해 모두 별도의 테스트용 컨버터를 준비해야 합니다. 따라서 저는 H2와 MySQL 모두에게서 차이를 타협할 방법을 원했기에 Text타입으로 저장하는 두번째 방법을 선택했습니다.

Data JPA에서는 이 문제를 어떻게 해결할까요?
============================

JPA 에서는 2021년 MySQL, Oracle, PostgreSQL, H2 와 같은 DB를 사용할 때 JSON 타입 변환을 가능하게 한 [hypersistence-utils](https://github.com/vladmihalcea/hypersistence-utils) 를 Vlad Mihalcea님이 만들어 주셨습니다.

```java
@Getter
@NoArgsConstructor
@ToString
@Table(name = "MEMBER")
@Entity
public class JpaMember {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "MEMBER_ID")
    private Long id;

    private String username;

    @Type(JsonType.class) // 이 애너테이션 하나 달아주면 H2에 JSON 타입으로 저장
    @Convert(converter = AddressConverter.class)
    private Address address;

    public JpaMember(String username, Address address) {
        this.username = username;
        this.address = address;
    }
}
```

아직 이 코드를 분해해 보지는 않았지만, JSON 데이터를 INSERT 할 때 어떤 방법을 적용했는지 힌트를 얻을 수 있지 않을까 생각해서 추후 시도해 보겠습니다.

아래의 링크를 참고해주세요!

[How to map a JSON column with H2, JPA, and Hibernate](https://stackoverflow.com/questions/39620317/how-to-map-a-json-column-with-h2-jpa-and-hibernate/67467069?stw=2&source=post_page-----bedaa2ac3938--------------------------------#67467069)