![image](https://user-images.githubusercontent.com/487999/79708354-29074a80-82fa-11ea-80df-0db3962fb453.png)

# 서비스 시나리오

배달의 민족 : 마이크로서비스 분석/설계 및 구현


기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다.
1. 고객이 선택한 메뉴에 대해 결제한다.
1. 주문이 되면 주문 내역이 입점상점주인에게 주문정보가 전달된다.
1. 상점주는 주문을 수락하거나 거절할 수 있다.
1. 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다.
1. 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다.
1. 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다.
1. 라이더가 해당 요리를 Pick한 후, 앱을 통해 통보한다.
1. 고객이 주문상태를 중간중간 조회한다.
1. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다.
1. 라이더의 배달이 끝나면 배송확인 버튼으로 모든 거래가 완료된다.


비기능적 요구사항
1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven


# 체크포인트

1. Saga (Pub / Sub)
2. CQRS
3. Compensation / Correlation


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과
  ![image](https://user-images.githubusercontent.com/93691092/211801743-06ebeb97-e2d8-4a2e-9888-c9f1370e8284.png)


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 Java로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd front
mvn spring-boot:run

cd store
mvn spring-boot:run 

cd customer
mvn spring-boot:run  

cd rider
mvn spring-boot:run

cd history
mvn spring-boot:run
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 front 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.

```
package delivery.prj.domain;

import delivery.prj.FrontApplication;
import delivery.prj.domain.OrderCanceled;
import delivery.prj.domain.OrderPlaced;
import java.util.Date;
import java.util.List;
import javax.persistence.*;
import lombok.Data;

@Entity
@Table(name = "Order_table")
@Data
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private Long userId;

    private Long storeId;

    private Long menuId;

    private String qty;

    private String status;
    
    ...
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다.
```
package delivery.prj.domain;

import delivery.prj.domain.*;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel = "orders", path = "orders")
public interface OrderRepository
    extends PagingAndSortingRepository<Order, Long> {
    }

```
- 적용 후 REST API 의 테스트
```
# front 서비스의 주문처리
http localhost:8081/orders menuId=20

# store 서비스의 주문처리
http PATCH localhost:8082/storeOrders status="요리시작"

# 주문 상태 확인 (CQRS)
http localhost:8085/histories/1

```

# 체크포인트 구현:


## 1. Saga (Pub / Sub)
kafka를 통한 Pub/Sub 비동기 통신
- Publish 예제 코드
```
    @PostPersist
    public void onPostPersist() {
        OrderPlaced orderPlaced = new OrderPlaced(this);
        orderPlaced.publishAfterCommit();
    }
    
```
- Subscribe 예제 코드
```
@Service
@Transactional
public class PolicyHandler {

    @Autowired
    StoreOrderRepository storeOrderRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void whatever(@Payload String eventString) {}

    @StreamListener(
        value = KafkaProcessor.INPUT,
        condition = "headers['type']=='OrderPlaced'"
    )
    public void wheneverOrderPlaced_OrderInfoTransfer(
        @Payload OrderPlaced orderPlaced
    ) {
        OrderPlaced event = orderPlaced;
        System.out.println(
            "\n\n##### listener OrderInfoTransfer : " + orderPlaced + "\n\n"
        );

        StoreOrder.orderInfoTransfer(event);
    }

```

## 2. CQRS
History를 통한 오더상태 업데이트 정보 조회
- History Aggregate
```
package delivery.prj.domain;

import java.util.Date;
import java.util.List;
import javax.persistence.*;
import lombok.Data;

@Entity
@Table(name = "History_table")
@Data
public class History {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;

    private Long orderId;
    private Long userId;
    private Long storeId;
    private Long menuId;
    private String qty;
    private String orderStatus;
    private String storeStatus;
    private String deliveryStatus;
}

```
- History View Handler
```
package delivery.prj.infra;

import delivery.prj.config.kafka.KafkaProcessor;
import delivery.prj.domain.*;
import java.io.IOException;
import java.util.List;
import java.util.Optional;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class HistoryViewHandler {

    @Autowired
    private HistoryRepository historyRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderPlaced_then_CREATE_1(
        @Payload OrderPlaced orderPlaced
    ) {
        try {
            if (!orderPlaced.validate()) return;

            // view 객체 생성
            History history = new History();
            // view 객체에 이벤트의 Value 를 set 함
            history.setUserId(orderPlaced.getUserId());
            history.setStoreId(orderPlaced.getStoreId());
            history.setMenuId(orderPlaced.getMenuId());
            history.setQty(orderPlaced.getQty());
            history.setOrderId(orderPlaced.getId());
            history.setOrderStatus("주문완료");
            // view 레파지 토리에 save
            historyRepository.save(history);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
- 주문 command
    ![image](https://user-images.githubusercontent.com/93691092/212068554-1e0e894a-e956-4632-ac8e-a39cc875837c.png)
- kafka monitoring 
    ![image](https://user-images.githubusercontent.com/93691092/212069459-ed71c1c6-c538-4559-89cb-dd07dc59d8ea.png)
- update 된 history 
    ![image](https://user-images.githubusercontent.com/93691092/212069497-69a4a1f2-0d62-4a51-bd9b-c437e85c41bc.png)

## 3. Compensation / Correlation
3-1. 주문 시 상점정보에 update 하는 예제 (Correlation : orderId)
- Store Order Repository
```
@RepositoryRestResource(
    collectionResourceRel = "storeOrders",
    path = "storeOrders"
)
public interface StoreOrderRepository
    extends PagingAndSortingRepository<StoreOrder, Long> {}

```
- 주문 event 수신
```
    @StreamListener(
        value = KafkaProcessor.INPUT,
        condition = "headers['type']=='OrderPlaced'"
    )
    public void wheneverOrderPlaced_OrderInfoTransfer(
        @Payload OrderPlaced orderPlaced
    ) {
        OrderPlaced event = orderPlaced;
        System.out.println(
            "\n\n##### listener OrderInfoTransfer : " + orderPlaced + "\n\n"
        );

        // Sample Logic //
        StoreOrder.orderInfoTransfer(event);
    }
```
- 상점 정보 update
    ![image](https://user-images.githubusercontent.com/93691092/212070887-50ac43dd-922d-4eb0-89ca-9e45566c7e50.png)
- update 결과 확인
    ![image](https://user-images.githubusercontent.com/93691092/212070975-e66e5c99-93dd-4302-9950-fb3ee31fd880.png)

3-2. 요리시작 시 event 수신 후 사용자 앞 notify 예제 (Correlation : orderId)
- 요리시작 event publish (StoreOrder)
    ![image](https://user-images.githubusercontent.com/93691092/212071605-0b2f49d4-7d3a-473c-9a92-266df55a80eb.png)
- kafak monitoring
    ![image](https://user-images.githubusercontent.com/93691092/212071817-b09c267b-4f1b-4323-b93a-8a8029079f8b.png)
- 요리시작 event 수신 (customer.PolicyHandler)
```
    @StreamListener(
        value = KafkaProcessor.INPUT,
        condition = "headers['type']=='CookStarted'"
    )
    public void wheneverCookStarted_NotifyViaKakao(
        @Payload CookStarted cookStarted
    ) {
        CookStarted event = cookStarted;
        System.out.println(
            "\n\n##### listener NotifyViaKakao : " + cookStarted + "\n\n"
        );

        Notification.notifyViaKakao(event);
    }
```
- 주문정보 notify
    ![image](https://user-images.githubusercontent.com/93691092/212072396-a8c3ca1c-9ddd-4537-8014-a910e83c7c0b.png)
- notify 결과 확인
    ![image](https://user-images.githubusercontent.com/93691092/212072463-aa39a423-42ac-4a8c-acc9-fecaed72c926.png)





