# AWS RDS
Amazon Web Service - Releational Database Service.\
관계형 데이터베이스들을 제공하는 AWS의 서비스로, AWS 환경에 특화된 RDS는 `Aurora` 시리즈로 분류한다.

`Aurora`는 AWS에서 기존의 DBMS를 Customizing 하여 클라우드 환경에 최적화 시킨 것으로,
성능 최적화 및 확장, HA(High Availavility) 그리고 Snapshot 기능 등을 유연하게 사용할 수 있게 한 계열이다.\
위와 같은 장점들이 있기에 Serverless class의 instance도 생성이 가능하다.

`Aurora`를 지원하는 DBMS 엔진으로는 `MySQL`과 `PostgreSQL`이 있으며, Customized기 때문에 Compatible 이라는 표현을 사용한다.

| <img width="60%" src="https://github.com/134130/dev-note/assets/50487467/164e1c59-60a3-4128-bd5a-8d3bb06c71e3" /> |
|:--:| 
| *RDS > Create database > Engine options<br>Compatible이라는 표현이 붙어있다* |

## Database
RDS의 근본으로, Cluster와 Instance가 있다.

- Cluster
  - consists of one or more DB instances
  - Endpoints ([aws-docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html))
    - Cluster endpoint (writer endpoint)
      - Endpoint Aliases for Leader of Writer Instances
      - DNS에서 CNAME이라고 대충 이해하면 편하다.
    - Reader endpoint
      - Cluster endpoint와 비슷하나 Reader Instances 중 하나로 Load Balancing을 해준다.
      - 일반적으로 Database는 쓰기작업보다 읽기작업이 훨씬 많기에 Read 작업을 분리하여 사용하는 경우가 많다. ([CQRS pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs))
      - > Reader Instance가 하나도 없을 수 도 있다. 그런경우 Reader endpoint가 Writer Instance를 가리킬 수 도 있다.\
        > 즉, **Reader endpoint는 Read-only endpoint가 아니다.** Read를 할 수 있는 endpoint들 중 하나로 Load Balancing을 하는 것이다. 때문에 Reader endpoint로 접속하더라도, Writer Instance에 접속되어 Write를 할 수 있는 경우도 있다.
    - Custom endpoints
      - 다른 VPC를 통한 접근들을 위해 custom endpoint를 사용할 수 있다.
- Instance
  - Cluster 구성요소. Cluster 설정을 통해 auto scailing 되며, 실제 컴퓨팅 리소스 단위이다.
  - Instance에는 하나의 Endpoint가 부여된다.
  - 역할에 따라 Writer Instance와 Reader Instance가 있으며, 각각에 최적화된 컴퓨팅 리소스가 부여된다.
  - Reader Instance에는 `SELECT` 구문 (Query)만 실행 가능하다
  - 일반적으로 직접 접속할 일은 없으며, Cluster 장애 발생 시 세부적인 진단 및 관리를 위해 직접 접속하는 경우도 있다.
  - > Reader Instance는 여러개를 두는 경우가 많으며, 데이터 조회 시 발생하는 Read Lock이라는 상태가 다른 프로세스들의 Read를 막기 때문에 그렇다.

## Proxy
Cluster는 실제 Instance로 redirecting 해주는 역할이지만, Proxy는 요청을 대신 처리한다.
| <img width="60%" src="https://github.com/134130/dev-note/assets/50487467/d9b89a8d-18b7-4b17-a9b5-1bb96126e137" /> |
|:--:|
| *Proxy Flow* |

\
Proxy는 Target을 지정하여 생성되며, Target은 Cluster 혹은 Instance다. (그림상의 파란 육각기둥이 Target이 될 수 있다)
위와같은 구조기 때문에, Cluster는 DBMS와 계정정보를 공유하지 않는다.

| <img width="60%" alt="image" src="https://github.com/134130/dev-note/assets/50487467/aadd30f2-9569-4b53-9714-eb8e70ae960b"> |
|:--:|
| *RDS > Proxies > Create Proxy* |

위 사진과 같이 Secrets Manager를 통해 Proxy를 사용할 때 사용할 계정정보를 별도로 관리한다. (RDS 및 DBMS가 관리하지 않는다)

경우에 따라 IAM Authentication을 사용할 수 도 있다. \
이 경우 사용자 IAM Role을 통한 인증이 이뤄지게 되며, AWS SDK를 사용하지 않는 MySQL Client에서는 plugin-authentication 방식 중 하나인 clear-text-password를 이용한다. ([iam-dbauth-cli](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.Connecting.AWSCLI.html))

```bash
# Generate IAM Authentication Token
aws rds generate-db-auth-token \
   --hostname rdsmysql.123456789012.us-west-2.rds.amazonaws.com \
   --port 3306 \
   --region us-west-2 \
   --username jane_doe

# Connect to RDS Proxy with Authentication Token
mysql --host=hostName --port=portNumber \
      --ssl-ca=full_path_to_ssl_certificate --enable-cleartext-plugin \
      --user=userName --password=authToken
```

\
Proxy는 기본적으로 Read/write Endpoint를 가지고 있으며, Custom Endpoints를 추가할 수 있다.
Custom Endpoint에는 Read/write 혹은 Read-only 중 하나의 role을 설정할 수 있으며, vpc를 별도로 지정하여 사용할 수 있다.
Read-only role이 부여된 Proxy Endpoint의 Target에 Reader Instance가 없다면 오류를 내며 접속할 수 없게된다.

| <img width="60%" src="https://github.com/134130/dev-note/assets/50487467/04daec9c-1fba-4906-a780-a3e4fe063fac" /> |
|:--:|
| *Read-only proxy endpoint에 접속했으나,<br>Target인 Cluster에 Reader Instance가 없는 경우 발생하는 에러* |






