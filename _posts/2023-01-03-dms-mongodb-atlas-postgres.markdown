---
layout: post
title:  "DMS를 이용한 mongoDB Atlas → PostgreSQL 마이그레이션"
date:   2023-01-03 00:55:00 +0900
categories: AWS
---

AWS 서비스 중에 DMS라는게 있다. Database Migration Service의 약자로, 동종 혹은 이종간의 데이터베이스 복제 및 마이그레이션을 가능하게 해준다.

예를 들어, MySQL → PostgreSQL과 같이 RDBMS간의 복제는 물론이고, mongoDB → Redshift의 경우와 같이 소스 및 대상의 스키마가 각각 비정형/정형 형태로 다르다고 해도 마이그레이션이 가능하다. 

그리고 지속적 복제, 즉 CDC(Change data capture)와 같은 작업도 가능한데, 대상 데이터베이스에 Kafka, Kinesis Data Streams 등이 있는 것으로 보아 변경 데이터를 스트리밍하는 것도 가능해보인다.

실제 활용 사례는 mongoDB Atlas → PostgreSQL로의 마이그레이션을 진행했는데, PostgreSQL은 데이터 웨어하우스의 용도로 선정했고, Redshift의 기반 엔진 자체가 PostgreSQL을 사용하므로 선정했다. 지금은 데이터 규모가 크지 않으므로 Redshift가 필요한 정도는 아니다.

## 아키텍처

다음은 간단한 아키텍처다

```
  source (mongodb)             data migration service
 ┌───────────────┐          ┌─────────────────────────┐
 │  ┌─────────┐  │   (2)    │                         │           
 │  │replica A│  │ source   │                         │   (2)
 │  └─────────┘  │ endpoint │                         │ destination     target 
 │  ┌─────────┐  │          │  ┌───────────────────┐  │ endpoint    ┌──────────────┐
 │  │replica B│  ◄─────────────┤     task (4)      ├──────────────► │ postgreSQL   │  
 │  └─────────┘  ├──────────►  └───────────────────┘  │ migrate     └──────────────┘
 │  ┌─────────┐  │ load     │     transform schema    │   (5)         
 │  │replica C│  │ & CDC    │                         │
 │  └─────────┘  │   (5)    │                         │
 └───────────────┘          └─────────────────────────┘
         (1)                        (3) instance
```

1. MongoDB Atlas에는 아주 작은 티어를 사용해도 레플리카셋을 기본적으로 세개를 지원한다.
2. 소스 및 대상이 될 엔드포인트를 만든다. 데이터베이스 엔진과 connection string만 잘 적어주면 끝나는 일이다.
3. 마이그레이션 작업을 실행할 복제 인스턴스를 만들어야 한다. VPC 및 서브넷 그룹에 속하며, 퍼블릭/프라이빗 IP이 할당되는 등 기본적인 개념은 EC2와 크게 다르지 않다. 작업을 실행할 컴퓨터일 뿐이다. 다만 차이점이라면 RDS 인스턴스처럼 Multi-AZ를 설정할 수 있다. 클래스에 `dms.t3.micro` 와 같이 `dms`라는 Prefix가 붙는다.
4. 태스크는 마이그레이션 작업을 정의하는 단위로, 여기서 실질적으로 소스와 대상을 설정하고, 어떤 형태로 로드하고, 변환 및 마이그레이션을 할지 결정한다.
5. 태스크를 실행하면, 소스 엔드포인트로부터 데이터를 로드하고, 설정에 따라 CDC 작업을 진행하며, 대상 엔드포인트로 마이그레이션한다.

오해하지 말아야 할 것들이 있다.

- 변환 과정은 스키마 변환일 뿐이다. 이종 데이터베이스 간의 데이터 타입 정의가 다른 경우에 유용하다. **데이터 값 그 자체를 변환할 수는 없다.**

컨셉은 매우 간단하므로, 크게 어려울 것은 없지만 몇가지 트러블슈팅을 기록하고자 한다.

## mongoDB Atlas 엔드포인트 설정

mongoDB Atlas의 경우, 일반적인 connection string을 이용하는 것이 아니라, 직접 레플리카셋에 접속해서 접근해야 한다.

