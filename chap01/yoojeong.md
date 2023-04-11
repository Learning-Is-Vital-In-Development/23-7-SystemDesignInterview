## 1장: 사용자 수에 따른 규모 확장성

### 단일서버

<font color='lightgreen'>모든 컴포넌트(웹, 앱, 데이터베이스, 캐시)가 한 대의 서버에서 실행되는 간단한 시스템</font>

사용자 요청 처리 흐름

1. DNS 질의
2. 대상 IP 주소로 HTTP 요청
3. 웹서버에서 응답 반환

> `Monolithic / MSA`

### 데이터베이스, 다중화

<font color='lightgreen'>Application Server / DB Server 분리</font>

|    분류     |                         RDBMS                          |                                             NoSQL                                              |
| :---------: | :----------------------------------------------------: | :--------------------------------------------------------------------------------------------: |
|     DB      |           MySQL, Aurora, Oracle, PostgreSQL            |                               MongoDB, Neo4j, Cassandra, HBase,                                |
| 데이터 모델 |   테이블 간 관계를 기반으로 한 정형화된 데이터 모델    |   비정형화된 데이터 모델<br> 문서(Document), 그래프(Graph), 열(Column), 키-값(Key-Value) 등    |
|   확장성    | 수직적 확장(Vertical Scaling) 방식으로 확장성이 제한적 |                     수평적 확장(Horizontal Scaling) 방식으로 확장성이 우수                     |
|  트랜잭션   |  ACID(원자성, 일관성, 격리성, 지속성) 트랜잭션을 지원  | ACID 트랜잭션보다는 덜 엄격한 BASE(Basically Available, Soft state, Eventual consistency) 모델 |
|  적용 분야  |       고정적인 스키마, 복잡한 관계를 갖는 데이터       |                      대규모 데이터, 높은 확장성, 유연한 스키마 등에 적합                       |
| 데이터 예시 |            금융권, 주문 관리, 인사 관리 등             |                     소셜 미디어, IoT 기기 데이터 수집, 로그 데이터 분석 등                     |

### 수직적 규모 확장 vs 수평적 규모 확장

<font color='lightgreen'> 더 좋은 인스턴스를 사용하거나, 인스턴스 오토스케일링 적용</font>

`스케일 업(scale up)` : 수직적 규모 확장은 서버에 고사양 자원(더 좋은 CPU, 더 많은 RAM)을 추가하는 것을 말합니다. ( = 장비 스펙업)

`스케일 아웃(scale out)` : 수평적 규모 확장은 더 많은 서버를 추가하여 성능을 개선하는 행위를 말합니다. ( = 장비 갯수 업)

> 스펙업은 단일 서비스의 앱이 사용하는 리소스가 늘었을 때(캐시로 인한 메모리 부족, cpu 과부하 등) 주로 하고 일반적, 트래픽을 못견디는 경우에는 스케일 아웃함(오토 스케일링 등)

### 로드 밸런서

<font color='lightgreen'> 오토스케일링된 서비스에 LB를 통해 프록싱/부하 분산</font>

로드밸런서 : 사용자는 LB의 주소만 알면 된다. 트래픽 분산, failover, HA

<details>
<summary>LB HealthCheck 문제</summary>

LB 에서 지정한 HealthCheck Time 보다 서버가 올라오는 시간이 느리면 Unhealthy Statement 로 판단해서 서버가 Graceful Shutdown 되는 현상이 있었음
</details>
<details>
<summary>LB 알고리즘</summary>
  
`라운드 로빈`
  
- 서버에 들어온 요청을 순서대로 돌아가며 배정하는 방식
- 클라이언트의 요청을 순서대로 분배하기 때문에 여러 대의 서버가 동일한 스펙을 갖고 있고, 서버와의 연결(세션)이 오래 지속되지 않는 경우에 활용하기 적합함.
  
`가중 라운드 로빈`
  
- 각각의 서버마다 가중치를 매기고 가중치가 높은 서버에 클라이언트 요청을 우선적으로 배분하는 방식
- 주로 서버의 트래픽 처리 능력이 상이한 경우 사용되는 부하 분산 방식이다. 예를 들어 A라는 서버가 5라는 가중치를 갖고 B라는 서버가 2라는 가중치를 갖는다면, 로드밸런서는 라운드로빈 방식으로 A 서버에 5개 B 서버에 2개의 요청을 전달함.

