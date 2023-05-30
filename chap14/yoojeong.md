## 14장: 유투브 설계

### 문제 이해 및 설계 범위 확정

**요구사항**

```
- 빠른 비디오 업로드
- 원활한 비디오 재생
- 재생 품질 선택 기능
- 낮은 인프라 비용
- 높은 가용성과 규모 확장성 및 안정성
- 지원 클라이언트: 모바일 앱, 웹 브라우저, 스마트 TV
```

**개략적 규모 추정**

- DAU 5 백만
- 한 사용자는 하루에 평균 5 개 비디오 시청
- 10%의 사용자가 하루에 1 비디오 업로드, 비디오 평균 크기는 300MB 
  - 일 신규 저장 용량=150TB (5백만 * 10% * 300MB)
- CDN 비용 = $150,000 (5백만 * 5비디오 * 0.3B * $0.02)


### 개략적인 설계

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/638117ce-042e-455c-b041-b11d4f227a41)

- `Client` : 컴퓨터, 휴대폰, 스마트 티비 등
- `CDN` : 영상은 CDN에 저장되고, 재생버튼을 누를 때 CDN에서 영상이 스트림된다
- `API servers` : 추천, 영상 URL 생성, 유저 등 영상 재생을 제외한 모든 것들은 API 서버에서 처리

**비디오 업로드 절차**

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/3c676263-c062-4660-862d-c16731cccbe1)


- `API servers`: 영상 스트리밍을 제외한 모든 요청을 처리
- `Metadata DB` : 영상 metadata를 저장.
- `Metadata cache`: 성능을 위해 영상 메타데이터와 유저 object 캐시
- `원본 저장소` : 원본 비디오를 보관할 BLOB(binary large object) storage
- `Transcoding servers:` : 영상 포맷 변환(인코딩), 단말이나 대역폭 요구사항에 맞는 최적의 비디오 스트림 제공하기 위해 필요
 - 비디오 업로드와 메타데이터 갱신은 병렬적으로 이루어짐
- `Transcoded storage` : 트랜스코딩이 완료된 비디오를 저장하는 BLOB storage
- `Completion queue` : 트랜스코딩 완료 이벤트를 보관할 메시지큐
- `Completion handler` : 트랜스코딩 완료 큐에서 이벤트 데이터를 꺼내어 메타데이터 캐시와 데이터베이스를 갱신할 작업 서버들


**비디오 스트리밍 절차**

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/6e196e26-16fe-4bc4-9138-6c251ddcf163)

- 프로토콜마다 지원하는 비디오 인코딩이 다르고 플레이어도 다르므로 서비스 용례에 맞는 프로토콜을 잘 골라 사용
- 비디오 스트리밍을 위해 사용하는 스트리밍 프로토콜 종류
    - MPEG–DASH. MPEG stands for “Moving Picture Experts Group” and DASH stands for "Dynamic Adaptive Streaming over HTTP".
    - Apple HLS. HLS stands for “HTTP Live Streaming”.
    - Microsoft Smooth Streaming.
    - Adobe HTTP Dynamic Streaming (HDS).

### 상세 설계

**비디오 트랜스코딩**

- 업로드된 비디오가 다른 단말에서 순조롭게 재생되려면 다른 단말과 호환되는 비트레이트와 포맷으로 저장되어야 함
  - 비트레이트가 높을 수록(고화질) 높은 성능, 빠른 네트워크가 필요함
- 네트워크 대역폭이 충분하지 않으면 저화질로 보내야 끊김없이 재생 가능
  - 모바일기기는 네트워크 상태가 수시로 달라지므로 비디오 화질이 자동/수동으로 변경 가능해야함
- 가공되지 않은 원본 비디오는 저장 공간을 많이 차지함
- 영상 인코딩 포맷
  - Container: 영상 파일, 음성파일, Metadata 를 보관 (.avi .mp4 등)
  - Codecs: 영상 퀄리티를 유지하면서 영상 사이즈를 줄일수 있게 하는 압축 알고리즘 (HEVC, VP9, H.264 등)


**Directed acyclic graph (DAG) 모델**

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/67dd4b03-a736-4285-9a4c-ebe3f7d4f910)


- 영상 제작자에 따라 여러 요구사항이 있을 수 있으므로 (워터마크, 섬네일 이미지, 등등) 유저가 직접 프로세스를 정의할 수 있는 파이프라인을 제공


### 비디오 트랜스코딩 아키텍처**

**전처리기**

- 비디오 분활
  - 비디오 스트림을 GOP(Group of Pictures) 단위로 분할 (GOP = 몇초 정도 되는 영상 조각)
  - 오래된 단말이나 브라우저는 GOP 단위의 비디오 분활을 지원하지 않으므로 전처리기가 비디오 분활을 대신한다
- DAG 생성
  - 클라이언트가 정의한 Config 파일에 따라 DAG 생성
  - ![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/ed7033af-daab-4adb-8483-c484fe4b14e6)
- 데이터 캐시
  - 안정성을 위해 GOP와 metadata를 임시 저장
  - 인코딩 실패 시 캐시를 활용해 인코딩을 재개

**DAG 스케줄러**

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/64f24549-7f57-4bd7-846f-d0d12e907234)

- DAG 그래프를 단계별로 분할해 각 자원 관리자의 작업 큐에 집어넣는다


**자원 관리자**

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/715c2081-1be2-449f-a82f-457d7645b40f)

