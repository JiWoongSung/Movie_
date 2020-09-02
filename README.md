<img width="300" alt="스크린샷 2020-09-01 오후 2 39 42" src="https://user-images.githubusercontent.com/26249603/91798946-02223300-ec61-11ea-9a97-dba05ddb5279.png">


# 예제 - 영화 예매 시스템 

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 영화예매]
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd-설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출서킷-브레이킹장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)


# 서비스 시나리오

기능적 요구사항
1. 고객이 영화를 선택하여 예매한다.
2. 고객이 결제를 한다. (동기, 결제 서비스)
3. 결제가 되면 해당 좌석이 점유된다. (비동기, 좌석 서비스)
4. 결제가 완료(정상 승인)되면, 결제 상태를 사용자에게 전달한다. (비동기, 알람 서비스)
(수정)
5. 고객은 본인의 예약을 취소할 수 있다. 
6. 예약이 취소되면 결제가 취소된다. (비동기, 결제 서비스)
7. 결제가 취소되면, 결제 취소 내용을 사용자에게 전달한다. (비동기, 알림서비스)
8. 결제가 취소되면 해당 좌석의 점유가 해제된다. (비동기, 좌석 서비스)
9. 결제가 취소되면 예약의 상태가 변경된다. (비동기, 예약 서비스)
(수정)


비기능적 요구사항
  
1. 트랜잭션
    1. 결제가 되지 않은 예약 건은 아예 예약이 성립되지 않아야 한다. - Sync 호출
1. 장애격리
    1. 알림 서비스 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
    1. 결제 시스템이 과중되면 사용자를 잠시 받지 않고 결제를 잠시 후에 하도록 유도한다 Circuit breaker, fallback
1. 성능
    1. 예결제 상태가 바뀔 때마다 SMS(카톡) 등으로 알림을 줄 수 있어야 한다.  Event driven


# 체크포인트

- 분석 설계


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
    
- 구현
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
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계
## AS-IS 조직 (Horizontally-Aligned)
<img width="800" alt="스크린샷 2020-09-01 오전 10 32 18" src="https://user-images.githubusercontent.com/26249603/91784855-db530500-ec3e-11ea-8199-e256d0760134.png">


## TO-BE 조직 (Vertically-Aligned)
<img width="800" alt="스크린샷 2020-09-01 오후 2 35 27" src="https://user-images.githubusercontent.com/26249603/91798686-6b557680-ec60-11ea-9bd0-79e977822c29.png">


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: 
링크 추후 첨부

### 이벤트 도출
<img width="800" alt="스크린샷 2020-09-01 오전 11 48 15" src="https://user-images.githubusercontent.com/26249603/91789213-2540e880-ec49-11ea-90e1-2099d742161b.png">

### 액터, 커맨드 부착하여 읽기 좋게
<img width="800" alt="스크린샷 2020-09-01 오후 2 50 01" src="https://user-images.githubusercontent.com/26249603/91799758-b1133e80-ec62-11ea-8859-43ec061f8360.png">

### 어그리게잇으로 묶기
<img width="800" alt="스크린샷 2020-09-01 오후 2 50 36" src="https://user-images.githubusercontent.com/26249603/91799764-b3759880-ec62-11ea-8fcc-2fdc679a35da.png">
    
### 바운디드 컨텍스트로 묶기
<img width="800" alt="스크린샷 2020-09-01 오후 2 50 41" src="https://user-images.githubusercontent.com/26249603/91799768-b5d7f280-ec62-11ea-9140-79b956704900.png">

    - 도메인 서열 분리 
        - Core Domain: Booking(예약관리), Seat(좌석관리) : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준은 예약관리 99.999% / 재고관리 90% 목표, 배포주기는 예약관리 1주일 1회 미만/ 고객관리 2주 1회 미만으로 함
        - General Domain: payment(결제관리) : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음
        - Supporting Domain:  notification(고객관리) : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포 주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함

### 폴리시 부착 
<img width="800" alt="스크린샷 2020-09-01 오후 2 50 48" src="https://user-images.githubusercontent.com/26249603/91799773-b7a1b600-ec62-11ea-8d45-286c48d86109.png">


### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Res) 
<img width="900" alt="스크린샷 2020-09-01 오후 3 47 21" src="https://user-images.githubusercontent.com/26249603/91811513-9775f500-ec6a-11ea-8e63-c2a9bb5dc9f9.png">


