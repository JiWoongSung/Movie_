# movie 

# 예제 - 영화 예매 시스템 

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 영화예매](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)


# 서비스 시나리오

기능적 요구사항
1. 재고 관리자가 도서 입고 처리를 한다.
2. 재고 입고 시 입고 수량 만큼 재고가 증가한다.
3. 고객이 도서 입고 리스트를 보고 도서 예약 신청을 한다.
4. 도서 예약은 1권으로 제약한다.
5. 도서 재고가 있으면 예약에 성공한다.
6. 도서 재고가 없으면 예약에 실패한다.
7. 예약 성공 시 재고는 1 차감된다.
8. 고객이 도서 예약을 취소한다.
9. 예약이 취소되면 재고가 1 증가한다.
10. 예약이 성공, 취소하면 카톡 등으로 알람을 보낸다.

비기능적 요구사항
1. 트랜잭션
    1. 모든 트랜잭션은 비동기 식으로 구성한다.
2. 장애격리
    1. 재고 관리 기능이 수행되지 않더라도 도서 예약은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
3. 성능
    1. 고객이 재고관리에서 확인할 수 있는 도서 재고를 예약(프론트엔드)에서 확인할 수 있어야 한다. CQRS
    1. 예약이 완료되면 카톡 등으로 알림을 줄 수 있어야 한다. Event driven



# 분석/설계
## AS-IS 조직 (Horizontally-Aligned)
<img width="800" alt="스크린샷 2020-09-01 오전 10 32 18" src="https://user-images.githubusercontent.com/26249603/91784855-db530500-ec3e-11ea-8199-e256d0760134.png">



## TO-BE 조직 (Vertically-Aligned)
<img width="800" alt="스크린샷 2020-09-01 오전 10 56 26" src="https://user-images.githubusercontent.com/26249603/91786097-d2176780-ec41-11ea-91ee-89c2ba77784d.png">


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: 
링크 추후 첨부

### 이벤트 도출
<img width="800" alt="스크린샷 2020-09-01 오전 11 48 15" src="https://user-images.githubusercontent.com/26249603/91789213-2540e880-ec49-11ea-90e1-2099d742161b.png">

### 액터, 커맨드 부착하여 읽기 좋게
<img width="800" alt="스크린샷 2020-09-01 오전 11 48 20" src="https://user-images.githubusercontent.com/26249603/91789218-27a34280-ec49-11ea-9f83-d22ef7ece82b.png">

### 어그리게잇으로 묶기
<img width="800" alt="스크린샷 2020-09-01 오전 11 48 27" src="https://user-images.githubusercontent.com/26249603/91789224-2d008d00-ec49-11ea-8664-f4b68eca0dd2.png">
    
### 바운디드 컨텍스트로 묶기
<img width="800" alt="스크린샷 2020-09-01 오전 11 48 33" src="https://user-images.githubusercontent.com/26249603/91789235-312caa80-ec49-11ea-8637-4869da5e2cd9.png">

    - 도메인 서열 분리 
        - Core Domain: 예약관리(front), 재고관리 : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준은 예약관리 99.999% / 재고관리 90% 목표, 배포주기는 예약관리 1주일 1회 미만/ 고객관리 2주 1회 미만으로 함
        - Supporting Domain:  고객관리 : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        
### 폴리시 부착 
<img width="800" alt="스크린샷 2020-09-01 오전 11 48 33" src="https://user-images.githubusercontent.com/26249603/91789235-312caa80-ec49-11ea-8637-4869da5e2cd9.png">

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Res) 


### 1차 완성된 모형


### 완성된 모형

![image](https://user-images.githubusercontent.com/63623995/81639169-2b227c00-9456-11ea-8e93-3a30d4344660.png)



### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

1. 재고 관리자가 도서 입고 처리를 한다. (ok)
2. 고객이 도서 입고 리스트를 보고 도서 예약 신청을 한다.(View 추가로 ok)
3. 도서 재고가 있으면 예약에 성공한다.(ok)
4. 도서 재고가 없으면 예약에 실패한다.(ok)
5. 고객이 도서 예약을 취소한다.(ok)
6. 예약이 성공되면 카톡 등으로 알람을 보낸다.(ok)

--> 완성된 모델은 모든 기능 요구사항을 커버함.

### 비기능 요구사항에 대한 검증

 - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
  : 모든 inter-microservice 트랜잭션이 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.
    

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/63623995/81639426-e0553400-9456-11ea-8346-be1d2d681305.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 표현
    - 서브 도메인과 바운디드 컨텍스트의 분리: 각 팀의 KPI 별로 아래와 같이 관심 구현 스토리지를 나눠가짐



# 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8083 이다)

```
cd bookreservation
mvn spring-boot:run

cd stockmanagement
mvn spring-boot:run 

cd customermanagement
mvn spring-boot:run  

```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 stock 마이크로 서비스). 

