# SKT_KANU
SKT Kanu (Azure project)
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW

# Table of contents
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
    - [Event Storming 결과](#Event-Storming-결과)
    - [헥사고날 아키텍처 다이어그램 도출](#헥사고날-아키텍처-다이어그램-도출)
  - [구현:](#구현:)
    - [DDD 의 적용](#DDD-의-적용)
    - [기능적 요구사항 검증](#기능적-요구사항-검증)
    - [비기능적 요구사항 검증](#비기능적-요구사항-검증)
    - [Saga](#saga)
    - [CQRS](#cqrs)
    - [Correlation](#correlation)
    - [GateWay](#gateway)
    - [Polyglot](#polyglot)
    - [동기식 호출(Req/Resp) 패턴](#동기식-호출reqresp-패턴)
    - [비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트](#비동기식-호출--시간적-디커플링--장애격리--최종-eventual-일관성-테스트)
  - [운영](#운영)
    - [Deploy / Pipeline](#deploy--pipeline)
    - [Config Map](#configmap)
    - [Secret](#secret)
    - [Circuit Breaker와 Fallback 처리](#circuit-breaker와-fallback-처리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [Zero-downtime deploy (Readiness Probe) 무정지 재배포](#zero-downtime-deploy-readiness-probe-무정지-재배포)
    - [Self-healing (Liveness Probe))](#self-healing-liveness-probe)

# 서비스 시나리오

기능적 요구사항
1. 고객이 커피(음료)를 주문(Order)한다.
2. 고객이 지불(Pay)하고, 포인트(Point)가 적립된다.
3. 결제모듈(payment)에 결제를 진행하게 되고 '지불'처리 된다.
4. 결제 '승인' 처리가 되면 주방에서 음료를 제조한다.
5. 고객과 매니저는 마이페이지를 통해 진행상태(OrderTrace)를 확인할 수 있다.
6. 음료가 준비되면 배달(Delivery)을 한다.
7. 고객이 취소(Cancel)하는 경우 지불 및 제조, 배달이 취소가 된다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 등록이 성립되지 않는다. - Sync 호출
2. 장애격리
    1. 지불이 수행되지 않더라도 주문과 결제는 365일 24시간 받을 수 있어야 한다  - Async(event-driven), Eventual Consistency
    2. 결제 시스템이 과중되면 주문(Order)을 잠시 후 처리하도록 유도한다  - Circuit breaker, fallback
3. 성능
    1. 마이페이지에서 주문상태(OrderTrace) 확인  - CQRS

# 분석/설계


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: http://www.msaez.io/#/storming/QFPJP8hKmlRp2MfooEVdmMfG9B72/mine/ccf667a1ea140f64e4144c2628864dfd

### 이벤트 도출
![image](https://user-images.githubusercontent.com/19682978/123178135-8af4ae80-d4c1-11eb-88b1-fb3ccf96a583.png)

### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/19682978/123178278-d3ac6780-d4c1-11eb-98fa-15c134e9d11d.png)

### 완성된 1차 모형
![image](https://user-images.githubusercontent.com/19682978/123178376-08b8ba00-d4c2-11eb-9585-3a94560cc26f.png)

### 완성된 최종 모형 ( 시나리오 점검 후 )
![image](https://user-images.githubusercontent.com/19682978/123271535-e794af80-d53b-11eb-8ad1-2fe9982f72c4.png)

## 헥사고날 아키텍처 다이어그램 도출 
![image](https://user-images.githubusercontent.com/20077391/121859335-88ac8a80-cd32-11eb-9159-9599abcf67cf.png)

# 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라,구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다.
(각자의 포트넘버는 8081 ~ 8084, 8088 이다.)
```shell
cd book
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd space
mvn spring-boot:run 

cd mypage 
mvn spring-boot:run

cd gateway
mvn spring-boot:run 
```
***

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate 단위로 Entity 로 선언하였다. 
- Spring Data REST의 RestRepository 적용 ( Entity / Repository Pattern 적용 위해 )

```
book 서비스 : book.java

package spacerent;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Book_table")
public class Book {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long bookid;
    private Long userid;
    private String status;
    private String spacename;

    @PostPersist
    public void onPostPersist(){
        Booked booked = new Booked();

        try{
            booked.setStatus("booking");
        }
        catch(Exception e)
        {
            e.printStackTrace();
            booked.setStatus("Timeout");
            System.out.println("**** Payment Service TIMEOUT ****");
        }

        BeanUtils.copyProperties(this, booked);     
        booked.publishAfterCommit();

        spacerent.external.Payment payment = new spacerent.external.Payment();
        payment.setBookid(booked.getBookid());
        payment.setSpacename(booked.getSpacename());
        payment.setStatus("success-pay");
        payment.setUserid(booked.getUserid());
        BookApplication.applicationContext.getBean(spacerent.external.PaymentService.class).pay(payment);


    }

    @PostUpdate
    public void onPostUpdate(){
         
        System.out.println("\n\n##### app onPostUpdate, getStatus() : " + getStatus() + "\n\n");
        if(getStatus().equals("cancel-booking")) {
            Bookcancelled bookcancelled = new Bookcancelled();
            BeanUtils.copyProperties(this, bookcancelled);
            bookcancelled.setBookid(this.getBookid());
            bookcancelled.setSpacename(this.getSpacename());
            bookcancelled.setUserid(this.getUserid());
            bookcancelled.setStatus("cancel-booking");                                
            bookcancelled.publishAfterCommit();
            
//            spacerent.external.Payment payment = new spacerent.external.Payment();
//            payment.setBookid(bookcancelled.getBookid());
//            payment.setSpacename(bookcancelled.getSpacename());
//            payment.setStatus("cancel-booking");
//            payment.setUserid(bookcancelled.getBookid());
//            BookApplication.applicationContext.getBean(spacerent.external.PaymentService.class).pay(payment);            
        }        


    }


    public Long getBookid() {
        return bookid;
    }

    public void setBookid(Long bookid) {
        this.bookid = bookid;
    }
    public Long getUserid() {
        return userid;
    }

    public void setUserid(Long userid) {
        this.userid = userid;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    public String getSpacename() {
        return spacename;
    }

    public void setSpacename(String spacename) {
        this.spacename = spacename;
    }

    public class findByBookId {
    }




}

```
- book 서비스 : BookRepository.java
```
package spacerent;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="books", path="books")
public interface BookRepository extends PagingAndSortingRepository<Book, Long>{

      Book findByBookId(Long bookid);
      
}
```

book 서비스 : PolicyHandler.java
```
package spacerent;

import spacerent.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class PolicyHandler{
    @Autowired BookRepository bookRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRegistered_Updatestate(@Payload Registered registered){

        if(!registered.validate()) return;

        System.out.println("\n\n##### listener Updatestate : " + registered.toJson() + "\n\n");

        // booking 성공 상태 저장  //
        Book book = bookRepository.findByBookId(registered.getBookid());
        book.setStatus(registered.getStatus());
        bookRepository.save(book);
            
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRegistercancelled_Updatestate(@Payload Registercancelled registercancelled){

        if(!registercancelled.validate()) return;

        System.out.println("\n\n##### listener Updatestate : " + registercancelled.toJson() + "\n\n");

        // booking 취소 상태 저장  //
        Book book = bookRepository.findByBookId(registercancelled.getBookid());
        book.setStatus(registercancelled.getStatus());
        bookRepository.save(book);
            
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void whatever(@Payload String eventString){}


}
```
- 적용 후 REST API 의 테스트
```
# 공간 예약(booking)
http POST http://localhost:8081/books userid="1" bookid="1" spacename="numberone" status="booking"
```
![image](https://user-images.githubusercontent.com/19682978/123274827-d0a38c80-d53e-11eb-9f60-7f19c9f8b93d.png)


```
# 결제 확인(payment)
http GET http://localhost:8083/payments/1
```
![image](https://user-images.githubusercontent.com/19682978/123274889-dac58b00-d53e-11eb-8cf0-7b848faefabb.png)

```

# 공간 등록 확인(register)
http GET http://localhost:8082/spaces/1
```
![image](https://user-images.githubusercontent.com/19682978/123274939-e44ef300-d53e-11eb-9b91-6e02d674ac95.png)
```

# 공간 예약 취소(booking)
http PATCH http://localhost:8081/books/2 status="cancel-booking"
```
![image](https://user-images.githubusercontent.com/19682978/123274996-f03ab500-d53e-11eb-99e5-a3da37950003.png)
```

# 결제 취소 확인(payment)
http GET http://localhost:8083/payments/2
```
![image](https://user-images.githubusercontent.com/19682978/123275049-f9c41d00-d53e-11eb-8b0c-544ec7449191.png)
```

# 공간 취소 확인(register)
http GET http://localhost:8082/spaces/2
```
![image](https://user-images.githubusercontent.com/19682978/123275082-02b4ee80-d53f-11eb-86e5-5965f8f86355.png)
```
```
## Gateway 적용
API GateWay를 통하여 마이크로 서비스들의 진입점을 통일할 수 있다. 
아래와 같이 GateWay를 적용하여 마이크로서비스들은 http://localhost:8088/{context}로 접근 .

```
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: book
          uri: http://localhost:8081
          predicates:
            - Path=/books/** 
        - id: space
          uri: http://localhost:8082
          predicates:
            - Path=/spaces/** 
        - id: payment
          uri: http://localhost:8083
          predicates:
            - Path=/payments/** 
        - id: mypage
          uri: http://localhost:8084
          predicates:
            - Path= /mypages/**
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
        - id: book
          uri: http://book:8080
          predicates:
            - Path=/books/** 
        - id: space
          uri: http://space:8080
          predicates:
            - Path=/spaces/** 
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/** 
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path= /mypages/**
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
# 게이트웨이를 통해 등록 확인
![image](https://user-images.githubusercontent.com/19682978/123275455-4b6ca780-d53f-11eb-9f07-c4567235baf5.png)

# 마이페이지 확인
![image](https://user-images.githubusercontent.com/19682978/123275505-558ea600-d53f-11eb-9f06-acd43d05d866.png)


## CQRS
타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이)도 내 서비스의 공간 예약 내역 조회가 가능하게 구현해 두었다.
본 프로젝트에서 View 역할은 mypage 서비스가 수행한다.

CQRS를 구현하여 마이페이지를 통해 조회할 수 있도록 구현

![image](https://user-images.githubusercontent.com/19682978/123275455-4b6ca780-d53f-11eb-9f07-c4567235baf5.png)

![image](https://user-images.githubusercontent.com/19682978/123275505-558ea600-d53f-11eb-9f06-acd43d05d866.png)

## Polyglot 

타 서비스들과 다른 DB를 사용하여 각 마이크로서비스의 다양한 요구사항과 서로 다른 종류의 DB간에도 문제 없이 능동적으로 대처가능한 다형성을 만족하는지 확인
( mypage를 제외한 나머지는 h2로 구현 )
![image](https://user-images.githubusercontent.com/19682978/123297016-1fa6ed00-d552-11eb-9b04-64bb4d862fd5.png)


## 동기식 호출

공간예약(book)->결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.

book 서비스 내 external.PaymentService

- payment 서비스 내린 후 확인 한 상태
![image](https://user-images.githubusercontent.com/19682978/123289575-8aa0f580-d54b-11eb-8009-e23b01216f93.png)

```
package spacerent.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="payment", url="${api.url.Payment}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.GET, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```

book 서비스 내에서 pay 메소드 호출

```
        spacerent.external.Payment payment = new spacerent.external.Payment();
        payment.setBookid(booked.getBookid());
        payment.setSpacename(booked.getSpacename());
        payment.setStatus("success-pay");
        payment.setUserid(booked.getUserid());
        BookApplication.applicationContext.getBean(spacerent.external.PaymentService.class).pay(payment);
```

# 운영

1. git에서 소스 가져오기

```
git clone https://github.com/cnuhoya/spacerent.git
```

- Build 하기
```
bash
cd book
mvn package

cd payment
mvn package

cd space
mvn package

cd mypage
mvn package

cd gateway
mvn package
```

- Docker Image Build/Push, deploy/service 생성 (yml 이용)

-- namespace 생성

```
kubectl create ns spacerent

# book
cd book
az acr build --registry skccuser22acr --image skccuser22acr.azurecr.io/book:latest .
cd kubernetes
kubectl create -f deployment.yml -n spacerent
kubectl create -f service.yaml -n spacerent

# payment
cd payment
az acr build --registry skccuser22acr --image skccuser22acr.azurecr.io/payment:latest .
cd kubernetes
kubectl create -f deployment.yml -n spacerent
kubectl create -f service.yaml -n spacerent

# space
cd space
az acr build --registry skccuser22acr --image skccuser22acr.azurecr.io/space:latest .
cd kubernetes
kubectl create -f deployment.yml -n spacerent
kubectl create -f service.yaml -n spacerent

# mypage
cd mypage
az acr build --registry skccuser22acr --image skccuser22acr.azurecr.io/mypage:latest .
cd kubernetes
kubectl create -f deployment.yml -n spacerent
kubectl create -f service.yaml -n spacerent

# gateway
cd gateway
az acr build --registry skccuser22acr --image skccuser22acr.azurecr.io/gateway:latest .
cd kubernetes
kubectl create -f deployment.yml -n spacerent
kubectl create -f service.yaml -n spacerent

```

- Yaml 파일을 이용한 Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: spacerent
  labels:
    app: gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      containers:
        - name: gateway
          image: skccuser22acr.azurecr.io/gateway:latest
          ports:
            - containerPort: 8080
``` 
```
apiVersion: v1
kind: Service
metadata:
  name: gateway
  namespace: spacerent
  labels:
    app: gateway
spec:
  ports:
    - port: 8080
      targetPort: 8080
  type: LoadBalancer
  selector:
    app: gateway
```
## Deploy / Pipeline
- Deployment 확인
  
 ![image](https://user-images.githubusercontent.com/19682978/123289793-b7eda380-d54b-11eb-8440-d9fba07a1b23.png)

## 동기호출 / Circuit braker / 장애

- 서킷 브레이크는 Spring FeignClient + Hystrix 옵션을 사용 
- 시나리오는 예약(book) -> 결제(payment) 시의 연결을 RESTful Request/Response로 연동하여 구현
- 결제 요청이 과도할 경우 서킷브레이크 

- Hystrix를 설정: 요청처리 쓰레드에서 처리시간이 2000 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 2000
```
- payment처리

```java
# (payment) Payment.java

@PostPersist
    public void onPostPersist(){

        System.out.println("\n\n$$$ 과연확인 : " + this.status + "\n\n");        
        Approved approved = new Approved();
        BeanUtils.copyProperties(this, approved);
        System.out.println("\n\nCircuit braker 확인 spacename: braker: " + approved.getSpacename() + "\n\n");      
        if(approved.getSpacename().equals("braker")){ 
            try{
              
                Thread.sleep(2500);
            }
            catch(Exception e){
                e.printStackTrace();
                System.out.println();
            }
        }        
```
- 부하 테스터 siege 툴을 통한 서킷 브레이커 동작 확인 . 동시 사용자 100명, 60초 동안 실시
```
kubectl exec -it pod/siege -c siege -n spacerent -- /bin/bash
siege -c100 -t60S -v --content-type "application/json" 'http://book:8080/books POST {"userId":"4", "bookid":"4", "spacename":"numberfour", "status":"booking" }'
```
![image](https://user-images.githubusercontent.com/19682978/123290012-e4092480-d54b-11eb-91b0-f616be380952.png)
![image](https://user-images.githubusercontent.com/19682978/123290147-00a55c80-d54c-11eb-9fcb-fe23fd9b882f.png)

## AutuScale (HPA)
서킷브레이크 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.
- 결제 서비스에 리소스에 대한 사용량을 정의.
payment/kubernetes/deployment.yml
```
  resources:
    limits:
      cpu: 500m
    requests:
      cpu: 200m
```
- 결제 서비스에 대한 replica를 동적으로 늘려주도록 HPA를 설정 ( CPU 사용량이 15프로를 넘어서면 replica를 10개까지 증가 )
```
kubectl autoscale deploy payment --min=1 --max=10 --cpu-percent=15 -n spacerent
```
- 서킷브레이크와 동일하게 워크로드를 1분 30초간 부여.
```
kubectl exec -it pod/siege -c siege -n edu -- /bin/bash
siege -c100 -t90S -v --content-type "application/json" 'http://book:8080/books POST {"userId":"5", "bookid":"5", "spacename":"numberfive", "status":"booking" }'
```
- 오토스케일 확인을 위해 모니터링을 걸어둔다.
```
watch kubectl get all -n edu
```
- 확인.  
![image](https://user-images.githubusercontent.com/19682978/123290355-29c5ed00-d54c-11eb-9877-e7cfd6da90f6.png)

## Configmap

- 변경 가능성이 있는 설정을 ConfigMap을 사용하여 관리  
  - Book에서 payment 서비스 url을 ConfigMap 사용하여 구현

- in book src (book/src/main/java/edu/external/PaymentService.java)  
    ![image](https://user-images.githubusercontent.com/19682978/123294333-b625df00-d54f-11eb-9aba-b1b364c746f3.png)

- book application.yml (book/src/main/resources/application.yml)
    ![image](https://user-images.githubusercontent.com/19682978/123294524-e5d4e700-d54f-11eb-88c1-70d1cd3812ed.png)

- book deploy yml (book/kubernetes/deployment.yml)  
    ![image](https://user-images.githubusercontent.com/19682978/123294589-f38a6c80-d54f-11eb-956e-f51cb5ac94b8.png)

- configmap 생성 후 조회

    ```
    kubectl create configmap paymenturl --from-literal=url=http://payment:8080 -n spacerent
    kubectl get configmap paymenturl -o yaml -n spacerent
    ```
   ![image](https://user-images.githubusercontent.com/19682978/123290493-4c580600-d54c-11eb-8143-3f0180d13a62.png)
    
- book pod 내부 환경변수도 확인
    ```
    kubectl exec -it pod/book-5d956f5564-cnt8p -n spacerent -- /bin/sh
    $ env
    ```
![image](https://user-images.githubusercontent.com/19682978/123290515-4feb8d00-d54c-11eb-98a4-9cb60408a545.png)

## Liveness - selfhealing
- port 와 path 잘못된 값(원래는 path : /actuator/health , port : 8080)으로 설정한 후 deploy 재배포후 확인
    ```
          livenessProbe:
            httpGet:
              path: '/actuator/failed'
              port: 8090
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
    ```    
배포전)

![image](https://user-images.githubusercontent.com/19682978/123353046-987e6700-d59b-11eb-9b65-5af2bdf56d1f.png)

배포후)

![image](https://user-images.githubusercontent.com/19682978/123353207-f7dc7700-d59b-11eb-938b-10569a6ebac8.png)


## Readness - 무정지재배포

- Readness 설정 제거 -
    ```
          # readinessProbe:
          #   httpGet:
          #     path: '/actuator/health'
          #     port: 8080
          #   initialDelaySeconds: 10
          #   timeoutSeconds: 2
          #   periodSeconds: 5
          #   failureThreshold: 10    
    ```
- 버전을 바꿈과 두개의 book이 공존함을 확인

![image](https://user-images.githubusercontent.com/19682978/123354112-ec8a4b00-d59d-11eb-8a09-401468c58936.png)

![image](https://user-images.githubusercontent.com/19682978/123354122-f2802c00-d59d-11eb-8a8f-60a7066b4d3b.png)

