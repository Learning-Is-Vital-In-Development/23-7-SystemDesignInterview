## 5장: 안정 해시 설계

분산 환경에서 요청 또는 데이터를 균등하게 분배해야하고, 안정 해시(Consistent Hashing) 는 이를 위한 기술
<br>1장 DB 레플리카, 샤딩에서 나온 재샤딩 문제의 해결법으로도 언급됨!

### 해시 키 재배치(rehash) 문제

#### mod 방식 key 분배

```python
serverIndex = hash(key) % N
```

<img src = "https://user-images.githubusercontent.com/75432228/233836637-a26f5c64-c8d9-476e-9c47-06b61aa61837.png" width="70%">


서버의 갯수가 고정되어 있는 경우 문제가 없지만, 서버가 추가 되거나 삭제 될 경우 문제 발생
- 특정 서버에 key 가 몰리는 등 key 가 불균등하게 분배됨
- 특정 key 에 대한 request 가 이전과 다른 서버로 전송되어 대규모 cache miss 발생
- 따라서 대부분의 해시키가 재분배되고, 이는 확장성(scalability)에 취약하다는 의미


<img src = "https://user-images.githubusercontent.com/75432228/233836646-c3983b13-b67c-4472-adcf-076af945140a.png" width="70%">


### 안정 해시

- 안정 해시는 해시 테이블의 크기가 조정될 때 평균적으로 k/n개의 키만 재배치하는 해시 기술(k = 키의 개수, n = 슬롯 개수)
- hash space 의 양 끝단을 붙여서 hash ring 을 만들고, key 값으로부터 시계방향으로 이동하면서 가장 먼저 만나게 되는 server 에 key 값을 매칭

<img src = "https://user-images.githubusercontent.com/75432228/233836842-c981a035-2371-42f4-ab34-4740739f1f9b.png" width="70%">


`Add server`

서버 추가 시 다른 Key들을 변경 할 필요 없이 k0를 보관하는 위치만 s0에서 s4로 변경
<img src = "https://user-images.githubusercontent.com/75432228/233836861-c3ebbf80-a1e0-42a3-8b4c-2c5d0e9b2109.png" width="70%">

`Remove server`

<img src = "https://user-images.githubusercontent.com/75432228/233836892-e14c1eec-0d8b-4065-a646-00d8aa8a1a6e.png" width="70%">



| 장점                                                                                                                                                                                     | 한계                                                                                                               |
| :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------- |
| - 서버가 추가되거나 제거될 때 재분배해야 하는 key 를 최소화<br>- Key 가 비교적 균일하게 분배되어 있어 scale out 에 유리<br>- 데이터를 좀 더 균등하게 분배하므로 Hotspot key problem 해결 | - 파티션(인접한 서버 사이의 해시 공간)의 크기를 균등하게 유지하는게 불가능<br>- 키의 균등 분포를 달성하기가 어려움 |

파티션 크기 불균형 - s0와 s2사이의 거리가 s0와 s3사이의 거리의 2배

<img src = "https://user-images.githubusercontent.com/75432228/233837111-60690c4b-a240-4b6a-a2ae-cbc60ed95a97.png" width="50%">

키 균등 분포 실패 - 대부분의 Key들은 s2에 보관

<img src = "https://user-images.githubusercontent.com/75432228/233837134-42194788-e2b0-4949-830a-f3d7075f954c.png" width="50%">



### 가상 노드

가상 노드는 실제 노드 또는 서버를 가리키는 노드로서, 하나의 서버는 링 위에 여러 개의 가상 노드를 가질 수 있다.(기본 구현에는 링 위에 서버는 1개의 노드만 가지고 있음)

<img src = "https://user-images.githubusercontent.com/75432228/233837241-c62f9ad0-1245-4657-b81a-eeaa59b1fd10.png" width="70%">

가상노드 개수가 늘어날수록 표준편차가 작아져 데이터가 더 균등하게 분포되지만, 가상노드를 저장할 공간은 더 많이 필요해지기 때문에 적절한 trade-off가 필요하다.


#### Ref.

[Redis 의 안정 해시](https://www.youtube.com/watch?v=mPB2CZiAkKM&t=3580s&ab_channel=%EC%9A%B0%EC%95%84%ED%95%9C%ED%85%8C%ED%81%AC)<br>
[Consistent Hashing](https://liuzhenglaichn.gitbook.io/system-design/advanced/consistent-hashing)

#### Appendix.

> 해시 함수 종류

<img src = "https://user-images.githubusercontent.com/75432228/233836699-20def45a-c6b6-48f7-b4c9-da28eb20c91c.png" width="70%">

[AWS DynamoDB 파티셔닝](https://hello-world.kr/19)
[Dynamo: amazon's highly available key-value store](https://dl.acm.org/doi/10.1145/1294261.1294281)
[아파치 카산드라 클러스터에서의 데이터 파티셔닝](https://woooongs.tistory.com/89)
[디스코드 채팅 어플리케이션](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
아카마이 CDN
매그레프 네트워크 부하 분산기