DMS는 공식적으로 Atlas를 지원하는 것은 아니다. Atlas의 connection string을 살펴보면 `mongodb+srv` 라는 프로토콜이 사용된다. 이는 엔드포인트로 접속할 때, 레플리카셋 중 하나를 특정해서 접속할 수 있게 한다. [(1)](https://www.mongodb.com/developer/products/mongodb/srv-connection-strings/)

```
mongodb+srv://USERNAME:PASSWORD@cluster0.oford.mongodb.net/test
```

여기서 SRV는 본질적으로 각 레플리카의 주소를 seed list로 기록해놓은 DNS 레코드이다.

오래된 mongoDB 드라이버를 사용하면, `mongodb+srv` 프로토콜을 사용할 수 없는데, 이 때는 `mongodb` 프로토콜을 이용하고, 각 레플리카를 가리키는 주소를 입력하는 방식으로 접속할 수 있다. 

```
mongodb://USERNAME:PASSWORD@cluster0-shard-00-00.oford.mongodb.net,cluster0-shard-01-00.oford.mongodb.net,cluster0-shard-02-00.oford.mongodb.net/test
```

DMS에서는, 레플리카 주소를 명시적으로 적어줘야 엔드포인트를 통해 접속이 가능하다. 다음 엔드포인트를 참고하면 콘솔에서 어떻게 설정해야 하는지 힌트를 얻을 수 있다.

```json
{
    "Username": "cdc",
    "ServerName": "cluster0-shard-00-00.oford.mongodb.net",
    "Port": 27017,
    "DatabaseName": "my-service",
    "AuthType": "password",
    "AuthMechanism": "scram-sha-1",
    "NestingLevel": "none",
    "ExtractDocId": "false",
    "AuthSource": "admin"
}
```

- ServerName은 레플리카 주소를 입력한다.
- Atlas는 인증 방식은 SCRAM-SHA-1을 이용한다.
- NestingLevel은 알아둘 필요가 있는 옵션이다. [(2)](https://docs.aws.amazon.com/ko_kr/dms/latest/APIReference/API_MongoDbSettings.html)
    - 문서모드 `none` 또는 테이블 모드 `one`로 지정할 수 있는데, 테이블 모드 `one` 로 설정해놓으면, JSON 문서의 가장 상위 키-값을 행과 열로 평면화할 수 있다. [(3)](https://docs.aws.amazon.com/ko_kr/dms/latest/userguide/CHAP_Source.MongoDB.html)

### 문서 모드

| oid_id | _doc |
| --- | --- |
| 5a94815f40bd44d1b02bdfe0 | { "a" : 1, "b" : 2, "c" : 3 } |
| 5a94815f40bd44d1b02bdfe1 | { "a" : 4, "b" : 5, "c" : 6 } |

### 테이블 모드

| oid_id | a | b | c |
| --- | --- | --- | --- |
| 5a94815f40bd44d1b02bdfe0 | 1 | 2 | 3 |
| 5a94815f40bd44d1b02bdfe1 | 4 | 5 | 6 |

## 인스턴스 생성과 연결 테스트

엔드포인트 연결 테스트는, 인스턴스 생성을 반드시 필요로 한다. 원리를 생각해보면 당연한 일이다. 이 서비스는 결국, 인스턴스의 작동 시간에 따라 과금된다.

## MySQL로부터의 마이그레이션

MySQL은 특별한 설정을 해주지 않으면 지속적 복제가 불가능하다. MySQL에 바이너리 로깅 옵션을 활성화해야 한다. MySQL 호환 데이터베이스는, 가능한 빨리 바이너리 로그를 삭제하게끔 작동한다고 한다. 따라서 로그 보존 기간을 늘리는 옵션을 해야 한다. MySQL의 경우 직접적인 설정 권한이 없어서, 실제로 적용하지 못했다. 

## 사용 소감

- DMS는 컨셉과 사용이 매우 간단한 서비스이다. connection string 문제만 해결되면 테이블 지정 등만 해주고, 나머지는 기본값으로 태스크를 만들어도 잘 작동하는 서비스다.
- 스키마의 이름을 바꾸는 등의 변환 작업을 할 수 있는데, 아직 필요성을 못느꼈다.
- 처음엔 일반적인 ETL 과정처럼 값을 변환해서 마이그레이션 하는 방법을 생각했는데, 그런 용도는 아니다. 그런 작업을 하고 싶다면 AWS Glue를 사용해야 한다.
- CDC 작업이 아주 쾌적하게 작동된다. 인스턴스를 굳이 만들지 않고 CDC 이벤트만 받고 싶은 요구가 불쑥불쑥 생긴다. Glue만을 이용해 CDC 작업을 하기엔 어려움이 있어보인다. 다만 중간에 값싼 S3 데이터 레이크를 두고, Glue가 작업을 하게 만들 수는 있다. [(4)](https://aws.amazon.com/ko/blogs/big-data/implement-a-cdc-based-upsert-in-a-data-lake-using-apache-iceberg-and-aws-glue/) 물론 여전히 복제 인스턴스가 필요하다. 데이터 변경 이벤트가 드문 경우 배치 작업을 하는 편이 경제적으로 낫다.