### 완성된 모형
<img width="900" alt="스크린샷 2020-09-01 오후 3 26 21" src="https://user-images.githubusercontent.com/26249603/91808602-88417800-ec67-11ea-9b83-13636fb1aae0.png">


### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

<img width="800" alt="그림1" src="https://user-images.githubusercontent.com/26249603/91811892-300c7500-ec6b-11ea-83af-76066c5e2566.png">

    - 고객이 영화를 선택하여 예매한다 (ok)
    - 고객이 결제한다 (ok)
    - 결제가 되면 결제 내역이 고객에게 알림으로 전달된다 (ok)
    - 예약이 완료된 좌석이 점유된다 (ok)


<img width="800" alt="그림12" src="https://user-images.githubusercontent.com/26249603/91811901-326ecf00-ec6b-11ea-9def-4fcb47c1f7c6.png">

    - 고객이 영화 예매를 취소할 수 있다 (ok)
    - 예약이 취소되면 결제가 취소된다 (ok)
    - 결제가 취소되면 해당 내역이 고객에게 알림으로 전달된다 (ok)
    - 예약이 완료된 좌석이 점유 해제된다 (ok)
    - 예약 시스템에서 예약의 상태가 변경된다 (ok)


### 비기능 요구사항에 대한 검증

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리 : 모든 inter-microservice 트랜잭션이 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함


## 헥사고날 아키텍처 다이어그램 도출 
    
<img width="800" alt="스크린샷 2020-09-01 오후 4 50 07" src="https://user-images.githubusercontent.com/26249603/91822828-49192400-ec73-11ea-8bfc-df3433e8a4bf.png">


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 표현
    - 서브 도메인과 바운디드 컨텍스트의 분리: 각 팀의 KPI 별로 아래와 같이 관심 구현 스토리지를 나눠가짐




# 구현

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd teamb-booking
mvn spring-boot:run

cd teamb-payment
mvn spring-boot:run 

cd teamb-seat
mvn spring-boot:run  

cd teamb-notification
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스 내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다.
(예시는 Payment 마이크로 서비스) 이 때 가능한 현업에서 사용하는 언어(유비쿼터스 랭퀴지)를 그대로 사용하려고 노력했다. 


```
package movieTicket;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long paymentId;
    private Long bookingId;
    private Double totalPrice;
    private String paymentStatus;
    private String paymentType;

    
    public Long getPaymentId() {
        return paymentId;
    }

    public void setPaymentId(Long paymentId) {
        this.paymentId = paymentId;
    }
    public Long getBookingId() {
        return bookingId;
    }

    public void setBookingId(Long bookingId) {
        this.bookingId = bookingId;
    }
    public Double getTotalPrice() {
        return totalPrice;
    }

    public void setTotalPrice(Double totalPrice) {
        this.totalPrice = totalPrice;
    }
    public String getPaymentStatus() {
        return paymentStatus;
    }

    public void setPaymentStatus(String paymentStatus) {
        this.paymentStatus = paymentStatus;
    }
    public String getPaymentType() {
        return paymentType;
    }

    public void setPaymentType(String paymentType) {
        this.paymentType = paymentType;
    }
}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package movieTicket;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface PaymentRepository extends PagingAndSortingRepository<Payment, Long>{

}
```
- (수정예정)적용 후 REST API 의 테스트
```
# booking 서비스의 예약 처리
http localhost:8081/bookings bookingId=1 customerId=1 seatIdList=1,2 quantity=2 price=20000 bookingStatus=”booked”
 status로 예약 처리할 때 같이 넣어주는 건지 헷깔려서.. 일단 넣었습니다.
```

## 폴리글랏 퍼시스턴스

앱프런트 (app) 는 서비스 특성상 많은 사용자의 유입과 상품 정보의 다양한 콘텐츠를 저장해야 하는 특징으로 인해 RDB 보다는 Document DB / NoSQL 계열의 데이터베이스인 Mongo DB 를 사용하기로 하였다. 이를 위해 order 의 선언에는 @Entity 가 아닌 @Document 로 마킹되었으며, 별다른 작업없이 기존의 Entity Pattern 과 Repository Pattern 적용과 데이터베이스 제품의 설정 (application.yml) 만으로 MongoDB 에 부착시켰다

