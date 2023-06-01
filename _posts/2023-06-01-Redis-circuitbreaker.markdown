---
layout: post
title:  "Redis 서킷브레이크 적용 - Spring Boot(kotlin)"
description:  Spring Boot(kotlin) 환경에서 DB master / slave 처리 
date:   2023-06-01 00:03:00 +000
categories: Spring boot Redis resilience4j
---
## 개요
Redis 캐시 사용시 Redis접속이 문제가 생길경우 DB 조회를 하기 위해 준비 해보자<br>
netflix/hystrix을 사용하였으나 더이상 업데이트는 안한다는 내용이 있고 2019년이 마지막 버전 이여서 이참에 바꿔 보았다  


## 환경 
### bulid.gradle.kts
```gradle.kst

dependencies {
    ...    
    implementation("io.github.resilience4j:resilience4j-spring-boot2:2.0.2")
    ...
}
```

### apllication.properties
```properties
// 세부 옵션은 구글링으로 찾으면 설명이 잘되어 있다 
resilience4j.circuitbreaker.instances.default.failure-rate-threshold=10
resilience4j.circuitbreaker.instances.default.wait-duration-in-open-state=100ms
resilience4j.circuitbreaker.instances.default.sliding-window-size=1
```


### 사용 Service 에서 호출 예시 
```kotlin 
// 캐시설정은 Controller 에 한적도 있고 Service 에 한적도 있는데 개인적으로 Service 에서 관리하는게 난 좀더 편했다 ..
@Service 
class CacheService(
    private val testRepository:TestRepository
){
    @Cacheable(cacheNames = ["TestCache"],key="#key",cacheManager="cacheManager")
    @CircuitBreaker(name="TestCache", fallbackMethod = "getTestCall")
    fun testCall(key:Long): Test {
        return this.getTestCall()
    }

    @Transactional(readOnly = true)
    protected fun getTestCall(t:Throwable?=null): Test {
        return testRepository.findById(key)
    }
    
}
```

## 결과
설정없이 호출 할 경우 redis 연결이 안된다면 에러가 발생된다<br>
하지만 위와 같이 간단하게 설정하고 사용한다면 redis 연결이 실패 해도 DB 조회를 하기 때문에 에러가 발생되지 않는다<br> 

하지만 다음과 같은 문제가 발생되었다 <br>
1분동안 지연이 발생된다 ....... 왜지 ?!<br>
레디스 연결 타임아웃도 하고 서킷브레이크 10% 로 하고 retry로도 1회로 하여 테스트 하였다 하지만 동일했다 1분지연이 발생하는 유형은 다음과 같다<br> 

api 기동시 레디스가 꺼져 있는경우 : 지연없이 DB 즉시 조회 처리  ( 이때는 문제가 없다 )<br>
api 기동시 레디스가 연결되어 있다가 서비스 도중 레디스연결이 끊어 진경우  : 1분 딜레이가 걸린다 ...... 이게 문제이다<br> 
해당 문제를 이미 이슈업이 된 내용이 있었다 하지만 그 후 개선된 내용은 확인하지 못하였다 <br>
<br>

그래서 다음과 같이 설정을 일부 더 추가 하였다<br>

### redisconfig 
```kotlin
...
기존 내용 
@Bean
fun redisConnectionFactory(): RedisConnectionFactory {
    ....
    return LettuceConnectionFactory(redisStandaloneConfiguration) 
}

변경 내용  
@Bean
fun redisConnectionFactory(): RedisConnectionFactory {
    ....
    var lettuceClientConfiguration: LettuceClientConfiguration = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofMillis(1000)) // 연결 타임아웃 설정
            .build()
    return LettuceConnectionFactory(redisStandaloneConfiguration,lettuceClientConfiguration) 
}
...
```


### apllication.properties
```properties
pspring.redis.lettuce.pool.enabled=false  // 추가 
```
되기는 하는거 같다 하지만 좀더 찾아 보고 사용할수 있을지 봐야 겠다 ... 


## 아쉬운점
netflix/hystrix 가 더 잘되던가 같다...
