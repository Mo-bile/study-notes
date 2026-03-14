
# **ECS 전환 회의안 – 주요 키워드 개념 정리**
# 1) **AWS API Gateway**

**정의:**  
클라이언트가 백엔드 API에 접근할 수 있는 **완전 관리형 API 엔드포인트** 서비스.

**기능:**

- 인증/인가
- 라우팅
- 요청/응답 변환
- Rate limit
- 버전 관리
- 트래픽 제어

**왜 쓰는가:**  
EC2에 직접 만든 Node 게이트웨이를 없애고  
AWS에서 제공하는 안정적/확장성 있는 API Gateway로 통합하려는 것.

---

# 2) **CloudFront**

**정의:**  
AWS CDN(Content Delivery Network) 서비스.

**왜 API 앞단에 쓰는가?**

- 전 세계 엣지에서 API 응답 속도 개선
    
- WAF와 자연스럽게 연동
    
- 공격을 엣지 레벨에서 차단
    
- API Gateway 도메인 호스팅 용도
    

---

# 3) **WAF (Web Application Firewall)**

**정의:**  
HTTP(S) 트래픽을 검사하여 공격을 차단하는 방화벽.

**막는 것:**

- SQL Injection
    
- XSS
    
- L7 공격
    
- Bot 트래픽
    
- 비정상 요청 패턴
    

**역할:**  
API Gateway 앞단에서 1차 보안.

---

# 4) **CloudMap (Service Discovery)**

**정의:**  
ECS 서비스끼리 내부에서 서로 호출할 수 있는 **서비스 디스커버리** 솔루션.

**예시 주소 형태:**

```
http://wallet.dev-cluster.local:8080/api
```

**왜 필요한가?**

- 내부 서비스 간 통신 시 ALB 없이 “이름 기반” 통신 가능
    
- IP가 동적으로 바뀌는 ECS Task를 자동으로 매핑
    

---

# 5) **Internal ALB**

**정의:**  
내부 통신만 가능(인터넷 노출 X)한 로드밸런서.

**사용 이유:**

- bridge 네트워크 모드 컨테이너는 CloudMap에 직접 연결 안될 수 있음
    
- 기존 internal ALB 주소로 라우팅해 해결
    

---

# 6) **ECS (Elastic Container Service)**

**정의:**  
AWS의 컨테이너 오케스트레이션 서비스.

**특징:**

- EC2 기반, Fargate 기반 둘 다 지원
    
- Task/Service 개념
    
- 배포 전략(rolling, blue/green) 적용 가능
    

---

# 7) **CodePipeline**

**정의:**  
AWS의 CI/CD 오케스트레이션 서비스.

**역할:**

- Git 커밋 → 빌드 → 배포  
    을 자동으로 연결하는 파이프라인.
    

---

# 8) **CodeBuild**

**정의:**  
컨테이너 기반으로 **빌드 작업**을 수행하는 서비스.

**비용 핵심:**

- CodeBuild 빌드 시간 × 사용 리소스  
    이 대부분의 CI/CD 비용을 차지.
    

---

# 9) **CodeDeploy (ECS)**

**정의:**  
ECS 서비스의 **배포/롤백을 책임지는 서비스**.

**기능:**

- Rolling 업데이트
    
- Blue/Green 배포
    
- 트래픽 전환
    
- 배포 실패 시 자동 롤백
    

---

# 10) **Blue/Green 배포**

**정의:**  
새 버전(Green)을 띄운 뒤 테스트 → 문제 없으면 트래픽 전환.

**장점:**

- 안정성 최고
    
- 문제 시 즉시 원래 버전(Blue)로 롤백 가능
    

**단점:**

- 배포 시간이 길다 (기존 대비 2배 이상)
    
- 새 환경을 따로 띄우기 때문에 리소스 증가
    

---

# 11) **Rolling 업데이트**

**정의:**  
기존 ECS Task를 하나씩 교체하며 배포하는 방식.

**장점:**

- 빠름
    
- 리소스 추가 부담 적음
    

**단점:**

- 배포 중 일부 트래픽이 구/신 버전 섞일 수 있음
    
- 장애 발생 시 복구 속도가 Blue/Green보다 낮음
    

---

# 12) **Bridge 네트워크 / awsvpc 네트워크 모드**

### Bridge

- Docker 기본 네트워크
    
- ECS Fargate에서는 미지원
    
- 서비스 디스커버리(CloudMap) 사용 어려움
    

### awsvpc

- 컨테이너가 **개별 ENI(IP)**를 받음
    
- CloudMap 사용 가능
    
- 실제 서버처럼 동작
    

---

# 13) **Public Subnet / Private Subnet**

**Public Subnet**

- Internet Gateway에 연결되어 외부 통신 가능
    

**Private Subnet**

- 외부에서 직접 접근 불가
    
- 내부(백엔드, DB) 구성에 사용
    

---

# 14) **Ingress / Egress 구조 변화**

새 구조에서는:

- 외부 접근 → CloudFront → WAF → API Gateway
    
- 내부 서비스 → CloudMap
    

기존 구조(ECS 전환 전)에서는:

- EC2 API-GW를 경유해 Internal ALB로 프록시
    
- 개발자는 VPN + 직접 호출 가능
    

---

# 15) **배포 속도 vs 안정성**

- Rolling → 빠름, 리소스 적음
    
- Blue/Green → 안정성 뛰어남, 리소스와 시간 2배 필요
    
- 개발 환경에서는 Blue/Green이 비효율적일 수 있음
    

---

# 16) **개발 환경 2중화 논의**

