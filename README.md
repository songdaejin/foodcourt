
# 개인 과제 수행 Project
![image](https://user-images.githubusercontent.com/88808280/133920732-a2d6a859-68d4-4588-a1fa-f595660c7daf.png)
## 푸트코트(food court) 주문    

# Table of contents

- [푸드코트 주문 시스템](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
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


# 서비스 시나리오

기능적 요구사항

A. 고객 주문(키오스크)
1. 고객 음식주문을 위해 품목을 조회할수 있다 
2. 고객은 식당, 음식 및 수량을 선택 후 주문할수 있다
3. 고객은 조리 접수 되지 않은 주문에 대해 취소 요청할수 있다

B. 매장 주문관리
1. 해당 매장으로 주문된 주문을 조회 할수 있다.
2. 조리전에 주문건에 대해 접수 처리 후 조리 시작한다
3. 조리가 완료된 주문건에 대해서는 주문완료 처리한다
4. 주문 완료 처리한 주문은 알림판을 통해 알림 처리된다.  
 
 C. 주문처리현황
 1. 주문건에 대한 주문, 조리 상태를 조회 할수 있다.  

비기능적 요구사항
1. 트랜잭션
    1. 주문취소를 위해 해당주문의 상태를 조회해야 한다.(조리시작된 주문건은 취소불가)  Sync 호출 
1. 장애격리
    1. 고객주문 기능이 수행되지 않더라도 주문관리 시스템은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 주문이 과중되면 사용자를 잠시동안 받지 않고 잠시후에 주문접수 되도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 매장별 주문 및 조리 상태 현황을 조회 할 수 있어야 한다  CQRS
    1. 주문에 대한 조리가 완료되면 알림판을 통해알림을 줄 수 있어야 한다  Event driven


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
![image](https://user-images.githubusercontent.com/88808280/134804087-ccad9c5e-de5c-4c55-b385-0a6cd92ed444.png)

## TO-BE 조직 (Vertically-Aligned)
![image](https://user-images.githubusercontent.com/88808280/134804111-b8010333-b3d4-4972-8cd7-579c123f4f23.png)

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  https://www.msaez.io/#/storming/CWH9U9ZXJmRVhWyyz88s1h12bLz1/0aedf2d2760e55f8b59356cf29773139


### 이벤트 도출
![image](https://user-images.githubusercontent.com/88808280/134804267-6404ff1a-cdfb-470e-8bc8-161708a14da6.png)

### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/88808280/134804307-87390d5a-0e29-40ae-9893-e5b8fcbf8571.png)


### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/88808280/134804489-b80c8641-bbed-44c7-935b-c9664ee8fae8.png)

### 어그리게잇
![image](https://user-images.githubusercontent.com/88808280/134804679-1d6f77d9-acba-422d-9c93-fc829db8491d.png)

### 바운디드 컨텍스트로 묶기
![image](https://user-images.githubusercontent.com/88808280/134809636-6c6deb6b-a589-4df8-9cbb-c81f1e6518be.png)

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)
![image](https://user-images.githubusercontent.com/88808280/134809764-fbdda5e8-95bb-4e1c-95a5-d44558f55055.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![image](https://user-images.githubusercontent.com/88808280/134809928-a8883dea-5108-4cfc-aec9-d5a68ee24547.png)

### 완성된 1차 모형
![image](https://user-images.githubusercontent.com/88808280/134810011-ac4bc771-b369-4fd6-b3e2-aa5fc6cff514.png)

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/88808280/134810728-a98e4569-48ea-4d92-baeb-2b25eb5a80e1.png)

1. 고객은 식당, 음식 및 수량을 선택 후 주문할수 있다
2. 고객은 조리 접수 되지 않은 주문에 대해 취소 요청할수 있다
3. 조리전에 주문건에 대해 접수 처리 후 조리 시작한다
4. 조리가 완료된 주문건에 대해서는 주문완료 처리한다
5. 주문 완료 처리한 주문은 알림판을 통해 알림 처리된다.
6. 주문건에 대한 주문, 조리 상태를 조회 할수 있다.  

## 헥사고날 아키텍처 다이어그램 도출
    
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리
    
![image](https://user-images.githubusercontent.com/88808280/134810874-57ca4cfe-5bf2-4600-9f53-ddd14d19fe89.png)


# 구현

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라,구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8086이다)

```shell
cd order
mvn spring-boot:run

cd cook
mvn spring-boot:run 

cd notification
mvn spring-boot:run 

cd shopaccount 
mvn spring-boot:run

cd gateway
mvn spring-boot:run 
```
## DDD(Domain-Driven-Design)의 적용
msaez Event-Storming을 통해 구현한 Aggregate 단위로 Entity 를 정의 하였으며,
Entity Pattern 과 Repository Pattern을 적용하기 위해 Spring Data REST 의 RestRepository 를 적용하였다.

foodcourt 서비스의 Order.java

```java

package foodcourt;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long orderId;
    private Long shopId;
    private Long menuId;
    private Long qty;
    private Long price;
    private String status;

    @PostPersist
    public void onPostPersist(){

        if("order".equals(this.status)){
            Ordered ordered = new Ordered();
            BeanUtils.copyProperties(this, ordered);
            ordered.publishAfterCommit();
        }
    }

    @PostUpdate
    public void onPreUpdate(){

         if("cancel".equals(this.status)){
            Canceled canceled = new Canceled();
            BeanUtils.copyProperties(this, canceled);
            canceled.publishAfterCommit();
        }
    }

    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }
    .. getter/setter Method 이하 생략
```


 RestController구현체 OrderController.java
 Order 처리 및 Order Cancel 처리한다.
```java
@RestController
 public class OrderController {


    @Autowired OrderRepository orderRepositroy;

    @PostMapping("orders/order")
    Order oderCreate(@RequestBody String data) {

        ObjectMapper mapper = new ObjectMapper();
        Order order = null;
        try {
             order = mapper.readValue(data, Order.class);
        } catch (IOException e) {
            e.printStackTrace();
        }

        return orderRepositroy.save(order);
    }

    @PostMapping("orders/cancel")
    Boolean oderCancel(@RequestBody String data) {
    
        ObjectMapper mapper = new ObjectMapper();
        Order orderInfo = null;
        try {
            orderInfo = mapper.readValue(data, Order.class);
        } catch (IOException e) {
            e.printStackTrace();
        }

        /* Cancel 가능한지 주문상태 조회 */
        String OrderStatus = OrderApplication.applicationContext.getBean(foodcourt.external.CookService.class)
        .orderCheck(orderInfo.getOrderId());

        if("order".equals(OrderStatus)){
            Optional<Order> orderOptional = orderRepositroy.findById(orderInfo.getOrderId());
            Order order = orderOptional.get();
            order.setStatus(orderInfo.getStatus());
            orderRepositroy.save(order);
            
            return true;
        }else{
            System.out.println("########### Order Cancel Failed - Reason_OerderStatus : "+ OrderStatus);
            
            return false;
        }


    }

 }

```

 Cook 서비스의 PolicyHandler.java


```java
@Service
public class PolicyHandler{
    @Autowired CookRepository cookRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrdered_OrderReceipt(@Payload Ordered ordered){

        if(!ordered.validate()) return;

        Cook cook = new Cook();
        cook.setShopId(ordered.getShopId());
        cook.setOrderId(ordered.getOrderId());
        cook.setMenuId(ordered.getMenuId());
        cook.setCookStatus(ordered.getStatus());
        
        cookRepository.save(cook);

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverCanceled_OrderCancel(@Payload Canceled canceled){

        if(!canceled.validate()) return;
        
        // 주문조회
        Long orderId = canceled.getOrderId();
        Optional<Cook> cookOptional = cookRepository.findByOrderId(orderId);
        Cook cook = cookOptional.get();
        
        // 주문 취소처리 
        cook.setCookStatus("cancel");
        cookRepository.save(cook);

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void whatever(@Payload String eventString){}
}
```

## 적용 후 REST API 의 테스트
각 서비스들의 Rest API 호출을 통하여 테스트를 수행하였음

```shell
음식 주문
http post http://localhost:8081/orders/order shopId=1 menuId=1 qty=1 price=1000 status=order

주문취소 
http post http://localhost:8081/orders/cancel orderId=1 status=cancel

조리 상태 변경 
. 조리시작: http  post  http://localhost:8082/cook/stausChange orderId=9 cookStatus=cook
. 조리완료: http  post  http://localhost:8082/cook/stausChange orderId=10 cookStatus=cooked

alarm 현황
http http://localhost:8083/alarms
```

## Gateway 적용
GateWay 구성를 통하여 각 서비스들의 진입점을 설정하여 라우팅 설정하였다.
```yaml
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: Order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/** 
        - id: Cook
          uri: http://localhost:8082
          predicates:
            - Path=/cooks/** 
        - id: Notification
          uri: http://localhost:8083
          predicates:
            - Path=/alarms/** 
        - id: ShopAccount
          uri: http://localhost:8084
          predicates:
            - Path=/views/** 
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


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: Order
          uri: http://Order:8080
          predicates:
            - Path=/orders/** 
        - id: Cook
          uri: http://Cook:8080
          predicates:
            - Path=/cooks/** 
        - id: Notification
          uri: http://Notification:8080
          predicates:
            - Path=/alarms/** 
        - id: ShopAccount
          uri: http://ShopAccount:8080
          predicates:
            - Path= /views/**
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

server:
  port: 8080

```
## CQRS 적용
shopAccount(View)는 Materialized View로 구현하여, 타 마이크로서비스의 데이터 원본에 Join SQL 등 구현 없이도 내 서비스의 화면 구성과 잦은 조회가 가능하게 구현 하였음.

주문(Order), 음식조리 상태변경(Cook) Transaction 발생 후 shopAccount 조회 결과 

- 주문 (order)

![image](https://user-images.githubusercontent.com/88808280/135024960-e135d091-8ce6-485d-b55c-5ce77d3be713.png)

- 조리완료 처리(cook)

![image](https://user-images.githubusercontent.com/88808280/135025034-b882662b-529a-458b-b08e-4b4a6a99e272.png)

- 전체 주문 현황(shopaccount)

![image](https://user-images.githubusercontent.com/88808280/135025250-fe663474-7f60-43b6-a3c3-5c09def82efd.png)



## 폴리글랏 퍼시스턴스
mypage 서비스의 DB와 Rental/Payment/Point 서비스의 DB를 다른 DB를 사용하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다.

|서비스|DB|pom.xml|
| :--: | :--: | :--: |
|Order| H2 |![image](https://user-images.githubusercontent.com/2360083/121104579-4f10e680-c83d-11eb-8cf3-002c3d7ff8dc.png)|
|Cook| H2 |![image](https://user-images.githubusercontent.com/2360083/121104579-4f10e680-c83d-11eb-8cf3-002c3d7ff8dc.png)|
|ShopAccount| H2 |![image](https://user-images.githubusercontent.com/2360083/121104579-4f10e680-c83d-11eb-8cf3-002c3d7ff8dc.png)|
|Alarm| HSQL |![image](https://user-images.githubusercontent.com/2360083/120982836-1842be00-c7b4-11eb-91de-ab01170133fd.png)|


## 동기식 호출과 Fallback 처리
주문 취소하기 위해서는 주문진행 상태가 주문접수 상태(order)에서 주문취소가능하며, 조리 시작(cook) 및 완료(cooked) 상태에서는 취소할수 없는 요구사항이 있음
해당 처리는 동기 호출이 필요하다고 판단하여 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 구현 하였음 

Order 서비스 내 CookService

```java
package foodcourt.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.Date;

@FeignClient(name="Cook", url="http://${api.url.Cook}:8080")
public interface CookService {
    @RequestMapping(method= RequestMethod.GET, path="/cook/cookStatusCheck")
    public String orderCheck(@RequestParam Long orderId);

}
```

Order 서비스 내 Req/Resp

```java
    @PostMapping("orders/cancel")
    Boolean oderCancel(@RequestBody String data) {
       
        ObjectMapper mapper = new ObjectMapper();
        Order orderInfo = null;
        try {
            orderInfo = mapper.readValue(data, Order.class);
        } catch (IOException e) {
            e.printStackTrace();
        }

        /* Cancel 가능한지 주문상태 조회 */
        String OrderStatus = OrderApplication.applicationContext.getBean(foodcourt.external.CookService.class)
        .orderCheck(orderInfo.getOrderId());

        if("order".equals(OrderStatus)){
            Optional<Order> orderOptional = orderRepositroy.findById(orderInfo.getOrderId());
            Order order = orderOptional.get();
            order.setStatus(orderInfo.getStatus());
            orderRepositroy.save(order);
            return true;
        }else{
            System.out.println("########### Order Cancel Failed - Reason_OerderStatus : "+ OrderStatus);
            return false;
        }
    }
```

Cook 서비스 내 Oder 서비스 Feign Client 요청 대상

```java
@RestController
 public class CookController {

    @Autowired CookRepository cookRepositroy;
    
    @RequestMapping(value = "/cook/cookStatusCheck",
    method = RequestMethod.GET,
    produces = "application/json;charset=UTF-8")
    public String cookStatusCheck(HttpServletRequest request, HttpServletResponse response) {

        Long orderId = Long.valueOf(request.getParameter("orderId"));
        Optional<Cook> cookOptional = cookRepositroy.findByOrderId(orderId);
        Cook cook = cookOptional.get();

        System.out.println("##### /cook/cookStatusCheck  cook Status : "+cook.getCookStatus());
        return cook.getCookStatus();
    }    
 }
```

동작 확인


주문상태가 주문접수 상태가 아닌 주문건에 대한 취소 요청시 처리  
![image](https://user-images.githubusercontent.com/88808280/135024686-99b50a6a-f855-499e-b595-54437cba2cfc.png)


## SAGA패턴 

SAGA 패턴은 각 서비스의 트랜잭션은 단일 서비스 내의 데이터를 갱신하는 일종의 로컬 트랜잭션 방법이고 서비스의 트랜잭션이 완료 후에 다음 서비스가 트리거 되어, 트랜잭션을 실행하는 방법입니다.

SAGA 패턴에 맞추어서 작성되어 있다.

**SAGA 패턴에 맞춘 트랜잭션 실행**

![image](https://user-images.githubusercontent.com/88808280/135009549-12d8900e-e163-4bdd-9186-f04da2482089.png)

SAGA 패턴에 맞추어서 order 서비스의 주문 생성이 완료되면 Cook 서비스를 트리거하게 되어 cook 상태를 업데이트하며,
Cook 상태가 완료 상태로 변경되면 Alarm 서비스에서 트리하게 되어있다.

아래는 Oder 서비스에 주문생성 실행한 결과이다.

![image](https://user-images.githubusercontent.com/88808280/135023646-f027a4c9-3f62-49ee-b0cc-22a1515f04d4.png)

위와 같이 Order 서비스에서 주문을 생성하게 될 경우 아래와 같이 Cook 서비스에서 Cook(조리요청) 생성 하게 된다. 

![image](https://user-images.githubusercontent.com/88808280/135023862-f091ea68-3da2-4813-845e-e7aa3c1f814f.png)


위와 같이 Cook 서비스에서 조리 상태를 완료로 업데이트 하면 이벤트를 발신하게 되고 이를 수신 받은 Alarm 서비스에서 수신하여 처리한다.

조리 상태변경(조리완료)

![image](https://user-images.githubusercontent.com/88808280/135024073-e5765612-7fbe-44d0-b526-b387f366e480.png)

알람 서비스(주문건에 대한 조리 완료)

![image](https://user-images.githubusercontent.com/88808280/135024183-1768dbfa-a580-4a90-a279-662c4d1857bc.png)


# 운영

## Deploy/ Pipeline

- yml 파일 이용한 Docker Image Push/deploy/서비스생성

```sh

cd gateway
az acr build --registry user0808 --image user0808.azurecr.io/gateway:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd order
az acr build --registry user0808 --image user0808.azurecr.io/order:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd cook
az acr build --registry user0808 --image user0808.azurecr.io/cook:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

cd ..
cd notification
az acr build --registry user0808 --image user0808.azurecr.io/notification:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml


cd ..
cd shopAccount
az acr build --registry user0808 --image user0808.azurecr.io/shopaccount:latest .

cd kubernetes
kubectl apply -f deployment.yml
kubectl apply -f service.yaml

```

## Config Map
- 변경 가능성이 있는 항목은 ConfigMap을 사용하여 설정 변경 할수 있도록 구성   
  - Rental 서비스에서 바라보는 Point 서비스 url 일부분을 ConfigMap 사용하여 구현 함

configmap 설정 및 소스 반영

![image](https://user-images.githubusercontent.com/89369983/133117696-5f3b4fd7-e850-4caa-901b-b3171baca69b.png)

configmap 구성 내용 조회

![image](https://user-images.githubusercontent.com/89369983/133117908-934c3b40-99a2-4905-9963-6692d6c82a28.png)


## Autoscale (HPA)
프로모션 등으로 사용자 유입이 급격한 증가시 안정적인 운영을 위해 Rental 자동화된 확장 기능을 적용 함

Resource 설정 및 유입전 현황 
![image](https://user-images.githubusercontent.com/89369983/133118106-ab6141ad-bd66-40bd-a14c-bcf21ecf5e23.png)

- rental 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정(CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘리도록 설정)

```sh
$ kubectl autoscale deploy booking --min=1 --max=10 --cpu-percent=15
```

서비스에 Traffic 유입(siege를 통해 워크로드 발생)

![image](https://user-images.githubusercontent.com/89369983/133118622-1a8e337b-b522-44fa-81c2-4b67877144d3.png)

부하테스트 결과 HPA 반영
![image](https://user-images.githubusercontent.com/89369983/133118351-4315f1b0-85b9-46ea-b23a-9d90ac21f6d5.png)

## Circuit Breaker
  * Istio 다운로드 및 PATH 추가, 설치
  * rentbook namespace에 Istio주입
![image](https://user-images.githubusercontent.com/89369983/133118751-c6ce8e89-a0ba-4655-bd7d-3da68f269bed.png)

  * Transaction이 과도할 경우 Circuit Breaker 를 통하여 장애격리 처리
![image](https://user-images.githubusercontent.com/89369983/133118831-9e6c5580-bc7a-4690-9084-250fcfbb47ee.png)


## Zero-Downtime deploy (Readiness Probe)

  * readiness 미 설정 상태에서, 배포중 siege 테스트 진행 
  - rental 서비스 배포 중 정상 실행중 서비스 요청은 성공(201), 배포중인 서비스에 요청은 실패 (503 ) 확인
![image](https://user-images.githubusercontent.com/89369983/133118944-e973c8c8-6e3c-4072-9e3c-f6dff07b56bc.png)

  * deployment.yml에 readiness 설정 및 적용 후 siege 테스트 진행시 안정적인 서비스 응답확인
![image](https://user-images.githubusercontent.com/89369983/133119028-cdc334ef-72e0-43ac-a603-9a38ee5e0ed8.png)


    
## Self-healing (Liveness Probe)

  * Liveness Probe 설정 및 restart 시도 점검을 위해 잘못된값 설정으로 변경 후 RESTARTS 처리 테스트 결과
![image](https://user-images.githubusercontent.com/89369983/133119195-0aee658f-fe25-46dd-abc2-2b125372f55c.png)

