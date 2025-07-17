---
layout: post
title: 메시지 브로커
description: >
  분산 시스템에서 비동기 메시지 전달을 중재하는 메시지 브로커의 개념,
  필요성, 핵심 컴포넌트, 전송 패턴, 신뢰성 보장 메커니즘, 대표 솔루션 비교,
  간단 설정 예시까지 한 번에 정리합니다.
image: /assets/img/blog/MessageBrokers.png
sitemap: false
---


# 메시지 브로커 완전 정복 가이드

## 1. 메시지 브로커란?

* **메시지 지향 미들웨어(MOM)**
  애플리케이션 간 **비동기 메시지**를 송수신하도록 중재하는 미들웨어
* **주요 역할**

  * 생산자(Producer) ↔ 소비자(Consumer) 간 느슨한 결합
  * 비동기 처리로 시스템 부하 분산
  * 장애 시 메시지 보관·재처리 지원

## 2. 왜 메시지 브로커를 사용할까?

* **느슨한 결합 (Loosely Coupled)**
  서비스 간 직접 호출 없이 메시지로 통신
* **비동기 처리 (Asynchronous)**
  즉시 응답이 필요 없는 작업(이메일, 이미지 처리 등) 큐잉
* **장애 격리 (Fault Isolation)**
  소비자 다운 시에도 메시지 유실 없이 브로커에 보관
* **확장성 (Scalability)**
  소비자 인스턴스 추가로 처리량 수평 확장

## 3. 핵심 컴포넌트

| 컴포넌트     | 역할                                |
| -------- | --------------------------------- |
| Producer | 메시지를 생성해 브로커에 발행 (`publish`)      |
| Exchange | (AMQP 기준) 메시지를 라우팅 키/헤더에 따라 큐로 분배 |
| Queue    | 소비자가 꺼내갈 때까지 메시지를 보관하는 버퍼         |
| Binding  | Exchange ↔ Queue 간 라우팅 규칙         |
| Consumer | 브로커로부터 메시지를 받아 처리 (`consume`)     |

## 4. 메시지 전달 패턴

| 패턴                   | 설명                               |
| -------------------- | -------------------------------- |
| 점대점 (Point-to-Point) | 하나의 큐에 쌓인 메시지를 여러 소비자가 경쟁 소비     |
| Pub/Sub              | Exchange에서 모든 바인딩된 큐로 브로드캐스트     |
| 토픽 (Topic)           | 와일드카드 기반 라우팅 키 매칭 → 특정 큐 집합으로 분배 |

## 5. 신뢰성 보장 메커니즘

1. **퍼시스턴스 (Persistence)**

  * 큐(`durable=true`) + 메시지(`delivery_mode=2`) 설정 시 디스크에 저장
2. **확인 (Acknowledgment)**

  * **Auto Ack**: 전달 즉시 삭제
  * **Manual Ack**: `basic.ack` 전송 후 삭제, `nack/requeue` 가능
3. **Dead-Letter & TTL**

  * 처리 실패 메시지 → 별도 DLQ로 이동
  * 메시지 TTL(Time-to-Live) 만료 시 자동 재분배 또는 삭제

## 6. 대표 메시지 브로커 비교

| 솔루션           | 프로토콜              | 특징                          |
| ------------- | ----------------- | --------------------------- |
| RabbitMQ      | AMQP, MQTT, STOMP | 다양한 라우팅, 경량 운영, HA 클러스터링 지원 |
| Apache Kafka  | Kafka Protocol    | 디스크 기반 로그 저장, 초고속 스트리밍      |
| Redis Streams | Redis Commands    | 인메모리 큐잉, Pub/Sub 결합 가능      |

## 7. 간단 설정 예시 (Java + RabbitMQ AMQP)

```java
// 1) 연결 & 채널 생성
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
try (Connection conn = factory.newConnection();
     Channel channel = conn.createChannel()) {

  // 2) Exchange 선언
  channel.exchangeDeclare("logs", "fanout", true);

  // 3) 메시지 발행
  String message = "Hello, Broker!";
  channel.basicPublish("logs", "", null, message.getBytes());

  // 4) 큐 생성 & 바인딩
  String queueName = channel.queueDeclare().getQueue();
  channel.queueBind(queueName, "logs", "");

  // 5) 메시지 소비
  DeliverCallback callback = (tag, delivery) -> {
    String body = new String(delivery.getBody(), StandardCharsets.UTF_8);
    System.out.println("Received: " + body);
  };
  channel.basicConsume(queueName, true, callback, consumerTag -> {});
}  
```

## 8. 결론

* **비동기·느슨한 결합·확장성**이 필요하면 메시지 브로커 도입
* **유연한 라우팅**과 **간편 운영** → RabbitMQ
* **대용량 이벤트 스트리밍** → Kafka

메시지 브로커의 핵심 개념을 숙지한 뒤, 직접 설치·실습해 보세요!