구조 제안:

```
개발1 → 개발2(테스트) → 스테이지 → 운영
```

**이유:**

- Blue/Green 없이도 개발 안전성 확보
    
- 배포 속도 빠르게 가져가고
    
- 운영 품질은 스테이지에서 검증
    

---

# 17) **EC2 API-GW 제거 이유**

- 기존 개발사가 Node 기반 gateway 운영
    
- 유지·운영 상 비효율
    
- AWS Native API Gateway가 더 안정적
    
- WAF/CloudFront 연동 간편
    
- 관리 비용 감소
    

---

# 🔥 총정리 (암기용 10개 키워드)

```
API Gateway – 통합 API 엔드포인트
CloudFront – CDN + 도메인/WAF 연동
WAF – L7 보안
CloudMap – ECS 서비스 디스커버리
Internal ALB – 내부 전용 로드밸런서

CodePipeline – CI/CD 오케스트레이션
CodeBuild – 빌드 수행
CodeDeploy – ECS 배포/롤백 관리

Blue/Green – 안정성
Rolling Update – 속도

awsvpc 네트워크 – 컨테이너별 IP
Public/Private Subnet – 인터넷 접근 구분
```

---

# ✅ **질문 1. AWS API Gateway는 직접 만든 nginx 게이트웨이(리버스 프록시)를 대신하는 것인가?**

**결론:**

```
대체한다는 표현이 맞음.
(nginx reverse proxy 역할을 AWS API Gateway가 수행하게 됨)
```

**근거:**  
네가 준 회의안에서

> “노드로 하는 게이트웨이 없어짐”  
> “API 게이트웨이 호출하게끔 변경”

➡ 기존에는 **EC2 위에서 Node/NGINX 기반 reverse proxy**를 운영하고 있었음.  
➡ 전환 후에는 **AWS API Gateway가 모든 외부 API 진입점 역할**을 맡음.

즉,

- URL 라우팅
- 인증/보안
- 프록시 역할
- 요청/응답 처리

이전까지 nginx/Node에서 하던 기능을 API Gateway가 통합 처리한다고 보면 됨.

---

# ✅ **질문 2. ECS에서 컨테이너는 무엇인가?**

**정확한 답:**

```
컨테이너는 ‘독립된 실행 환경에서 돌아가는 애플리케이션 단위’임.
(보통 Docker 이미지를 기반으로 실행되는 프로세스)
```

더 풀어서 말하면:

- ECS는 “컨테이너 오케스트레이션 서비스”
- 컨테이너는 Docker 이미지 하나가 실행되는 **프로세스**
- VM처럼 OS 전체를 띄우지 않음
- 파일시스템·네트워크·환경변수 등이 격리된 실행 단위

**예시:**

```
wallet-service 컨테이너
renewal-service 컨테이너
notification-service 컨테이너
```

이런 식으로 서비스마다 컨테이너 하나씩 띄운다고 보면 됨.

---

# ✅ **질문 3. Blue는 왜 원래 버전이고 Green은 왜 신버전인가?**

**정확한 내용:**

```
Blue = 현재 운영 중인 버전(기존 버전)
Green = 새로 배포되는 버전(신버전)
```

이건 AWS가 정한 규칙이 아니라,  
**Blue/Green 배포라는 업계 표준 용어 자체가 이렇게 정의되어 있음.**

**배경 개념:**

- Blue(파란색) → 기존에 안정적으로 있는 시스템
- Green(초록색) → 새롭게 준비된 시스템

이 색 naming은 단순한 관례이고, 실제로는 “원래/새로운”이라는 대비를 주기 위한 것.

“왜 파란색이 기존이고 초록색이 새로운가?”는 특별한 이유가 있는 것이 아니라:

```
Blue = old  
Green = new
```

이라는 관습적 표현일 뿐.

---

# ✅ **질문 4. Rolling Update는 기존 방식처럼 빌드를 덮어씌우는 개념인가?**

**가장 정확한 표현:**

```
YES. 기존 방식(덮어쓰기 배포)에 가장 가까운 배포 전략이 Rolling Update임.
단, ECS는 자동으로 '한 개씩 교체하면서' 배포한다는 점이 다름.
```

### 💡 차이점을 정확히 정리하면

#### 🔹 기존 방식(단순 덮어쓰기)

- 서버 하나에 새 버전 올림
    
- 덮어쓰는 동안 잠깐 다운타임 생길 수 있음
    
- 운영 중인 앱이 바로 교체됨
    

#### 🔹 Rolling Update(ECS)

- 기존 태스크(Task)를 **하나씩** 종료
    
- 새 버전 태스크를 **하나씩** 띄움
    
- 둘이 섞여 있는 상태로 점진적으로 교체
    
- 다운타임 최소화
    

```
기존 서버 A -> 새 버전 A  
기존 서버 B -> 새 버전 B  
기존 서버 C -> 새 버전 C
```

이런 식으로 순차적으로 교체됨.

**따라서 개념적으로는 “덮어쓰기 배포”와 비슷하지만,  
운영 중 다운타임 없이 조금씩 교체한다는 점이 큰 차이.**

---

# 🔥 4문장 요약 (아주 짧게 암기용)

1. **API Gateway = nginx reverse proxy 대체 + AWS 관리형 API 진입점.**
2. **컨테이너 = Docker 기반으로 만들어진 격리된 앱 실행 단위.**    
3. **Blue = 기존 버전, Green = 신규 버전(업계 표준).**
4. **Rolling = 덮어싸는 배포와 비슷하지만 하나씩 교체하며 다운타임 최소화.**