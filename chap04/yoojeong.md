## 4장: 처리율 제한 장치의 설계

![image](https://user-images.githubusercontent.com/75432228/233833141-95873b47-6321-4f2b-b5ef-7f9d7951f96f.png)


처리율 제한 장치 (rate limiter) : 설정된 threshold 까지만 클라이언트의 요청을 허용하는 장치

> [Throttling, Debouncing](https://jins-dev.tistory.com/entry/%EC%9B%B9%ED%94%84%EB%A1%A0%ED%8A%B8%EC%97%94%EB%93%9C%EC%97%90%EC%84%9C-%EC%93%B0%EB%A1%9C%ED%8B%80%EB%A7%81Throttling%EA%B3%BC-%EB%94%94%EB%B0%94%EC%9A%B4%EC%8B%B1Debouncing%EC%9D%98-%EA%B0%9C%EB%85%90)

- Dos 공격에 의한 자원 고갈 방지
  - 트위터 : 3시간에 트윗 300개 제한
  - 구글 독스 : read per minute 300회 제한
- 비용 절감
  - 모든 요청을 수용할 필요가 없어 서버 수를 줄일 수 있음
  - 우선순위가 높은 API에 더 많은 자원 할당 가능
  - 유료 외부 API 를 사용 중일 경우 사용료 감축 가능
- 서비스 가용성 확보
  - bot 에서 오는 트래픽 등을 걸러내 과부하 방지
  - 레인보우 테이블 등 brute force attack 으로부터 방어
- Rate에 대해 과금을 부과하는 Business Model로 활용
  - Medium, 슬랙, 아웃스탠딩...

> crawler

### 1. 문제 이해 밑 설계 범위 확정

| 질문을 통해 확인한 조건                                                                                                                                           | 요구사항                                                                                                                                                                                                                                             |
| :---------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| - 클라이언트측 제한 / 서버측 제한<br>- 제한 기준 (IP Address, userId, API Key....)<br>- 시스템 환경 구성<br>- 독립 서비스 / 기존서비스에 추가<br>- 제제 사실 알림 | - threshold 초과 요청 제한<br>- 낮은 응답시간<br>- 메모리 사용량 최소화<br>- 분산형 처리율 제한(분산 환경에서 각 서버와 프로세스에서 공유되어야함)<br>- 예외 처리(사용자가 요청 제한 사실을 알아야함)<br>- 제한장치 장애가 시스템에 영향을 주면 안됨 |

> 쓰로틀링에서 에러는 최대한 빨리 내려주는게 좋다..! 백화현상이 있던 적이 있는데 요청 거절되어도 timeout 까지 대기하느라 장애가 그대로 노출돼버림

> Rate Limit 에러는 RFC6585 기준 `492 too many request` 사용이 권장되고, 일반적인 GW 사용 시 이 에러를 리턴함

### 2. 개략적인 설계안 제시 및 동의 구하기

클라이언트 요청은 위변조가 쉽고, 모든 클라를 통제하기도 어렵기 때문에 서버 혹은 게이트웨이등 미들웨어에 제한장치를 둔다.

서버와 GW 중 어디에 둘지는 상황마다 다르지만, 보통 아래 지침을 참고한다.

- 언어, 캐시 등 사용 중인 스택이 서버에 제한장치를 구현해도 되는 효율이 되는지 확인
- 처리율 제한 알고리즘을 선택할 때, GW를 쓰면 알고리즘 선택이 제한될 수도 있다
- 인증이나 IP 화이트리스트 등이 GW에 이미 구현돼있다면 GW에 하는게 좋을 수 있다
- 직접 구현은 리소스가 꽤 들기 때문에 상용 GW를 쓰는게 나을 수도 있다

#### 처리율 제한 알고리즘

|      알고리즘      | 구현                                                                                                                                                                           | 장점                                                                                           | 단점                                                                                                                      |
| :----------------: | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------ |
|    Token Bucket    | bucket은 최대 n 개의 토큰을 채울 수 있고, 일정 시간마다 refill rate 만큼 공급됨<br> 요청이 들어오면 토큰과 함께 보내고 토큰이 없으면 요청은 거절된다                           | 구현이 쉽고 메모리 사용율이 효율적.<br>Burst of request 처리 가능                              | 버킷 크기와 토큰 공급률을 적절하게 튜닝하는 것이 중요                                                                     |
|    Leak Bucket     | 요청 처리율이 고정된 토큰 버킷알고리즘으로, FIFO 큐로 구현됨<br> 시간당 n 개의 요청을 처리하는 큐를 생성해 큐가 비어있으면 요청을 추가하고, 가득찬 경우 요청은 버린다          | 큐의 크기가 고정되어 메모리 사용이 효율적 <br>일정하게 요청을 처리하기때문에 출력이 안정적이다 | 트래픽이 몰리거나 요청량에 비해 처리속도가 늦으면 새로운 요청은 버려지는 starve 현상 발생                                 |
|    Fixed Window    | 버킷에 정해진 시간 단위로 window 가 만들어지고, window 에는 요청 건수가 기록된다. 요청 건수가 threshold 보다 크면 요청은 거부된다                                              | 이해하기 쉽고 메모리 효율이 좋다<br> 새 요청의 starvation 이 발생하지 않음                     | 기간 경계의 트래픽 편향 문제: 경계 부근에서 트래픽이 몰리면 한도보다 많은 요청이 처리될 수 있음                           |
| Sliding Window Log | 타임 스탬프를 레디스의 Sorted Set에와 같은 캐시에 보관하며 새 요청이 오면 만료된 타임스탬프를 제거하고, 새 요청 타임 스탬프를 로그에 추가<br> 로그가 허용치보다 크면 요청 거부 | burst of traffic 처리 가능<br>일정한 request rate 유지 가능                                    | 거부된 요청의 타임스탬프를 보관하기 때문에 메모리 사용량이 높다                                                           |
| Sliding Window Log | Fixed Window Counter와 Sliding Window Log 알고리즘의 결합으로 이전 시간대의 평균 처리율에 따라 현재 윈도우의 상태를 계산                                                       | burst of traffic 처리 가능<br>메모리 효율이 좋다                                               | previous window 에 대해 approximation을 하기 때문에 정확한 계산을 하지는 못하나, 여전히 매우 높은 정확도라 큰 문제는 아님 |

- 처리율 제한 알고리즘의 기본 아이디어는 많은 요청이 접수되었는지를 추적할 수 있는 카운터를 추적 대상별로 두고 이 카운터의 값이 한도를 넘어서면 이후 요청은 거부하는 것
- 데이터베이스는 디스크 접근 때문에 느리니까, 메모리상에서 동작하는 캐시가 바람직하다.

#### 개략적인 아키텍쳐
![image](https://user-images.githubusercontent.com/75432228/233833995-e22f07e8-861d-46a7-8f31-a59f6f36d07f.png)

레디스는 처리율 제한 장치를 구현할 때 자주 사용되는 메모리 기반 저장소로, INCR과 EXPIRE의 두가지 명령어를 지원
- INCR: 메모리에 저장된 카운터의 값을 1만큼 증가시킨다.
- EXPIRE: 카운터에 타임아웃 값을 설정한다. 설정된 시간이 지나면 카운터는 자동으로 삭제된다.

### 3. 상세 설계

#### 처리율 제한 규칙

보통 config 파일 형태로 디스크에 저장되고, 작업 프로세스(worker)가 수시로 디스크에서 규칙을 읽어 캐시에 저장한다.

#### 처리율 한도 초과 트래픽의 처리

> HTTP 응답 헤더<br>- X-Ratelimit-Remaining: 윈도 내에 남은 처리 가능 요청 수<br>- X-Ratelimit-Limit: 매 윈도마다 클라이언트가 전송할 수 있는 요청 수<br>- X-Ratelimit-Retry-After: 한도 제한에 걸리지 않으려면 몇 초 뒤에 다시 요청을 보내야 하는지

429 반환 외에도 Queue에 담아뒀다가 나중에 처리할 수도 있다.

#### 분산 환경에서의 처리율 제한 장치 구현

- `race condition` 문제
  가장 널리 알려진 해결책은 lock 이지만, 시스템의 성능을 상당히 떨어뜨린다.
  락 대신 [Lua Script](https://stripe.com/blog/rate-limiters), [Sorted Set](https://engineering.classdojo.com/blog/2015/02/06/rolling-rate-limiter/) 이라고 불리는 레디스 자료구조를 사용하여 해결할 수 있다.
- `synchronization` 문제
  sticky session 을 활용할 수 있지만 추천하지 않음. 규모 면에서 확장 가능하지도 않고 유연하지도 않기 때문이다.
  더 나은 해결책은 레디스와 같은 중앙 집중형 데이터 저장소를 쓰는 것이다.

- 성능 최적화
  데센이 멀리 떨어진 사용자를 지원하면 latency가 증가하기 때문에 신경써야한다.
  제한 장치 간 데이터 동기화 시 최종 일관성 모델 사용
- 모니터링
  모니터링을 통해 채택한 처리율 제한 알고리즘과 정의한 처리율 제한 규칙의 효과를 확인해야한다.

### 4. 마무리

추가로 알아볼만한 정보

- 경성 처리율 제한과 연성 처리율 제한
- Application 계층 처리율 제한 외에 IPtables 를 이용한 네트워크 계층 처리율 제한
- 처리율 제한 회피
- 클라이언트측 처리율 제한 - [npm client-rate-limiter](https://www.npmjs.com/package/client-rate-limiter)


#### Ref.

[Designing an API Rate Limiter](https://goalgorithm.wordpress.com/2019/06/08/designing-an-api-rate-limiter/) Java 예제<br>
[Designing a Distributed Rate Limiter — Introduction](https://medium.com/wineofbits/designing-a-distributed-rate-limiter-introduction-731afd345a66)<br>
[Resilience4j RateLimiter](https://resilience4j.readme.io/docs/ratelimiter)<br>
[Istio rate limit(envoy)](https://istio.io/latest/docs/tasks/policy-enforcement/rate-limit/)<br>
[envoy 도 레디스 사용함](https://github.com/envoyproxy/ratelimit)<br>
[배민 처리율 제한 알고리즘](https://www.youtube.com/watch?v=uWcn7omddxs&t=276s&ab_channel=%EC%9A%B0%EC%95%84%ED%95%9C%ED%85%8C%ED%81%AC)


#### Appendix.

> Rate Limiter와 Circuit Breaker의 차이

||Rate Limiter| Circuit Breaker|
|:--:| :--:| :--:|
|보호 대상| Server의 과부하|Client를 위한 응답 속도|
|문제 원인 | Client의 과도한 요청|Server의 부분적 장애|
| 의의 | 이용자 폭증 대응<br> 특정 이용자의 악의적 이용 또는 자원 독점 차단<br> Dos 공격 방지 등|장애 서비스에 접속을 차단함으로써 에러 전파 방지 및 즉각적인 대체 응답 수행|
| 의의 | 이용자 폭증 대응<br> 특정 이용자의 악의적 이용 또는 자원 독점 차단<br> Dos 공격 방지 등|장애 서비스에 접속을 차단함으로써 에러 전파 방지 및 즉각적인 대체 응답 수행|
| |![image](https://user-images.githubusercontent.com/75432228/233834918-6e012c88-0778-4063-9c31-37ef27b3b195.png) | ![image](https://user-images.githubusercontent.com/75432228/233834913-95922ce2-295e-456f-943e-e96dbd0df31b.png)|