```
package bookrental;

import javax.persistence.*;

import bookrental.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.util.MimeTypeUtils;

@Entity
@Table(name="Stock_table")
public class Stock {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String bookid;
    private Long qty;
    private String status;

    @PostPersist
    public void onPostPersist(){
        Incomed incomed = new Incomed();
        incomed.setId(this.getId());
        incomed.setBookid(this.getBookid());
        incomed.setQty(this.getQty());
        ObjectMapper objectMapper = new ObjectMapper();
        String json = null;

        try {
            json = objectMapper.writeValueAsString(incomed);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON format exception", e);
        }

        KafkaProcessor processor = Application.applicationContext.getBean(KafkaProcessor.class);
        MessageChannel outputChannel = processor.outboundTopic();
        outputChannel.send(MessageBuilder
                .withPayload(json)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());
    }

    @PostUpdate
    public void onPostUpdate(){
        String status = this.getStatus();
        
        if(status.equals("revSucceeded")){
            Revsuccessed revsuccessed = new Revsuccessed();
            revsuccessed.setId(this.getId());
            revsuccessed.setBookid(this.getBookid());
            ObjectMapper objectMapper = new ObjectMapper();
            String json = null;

            try {
                json = objectMapper.writeValueAsString(revsuccessed);
            } catch (JsonProcessingException e) {
                throw new RuntimeException("JSON format exception", e);
            }

            KafkaProcessor processor = Application.applicationContext.getBean(KafkaProcessor.class);
            MessageChannel outputChannel = processor.outboundTopic();
            outputChannel.send(MessageBuilder
                    .withPayload(json)
                    .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                    .build());

        } else if(status.equals("revFailed")){
            Revfailed revfailed = new Revfailed();
            revfailed.setId(this.getId());
            revfailed.setBookid(this.getBookid());
            ObjectMapper objectMapper = new ObjectMapper();
            String json = null;

            try {
                json = objectMapper.writeValueAsString(revfailed);
            } catch (JsonProcessingException e) {
                throw new RuntimeException("JSON format exception", e);
            }

            KafkaProcessor processor = Application.applicationContext.getBean(KafkaProcessor.class);
            MessageChannel outputChannel = processor.outboundTopic();
            outputChannel.send(MessageBuilder
                    .withPayload(json)
                    .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                    .build());
        } else if(status.equals("revCanceled")) {
            Revcanceled revcanceled = new Revcanceled();
            revcanceled.setId(this.getId());
            revcanceled.setBookid(this.getBookid());
            ObjectMapper objectMapper = new ObjectMapper();
            String json = null;

            try {
                json = objectMapper.writeValueAsString(revcanceled);
            } catch (JsonProcessingException e) {
                throw new RuntimeException("JSON format exception", e);
            }
            
            KafkaProcessor processor = Application.applicationContext.getBean(KafkaProcessor.class);
            MessageChannel outputChannel = processor.outboundTopic();
            outputChannel.send(MessageBuilder
                    .withPayload(json)
                    .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                    .build());
        }
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getBookid() {
        return bookid;
    }

    public void setBookid(String bookid) {
        this.bookid = bookid;
    }
    public Long getQty() {
        return qty;
    }

    public void setQty(Long qty) {
        this.qty = qty;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package bookrental;

import org.springframework.data.repository.CrudRepository;
import java.util.Optional;

public interface StockRepository extends CrudRepository<Stock, Long>{
    Optional<Stock> findByBookid(String BookId);    # bookid로 찾기 위해 선언
}
```
- 적용 후 REST API 의 테스트
```
성공 케이스
// 재고 서비스 입고
1. http POST localhost:8082/stocks bookid="1" qty=3

// 예약 서비스에서 입고된 책 예약
2. http POST localhost:8081/reservations bookid="1" userid="test1"

// 예약 서비스에서 고객의 예약 상태가 '성공'임을 확인
3. http GET localhost:8081/reservations/*
```
![image](https://user-images.githubusercontent.com/63623995/81771217-5de37780-951d-11ea-9918-0d6b6dc30531.png)
```
// 재고 서비스에서 고객이 예약한 책의 재고 감소를 확인
4. http GET localhost:8082/stocks/*
```
![image](https://user-images.githubusercontent.com/63623995/81771292-8c615280-951d-11ea-8d2a-587ece40b771.png)
```
// 고객 서비스 콘솔을 통해 고객의 예약이 정상적으로 완료 되었는지 확인
```
![image](https://user-images.githubusercontent.com/63623995/81771166-3be9f500-951d-11ea-87bb-43f9f39b26d4.png)
```

실패 케이스
// 재고 서비스 입고 (단, 테스트를 위한 수량은 '0')
1. http POST localhost:8082/stocks bookid="2" qty=0

// 예약 서비스에서 입고된 책 예약
2. http POST localhost:8081/reservations userid="user" bookid="2"

// 예약 서비스에서 고객의 예약 상태가 '실패'임을 확인
3. http GET localhost:8081/reservations/*

취소 케이스
// 재고 서비스 입고
1. http POST localhost:8082/stocks bookid="1" qty=3

// 예약 서비스에서 입고된 책 예약
2. http POST localhost:8081/reservations bookid="1" userid="test1"

// 재고 서비스에서 고객이 예약한 책의 재고 감소를 확인
3. http GET localhost:8082/stocks/*
```
![image](https://user-images.githubusercontent.com/63623995/81771292-8c615280-951d-11ea-8d2a-587ece40b771.png)
```
// 예약 서비스에서 예약한 책을 취소 요청함
4. http PATCH localhost:8081/reservations/* status="Canceled"
```
![image](https://user-images.githubusercontent.com/63623995/81771410-d5b1a200-951d-11ea-85d5-c6a81bcba0fd.png)
```
// 예약 서비스에서 고객의 예약 상태가 '취소'임을 확인
5. http GET localhost:8081/reservations/*
```
![image](https://user-images.githubusercontent.com/63623995/81771432-ec57f900-951d-11ea-8940-38e24c625ee6.png)
```
// 재고 서비스에서 고객이 취소 요청한 책의 수량이 증가하였는지 확인한다.
6. http GET localhost:8082/stocks/*
```
![image](https://user-images.githubusercontent.com/63623995/81772095-db0fec00-951f-11ea-8c75-717ebcefed41.png)
```
// 고객 서비스 콘솔을 통해 고객의 예약이 정상적으로 취소 되었는지 확인
```
![image](https://user-images.githubusercontent.com/63623995/81772123-faa71480-951f-11ea-9d81-03253d199c3d.png)



## 퍼시스턴스
앱프런트(app) 는 H2DB를 활용하였으며 VO 선언시 @Entity 마킹되었음, 기존의 Entity Pattern 과 Repository Pattern 적용이 가능하도록 구현하였다.

```
# Reservation.java

@Entity
@Table(name="Reservation_table")
public class Reservation {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String userid;
    private String bookid;
    private String status;
    ..


# ReservationRepository.java

public interface ReservationRepository extends CrudRepository<Reservation, Long> {
    Optional<Reservation> findBybookid(String BookId);
}
```

## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


도서예약이 이루어진 후에 고객관리시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 고객관리시스템의 처리를 위하여 도서예약이 블로킹 되지 않도록 처리한다.
 
- 이를 위하여 도서예약 기록을 남긴 후에 곧바로 예약 요청이 되었다는 도메인 이벤트를 카프카로 송출한다
 
```
package bookrental;

@Entity
@Table(name="Reservation_table")
public class Reservation {

 ...
    @PostPersist
    public void onPostPersist(){
        Requested requested = new Requested();
        requested.setOrderId(this.getId());
        requested.setUserid(this.getUserid());
        requested.setBookid(this.getBookid());
        ObjectMapper objectMapper = new ObjectMapper();
        String json = null;

        try {
            json = objectMapper.writeValueAsString(requested);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON format exception", e);
        }

        KafkaProcessor processor = Application.applicationContext.getBean(KafkaProcessor.class);
        MessageChannel outputChannel = processor.outboundTopic();

        outputChannel.send(MessageBuilder
                .withPayload(json)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build());
    }

}
```
- 재고관리 서비스에서는 도서예약 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package bookrental;

...

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRequested_Checkstock(@Payload Requested requested){

        if(requested.isMe()) {
            stockRepository.findByBookid(requested.getBookid())
                    .ifPresent(
                            stock -> {
                                long qty = stock.getQty();

                                if (qty >= 1) {   # 재고량이 1개 이상 있으면
                                    stock.setQty(qty - 1);  # 재고 차감 후 
                                    stock.setStatus("revSucceeded");  # 예약완료 status set
                                    stockRepository.save(stock);
                                    System.out.println("set Stock -1");
                                } else {
                                    stock.setStatus("revFailed");   # 예약실패 status set
                                    stockRepository.save(stock);
                                    System.out.println("stock-out");
                                }
                            }
                    )
            ;
        }
    }

}

```
이후, 재고 차감에 성공하고 예약이 완료되면 카톡 등으로 고객에게 카톡 등으로 알람을 보낸다. 
  
```
    @Service
    public class PolicyHandler{

        @StreamListener(KafkaProcessor.INPUT)
        public void wheneverReserved_Orderresultalarm(@Payload Reserved reserved){

            if(reserved.isMe()){
                System.out.println("##### 예약 완료 되었습니다  #####" );//+ reserved.toJson());
            }

        }
    }

```

도서예약 시스템은 고객관리(알람) 시스템와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 고객관리 시스템이 유지보수로 인해 잠시 내려간 상태라도 예약을 받는데 문제가 없다:
```
# 고객관리 서비스 (customer) 를 잠시 내려놓음 (ctrl+c)

# 재고 서비스 : 입고
1. http POST localhost:8082/stocks bookid="1" qty=3

# 예약 서비스 : 입고된 책 예약
2. http POST localhost:8081/reservations bookid="1" userid="test1"    #Success

# 예약 서비스 : 고객의 예약 상태가 '성공'임을 확인
3. http GET localhost:8081/reservations/*
```
![image](https://user-images.githubusercontent.com/63623995/81771217-5de37780-951d-11ea-9918-0d6b6dc30531.png)
```
# 예약 완료 알람이 오지 않음

# 고객관리 서비스 기동
4. cd customermanagement
mvn spring-boot:run

# 고객관리 서비스 : 알람 확인
5. "##### 예약 완료 되었습니다  #####"
```
![image](https://user-images.githubusercontent.com/63623995/81771166-3be9f500-951d-11ea-87bb-43f9f39b26d4.png)



# 운영

## CI/CD 설정

각 구현체들은 각자의 Git source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였으며, pipeline build script 는 각 서비스 프로젝트 폴더 이하에 azure-pipelines.yml 에 포함되었다.


### 설정
- Pipeline 생성 (Reservation, Stockmanagement, Customermanagement)
![pipe](https://user-images.githubusercontent.com/63623995/81764531-c0804780-950c-11ea-9553-402bba09c0ca.png)

- Pipeline 설정(모두 동일)
![pipe2](https://user-images.githubusercontent.com/63623995/81764550-c9711900-950c-11ea-8210-030511bde916.png)

- Release 생성
![release](https://user-images.githubusercontent.com/63623995/81764553-cbd37300-950c-11ea-94f7-892dc0b68ddc.png)

- Release 설정(모두 동일)
![release2](https://user-images.githubusercontent.com/63623995/81764558-cd04a000-950c-11ea-91a4-018f0c2a3b87.png)
![release3](https://user-images.githubusercontent.com/63623995/81764561-ce35cd00-950c-11ea-9052-22c52c21e584.png)



## Auto scaling 테스트

■ 목표: 
부하가 과다하게 발생 할 경우, POD가 자동으로 증설됨

■ 테스트 절차: 
서비스 처리에 대한 부하를 과다하게 유발 시켜 POD Auto-scaling 적용
(로직에는 sleep 30적용을 통해 Java Thread 수행 시간이 오래 걸림)

가. 부하 발생 전

![image](https://user-images.githubusercontent.com/63623995/81773542-75256380-9523-11ea-8701-f64d2d1be91b.png)

■ 결과: 
Stress 테스트 통해 시스템 Capacity가 초과되는 Request 요청 시 HPA에 의해 POD자동 증가

![image](https://user-images.githubusercontent.com/63623995/81773629-b453b480-9523-11ea-8b47-3c52d3ed1b17.png)

■ 참고 자료: HPA설정 내역

![image](https://user-images.githubusercontent.com/63623995/81773688-d77e6400-9523-11ea-8ee0-3e7ff0646373.png)



## Auto-Healing 테스트

■ 목표: 
POD에 예상치 못한 장애가 발생 할 경우, POD가 자동으로 재부팅됨.

■ 테스트 절차
과도하게 오래 실행되는 로직을 다수 수행할 경우, WAS의 Java Thread 부족으로 인해 Hang 발생

■ 테스트 명령어

while [ true ]; do http POST bookstore.skcc.co.kr/reservations userid="user" bookid="1" status="selfHealingTest" & done

■ 결과: 
POD에 Hang 발생 시 liveness Probe 설정에 의해 자동으로 POD재부팅 됨.

![image](https://user-images.githubusercontent.com/63623995/81774269-211b7e80-9525-11ea-8c2a-cf5f4cf9df96.png)

■ 참고 자료: Probe 설정 내역

![image](https://user-images.githubusercontent.com/63623995/81774336-44462e00-9525-11ea-907b-c4a3132db38c.png)

## ConfigMap / Secret 적용

■ 목표: 
환경 변수 적용을 통해 용도 별 구분 인자값으로 활용

■ 결과: 
CnfigMap: 개발기/운영기 구분자료 활용
kubectl exec -ti reservations2-6dd99458f8-ncfng -n bookstore env | egrep EMBED_TOMCAT_JAVA_OPTS
EMBED_TOMCAT_JAVA_OPTS=-client

Secret: DB접속 정보 저장 목적으로 활용
kubectl exec -ti reservations2-6dd99458f8-ncfng -n bookstore env | egrep "DB_USER|DB_PASS"
DB_USER=myuser
DB_PASS=#skcc123

■ 참고 자료: ConfigMap/Secret 설정 내역

![image](https://user-images.githubusercontent.com/63623995/81774137-dac61f80-9524-11ea-8735-d26a99fe9aa2.png)
		

## Monitoring

■ 목표:

쿠버네티스 서비스 모니터링 구성
POD의 상태가 비정상이거나, Node의 CPU/Memory 사용량이 비정상적으로 과도할 경우, 운영자에게 SMS으로 Alert함.

■ 구성 내역: 

Azure Monitor를 통해 CPU, Memory, POD등이 임계치를 초과 할 경우 SMS 발송 되도록 구성 함.
![image](https://user-images.githubusercontent.com/63623995/81772858-e5cb8080-9521-11ea-8b46-59770e0ed5c0.png)


## Alerting

■ 목표: 

쿠버네티스 서비스 모니터링 및 Alerting 구성
POD의 상태가 비정상이거나, Node의 CPU/Memory 사용량이 비정상적으로 과도할 경우, 운영자에게 SMS으로 Alert함.

■ 구성 내역: 

Azure Monitor를 통해 CPU, Memory, POD등이 임계치를 초과 할 경우 SMS 발송 되도록 구성 함.
![image](https://user-images.githubusercontent.com/63623995/81772923-0e537a80-9522-11ea-9074-a2b8d713caa1.png)


## Persistent Volume

■ 목표: 

Persistent Volume 마운트를 통해 영구 데이터 저장
해당 스토리지는 첨부 파일들을 저장하는데 사용 가능하며, Dynamic으로 유동성있게 구성함.

■ 구성 결과: 

root@mypod-azurefiles:/# df -h | grep azure
//faee688dbf4284f62a1a567.file.core.windows.net/kubernetes-dynamic-pvc-d402c327-d428-4eb8-9f0d-072314cc6555  5.0G     0  5.0G   0% /mnt/azure

■ 설정 내역: 

[Storage Class]
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=0
  - gid=0
  - mfsymlinks
  - cache=strict
parameters:
  skuName: Standard_LRS

[PVC]
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azurefile
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile
  resources:
    requests:
      storage: 5Gi
[POD]
kind: Pod
apiVersion: v1
metadata:
  name: mypod-azurefiles
spec:
  containers:
  - name: mypod
    image: nginx:1.15.5
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi
    volumeMounts:
    - mountPath: "/mnt/azure"
      name: volume
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: bookstore-azurefile