1. Task Scheduler 가 가장 우선순위가 높은 일 가져옴
2. Task Scheduler  가장 효율적으로 가져온 일을 끝낼 수 있는 Worker를 가져옴
3. Worker 에게 일을 시킴
4. Task/Worker 정보를 Running Queue 넣음
5. 일이 끝나면 Running Queue 에서 정보 삭제

**작업 서버**

- DAG에 정의된 작업을 수행
- 작업 종류에 따라 서버 구분

**임시 저장소**

- Metadata 등 참조가 빈번하고 크기가 작은 데이터는 메모리에 캐싱
- 비디오/오디오 등은 BLOB 에 캐시
- 임시 저장소의 데이터는 프로세싱 완료 후 삭제


### 시스템 최적화

**속도 최적화: video uploading 병렬 처리**

 ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/99a6577f-3e40-4401-9c18-7cc082e5364c/Untitled.png)

- 하나의 큰 영상파일을 업로드 하는것은 비효율적이므로 영상을 GOP로 나눠서 업로드
- 업로드가 실패할 경우 빠르게 재실행 가능
- 클라이언트를 통해 구현할 수 있음


**속도 최적화: 업로드 센터를 사용자 근거리에 지정**

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/01502c16-041d-4243-8eb5-34e912b72871)


**속도 최적화: 병렬처리**

느슨한 결합을 통한 병렬성 증가


| AS-IS | TO-BE|
|:--:|:--:|
|![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/72d60d0a-67fc-40a7-b0e3-40be692008ad)|![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/796350d9-5e1c-4600-8c29-4ae23687df1d)|
|전 단계의 결과물이 다음 단계에 영향을 주어 병렬처리가 어려움 | 메세지큐 도입을 통한 이벤트 모듈 병렬 처리|



**안전성 최적화: pre-signed URL**

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/8f50e17f-f3a5-4625-afcb-0d769d9b7aa1)

- 클라우드 서비스에 따라 이름 상이 (Azure: Shared Access Signature)
- 오직 승인된 유저만 알맞은 위치에 영상을 업로드할 수 있도록 하기 위한 장치

    
**비용 최적화**

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/95a15082-1190-4896-a142-31eebccfc7b8)


- 유튜브 시청에 관한 리서치에 따르면 인기있는 영상은 많이 보지만 나머지 영상들은 거의 조회되지 않는다 
- 인기있는 영상들만 CDN에서 스트리밍, 다른 영상은 비디오 서버에서 제공하는 식의 최적화 가능2. 인기가 없는 영상의 경우 다른 포맷으로 인코딩된 버전을 저장하고 있을 필요가 없다
- 인기가 없거나 짧은 영상의 경우 On-demand 방식으로 인코딩
- 특정 지역에서만 인기가 많은 비디오는 다른 지역 CDN으로 업로드 해놓을 필요가 없음
- CDN 직접 제작 (Netflix)

> 근데 인코딩은 미리 하는데 인기 없을지 어케 알죠..?

**에러 처리**
- 회복 가능 오류
    - ex. 특정 비디오 세그먼트를 트랜스코딩 하다 실패하는 등
    - retry 가 어렵다고 판단되면 클라이언트에게 적절한 오류 코드를 반환
- 회복 불가능 오류
    - ex. 비디오 포맷이 잘못됨
    - 해당 비디오에 대한 작업을 중단하고 클라이언트에게 오류 코드를 반환

### 오류에 대한 전형적 해결 방법

- 업로드 오류: rerty
- 비디오 분활 오류: 낡은 버전의 클라이언트가 GOP 경계에 따라 비디오를 분할하지 못한 경우라면 전체 비디오를 서버로 전송하고 서버가 해당 비디오 분활을 처리하도록 한다.
- 트랜스코딩 오류: rerty
- 전처리 오류: DAG 그래프를 재생성
- DAG 스케줄러 오류: 작업을 다시 스케줄링
- 자원 관리자 큐에 장애 발생: 캐시를 이용
- 작업 서버 장애: 다른 서버에서 해당 작업을 rerty
- API 서버 장애: API 서버는 stateless 이므로 신규 요청은 다른 API 서버로 우회될 것이다
- Metadata Cache 서버 장애: 데이터는 다중화되어 있으므로 다른 노드에서 데이터를 여전히 가져올수 있을것, 장애가 난 캐시 서버는 새로운 것으로 교체
- Metadata DB 서버 장애
    - 주 서버가 죽었다면 부 서버가운데 하나를 주 서버로 교체
    - 부 서버가 죽었다면 다른 부 서버를 통해 읽기 연산을 처리하고 죽은 서버는 새것으로 교체

## 마무리

- 영상 검열
  - 불법적인 영상물을 삭제하는 것
  - 업로드 과정에서 찾거나 유저의 피드백을 통해서 찾을 수 있다.
- 실시간 스트리밍
  - 현재 우리가 설계한 것과 많은 것이 비슷하나(업로드, 인코딩, 스트리밍) 몇 가지 다름
  - Latency 요구사항이 더 높다: 다른 streaming protocol을 사용해야 할 수도 있다
  - 병렬처리의 중요성이 낮다: 이미 작은 크기의 영상 데이터가 처리되고 있기 때문
  - 다른 에러 처리가 필요하다: 시간이 오래 걸리는 에러 처리방식은 사용할 수 없다