```
# Order.java
package fooddelivery;

@Document
public class Order {

    private String id; // mongo db 적용시엔 id 는 고정값으로 key가 자동 발급되는 필드기 때문에 @Id 나 @GeneratedValue 를 주지 않아도 된다.
    private String item;
    private Integer 수량;

}


# 주문Repository.java
package fooddelivery;

public interface 주문Repository extends JpaRepository<Order, UUID>{
}

# application.yml

  data:
    mongodb:
      host: mongodb.default.svc.cluster.local
    database: mongo-example

```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 예약(reservation)->결제(pay) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
package movieTicket.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="payment", url="http://payment:8080")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void makePayment(@RequestBody Payment payment);

}

```

- 주문을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
@PostPersist
public void onPostPersist(){

    //Following code causes dependency to external APIs
    // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

    movieTicket.external.Payment payment = new movieTicket.external.Payment();
    // mappings goes here
    BookingApplication.applicationContext.getBean(movieTicket.external.PaymentService.class)
        .makePayment(payment);

    Booked booked = new Booked();
    BeanUtils.copyProperties(this, booked);
    booked.publishAfterCommit();
}
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# 결제 (pay) 서비스를 잠시 내려놓음 (ctrl+c)

#예약처리
http localhost:8081/bookings bookingId=1 customerId=1 seatIdList=1,2 quantity=2 price=20000 bookingStatus=”booked” #Fail

#결제서비스 재기동
cd teamb-payment
mvn spring-boot:run

#예약 처리
http localhost:8081/bookings bookingId=1 customerId=1 seatIdList=1,2 quantity=2 price=20000 bookingStatus=”booked” #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)




## 비동기식 호출 과 Eventual Consistency

결제가 이루어진 후에 알람 시스템으로 사용자에게 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 예약 시스템의 처리를 위하여 결제주문이 블로킹 되지 않도록 처리한다.

- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package movieTicket;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long paymentId;
    private Long bookingId;
    private Double totalPrice;
    private String paymentStatus;
    private String paymentType;

    @PostPersist
    public void onPostPersist(){
        PaymentSucceed paymentSucceed = new PaymentSucceed();
        BeanUtils.copyProperties(this, paymentSucceed);
        paymentSucceed.publishAfterCommit();


    }
}
```
알람 서비스에서는 결제 승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다. 실제 구현을 하자면, 카톡 등으로 구현한다. 
  
```
package movieTicket;

import movieTicket.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentSucceed_SendNotification(@Payload PaymentSucceed paymentSucceed){

        if(paymentSucceed.isMe()){
            System.out.println("##### listener SendNotification : " + paymentSucceed.toJson());
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentCanceled_SendNotification(@Payload PaymentCanceled paymentCanceled){

        if(paymentCanceled.isMe()){
            System.out.println("##### listener SendNotification : " + paymentCanceled.toJson());
        }
    }

}

```

알림 시스템은 예약/결제와 완전히 분리되어 있으며, 이벤트 수신에 따라 처리되기 때문에, 알림 시스템이 유지보수로 인해 잠시 내려간 상태라도 예약을 받는데 문제가 없다.

```
#알림 서비스 (store) 를 잠시 내려놓음 (ctrl+c)

#예약 처리
http localhost:8081/bookings bookingId=1 customerId=1 seatIdList=1,2 quantity=2 price=20000 bookingStatus=”booked”

#예약상태 확인
http localhost:8081/bookings 

#알람 서비스 기동
cd teamb-alarm
mvn spring-boot:run

#상태 확인
http localhost:8081/bookings
```


# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 cloudbuild.yml 에 포함되었다.


## 동기식 호출/서킷 브레이킹/장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

- Hystrix 설정: 아래 조건에 해당될 경우 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

feign:
  hystrix:
    enabled: true
  client:
    config:
      default:
        loggerLevel: BASIC
        # feign의 전역 timeout 설정 : 5초
        connectTimeout: 300
        readTimeout: 300

