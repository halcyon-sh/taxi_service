# Taxi Service (카카오택시)

## Table of contents

- [예제 - Taxi Service](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [DDD 의 적용](#ddd-의-적용)
    - [Correlation](#Correlation)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 일관성 테스트](#비동기식-호출--시간적-디커플링--장애격리--최종-일관성-테스트)
  - [운영](#운영)
    - [CI/CD 설정](#CICD-설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출--서킷-브레이킹--장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)

## 서비스 시나리오

#### 기능적 요구사항
```
1. 고객은 목적지를 입력하여 차량배정 요청을 한다
2. 드라이버는 차량배정 요청 목록 중 선택하여 수락 여부를 결정한다
3. 고객이 탑승시, 드라이버는 운행을 시작한다 
4. 목적지에 도착시, 드라이버는 운행을 종료하고, 이때 발생한 요금정보가 계산되어 요금결제가 진행된다(1000원/분)
5. 고객은 운행 시작 이전 상태에서는 차량 배정 요청 취소가 가능하다. 취소시점에 드라이버가 운행을 수락한 상태라면, 해당 운행정보는 삭제된다.
6. 차량배정 요청 / 수락, 운행시작 / 종료, 요금결제 시 고객에게 메시지가 발송된다.
```
*****

#### 비기능적 요구사항
```
1. 트랜잭션 
  - 운행 종료시 자동으로 요금결제 처리 된다.(Sync)

2. 장애격리 
  - 차량배정 요청 및 수락 기능은 24시간 받을 수 있어야 한다. (Async 호출-event-driven)
  - 결제시스템이 과중되면 결제를 받지 않고 결제를 잠시 후에 하도록 유도한다. (Circuit breaker, fallback)

3. 성능
  - 고객은 차량배정 요청후 상태를 차량배정 여부 등 상태를 확인할수 있어야 한다. (CQRS)
  - 차량배정 요청 / 수락, 운행시작 / 종료, 요금결제 시 고객에게 알림을 줄 수 있어야 한다 (Event Driven)
```
*****

#### 체크포인트
- 분석설계 (40)
  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현 (35)
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영 (25)
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?
*****

# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)

## TO-BE 조직 (Vertically-Aligned)

## Event Storming 결과

### MSAEz 로 모델링한 이벤트스토밍 결과:
http://www.msaez.io/#/storming/kycX5k0eFueNvkNVODgeEyDOMSE3/mine/3d90ff6635560983f312407faa267f6d

### 이벤트 도출

### 부적격 이벤트 탈락

### 액터, 커맨드 부착하여 읽기 좋게

### 어그리게잇으로 묶기

### 바운디드 컨텍스트로 묶기

### 폴리시 부착 
- 괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

### 완성된 1차 모형
![최종 결과 이미지](https://user-images.githubusercontent.com/83382676/124608777-f9fed980-dea9-11eb-9031-19d1b3cd9b17.png)

### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

#### - 기능적 요구사항 검증
![기능요구사항 검증 1](https://user-images.githubusercontent.com/83382676/124608815-02efab00-deaa-11eb-8749-b9612b616d3b.png)

```
기능적 요구사항 - 1
 - 고객은 목적지를 입력하여 차량배정 요청한다. (OK)
 - 드라이버는 차량배정 요청 목록중 선택하여 수락여부를 결정한다. (OK)
 - 고객이 탑승시, 드라이버는 운행을 시작한다. (OK)
 - 목적지에 도착한 후, 드라이버는 운행을 종료한다. (OK)
 - 운행 종료시 요금이 자동 결제 된다. (OK)
```

![기능 요구사항 검증2](https://user-images.githubusercontent.com/83382676/124609244-6c6fb980-deaa-11eb-8e4e-800227b49cb4.png)

```
기능적 요구사항 - 2
 - 고객은 운행 시작전까지는 차량 배정 요청 취소가 가능하다. (OK-노란색)
 - 드라이버가 운행 수락한 상태라면, 해당 운행 예정 정보는 삭제한다. (OK-노란색)
 - 차량배정 요청/수락 , 운행 시작/종료, 요금결제시 고객에게 메시지를 발송한다. (Not OK - 초록색, 운행시작시 메시지 미발송, 요청 취소시 메시지 발송)
```

#### - [수정1] 운행시작시 메시지 발송, 요청 취소시 메시지 미발송으로 변경 
![수정된 최종 결과 이미지](https://user-images.githubusercontent.com/83382676/124610671-ac836c00-deab-11eb-9e86-faeef37d950a.png)


#### - 비기능적 요구사항 검증
```
비기능적 요구사항
 - 운행 종료시 자동으로 요금결제 처리 된다.(Sync) - ① ACID 트랜잭션 적용. 운행종료시 결제처리는 Request-Response 방식 처리
 - 고객 알람 기능 등 데이터 일관성이 크리티컬 하지 않은 Case 들은 - ② Eventual Consistency 방식 트랜잭션 처리
 - 고객은 차량배정 요청 후, 상태를 차량배정 여부 등 상태를 확인 할 수 있어야 한다.(CQRS)- ③ Dashboard
```


*****

### 헥사고날 아키텍처 다이어그램 도출
![헥사고날 아키텍처](https://user-images.githubusercontent.com/83382676/124612664-7ba43680-dead-11eb-9590-51c07e512aac.png)
```
- Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
- 호출관계에서 PubSub 과 Req/Resp 를 구분함
- 서브 도메인과 바운디드 컨텍스트의 분리
```
*****
# 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다.
```
mvn spring-boot:run

 1) CarAllocationRequest  - Port : 8081
 2) Driving               - Port : 8086
 3) Payment               - Port : 8083
 4) Dashboard             - Port : 8084
 5) Message               - Port : 8085
```
## CQRS
차량배정 상태, 운행 상태, 결제 금액 등 총 Status 에 대하여 조회 할 수 있도록 CQRS 로 구현하였다.
- CarAllocationRequest, Driving, Payment 개별 Aggregate Status 를 통합 조회하여 성능 Issue 를 사전에 예방할 수 있다.
- 비동기식으로 처리되어 발행된 이벤트 기반 Kafka 를 통해 수신/처리 되어 별도 Table 에 관리한다
- 모델링 (DrivingInfo)
```
    Long    carReqId;      // 차량요청 ID
    String  userId;        // 고객 ID
    String  destAddr;      // 목적지 주소
    String  allocStatus;   // 차량배정상태 (Requested, Approved, DrivingStarted, DrivingFinished)
    Long    drivingId;     // 운행 ID
    String  drivingStatus; // 운행 상태 (DrivingStarted, DrivingFinished)
    String  startTime;     // 운행 시작 시간
    String  finishTime;    // 운행 종료 시간
    Long    fee;           // 금액
    Long    paymentId;     // 결제ID
    String  paymentStatus; // 결제상태 (PayRequested, PaymentApproved)
```
- viewpage MSA ViewHandler 를 통해 구현 (차량 배정 요청(allocationRequested) 이벤트 발생 시, Pub/Sub 기반으로 별도 DrivingInfo 테이블에 저장)
```
# DrivingInfoViewHandler.java

    @StreamListener(KafkaProcessor.INPUT)
    public void whenAllocationRequested_then_CREATE_1 (@Payload AllocationRequested allocationRequested) {
        try {
            if (!allocationRequested.validate()) return;

            // view 객체 생성
            DrivingInfo drivingInfo = new DrivingInfo();
            // view 객체에 이벤트의 Value 를 set 함
            drivingInfo.setCarReqId(allocationRequested.getCarReqId());
            drivingInfo.setUserId(allocationRequested.getUserId());
            drivingInfo.setAllocStatus(allocationRequested.getAllocStatus());
            drivingInfo.setDestAddr(allocationRequested.getDestAddr());
            
            // view 레파지토리 save
            drivingInfoRepository.save(drivingInfo);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```
- 실제로 Dashboard view 페이지를 조회해 보면 모든 차량배정, 운행에 대한 전반적인 상태, 결제 상태 등의 정보를 종합적으로 알 수 있다
![CQRS](https://user-images.githubusercontent.com/83382676/124758741-dac78100-df69-11eb-8b23-1daec5f0bb83.png)


## 동기식 호출 과 Fallback 처리
요구사항 조건 중 하나로 운행 종료 시 요금 결제 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현
```
# PaymentService.java

@FeignClient(name="Driving", url="http://localhost:8083")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.GET, path="/payFee")
    public void payFee(@RequestParam("drivingId") Long drivingId,
                       @RequestParam("fee") Long fee, 
                       @RequestParam("paymentStatus") String paymentStatus );

}

```

- 운행 종료 직후(@PostUpdate) 요금 결제를 동기(Sync)로 요청하도록 처리
```
# Driving.java

    @PostUpdate
    public void onPostUpdate(){
    
        ..............
    
        else if ( this.getAllocStatus().equals("DrivingFinished") )
        {
            System.out.println("########################  onPostUpdate drivingFinished publishAfterCommit" );

            // 자동 요금결제 Req-Res (sync)
            taxiservice.external.Payment payment = new taxiservice.external.Payment();
            DrivingApplication.applicationContext.getBean(taxiservice.external.PaymentService.class)
                .payFee(this.drivingId, this.fee, "PayRequested");
            
            // 결재처리 후 운행종료
            DrivingFinished drivingFinished = new DrivingFinished();
            BeanUtils.copyProperties(this, drivingFinished);
            drivingFinished.publishAfterCommit();
        }
        
        ..............
```
- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 운행 종료 불가함을 확인
```
# 결제 (payment) 서비스를 잠시 내려놓음 (ctrl+c)

# 운행 종료 실패
http PATCH http://localhost:8086/drivings/2 allocStatus="DrivingFinished" drivingStatus="DrivingFinished"
```
![운행 종료 오류](https://user-images.githubusercontent.com/83382676/124761608-ef594880-df6c-11eb-934b-cb14706d4e2a.png)


```
# 결제서비스 재기동
cd payment
mvn spring-boot:run

# 운행 종료 성공
http PATCH http://localhost:8086/drivings/2 allocStatus="DrivingFinished" drivingStatus="DrivingFinished"
```
![운행 종료 성공](https://user-images.githubusercontent.com/83382676/124761633-f2eccf80-df6c-11eb-8025-a39334e9dbea.png)


## DDD 의 적용
각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. (예시는 CarAllocationRequest 마이크로 서비스). 현실에서 발생가능한 이벤트에 의하여 마이크로 서비스들이 상호 작용하기 좋은 모델링으로 구현을 하였다.
```
# CarAllocationRequest.java

@Entity
@Table(name="CarAllocationRequest_table")
public class CarAllocationRequest {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long carReqId;
    private String userId;
    private String destAddr;
    private String allocStatus;

    .....
    
    public Long getCarReqId() {
        return carReqId;
    }

    public void setCarReqId(Long carReqId) {
        this.carReqId = carReqId;
    }
    
    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }
    
    public String getDestAddr() {
        return destAddr;
    }

    public void setDestAddr(String destAddr) {
        this.destAddr = destAddr;
    }
    
    public String getAllocStatus() {
        return allocStatus;
    }

    public void setAllocStatus(String allocStatus) {
        this.allocStatus = allocStatus;
    }
}
```

## Correlation
본 예제에서는 PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현을 이벤트 클래스 안의 변수로 전달받아 서비스간 연관된 처리를 구현하였다.

아래의 예제를 보면

차량배정요청 건에 대하여 수락(AllocationRequestApproved)을 하면 동시에 연관된 차량배정요청(CarAllocationRequest)의 상태가 변경 되고, 운행 시작 / 종료에 따라 다시 연관된 차량배정요청(CarAllocationRequest), 결제(Payment) 서비스의 상태 데이터가 변경되는 것을 확인할 수 

1) 차량배정 요청

![차량 요청](https://user-images.githubusercontent.com/83382676/124758783-e5821600-df69-11eb-8390-ffd3aad712bf.png)

2) 차량배정 요청 수락

![요청 수락](https://user-images.githubusercontent.com/83382676/124758801-e9159d00-df69-11eb-93f9-1da4f2248174.png)

3) 수락 후 - 차량배정요청 상태

![수락 후 상태](https://user-images.githubusercontent.com/83382676/124758813-eca92400-df69-11eb-8dae-d734473aac33.png)

4) 운행시작

![운행 시작](https://user-images.githubusercontent.com/83382676/124758833-f0d54180-df69-11eb-99b6-66373793cbe8.png)

5) 운행종료

![운행 종료](https://user-images.githubusercontent.com/83382676/124758848-f3d03200-df69-11eb-9973-c3a5c5307e3a.png)

5) 운행종료 후 - 결재 상태

![운행 종료 후 결제 상태](https://user-images.githubusercontent.com/83382676/124758857-f6cb2280-df69-11eb-8061-149171b433b1.png)



## 폴리글랏 퍼시스턴스

## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 일관성 테스트
차량배정 요청 수락, 운행시작, 운행종료 시 차량배정요청 상태가 업데이트 되며, 이때 메시지가 전송되는 시스템과의 통신 행위는 비동기식으로 처리한다. 

- 차량 배정요청 수락, 운행시작, 운행종료 시에는 각각의 이벤트를 카프카로 송출한다.(publish)
```
# Driving.java

    @PostUpdate
    public void onPostUpdate(){

        System.out.println("######################## onPostUpdate " + this.getAllocStatus() );
        
        if ( this.getAllocStatus().equals("Approved")  )
        {
            System.out.println("########################  onPostUpdate allocationRequestApproved publishAfterCommit" );
            AllocationRequestApproved allocationRequestApproved = new AllocationRequestApproved();
            BeanUtils.copyProperties(this, allocationRequestApproved);
            allocationRequestApproved.publishAfterCommit();
        }
        else if ( this.getAllocStatus().equals("DrivingStarted") )
        {
            System.out.println("########################  onPostUpdate drivingStarted publishAfterCommit" );
            DrivingStarted drivingStarted = new DrivingStarted();
            BeanUtils.copyProperties(this, drivingStarted);
            drivingStarted.publishAfterCommit();
        }
        else if ( this.getAllocStatus().equals("DrivingFinished") )
        {
            System.out.println("########################  onPostUpdate drivingFinished publishAfterCommit" );

            // 자동 요금결제 Req-Res
            taxiservice.external.Payment payment = new taxiservice.external.Payment();

            DrivingApplication.applicationContext.getBean(taxiservice.external.PaymentService.class)
                .payFee(this.drivingId, this.fee, "PayRequested");
            
            // 결재처리 후 운행종료
            DrivingFinished drivingFinished = new DrivingFinished();
            BeanUtils.copyProperties(this, drivingFinished);
            drivingFinished.publishAfterCommit();
        }
        else return;

    }
```

- 차량배정요청 시스템에서는 각각의 이벤트에 대해서 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.
```
# CarAllocationRequest > PolicyHandler.java

@Service
public class PolicyHandler{
    @Autowired CarAllocationRequestRepository carAllocationRequestRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverAllocationRequestApproved_UpdateAllocation(@Payload AllocationRequestApproved allocationRequestApproved){

        if(!allocationRequestApproved.validate()) return;

        System.out.println("\n\n##### listener UpdateAllocation : " + allocationRequestApproved.toJson() + "\n\n");

        // Sample Logic //
        CarAllocationRequest carAllocationRequest = carAllocationRequestRepository.findByCarReqId( allocationRequestApproved.getCarReqId() );
        carAllocationRequest.setAllocStatus( allocationRequestApproved.getAllocStatus() );
        carAllocationRequestRepository.save(carAllocationRequest);
            
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverDrivingStarted_UpdateAllocation(@Payload DrivingStarted drivingStarted){

        if(!drivingStarted.validate()) return;

        System.out.println("\n\n##### listener UpdateAllocation : " + drivingStarted.toJson() + "\n\n");

        // Sample Logic //
        CarAllocationRequest carAllocationRequest = carAllocationRequestRepository.findByCarReqId( drivingStarted.getCarReqId() );
        carAllocationRequest.setAllocStatus( drivingStarted.getAllocStatus() );
        carAllocationRequestRepository.save(carAllocationRequest);
            
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverDrivingFinished_UpdateAllocation(@Payload DrivingFinished drivingFinished){

        if(!drivingFinished.validate()) return;

        System.out.println("\n\n##### listener UpdateAllocation : " + drivingFinished.toJson() + "\n\n");

        // Sample Logic //
        CarAllocationRequest carAllocationRequest = carAllocationRequestRepository.findByCarReqId( drivingFinished.getCarReqId() );
        carAllocationRequest.setAllocStatus( drivingFinished.getAllocStatus() );
        carAllocationRequestRepository.save(carAllocationRequest);
            
    }
```

- 그 외 메시지 서비스는 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 메시지 서비스가 유지보수로 인해 잠시 내려간 상태 라도 차량요청, 운행 서비스에는 문제가 없다.
```
# 메시지 (Message) 서비스를 잠시 내려놓음 (ctrl+c)

# 차량 배정 요청 - 정상
http POST localhost:8081/carAllocationRequests userId="USER1" destAddr="Misa" allocStatus="Requested"

# 요청수락 - 정상
http PATCH http://localhost:8086/drivings/1 allocStatus="Approved"
   
# 운행시작 - 정상
http PATCH http://localhost:8086/drivings/1 allocStatus="DrivingStarted" drivingStatus="DrivingStarted"
   
# 운행종료 - 정상
http PATCH http://localhost:8086/drivings/1 allocStatus="DrivingFinished" drivingStatus="DrivingFinished"
   
```

## Gateway
Gateway 생성을 통하여 마이크로서비스들의 진입점을 통일시킴
```
# gateway > src > resources > application.yml

server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: CarAllocationRequest
          uri: http://localhost:8081
          predicates:
            - Path=/carAllocationRequests/** 
        - id: Driving
          uri: http://localhost:8086
          predicates:
            - Path=/drivings/** 
        - id: Payment
          uri: http://localhost:8083
          predicates:
            - Path=/payments/** 
        - id: Dashboard
          uri: http://localhost:8084
          predicates:
            - Path= /drivingInfoes/**
        - id: Message
          uri: http://localhost:8085
          predicates:
            - Path=/messages/** 
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true
```

- Gateway 서비스 기동 후 Gateway Port 8088을 통해 각각의 서비스에 접근 가능 여부 확인

1) 차량 요청

![gateway1](https://user-images.githubusercontent.com/83382676/124851048-bbb60700-dfdc-11eb-98fc-93f956102472.png)


2) 운행 종료 후 메시지 조회

![gateway2](https://user-images.githubusercontent.com/83382676/124851055-beb0f780-dfdc-11eb-8130-17c78ac49408.png)


3) 운행 종료 후 Dashboard 조회

![gateway3](https://user-images.githubusercontent.com/83382676/124851060-c1abe800-dfdc-11eb-9932-0c931cd29b34.png)




*****
# 운영

## CI/CD 설정


## 동기식 호출 / 서킷 브레이킹 / 장애격리


## 오토스케일 아웃


## 무정지 재배포


