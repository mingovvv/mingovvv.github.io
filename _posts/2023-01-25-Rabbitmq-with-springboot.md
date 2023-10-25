---
title: springboot에서 rabbitmq 맛보기
author: mingo
date: 2023-01-25 23:21:09 +0900
categories: [rabbitmq]
tags: [rabbitmq, springboot]
---

----
springboot 환경에서 메시지-브로커 오픈소스인 `rabbitmq`를 적용해 간단한 announce 프로젝트를 구축해보자.

## 초기설정
부트에서 라이브러리 의존성을 추가하고 rabbitmq 서버는 docker를 통해 구성한다.

```text
// build.gradle
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```
```text
docker pull rabbitmq
docker run -d --name my-rabbit -p 5672:5672 -p 15672:15672 rabbitmq
```

## 구조
Exchange는 `Direct`, `Topic`, `Fanout`가 각각 클라이언트에게 주어지고 해당 클라이언트는 원하는 Exchange를 바탕으로 원하는 수신자에게
메시지를 전달한다.

## 소스코드

### config
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    queue-name: user.human.eric,user.human.aron,user.human.adam,user.devil.emma,user.devil.issabel,admin.none.gm
    direct-exchange: mingo.direct
    topic-exchange: mingo.topic
    fanout-exchange: mingo.fanout
```
사용자는 메시지를 전달받을 각각의 Queue가 존재하고 Queue 이름을 아래와 같이 구성된다.
- user.human
  - eric, aron, adam
- user.devil
  - emma, isaabel
- admin.none
  - gm

### Exchange과 Queue를 Binding 구현체
```java
@Bean
public Declarables fanoutBindings() {
    List<String> queueName = rabbitProperties.getQueueName().stream().toList();

    List<Declarable> declarables = new ArrayList<>();
    FanoutExchange fanoutExchange = new FanoutExchange(rabbitProperties.getFanoutExchange());
    List<Binding> bindings = new ArrayList<>();

    queueName.forEach(queue -> {
        Queue q = new Queue(queue);
        declarables.add(q);
        bindings.add(BindingBuilder.bind(q).to(fanoutExchange));
    });

    declarables.add(fanoutExchange);
    declarables.addAll(bindings);

    return new Declarables(declarables);
}

@Bean
public Declarables topicHumanBindings() {

    List<String> queueName = rabbitProperties.getQueueName().stream().filter(q -> q.startsWith("user.human")).toList();

    List<Declarable> declarables = new ArrayList<>();
    TopicExchange topicExchange = new TopicExchange(rabbitProperties.getTopicExchange());
    List<Binding> bindings = new ArrayList<>();

    queueName.forEach(queue -> {
        Queue q = new Queue(queue);
        declarables.add(q);
        bindings.add(BindingBuilder.bind(q).to(topicExchange).with("human.*"));
    });

    declarables.add(topicExchange);
    declarables.addAll(bindings);

    return new Declarables(declarables);

}

@Bean
public Declarables topicDevilBindings() {

    List<String> queueName = rabbitProperties.getQueueName().stream().filter(q -> q.startsWith("user.devil")).toList();

    List<Declarable> declarables = new ArrayList<>();
    TopicExchange topicExchange = new TopicExchange(rabbitProperties.getTopicExchange());
    List<Binding> bindings = new ArrayList<>();

    queueName.forEach(queue -> {
        Queue q = new Queue(queue);
        declarables.add(q);
        bindings.add(BindingBuilder.bind(q).to(topicExchange).with("devil.*"));
    });

    declarables.add(topicExchange);
    declarables.addAll(bindings);

    return new Declarables(declarables);

}

@Bean
public Declarables directBindings() {

    List<String> queueName = rabbitProperties.getQueueName().stream().toList();

    List<Declarable> declarables = new ArrayList<>();
    DirectExchange directExchange = new DirectExchange(rabbitProperties.getDirectExchange());
    List<Binding> bindings = new ArrayList<>();

    queueName.forEach(queue -> {
        Queue q = new Queue(queue);
        declarables.add(q);
        bindings.add(BindingBuilder.bind(q).to(directExchange).with(queue));
    });

    declarables.add(directExchange);
    declarables.addAll(bindings);

    return new Declarables(declarables);

}
```
 - Declarables 객체를 통하여 한번에 Exchange <-> Queue 사이를 Binding 함
 - announce 기능과 역할에 맞도록 메시지 수신자 바인딩
   - FanoutExchange는 모든 user에게 메시지를 전달한다.
     - FanoutExchange에 연결된 모든 Queue가 대상이 된다.
   - TopicExchange는 라우팅 키(routing-key)의 패턴이 부합한 user에게 메시지를 전달한다.
     - user.human : routing-key (human.*) 
     - user.devil : routing-key (devil.*)
   - DirectExchange는 라우팅 키(routing-Key)가 일치한 user에게 메시지를 전달한다.
     - user.devil.{username} : routing-key (user.devil.{username})

### Consumer 구현체
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class UserListener {

    @RabbitListener(queues = "#{'${spring.rabbitmq.queue-name}'.split(',')[0]}")
    public void receiveMessage1(MessageDto MessageDto) {
        log.info("Received Queue name : [{}], message: [{}]", "user.human.eric", MessageDto.toString());
    }

    @RabbitListener(queues = "#{'${spring.rabbitmq.queue-name}'.split(',')[1]}")
    public void receiveMessage2(MessageDto MessageDto) {
        log.info("Received Queue name : [{}], message: [{}]", "user.human.aron", MessageDto.toString());
    }

    @RabbitListener(queues = "#{'${spring.rabbitmq.queue-name}'.split(',')[2]}")
    public void receiveMessage3(MessageDto MessageDto) {
        log.info("Received Queue name : [{}], message: [{}]", "user.human.adam", MessageDto.toString());
    }

    @RabbitListener(queues = "#{'${spring.rabbitmq.queue-name}'.split(',')[3]}")
    public void receiveMessage4(MessageDto MessageDto) {
        log.info("Received Queue name : [{}], message: [{}]", "user.devil.emma", MessageDto.toString());
    }

    @RabbitListener(queues = "#{'${spring.rabbitmq.queue-name}'.split(',')[4]}")
    public void receiveMessage5(MessageDto MessageDto) {
        log.info("Received Queue name : [{}], message: [{}]", "user.devil.issabel", MessageDto.toString());
    }

    @RabbitListener(queues = "#{'${spring.rabbitmq.queue-name}'.split(',')[5]}")
    public void receiveMessage6(MessageDto MessageDto) {
        log.info("Received Queue name : [{}], message: [{}]", "admin.none.gm", MessageDto.toString());
    }

}
```
 - user의 name을 기반으로 Qeueu 메시지를 수신하는 리스너 생성

