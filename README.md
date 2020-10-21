# Table of contents

- [렌트카 서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오



기능적 요구사항
1.	관리자는 차량을 등록할 수 있다.
2.	고객이 차량을 선택해 주문하면 결제가 진행된다.
3.	주문이 결제되면 차량의 가능 여부가 확정된다.
4.	고객은 주문 및 결제를 취소할 수 있다.
5.	주문이 취소되면 결제가 취소되고 차량의 주문 가능 여부가 변경된다.
6.	고객은 렌트 가능 차량의 수를 확인할 수 있다



비기능적 요구사항
1. 트랜잭션
    1. 결제가 완료 되지 않은 주문 건은 주문이 성립되지 않는다.  Sync 호출
    1. 주문과 결제는 동시에 진행된다.  Sync 호출
1. 장애격리
    1. 렌트 시스템이 수행되지 않더라도 주문 / 결제는365일 24시간 받을 수 있어야 한다.  Async 호출 (event-driven)
    1. 렌트 시스템이 과중 되면 주문 / 결제를 받지 않고 결제 취소를 잠시 후에 하도록 유도한다.  Circuit breaker, fallback
1. 성능
    1. 고객은 렌트 잔여 수량을 확인할 수 있다.  CQRS
    1. 주문/결제 취소 정보가 변경 될 때마다 렌트 가능 여부가 변경 될 수 있어야 한다.  Event Driven





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
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/NMhho4mGA8eu4HsduIIw1ujbKLn2/every/68247b3541f0c829d6f4f224a2765bf2/-MJp6OmjlMbSMJL-_PuI


### 이벤트 도출
![1](https://user-images.githubusercontent.com/64885343/96411189-08756a00-1223-11eb-9927-cf066cf89ea2.png)



### 어그리게잇으로 묶기 / 액터, 커맨드 부착하여 읽기 좋게
![2](https://user-images.githubusercontent.com/64885343/96411192-090e0080-1223-11eb-995a-11b2656a5bc5.png)

    - 차량렌트 주문, 결제, 관리 등은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어준다.

### 바운디드 컨텍스트로 묶기

![3](https://user-images.githubusercontent.com/64885343/96411193-09a69700-1223-11eb-9a23-54add65b93af.png)

    - 도메인 서열 분리 
    	- 주문: 고객 주문 오류를 최소화 한다. (Core)
    	- 결제 : 결제 오류를 최소화 한다. (Supporting)
    -	 차량 : 차량 상태 오류를 최소화 한다. (Supporting)


### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![4](https://user-images.githubusercontent.com/64885343/96411194-0a3f2d80-1223-11eb-847d-99ae092070de.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![5](https://user-images.githubusercontent.com/64885343/96411196-0a3f2d80-1223-11eb-8b10-e89c5a6f0113.png)

### 완성된 1차 모형

![6](https://user-images.githubusercontent.com/64885343/96411197-0ad7c400-1223-11eb-9b60-40ee68bd4c98.png)



### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![7](https://user-images.githubusercontent.com/64885343/96411199-0ad7c400-1223-11eb-9dd6-c2c3b0fc9423.png)

	- 고객이 차량를 선택해 주문을 진행한다. (OK)
	- 주문 시 자동으로 결제가 진행된다. (OK)
	- 결제가 성공하면 차량은 예약불가 상태가 된다. (OK)
	- 차량이 렌트가 되면 보유 차량의 수량이 변경된다. (OK)
   
    






![8](https://user-images.githubusercontent.com/64885343/96411203-0b705a80-1223-11eb-9294-e9f24479a7cc.png)


	- 고객이 예약/결제를 취소한다. (OK)
	- 예약/결제 취소 시 자동 예약/결제 취소된다. (OK)
	- 결제가 취소되면 보유 차량 수량이 증가한다. (OK)







### 모델 수정

![9](https://user-images.githubusercontent.com/64885343/96411206-0b705a80-1223-11eb-9738-7c419322dfcc.png)
    
    - View Model 추가
    - 수정된 모델은 모든 요구사항을 커버함.


### 비기능 요구사항에 대한 검증

![9](https://user-images.githubusercontent.com/64885343/96411206-0b705a80-1223-11eb-9738-7c419322dfcc.png)


	- 차량 등록 서비스를 주문/결제 서비스와 격리하여 차량 등록 서비스 장애 시에도 예약이 가능
	- 차량 주문 불가 상태일 경우 예약 확정이 불가함
   









## 헥사고날 아키텍처 다이어그램 도출
    
![화면 캡처 2020-10-19 162339](https://user-images.githubusercontent.com/64885343/96414049-8b98bf00-1227-11eb-910d-202937f223f1.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd order
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd rentcar
mvn spring-boot:run  

cd system
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. 

```
package carRent;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Pay_table")
public class Pay {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long carId;
    private Long orderId;
    private String stauts;
    private Integer qty;

    .../... 중략  .../...
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getCarId() {
        return carId;
    }

    public void setCarId(Long carId) {
        this.carId = carId;
    }
    .../... 중략  .../...
    public void setQty(Integer qty) {
        this.qty = qty;
    }
}

```

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록    
데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용
```
package carRent;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface PayRepository extends PagingAndSortingRepository<Pay, Long>{


}
```
   
   
---
#### 적용 후 REST API 의 테스트

1. 차량 등록

![차량 등록 캡쳐](https://user-images.githubusercontent.com/64885343/96722043-e711ab00-13e7-11eb-8b81-7b9871cc0235.png)


2. 차량 예약 

![차량 예약 캡쳐](https://user-images.githubusercontent.com/64885343/96722073-ee38b900-13e7-11eb-9ba4-378785482fe5.png)


3. 차량 보기

![조회](https://user-images.githubusercontent.com/64885343/96722082-f09b1300-13e7-11eb-861f-bf623eed244f.png)


---


## 폴리글랏 퍼시스턴스

각 마이크로서비스는 별도의 H2 DB를 가지고 있으며 CQRS를 위한 system에서는 H2가 아닌 HSQLDB를 적용하였다.

```
# system의 pom.xml에 dependency 추가
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>


```





## 동기식 호출과 Fallback 처리
Order → Payment 간 호출은 동기식 일관성 유지하는 트랜잭션으로 처리.     
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출.     

```
package carRent;
import carRent.config.kafka.KafkaProcessor;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.openfeign.EnableFeignClients;


@SpringBootApplication
@EnableBinding(KafkaProcessor.class)
@EnableFeignClients
public class OrderApplication {
    protected static ApplicationContext applicationContext;
    public static void main(String[] args) {
        applicationContext = SpringApplication.run(OrderApplication.class, args);
    }
}
```

FeignClient 방식을 통해서 Request-Response 처리.     
Feign 방식은 넷플릭스에서 만든 Http Client로 Http call을 할 때, 도메인의 변화를 최소화 하기 위하여 interface 로 구현체를 추상화.    
→ 실제 Request/Response 에러 시 Fegin Error 나는 것 확인   




- 예약 받은 직후(@PostPersist) 결제 요청함
```
-- Order.java
        @PrePersist
    public void onPrePersist(){

        try {
            Thread.currentThread().sleep((long) (800 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        carRent.external.Pay pay = new carRent.external.Pay();
        pay.setOrderId(ordered.getId());
        pay.setQty(ordered.getQty());
        pay.setCarId(ordered.getCarId());
        System.out.println("##### 오더아이디 어디감 : " + ordered.getId());
        pay.setStauts("APPROVED");

        // mappings goes here
        OrderApplication.applicationContext.getBean(carRent.external.PayService.class)
            .payreq(pay);


    }
```



- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인함.   
```
Order -- (http request/response) --> Payment

# Payment 서비스 종료

# Order 등록
http http://localhost:8081/orders id=1 status=ORDERED carId=1 orderId=1     #Fail!!!!
```
Payment를 종료한 시점에서 상기 Car 등록 Script 실행 시, 500 Error 발생.
("Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction")   


---
## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

Payment가 이루어진 후에(PAID) Car시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리.   
Car 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리.   
이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish).   

- Car 서비스에서는 PAID 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:   
```

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @Autowired
    CarRepository carRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Rented(@Payload Paid paid){

        if(paid.isMe()){
            System.out.println("##### listener Rented : " + paid.toJson());


            Optional<Car> carOptional = carRepository.findById(paid.getCarId());

            Car car = carOptional.get();
            car.setCnt(car.getCnt()-paid.getQty());


            carRepository.save(car);
        }
    }
```

- Car 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, Car 시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# Car Service 를 잠시 내려놓음 (ctrl+c)

#PAID 처리
http http://localhost:8082/payments id=1 status=PAID carId=1 orderId=1          #Success!!

#결제상태 확인
http http://localhost:8082/payments  #제대로 Data 들어옴   

#House 서비스 기동
cd rentcar
mvn spring-boot:run

#Car 상태 확인
http http://localhost:8083/cars     # 제대로 kafka로 부터 data 수신 함을 확인
```


---



# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.

![CICD codebuild](https://user-images.githubusercontent.com/64885343/96725086-7cfb0500-13eb-11eb-96ed-a3fed1713848.png)

Webhook으로 연결되어 github에서 수정 시 혹은 codebuild에서 곧바로 빌드가 가능하다.

![포CICD 구성 빌드기록](https://user-images.githubusercontent.com/64885343/96725096-7f5d5f00-13eb-11eb-8f2a-46be3a094151.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient 을 사용하여 구현함

시나리오는 order -> payment 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.


- 피호출 서비스(결제:payment) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# Pay.java (Entity)
    @PostPersist
    public void onPostPersist(){

        try {
            Thread.currentThread().sleep((long) (800 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();
    }
```

일단 서킷브레이커 미적용 시, 모든 요청이 성공했음을 확인

![부하(100%)](https://user-images.githubusercontent.com/64885343/96725646-06aad280-13ec-11eb-8a99-8c22caf3a33d.png)


데스티네이션 룰 적용
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-payment
  namespace: istio-cb-ns
spec:
  host: skccuser26-payment
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1024           # 목적지로 가는 HTTP, TCP connection 최대 값. (Default 1024)
      http:
        http1MaxPendingRequests: 1  # 연결을 기다리는 request 수를 1개로 제한 (Default 
        maxRequestsPerConnection: 1 # keep alive 기능 disable
        maxRetries: 3               # 기다리는 동안 최대 재시도 수(Default 1024)
    outlierDetection:
      consecutiveErrors: 5          # 5xx 에러가 5번 발생하면
      interval: 5s                  # 5초마다 스캔 하여
      baseEjectionTime: 30s         # 30 초 동안 circuit breaking 처리   
      maxEjectionPercent: 10       # 10% 로 차단
EOF
```
적용 후 부하테스트 시 서킷브레이커의 동작으로 미연결된 결과가 보임
- 동시사용자 20명
- 20초 동안 실시
```
siege -c3 -t20S -v  --content-type "application/json" 'http://skccuser26-payment:8080/pays POST {"id":"1","carId":"1","orderId":"1","status":"ORDERD","qty":"10"}'
```

![image](https://user-images.githubusercontent.com/70302894/96579639-1f46ba00-1312-11eb-8b13-1c552b108711.JPG)

78%정도의 요청 성공률 확인

![image](https://user-images.githubusercontent.com/70302894/96579632-1ce46000-1312-11eb-8c7a-8aff5c351056.JPG)

키알리 화면 캡쳐
![image](https://user-images.githubusercontent.com/70302894/96579638-1eae2380-1312-11eb-8b9e-c3e21aec7d75.JPG)

예거 화면 캡쳐
![예거](https://user-images.githubusercontent.com/70302894/96673034-8743e180-13a0-11eb-9617-d9ab5149590f.JPG)



- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 더 과부하를 걸면 반 이상이 실패 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.


- Retry 의 설정 (istio)

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-order-network-rule
  namespace: istio-cb-ns
spec:
  hosts:
  - book
  http:
  - route:
    - destination:
        host: payment
    timeout: 3s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,retriable-4xx,gateway-error,connect-failure,refused-stream
EOF
```


- Availability 가 높아진 것을 확인 (siege)





### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 10프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy skccuser04-payment --min=1 --max=10 --cpu-percent=10 -n istio-cb-ns

kubectl autoscale deployment.apps/skccuser04-payment --cpu-percent=10 --min=1 --max=10 -n istio-cb-ns
```

오토스케일을 위한 metrics-server를 설치하고 배포한다. 적용한 istrio injection을 해제한다.
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml

kubectl get deployment metrics-server -n kube-system

kubectl label namespace istio-cb-ns istio-injection=disabled --overwrite

```
- CB 에서 했던 방식대로 워크로드를 120초 동안 걸어준다.
```
siege -c20 -t120S -v  --content-type "application/json" 'http://skccuser04-payment:8080/payments POST {"id":"1","houseId":"1","bookId":"1","status":"BOOKED"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
watch kubectl get deploy skccuser04-payment -n istio-cb-ns
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![image](https://user-images.githubusercontent.com/70302894/96588282-6d61ba80-131e-11eb-8f75-90e10ef6f203.JPG)

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

![오토스케일링 결과](https://user-images.githubusercontent.com/70302894/96664104-07604c00-138d-11eb-9fed-9c7bd56bf879.JPG)



## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c20 -t20S -v  --content-type "application/json" 'http://skccuser04-payment:8080/payments POST {"id":"1","houseId":"1","bookId":"1","status":"BOOKED"}'
```

- 코드빌드에서 재빌드 

![image](https://user-images.githubusercontent.com/70302894/96588975-3dff7d80-131f-11eb-9018-6527b1907591.JPG)



- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

<img width="594" alt="무정지" src="https://user-images.githubusercontent.com/7261288/96663462-abe18e80-138b-11eb-8a5d-59d11ac07491.png">

배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 와 liveness Prove 설정을 다시 추가:

```
# deployment.yaml 에 readiness probe / liveness prove  설정:
      containers:
        - name: payment
          image: username/payment:latest
          ports:
            - containerPort: 8080
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
- git commit 이후 자동배포 시 siege 돌리고 Availability 확인:

![image](https://user-images.githubusercontent.com/70302894/96663164-1b0ab300-138b-11eb-9286-94a73c09cff4.JPG)


배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

또한 Liveness Probe가 적용되어있어 kubectl get all -n istio-cb-ns에서 확인 시 자동으로 Restart 됨 (하단이미지 Restart 횟수 확인가능)

![라이브네스 전](https://user-images.githubusercontent.com/70302894/96665219-5c9d5d00-138f-11eb-8c62-ad9ade0bc248.JPG)


# configmap
house 서비스의 경우, 국가와 지역에 따라 설정이 변할 수도 있음을 가정할 수 있다.   
configmap에 설정된 국가와 지역 설정을 house 서비스에서 받아 사용 할 수 있도록 한다.   
   
아래와 같이 configmap을 생성한다.   
data 필드에 보면 country와 region정보가 설정 되어있다. 
##### configmap 생성
```
kubectl apply -f - <<EOF 
apiVersion: v1
kind: ConfigMap
metadata:
  name: house-region
  namespace: istio-cb-ns
data:
  country: "korea"
  region: "seoul"
EOF
```
 
house deployment를 위에서 생성한 house-region(cm)의 값을 사용 할 수 있도록 수정한다.
###### configmap내용을 deployment에 적용 
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: house
  labels:
    app: house
...
    spec:
      containers:
        - name: house
          env:                                                 ##### 컨테이너에서 사용할 환경 변수 설정
            - name: COUNTRY
              valueFrom:
                configMapKeyRef:
                  name: house-region
                  key: country
            - name: REGION
              valueFrom:
                configMapKeyRef:
                  name: house-region
                  key: region
          volumeMounts:                                                 ##### CM볼륨을 바인딩
          - name: config
            mountPath: "/config"
            readOnly: true
...
      volumes:                                                 ##### CM 볼륨 
      - name: config
        configMap:
          name: house-region
```
configmap describe 시 확인 가능

![컨픽맵2](https://user-images.githubusercontent.com/70302894/96668946-37145180-1397-11eb-8465-0a34c4e271f4.JPG)

