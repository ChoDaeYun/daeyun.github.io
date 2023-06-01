---
layout: post
title:  "Redis Pub/Sub - Spring Boot(Java)"
description:  Redis를 활용한 Pub/Sub 적용 
date:   2021-08-04 00:00:00 +000
categories: Redis JAVA SpringBoot 
---
## 개요
Java Spring Boot 환경에서 Redis를 이용한 Pub/Sub 설정 및 사용

## Redis 설정 추가 
$ config set notify-keyspace-events KEA <br>
참조 : https://redis.io/topics/notifications

## dependencies 추가
org.springframework.boot 2.4.2

## RedisConfig.java 
```java
public class RedisConfig {
    
    ...

    @Bean
    public RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory, RedisKeyExpirationListener expirationListener){
        RedisMessageListenerContainer listenerContainer = new RedisMessageListenerContainer();
        listenerContainer.setConnectionFactory(connectionFactory);
        listenerContainer.addMessageListener(expirationListener, new PatternTopic("__keyevent@*__:expired"));
        listenerContainer.setErrorHandler(e -> log.error("There was an error in redis key expiration listener container", e));
        return listenerContainer;
    }
}
```

## RedisKeyExpirationListener.java
```java
@Component
public class RedisKeyExpirationListener implements MessageListener {


    @Override
    public void onMessage(Message message,byte[] pattern){
        String expiredKey = message.toString();
        System.out.println(expiredKey); // 만료되는 키정보 
    }
}
```

## 결과
키 만료 후 서버로 전송 로그 확인

## 아쉬운점 
키만 전달되기 때문에 onMessage 처리시 데이터를 사용할수 있는 방법이 없다.<br>
그래서 임시 활용으로 만료시 활용해야 하는 데이터에 대해서는 다음과 같이 이중으로 처리<br>
만료 예정 5초 <br>
1. 첫번째 5초 만료로 등록<br> 
2. 두번째 5+@초 만료로 등록 <br>

만약 key명이 abcd 라고 한다면 1번에 abcd_expiry 으로 생성 2번 abcd 로 5+@ 초로 생성하여<br> 
1번에서 만료 될 경우 _expiry를 제외한 abcd 추출하여 데이터 확인  <br>

## 문제점 
레디스의 의존성을 높이므로써 레디스가 죽으면 서버가 reconnect 로그가 남으며 서비스가 정상 작동되지 않는 현상 발생 <br>
DB만큼 레디스도 중요해진 상황 불편하다... 결국 장애 빈도를 줄이기 위해 직접 설치가 아닌 AWS elasticache 사용 
