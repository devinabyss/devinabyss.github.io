---
published: true
layout: single
toc: true
title: Spring Ehcache 환경설정
category: library
tags: ehcache, spring
comments: true
---

데이터 중 매번 실시간 데이터를 반드시 반영해야 하는 데이터는 외외로 많지가 않아서 적당한 Interval 을 적용한 caching 은 어플리케이션 성능에 분명 도움을 준다. 특히 복잡한 쿼리로 가져오는 데이터라면 두말할 필요도 없고. Ehcache 는 Java 생태계에서 간편하게 쓰이는 캐시 라이브러리다. MyBatis 를 사용한다면 MyBatis 자체에서 제공하는 쿼리 캐시 기능을 사용해도 되고, MyBatis 위에 Ehcache 를 붙일 수도 있는 것 같지만 Spring Framework 를 사용한다면 굳이 따로 관리되는 설정을 사용하는 것 보다는 ehcache 를 Spring 이 제공하는 Cache 인터페이스에 직접 붙이는게 나을 것이다.



[**Practice Ehcache With SpringMVC - GitHub**](https://github.com/dwuthk/practice-ehcache-with-springmvc)



## Test Enviorment

\- Spring Framework 4.2.5 (<https://projects.spring.io/spring-framework/>)

\- MyBatis 3.3.1 (<http://www.mybatis.org/mybatis-3/ko/>)

\- Ehcache 2.10.1 (<http://www.ehcache.org/>)

\- H2 Database 1.4.191 (<http://www.h2database.com/html/main.html>)



## Maven Dependencies

```xml
<!-- Ehcache -->
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>${version.ehcache}</version>
</dependency>

<!-- Ehcache 적용을 위한 Spring artifact -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${version.springframework}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>${version.springframework}</version>
</dependency>
```

\- ehcache artifact 는 당연히 캐싱을 관장할 라이브러리 이므로 의존성에 추가

\- spring-context artifact 는 스프링의 캐시 관리를 위한 인터페이스를 제공하기에 의존성 추가 (@Annotation 등)

\- spring-context-support artifact 는 스프링의 캐시 관리 기능에 Ehcache 를 위한 추가 패키지들이 포함되어 있어 의존성을 추가한다. ([참조](http://static.springsource.org/spring-framework/docs/3.2.0.RELEASE/spring-framework-reference/html/migration-3.2.html))



## Spring 환경 설정

```xml
<!-- 
    http://www.springframework.org/schema/cache namespace dtd 필요 
    annotation 기반 cache 지정을 위한 설정
-->
<cache:annotation-driven cache-manager="cacheManager" key-generator="customKeyGenerator" />

<!-- Custom Cache Key Generator 지정. 커스터마이징 없이 DefaultKeyGenerator 를 사용해도 무방 -->
<bean id="customKeyGenerator" class="com.dwuthk.practice.ehcache.generator.CustomKeyGenerator" />

<!-- Cache Manager 설정 -->
<bean id="cacheManager" class="org.springframework.cache.ehcache.EhCacheCacheManager">
    <property name="cacheManager">
        <bean class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean">
            <property name="configLocation" value="classpath:ehcache.xml" />
            <property name="shared" value="true" />
        </bean>
    </property>
</bean>
```

Spring 설정에서 주의할 점은, 여타 설정 (tx:annotation-driven 등) 과 마찬가지로 Spring AOP 를 이용한 @Annotation 기반으로 코드 레벨에서 Cache 를 지정하기 위해서는 annotation-driven 이 설정된 레벨 내에서 @Annotation 을 지정할 Bean 들이 관리되어야 한다는 것이다. 

GitHub 예제의 경우 cache:annotation-drive 설정과 Cache 지정을 위한 Service Layer Bean 들이 Context 레벨에서 관리되고 있는데, 만약 Service Layer Bean 들을 Servlet 레벨에서 스캔하여 관리하도록 해놓고 cache 설정은 Context 레벨에서 명시했다면 Servlet 레벨에서 관리중인 Service Layer Bean 들에 @Annotation 을 통한 캐시 명시가 적용되지 않는다. 이는 굳이 Cache 설정에 국한된 것이 아닌 Spring 환경 설정에 익숙치 않은 개발자들이 자주 범하는 실수다.



## @Annotation 을 이용한 코드 레벨 캐시 명시

```java
public Map<String, Object> getNormalDataNonCaching(String param) {

    return generateMap(param);
}

@Cacheable(value = "normal", key = "#param")
public Map<String, Object> getNormalDataCaching(String param) {

    return generateMap(param);

}

@CacheEvict(value = "normal", key = "#param")
public Map<String, Object> getNormalDataEraseCache(String param) {

    return generateMap(param);
}

/**
 * 임의의 Long 객체 생성하여 Map 에 저장하여 리턴
 *
 * @param param
 * @return
 */
private Map generateMap(String param) {

    Map<String, Object> map = new HashMap<>();
    map.put("long", random.nextLong());

    logger.info("## PARAM : {}", param);
    logger.info("## RESULT : {}", map);

    return map;
}
```

SampleService.java 를 보면 서비스 레이어에서 난수를 Long 으로 생성하여 Map 으로 감싸 리턴하는 공통 로직인 generateMap 메소드를 수행하는 4개의 메소드가 존재한다. 이들 중 최초의 3개는 Ehcache 를 이용한 데이터 캐싱을 간단히 테스트할 수 있는 각 케이스들이다.

**1. 캐싱 없이 매번 데이터를 생성하여 리턴**

**2. 지정된 설정에 따라 데이터를 캐싱하여 리턴**

@Cacheable 은 메소드 (ElementType.METHOD) 와 클래스 (ElementType.CLASS) 에 지정 가능하며, 어노테이션이 지정된 대상은 아래의 옵션에 따라 데이터가 한번 생성되면 캐시 설정에 따라 데이터가 캐싱되며, 다음 호출 시에 데이터를 생성 혹은 가져오는 과정을 생략하고 캐시에 저장된 데이터가 바로 리턴 된다.

@Cacheable 은 총 8가지의 옵션 요소가 존재하는데 ([참조](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html)) , 이들 중 주로 사용되는 옵션은 value, key, keyGenerator 등 이다.

value - 데이터를 저장할 캐싱 공간의 대표 명칭

key - 저장될 데이터의 key 생성의 재료가 될 파라메터 (혹은 필드) 를 SpringEL 을 통해 지정 

(지정하지 않으면 DefaultKeyGenerator 에 의해 모든 파라메터들의 hashcode 를 조합한 Integer 값을 key 값으로 생성한다.)

keyGenerator - 지정된, 혹은 전체 key 생성 재료들을 기반으로 데이터의 Key 값을 생성하는 방법이 구현된 빈 객체를 지정.

메소드 레벨에서 @Cacheable 을 사용하고 별도의 key 지정을 하지 않으면 메소드의 파라메터로 Key 값이 생성되는데, 만약 파라메터가 원시형 (래퍼 포함) 이나 String 이 아닌 객체 (데이터 Entity 와 같은) 일 경우 객체의 hashcode 를 기준으로 DefaultKeyGenerator 가 Key 값을 생성한다. 대게 데이터의 Key 값은 객체가 담고 있는 데이터가 eqauls 일 경우 같아야 데이터 캐싱이 의미가 있는 경우가 많으므로 일반 객체를 파라메터로 사용할 경우 CustomKeyGenerator 를 객체에 맞게 구현하여 지정해야 한다.

cacheManager - 상황에 따라 다른 설정의 캐시 매니저를 사용해야 할 경우 상황 별로 cacheManager 지정

cacheNames - 데이터를 저장할 캐싱 공간의 명칭 집합 

cacheResolver - 커스텀 캐시 리졸버를 지정

condition - 메소드가 호출되었을 경우 결과 데이터가 캐싱되어야 할 조건을 SpringEL 로 지정

unless - 메소드가 호출되었을 경우 결과 데이터가 캐싱되지 않아야 할 조건을 SpringEL 로 지정



## 저장된 캐시를 지우고 새로 데이터를 생성하여 리턴

@CacheEvict 는 @Cacheable 로 저장된 캐시 데이터의 삭제를 명시하며, 옵션의 사용은 @Cacheable 과 거의 유사하다. ([참조](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/CacheEvict.html#beforeInvocation--))

value - 데이터를 제거할 캐싱 공간의 대표 명칭

key - 제거할 데이터의 key 생성의 재료가 될 파라메터 (혹은 필드) 를 SpringEL 을 통해 지정 (지정하지 않으면 DefaultKeyGenerator 에 의해 모든 파라메터들의 hashcode 를 조합한 임의의 Integer 값을 key 값으로 생성한다.)

keyGenerator - 지정된, 혹은 전체 key 생성 재료들을 기반으로 제거할 데이터의 Key 값을 생성하는 방법이 구현된 빈 객체를 지정. 

cacheManager - 상황에 따라 다른 설정의 캐시 매니저를 사용해야 할 경우 상황 별로 cacheManager 지정

cacheNames - 데이터가 저장된 캐싱 공간의 명칭 집합 

cacheResolver - 커스텀 캐시 리졸버를 지정

condition - 캐싱 데이터의 제거를 진행 할 조건을 SpringEL 로 지정

beforeInvocation - 데이터의 제거를 메소드 호출 프로세스가 종료되는 시점에서 할 것인지, 호출 시점에서 지울 것인지 지정 (Default = false)

allEntries - key 값에 상관없이 지정된 캐싱 공간의 모든 데이터를 제거 (Default = false)



## 기타 코드 레벨 관련 사항

\- MyBatis Mapper 로 DB 에 접근하는 코드가 포함된 서비스 메소드에는 별다른 추가 설정 없이 최종 리턴되는 데이터가 캐싱된다.

인터넷 상에 검색을 하다보면 MyBatis 를 위해서는 별도의 설정이 필요하거나 불가능하다는 의견들이 간혹 보이는데, 이는 잘못된 정보다.

단, Mapper 인터페이스 메소드에는 @Cacheable 이나 @CacheEvict 와 같은 명시가 동작하지 않고 오히려 스프링이 기동 중 에러를 뿜는다.

별다른 추가 설정 없이 Mapper 의 메소드 단위로 데이터를 캐싱하고자 한다면 Interface 인 Mapper 를 구현한 클래스 (고전 iBatis 처럼 sqlMapper 를 인젝션받아 구현하는..) 를 사용해야 한다.

\- 역시 검색을 하다보면 Interface 의 구현체의 메소드에만 어노테이션 명시가 동작한다는 정보가 간혹 있는데, 적어도 최신 버전인 2.10 에서는 굳이 Interface 의 구현체가 아니더라도 메소드 레벨이면 어노테이션 명시가 가능하다.

\- 기본 설정인 proxy 모드 (annotation-driven mode="proxy") 로 어노테이션 명시를 사용하면 Spring AOP 가 가지는 제한 사항을 동일하게 가진다. 

즉 self-invocation 메소드에 어노테이션을 명시해도 캐싱이 동작하지 않는다. 이는 service 에 service 빈을 인젝션 받아 사용하는 꼼수를 쓸 수도 있지만, 좀 더 유연하고 타이트한 캐싱 시나리오를 설계하고 싶다면 aspectj 모드를 사용해야 한다.



## 캐시 저장 공간 설정

@Cacheable 이나 @CacheEvict 를 사용하면서 가장 중요한 포인트는 데이터가 캐싱될 (value 로 지정할 대표 명칭 (alias) ) 공간이다.

value 옵션으로 지정한 각 저장 공간들은 Spring 환경 설정에서 지정한 ehcache.xml 파일에서 공간 별로 여러 옵션의 설정이 가능하다.

(저장공간 설정 뿐 아니라 전체적인 캐시 매니저의 방침에 대해 설정할 수 있다. 다만 일일이 설정들을 찾아보기엔 다소 무리일 듯)

- [설정 가이드](https://www.google.co.kr/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&ved=0ahUKEwjboN77lfrLAhVC3qYKHSVpDDYQFgglMAI&url=http%3A%2F%2Fwww.ehcache.org%2Fgenerated%2F2.10.1%2Fhtml%2Fehc-all%2F&usg=AFQjCNFrS1hCHCCfKmwSclz13wIyXGpIFQ&sig2=h30hGIx1Y_KGynPPgOgfpQ&bvm=bv.118443451,d.dGY)

- [설정 및 설명 예제](http://www.ehcache.org/ehcache.xml)

- [CacheConfiguration JavaDoc](http://www.ehcache.org/apidocs/2.10.1/net/sf/ehcache/config/CacheConfiguration.html)

```
<cache name="normal" 
    maxEntriesLocalHeap="10000"
    eternal="false" 
    timeToIdleSeconds="300"
    timeToLiveSeconds="600"
    overflowToDisk="false" 
    memoryStoreEvictionPolicy="LFU" 
    transactionalMode="off">
</cache>
```

매우 기본적인 옵션들만 간단하게 설펴보면 다음과 같다.

eternal - 캐싱 데이터를 영원히 유지할 것인지 여부

maxEntriesLocalHeap - 힙 메모리에 보유할 최대 데이터 갯수

overflowToDisk - 메모리 저장 공간이 부족할 때 Disk 사용 여부

timeToIdelSeconds - 저장된 데이터가 지정된 시간 (초) 동안 재호출되지 않으면 휘발됨

timeToLiveSeconds - 한번 저장된 데이터의 최대 저장 유지 시간 (초)

memoryStoreEvictionPolicy - 저장 공간이 꽉 찰 경우 데이터를 제거하는 규칙 지정.

LRU - 데이터의 접근 시점을 기준으로 최근 접근 시점이 오래된 데이터부터 삭제

LFU - 데이터의 이용 빈도 수를 기준으로 이용 빈도가 가장 낮은 것부터 삭제

FIFO - 먼저 저장된 데이터를 우선 삭제

캐싱될 데이터의 양이 많고, @Cacheable 로 저장된 캐시가 @CacheEvict 로 제거될 명확한 시나리오가 수립되어 있다면 여러 메소드 or 클래스에서 공통의 캐시 공간을 공유할 수 있겠지만, 굳이 복잡한 캐싱 데이터 구조를 가져야 할 제약 환경이 아니라면 각 Case 별로 캐싱 공간을 별도 지정하는 것이 관리 측면에서 더 낫지 않을까 싶다. (대규모 캐싱이 필요한 시스템은 요즘 Redis 가 대세이니 Ehcache 를 사용하는 대부분의 Case 에서는 무방하지 않을까? 단지 큰 물에서 놀아본 경험이 없는 개발자의 근거 없는 추측일 뿐이니 깊게 받아들이면 오히려 위험하다.)



## 기타

매우 기본적인 Ehcache 적용 방법에 대해 간단히 정리해 보았다.

보다 심층적인 활용을 위해서는 Cache 모니터링 관련 설정이나 다른 인스턴스간의 캐싱 데이터 동기화에 대한 설정을 추가로 확인해봐야 할 듯.

다만 모니터링은 유용하게 쓸 수 있을 것 같지만, 

인스턴스간 캐싱 동기화가 필요한 수준이라면 차라리 Redis 를 활용하는게 더 낫지 않을까 싶다.

Ehcache 심화 활용을 위한 비용과 Redis 를 새로 적용하는 비용을 굳이 따지자면 전자가 더 낮긴 하겠지만, 후일을 생각한다면..