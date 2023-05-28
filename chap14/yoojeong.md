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

** 비디오 업로드 절차 **

![image](https://github.com/rachel5004/23-7-SystemDesignInterview/assets/75432228/0395f767-9059-4790-a4b0-b6f75a96d803)
