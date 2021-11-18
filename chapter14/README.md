### 1단계 : 문제 이해 및 설계 범위

- **문제 정의**
    - 핵심 기능 : 비디오를 올리는 기능, 시청하는 기능
    - 클라이언트 : 모바일 앱, 브라우저, 스마트 TV
    - 일간 능동 사용자수 : 5백만, 10% 사용자가 하루 1개의 비디오를 업로드, 하루 평균 5개의 비디오를 시청
    - 다국어 지원
    - 현존하는 비디오 종류와 해상도를 모두 지원
    - 암호화
    - 비디오 크기 : 최대 1GB, 평균 300MB
    - 클라우드 서비스 활용 가능
- **개략적 비용**
    - AWS의 CloudFront 를 CDN 솔루션으로 사용할 경우, 100% 트래픽이 미국에서 발생한다고 가정하면 1GB당 $0.02
    - CDN 비용 : 5백만 x 5비디오 x 0.3GB x $0.02 = $150,000

### 2단계 : 개략적 설계안 제시 및 동의 구하기

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d14d0c02-12ac-4430-ad89-b53b1c81599f/Untitled.png)

**비디오 업로드 절차**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6ba5a5ae-e77d-4f46-8455-3b3315b9c594/Untitled.png)

**비디오 스트리밍 절차**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/21726f2d-2c05-4426-ac37-8e864646d953/Untitled.png)

---

### 3단계 : 상세 설계

- **유향 비순환 그래프(DAG) 모델**
    
    콘텐츠 창작자의 요구사항을 반영하기 위해, 각기 다른 유형의 비디오 프로세싱이 필요함
    
    적절한 추상화를 통해 클라이언트로 하여금 실행할 작업을 손수 정의할 수 있도록 해야함
    
    페이스북은 이를 위해 유향 비순환 그래프 프로그래밍 모델을 도입하여 일련의 작업들이 순차적으로 또는 병렬적으로 실행될 수 있게 하고 있음
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/243c52d1-f97d-40b8-8713-9abc9716eaed/Untitled.png)
    
- **비디오 트렌스코딩 아키택쳐**
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/068613ff-5d68-435c-80be-518e52944ccd/Untitled.png)
    
    1. 전처리기
        1. 비디오 분할 : 비디오 스트림을 GOP 라는 단위로 쪼갬
            1. GOP(Group Of Picture) :  특정 순서로 배열된 프레임 그룹, 독립적으로 재생 가능
        2. DAG 생성 : 클라이언트 프로그래머가 작성한 설정 파일에 따라 DAG를 만들어냄
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/22635c24-cb83-47fa-92f4-f48dbfffa2eb/Untitled.png)
        
        c. 데이터 캐시 : GOP와 메타데이터를 임시 저장소에 보관
        
    
    1. DAG 스케줄러
    
    DAG 그래프를 몇 개 단게로 분할하여 각각을 자원 관리자의 작업 큐에 집어넣음
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5ead7478-206d-4e6c-93c7-dd31ec2e0d69/Untitled.png)
    
    1. 자원 관리자
    
    자원 배분을 하는 역할로 아래 그림과 같이 3개의 큐와 작업 스캐줄러로 구성
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3cc36d6f-3434-4860-9f47-bc8eee7746c8/Untitled.png)
    
    - 작업 큐 : 실행할 작업이 보관되어 있는 우선순위 큐
    - 작업 서버 큐 : 작업 서버의 가용상태 정보가 보관되어 있는 큐
    - 실행 큐 : 현재 실행 중인 작업 미치 작업 서버 정보가 보관되어 있는 큐
    - 작업 스케줄러 : 최적의 작업/서버 조합을 골라, 해당 작업 서버가 작업을 수행하도록 지시하는 역할
    
    1. 작업 서버
    
    DAG에 정의된 작업을 수행한다. 작업 종류에 따라 작업 서버도 구분하여 관리한다.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0fb6ebfe-a45d-4531-8aab-7a91d5fe254d/Untitled.png)
    
- **시스템 최적화**
    1. 비디오 병렬 업로드
        
        작은 단위의 GOP 들로 분할하여 병렬적으로 업로드
        
        ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1dfc76c0-8f13-46c4-9f20-3f5b6c4b3a79/Untitled.png)
        
    2. 업로드 센터를 사용자 근거리에 지정
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3c2bc4fe-d8b4-4db6-80a7-f2fe5611d6a3/Untitled.png)
    
    1. 모든 절차를 병렬화
    
    느슨하게 결합된 시스템을 만들어서 병렬성을 높임
    
    ![강하게 결합되어있는 각 시스템들](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1c4125eb-368d-4460-a66d-162a12ad96fa/Untitled.png)
    
    강하게 결합되어있는 각 시스템들
    
    이러한 시스템의 결합도를 낮추기 위해 메세지 큐를 도입
    
    - 인코딩 모듈은 다운로드 모듈의 작업이 끝나기를 더 이상 기다릴 필요가 없음
    - 메세지 큐에 보관된 이벤트 각각을 인코딩 모듈은 병렬적으로 처리할 수 있음
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c98aee47-9758-4542-b9f4-420cca87b9d1/Untitled.png)
    
    1. 미리 사인된 업로드 URL
    
    허가받은 사용자만이 올바른 장소에 비디오를 업로드 할 수 하기 위해, 미리 사인된 업로드 URL을 이용한다.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a3d08b22-3420-4ee9-ab43-5ced044ac4d7/Untitled.png)
    
    1. POST 요청을 통해 미리 사인된 URL을 받는다.
        1. 미리 사인된 URL : s3 에서 사용하는 용어로 MS Azure는 접근공유 시그니쳐 라고 부름
    2. 해당 URL이 가리키는 위치에 비디오를 업로드한다.
    
    1. 비디오 보호
        1. DRM(Digital Rights Management) : 애플의 페어플레이, 구글의 와이드바인, 마이크로소프트의 플레이테디
        2. AES 암호화 : 비디오를 암호화 하고 접근권한을 설정하는 방식. 허락된 사용자만이 비디오 시청 가능
        3. 워터마크
    
    1. 비용 최적화
    
    연구결과에 따르면, 유튜브의 스트리밍은 롱테일 분포를 따름. 이에 착안해서 몇 가지 최적화가 가능
    
    1. 인기 비디오는 CDN을 통해 재생하되 다른 비디오는 비디오 서버를 통해 재생
    2. 인기 없는 비디오는 인코딩을 제한 할 수 있음
    3. 특정 지역에서만 인기 높은 비디오의 경우, 해당 비디오는 다른 지역에 옮길 필요가 없음
    4. CDN을 직접 구축하고 인터넷 서비스 제공자(ISP)와 제휴
