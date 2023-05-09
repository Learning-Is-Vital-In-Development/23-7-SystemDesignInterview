# 5. 안정 해시 설계

수평적 규모 확장성을 달성하기 위해 요청 또는 데이터를 서버에 균등하게 나누는 것이 중요하다. 이 목적을 달성하기 위해 보편적으로 사용되는 기술이 안정 해시 설계이다.

# 해시 키 재배치(rehash) 문제

N개의 캐시 서버가 있고 균등하게 부하를 나누는 보편적인 방식은 나머지 연산을 이용하는 것이다.

이 방법은 서버 풀의 크기가 고정되어 있을 때, 그리고 데이터의 분포가 균등할 때 잘 동작한다. 하지만 서버의 풀 크기가 달라질 때 문제가 생긴다.

서버 풀의 크기가 변경되어 나머지 연산을 적용하게 되면 같은 키에 대해서 기존과 다른 서버로 분배될 수 있고 대부분 캐시가 없어 대규모 캐시 미스가 발생하게 된다.

# 안정 해시

안정 해시(consistent hash)는 이를 해결하기 위한 방안이다.

안정 해시는 해시 테이블 크기가 조절 될 때, 평균 k/n개의 키만 재배치하는 해시 기술이다. 여기서 k는 키의 개수, n은 슬롯(slot)의 개수이다.

### 해시 공간과 해시 링

해시 공간(hash space)는 해시 함수에 따른 결과 범위를 의미한다. 이 해시 공간의 시작과 끝점을 닿게 하면 해시 링(hash ring)이 된다.

### 해시 서버

해시 함수를 사용해 서버 IP나 이름을 해시 링 위 한 점에 대응시킬 수 있다.

### 해시 키

캐시할 키 역시 동일하게 해시 링 위에 어느 지점에 해시 함수를 사용해 배치할 수 있다.

### 서버 조회

키가 저장될 서버는 키의 위치로부터 시계 방향(해시공간 양의 방향)으로 링을 탐색하여 가장 처음 만나는 서버다.

### 서버 추가

새로운 서버가 추가되면 서버의 위치가 할당되고 새로운 서버의 이전 서버로부터 새로운 서버 사이의 키들만 재 배치된다.

### 서버 제거

기존 서버가 제거되면, 제거된 서버의 이전 서버로부터 제거된 서버 사이의 키들 제거된 서버의 다음 서버로 재 배치된다.

### 기본 구현법의 두 가지 문제

기본 구현법을 정리하면 아래와 같다.

- 서버와 키를 균등 분포(uniform distribution) 해시 함수를 사용해 해시 링에 배치한다.
- 키의 위치에서 링을 시계 방향으로 탐색하다 만나는 최초의 서버가 키가 저장될 서버다.

두 가지 문제점

- 서버가 추가되거나 삭제되는 상황을 감안하면 파티션(partition)의 크기가 균등하게 유지하는 것을 불가능하다.
- 키의 균등 분포를 달성하히가 어렵다는 것이다.

이 문제를 해결하기 위해 제안된 기법이 가상 노드(virtual node) 또는 복제(replica)라고 불리는 기법이다.

## 가상 노드

가상 노드(virtual node)는 실제 노드 또는 서버를 가리키는 노드로서 하나의 서버는 링 위에 여러 개의 가상 노드를 가질 수 있다. 가상 노드의 수를 증가 시킬 수록 키의 분포는 점점 균등해진다. 표준 편차(stadard deviation)가 작아져서 데이터가 고르게 분포되기 때문이다.