### Publisher 구현체
```java
public interface RabbitExchange {
    void sendMessage(MessageDto messageDto);
}

@Slf4j
@Service
@RequiredArgsConstructor
public class DirectProducer implements RabbitExchange {

  private final RabbitTemplate rabbitTemplate;
  private final RabbitProperties rabbitProperties;

  @Override
  public void sendMessage(MessageDto messageDto) {
    log.info("Direct Message Sent: [{}]", messageDto.toString());
    rabbitTemplate.convertAndSend(rabbitProperties.getDirectExchange(), messageDto.getRoutingKey(), messageDto);
  }

}

@Slf4j
@Service
@RequiredArgsConstructor
public class TopicProducer implements RabbitExchange {

  private final RabbitTemplate rabbitTemplate;
  private final RabbitProperties rabbitProperties;
  
  @Override
  public void sendMessage(MessageDto messageDto) {
    log.info("Topic Message Sent: [{}]", messageDto.toString());
    rabbitTemplate.convertAndSend(rabbitProperties.getTopicExchange(), messageDto.getRoutingKey(), messageDto);
  }

}

@Slf4j
@Service
@RequiredArgsConstructor
public class FanoutProducer implements RabbitExchange {

  private final RabbitTemplate rabbitTemplate;
  private final RabbitProperties rabbitProperties;
  
  @Override
  public void sendMessage(MessageDto messageDto) {
    log.info("Fanout Message Sent: [{}]", messageDto.toString());
    rabbitTemplate.convertAndSend(rabbitProperties.getFanoutExchange(), null, messageDto);
  }

}
```
 - send() 메서드를 RabbitExchange 인터페이스를 구현
 - 클라이언트에게 어떤 Exchange를 사용해서 어떤 user에게 announce 할 것인지 선택권을 양도

### publisher 호출영역
```java
private final Map<ExchangeType, RabbitExchange> rabbitExchangeMap;

  @PostMapping("/rabbitmq/send")
  public ResponseEntity<String> produce(@RequestBody MessageDto messageDto) {
      rabbitExchangeMap.getOrDefault(messageDto.getExchangeType(), rabbitExchangeMap.get(ExchangeType.DIRECT))
              .sendMessage(messageDto);
      return ResponseEntity.ok(messageDto.getExchangeType().name() + " successfully send");
  }
```
 - Exchange type을 Map 형식으로 생성하고 스프링 빈 등록
 - 클라이언트의 요청에 따라 원하는 Exchange type을 기반으로 announce 할 수 있음

### 메시지 호출

![Desktop View](/assets/img/post/20230125/1.png){: width="600" height="600" }
 - 전체 공지 -> Exchange type : FANOUT

![Desktop View](/assets/img/post/20230125/2.png){: width="800" height="400" }
 - 전체 유저 메시지 수신 확인

![Desktop View](/assets/img/post/20230125/3.png){: width="600" height="600" }
 - 악마 종족 유저들에게 공지 -> Exchange type : TOPIC

![Desktop View](/assets/img/post/20230125/4.png){: width="800" height="400" }
 - 악마 종족 유저들 메시지 수신 확인

![Desktop View](/assets/img/post/20230125/5.png){: width="600" height="600" }
 - 'adam' 유저에게 공지 -> Exchange type : DIRECT

![Desktop View](/assets/img/post/20230125/6.png){: width="800" height="400" }
 - 'adam' 유저 메시지 수신 확인

### 마무리
Exchange와 Queue 그리고 이 둘을 연결하는 Binding에 대한 관계를 다시금 생각해볼수 있었다. 하나의 프로젝트로 구성되어
Publisher영역 Consumer영역이 같이 구현되었지만 실제 회사 프로젝트는 이렇게 간단하지 않다. 어느 영역에서 앞서 말한 객체들의 생성을
주관할 것이냐 하는 것은 비즈니스 모델에 따라 다르겠지만 개인적으로 아래 링크의 글을 보고 많은 도움을 얻었다.

https://gigi.nullneuron.net/gigilabs/rabbitmq-who-creates-the-queues-and-exchanges/