`IP hash`
  
- 클라이언트의 IP 주소를 특정 서버로 매핑하여 요청을 처리하는 방식
- 사용자의 IP를 Hashing해 로드를 분배하기 때문에 사용자가 항상 동일한 서버로 연결되는 것을 보장함.

`Least Connection`
  
- 요청이 들어온 시점에 가장 적은 연결상태를 보이는 서버에 우선적으로 트래픽을 배분하는 방식
- 자주 세션이 길어지거나, 서버에 분배된 트래픽들이 일정하지 않은 경우에 적합함.

`Least Response time`
  
- 서버의 현재 연결 상태와 응답시간(Response Time, 서버에 요청을 보내고 최초 응답을 받을 때까지 소요되는 시간)을 모두 고려하여 트래픽을 배분하는 방식
- 가장 적은 연결 상태와 가장 짧은 응답시간을 보이는 서버에 우선적으로 로드를 배분함.
</details>
  
> Nginx, SpringCloudGateway, Kong<br> `failover`, `rolling/Blue-Green 배포`, `graceful shutdown`

### 데이터베이스 다중화

<font color='lightgreen'>DB 도 스케일 아웃이 필요하다</font>

HA, Failover 를 위한 Source-Replica (Master-Slave, Primary-Secondary)

`Source DB` : Write 연산

`Replica DB` : Read 연산, 일반적인 어플리케이션은 읽기가 월씬 많으므로 Replica는 여러대를 둔다.

- `Performance` : 쓰기와 읽기 연산을 처리하는 DB 가 분리되어 병렬로 처리될 수 있는 쿼리의 수가 늘어나므로 성능이 좋아짐
- `안전성` : 데이터베이스를 여러 지역에 분산시켜 놓음으로써 서버가 고장나도 데이터를 보존할 수 있다.
- `HA` : 다중화를 통해 Source 서버에 장애가 발생하더라도 Replica 서버 중 하나를 선출하여 Source 서버로 승격 시켜서 장애 시간을 최소화 시킬 수 있다.
  네트워크나 서버의 상황에 따라 복제 지연이 발생할 수 있다.