hystrix:
  threadpool:
    default:
      coreSize: 100       #Hystrix 스레드 풀의 기본 크기
      maximumSize: 100    #Hystrix 스레드 풀의 최대 크기
      maxQueueSize: -1
      keepAliveTimeMinutes: 1 #hystrix.threadpool.default.maximumSize 값을 사용하기 위한 설정
      allowMaximumSizeToDivergeFromCoreSize: false
  command:
    default:
      execution:
        timeout:
          enabled: true
        isolation:
          #THREAD 방식에서는 서비스 호출이 별도의 스레드에서 수행, SEMAPHORE 방식에서는 서비스 호출을 위해 별도의 스레드를 만들지 않고 단지 각 서비스에 대한 동시 호출 수를 제한할 뿐
          strategy: THREAD             #THREAD/SEMAPHORE  SEMAPORE는 외부 IO가 없는 경우 사용 권장
          thread:
            # Ribbon의 각 timeout보다 커야 기대하는대로 동작함 (
            timeoutInMilliseconds: 610
            interruptOnTimeout: true            #Timeout걸리면 Thread 실행 중단
            interruptOnCancel: false            #Cancle되었을 때 Thread 실행 중단
      circuitBreaker:
        requestVolumeThreshold: 5           # 설정수 값만큼 요청이 들어온 경우만 circut open 여부 결정 함 - 즉 이 값이 20으로 설정되어있다면 10초간 19개의 요청이 들어와서 19개가 전부 실패하더라도 서킷 브레이커는 열리지않는다
        errorThresholdPercentage: 50        # requestVolumn값을 넘는 요청 중 설정 값이상 비율이 에러인 경우 circuit open
        sleepWindowInMilliseconds: 5000     # 한번 오픈되면 얼마나 오픈할 것인지
        enabled: true
        forceOpen: false
        forceClosed: false
      fallback:
        enabled: true


# PaymentFallback.java

@Component
public class PaymentFallback implements PaymentService{

    @Override
    public void makePayment(Payment payment) {
        System.out.println("hystrix!!!");
    }
}


# PaymentService.java

@FeignClient(name="payment", url="${api.url.payment}", fallback = PaymentFallback.class)
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void makePayment(@RequestBody Payment payment);

}

```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/bookings 

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.73 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.75 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.77 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.97 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.81 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.87 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.12 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.16 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.17 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.26 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.25 secs:     207 bytes ==> POST http://localhost:8081/bookings

* 요청이 과도하여 CB를 동작함 요청을 차단

HTTP/1.1 500     1.29 secs:     248 bytes ==> POST http://localhost:8081/bookings  
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     1.23 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     1.42 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     2.08 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.29 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/bookings

* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작

HTTP/1.1 201     1.46 secs:     207 bytes ==> POST http://localhost:8081/bookings 
HTTP/1.1 201     1.33 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.36 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.63 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.65 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.68 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.69 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.74 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.76 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     1.79 secs:     207 bytes ==> POST http://localhost:8081/bookings

* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리

HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/bookings    
HTTP/1.1 500     1.92 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/bookings

* 생각보다 빨리 상태 호전됨 - (건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복) => 요청 수락

HTTP/1.1 201     2.24 secs:     207 bytes ==> POST http://localhost:8081/bookings  
HTTP/1.1 201     2.32 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.16 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.21 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.29 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.30 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.38 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.59 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.61 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.62 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     2.64 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.01 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.27 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.33 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.45 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.52 secs:     207 bytes ==> POST http://localhost:8081/bookingss
HTTP/1.1 201     4.57 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.70 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/bookings

* 이후 이러한 패턴이 계속 반복되면서 시스템은 도미노 현상이나 자원 소모의 폭주 없이 잘 운영됨


HTTP/1.1 500     4.76 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.23 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.76 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.74 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.82 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.82 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.84 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.66 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     5.03 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.22 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.19 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.18 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     5.13 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.84 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.80 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.87 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.33 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.86 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.96 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.34 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 500     4.04 secs:     248 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.50 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.95 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.54 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/bookings


:
:

Transactions:		        1025 hits
Availability:		       63.55 %
Elapsed time:		       59.78 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
Successful transactions:        1025
Failed transactions:	         588
Longest transaction:	        9.20
Shortest transaction:	        0.00

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy pay --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/booings POST {"bookingId": 1}' 
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
pay     1         1         1            1           24s
pay     1         2         1            1           32s
pay     1         4         1            1           1m
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:		        5023 hits
Availability:		       93.45 %
Elapsed time:		       120 secs
Data transferred:	        0.32 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
```


## 무정지 재배포
* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/booings POST {"bookingId": 1}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/bookings
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/bookings
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3022 hits
Availability:		       71.15 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.63 secs
Transaction rate:	       17.12 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       95.01

```
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe/Liveness Probe 를 설정함

```
# deployment.yaml 의 readiness probe/liveness probe 설정:

 readinessProbe:
    httpGet:
      path: '/actuator/health'
      port: 8080
    initialDelaySeconds: 10
    timeoutSeconds: 2
    periodSeconds: 5
    failureThreshold: 10
 livenessProbe:
    httpGet:
      path: '/actuator/health'
      port: 8080
    initialDelaySeconds: 120
    timeoutSeconds: 2
    periodSeconds: 5
    failureThreshold: 5
```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