> [Master-Slave 데이터 동기화 구조](https://www.percona.com/blog/how-does-mysql-replication-really-work/)<br> [Spring 에서 master-slave 구조 라우팅](https://cornswrold.tistory.com/562)

<details>
<summary>Binlog 이외의 데이터 복제 방법</summary>

- `스냅샷 복제(Snapshot Replication)` : 마스터 DB의 스냅샷을 생성하고 이를 슬레이브 DB로 복사하는 방식. DB의 크기가 크고 데이터 변경이 적은 경우에 적합하고, binlog를 사용하지 않으므로 마스터 DB에 영향을 미치지 않는다
- `MySQL Router` : MySQL 프로토콜을 사용하여 마스터 DB와 슬레이브 DB 간의 트래픽을 라우팅하는 프록시 서버. MySQL Router를 사용하면 마스터 DB와 슬레이브 DB 간의 물리적 거리를 늘리거나, 복잡한 네트워크 토폴로지를 관리하는 경우에도 쉽게 데이터를 복제 가능
- `MySQL Group Replication` : 마스터-마스터(Master-Master) 모델에서 사용할 수 있는 MySQL의 고가용성(High Availability) 솔루션. 노드 간의 복제를 자동으로 관리하며 데이터의 일관성을 보장
- `MySQL NDB Cluster` : MySQL의 분산 데이터베이스 솔루션으로 노드 간의 동기화를 위해 binlog 대신 자체적인 데이터 복제 기술을 사용

</details>

### 캐시

<font color='lightgreen'> 자주 읽히는 데이터는 캐싱해서 사용</font>

캐시 : 값 비싼 연산 결과 또는 자주 참조되는 데이터를 메모리 안에 두고, 이후 요청은 빨리 처리될 수 있도록 하는 저장소

1. 요청을 받은 웹 서버는 캐시에 응답이 저장되어 있는지 확인
2. 캐시에 있다면 해당 데이터를 클라이언트에 반환
3. 캐시에 없다면 DB 에서 데이터를 찾아 캐시에 저장한 뒤 클라이언트에 반환

캐시의 종류

- 로컬캐시 (ehcache..)
- 글로벌캐시 (Redis..)
- CDN

> Redis는 보통 서버 어플리케이션까지 와서 조회하는데, 브라우저에서 304 Not-Modified 를 보내는 경우는 서버를 조회하지 않은 경우이다...여기서 브라우저가 조회한 캐시는 뭘까?

캐시전략의 종류

- 읽기 전략
  - Look Aside
  - Read Through
  
- 쓰기 전략
  - Write Back
  - Write Through
  - Write Around

캐시 사용 시 유의할 점

- `when cache?` : 캐시는 갱신은 자주 일어나지 않지만 참조는 빈번하게 일어난다면 고려해볼만 하다.
- `cache target?` : 영속적으로 보관할 데이터를 캐시에 두는 것은 바람직하지 않다.
- `cache expire?` : 만료된 데이터는 캐시에서 삭제되어야 한다. 아니면 OOM의 원인이 됨
- `consistency?` : DB의 원본과 캐시의 일관성을 보장하기 위해 DB 변경시 evict, 적당한 TTL 등이 필요
- `failover?` : 캐시 서버를 한 대만 두는 경우 해당 서버는 단일 장애 지점(SPOF)이 되어 버릴 수 있다.
- `size?` : 캐시 메모리 크기는 너무 크지도 작지도 않게 적절하게 잡아야 한다.
- `eviction policy?` : 캐시가 꽉 차버리면 LRU, LFU, FIFO 같은 정책들을 사용해서 사용해야 한다.

> 캐시 데이터 방출 정책- Window TinyLfu<br>
![image](https://user-images.githubusercontent.com/75432228/230772217-4859aa84-548b-4077-a615-1829472c264a.png)
<br>[cache 이론 정리](https://jins-dev.tistory.com/entry/Cache-%EC%97%90-%EB%8C%80%ED%95%9C-%EC%9D%B4%EB%A1%A0%EC%A0%81%EC%9D%B8-%EC%A0%95%EB%A6%AC)

<details>
<summary>[DB 변경시 evict - Mybatis cache](https://yunamom.tistory.com/40)</summary>
  
- 수정/삭제 시 자동으로 cache 에도 반영
- mybatis는 mapper의 @select, @insert, @update, @delete 어노테이션으로 쿼리를 판단
  - select 일경우 캐시에서 읽음
  - @insert, @update, @delete 의 경우 캐시 갱신
- but mapper 에 맞는 테이블만 해당되기 때문에 한 mapper 는 하나의 테이블만 조작하도록 설계해야한다.
  (ex. 유저 테이블에서 조인을 통해 게시글 테이블을 수정해도 게시글 매퍼캐시는 수정되지 않음)
</details>

### 콘텐츠 전송 네트워크(CDN)

CDN : 정적 콘텐츠를 전송하는 데 쓰이는 지리적으로 분산된 서버의 네트워크
이미지, 비디오, CSS, JavaScript 파일 등 정적 컨텐츠 캐시

CDN 사용 시, 고려 사항

- `비용`: 제3 자에 의해 운영 → CDN 들어가고 나가는 데이터 전송 양에 따라 요금 지불
- `적절한 만료시간 설정`: 만료기간이 중요한 콘텐츠의 경우, 만료시점 설정이 중요함
- `CDN 장애 대처방안`: CDN 죽었을 때, 어떻게 동작하는가에 대한 대책 필요
- `콘텐츠 무효화`: 만료되지 않은 콘텐츠라도 무효화 할 수 있는 방안 필요
  - CDN 사업자가 제공하는 API, Object versioning

### 무상태(stateless) 웹 계층

<font color='lightgreen'> 서버는 정보를 가지지 않음! 정보는 DB에 </font>

> 각 서버 로컬에서 가진 정보는 크러스터링을 어떻게 할 것인가?
> <br>ehcache 의 분산 로컬 캐싱
> <br>sticky session 과 session clustering

### 데이터 센터

지리적 라우팅 : 사용자는 가장 가까운 데이터 센터로 안내된다.

- 데이터 센터 장애시

  데이터 센터 중 하나에 심각한 장애가 발생하면 모든 트래픽은 장애가 없는 데이터 센터로 전송된다.<br>
  데이터 센터마다 데이터베이스를 운용중인데, 장애시 자동으로 failover되어 트래픽이 다른 데이터 센터로 간다고 해도 정상 데이터를 동일하게 내려줘야한다.

- 테스트 배포

  테스트 배포 또한 여러 지역에 웹사이트 또는 어플리케이션을 여러 위치에서 테스트하는 것이 중요
  ci/cd는 모든 데이터 센터에 동일한 서비스가 설치되도록 하는데 중요한 역할

> 넷플릭스는 여러 데이터센터에 걸쳐 데이터를 동기화하는중이라고 나오는데, 2016년에 데센 종료하고 클라우드로 옮겼음..

### 메세지 큐

메세지 큐 : 메세지의 무손실, 비동기 통신을 지원하는 컴포넌트

|      |                                                                     |
| :--: | :------------------------------------------------------------------ |
| 장점 | 비동기 처리<br>높은 확장성<br>데이터 복구<br>서비스간의 느슨한 결합 |
| 단점 | 실패시 에러 처리 구현 필요<br>구현 복잡성<br>관리비용               |

> 메시지 큐(Message Queue)와 EDD(Event Driven Design)<br>
> Message Driven vs. Event Driven

### 로그, 메트릭 그리고 자동화

`로그`

- 에러로그 모니터링 → 시스템/어플리케이션의 오류 및 문제 찾기 위함
- 검색 및 조회 기능 지원해야 편하다.

`메트릭`

- 시스템의 현재 상황 파악 , 사업 현황에 관한 유용한 정보
- 호스트 단위 메트릭: CPU, 메모리, 디스크 I/O 등
- 종합 메트릭: 데이터베이스 계층 성능, 캐시 계층 성능
- 핵심 비즈니스 메트릭: 일별 능동 사용자(DAU), 수익 등

`자동화`

- CI/CD
- 빌드/테스트/배포 절차 자동화 → 개발 생산성 향상

Tools
- `Sentry`, `Datadog`, `Grafana`, `Prometheus`
- `Jenkins`, `Travis`, `Github Actions`
- `ELK`

### 데이터베이스의 규모 확장

<font color='lightgreen'> DB 또한 스케일업보다는 스케일 아웃이 좋다</font>

파티셔닝과 샤딩

![image](https://user-images.githubusercontent.com/75432228/230772164-b7c1359f-4764-421d-8a3b-e883921c0549.png)

### 데이터의 재샤딩

<font color='lightgreen'> 데이터가 너무 많아져서 하나의 샤드로는 더 이상 감당하기 어려울 때, 샤드 간 데이터 분포가 균등하지않아 어떤 샤드가 다른 샤드에 비해 빨리 찰 때 재샤딩이 필요</font>

- 유명 인사(핫스팟 키) : 특정 샤드에 쿼리가 집중되어 서버에 과부하가 걸리는 문제
- 조인과 비정규화: 샤딩을 하면, 여러 샤드에 걸친 데이터를 조인하기가 힘들어진다. 이를 해결하는 한 가지 방법은 데이터베이스를 비정규화하여 하나의 테이블에서 쿼리가 수행될 수 있도록 하는 것이다.

샤딩의 방법

- hash sharding
  ![image](https://user-images.githubusercontent.com/75432228/230772129-07179486-89a3-401f-95a9-79bdcbfe6999.png)

  - [안정해시(hash ring) 을 사용하는 방법](https://binux.tistory.com/119)

- range sharding
  ![image](https://user-images.githubusercontent.com/75432228/230772134-832fb39e-1390-4643-b95b-92361d9e1174.png)
  
[DB분산처리를 위한 sharding](https://techblog.woowahan.com/2687/)<br>
[데이터베이스 샤딩이란 무엇인가요?](https://aws.amazon.com/ko/what-is/database-sharding/)

---

마무리

![image](https://user-images.githubusercontent.com/75432228/230772110-4b569fa5-93bb-4887-b629-db22743442e4.png)
