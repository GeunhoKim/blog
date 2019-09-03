---
author: "Kim, Geunho"
date: 2019-09-03
title: "Testcontainers - 컨테이너 기반 테스트 환경"
---


2017년 중순, 회사 내에서 완전히 새로운 개발/운영 플랫폼 도입을 위한 TF 진행을 하면서 다양한 레퍼런스를 조사했었다.  
그때 발견한 한 샘플 프로젝트가 [.NET Core](https://dotnet.microsoft.com)로 구현된 웹 서비스였는데, Visual Studio 2017 솔루션으로 구성되어 각 프로젝트를 모두 컨테이너화까지 해놓은 좋은 예제였다.  

재미있었던 것은 한번 빌드하면 자동으로 필요한 서브 시스템들(mongodb, redis, rabbitmq, ...)을 docker compose로 구성해서 빌드가 실행될 때 컨테이너가 함께 로컬에 띄워지도록 한 것이었다.  
각 웹 프로젝트들도 빌드 타임에 도커 이미지까지 빌드한 후 컨테이너를 띄워서 컨테이너 내 프로세스를 디버그하는 방식이었는데, 당시에는 신선한 충격이었다.  

# 테스트 환경
일관된 테스트 환경을 갖는 것은 어려운 일이다. 특히 개발 테스트 환경은 개발 환경에서 동작하는 별도의 데이터 소스를 바라보도록 구성되고, 유기적으로 연결된 각 서비스들은 서로의 개발 환경을 바라보고 있다.  
개발 환경에 띄워진 서비스들은 적절한 개발 브랜치 전략과 CI[^1] 구성이 되어있지 않으면 쉽게 망가질 수 있다. 특히 데이터 소스는 언제든 아무도 모르게 프로세스가 중단될 수도 있고 RDB의 경우에는 테스트용 데이터가 망가지거나, DB의 저장 프로시져가 바뀔 수도 있다.  

이렇듯, 우리가 테스트하고자 하는 하나의 서비스는 독립적으로 동작하지 않고 여러 서비스와 데이터 소스와 관계를 맺기 때문에 테스트하기가 쉽지 않다.  

# 컨테이너 기반 테스트 환경
"컨테이너 기반 테스트 환경"은 당시에 샘플 프로젝트를 보고 착안한 생각이었다. 상당히 러프한 아이디어였는데, 그 구조는 다음과 같다.

![테스트 환경 관리 시스템](/container-based-test-env-1.png) _그림 1. 서비스 관리자는 이미지와 테스트용 volume 데이터를 관리한다._

![로컬 개발 환경](/container-based-test-env-2.png) _그림 2. 로컬 개발 환경_

그림 1을 보면, 서비스 관리자가 도커 이미지와 테스트용 volume 데이터를 관리한다. 이때 테스트 환경 관리 시스템을 통해서 권한을 제어한다.  
그림 2에서는 어떻게 로컬 개발 환경이 구성되는지 알 수 있는데, 개발자는 서로 통신하는 외부 서비스를 로컬에 이미지로 내려받아 컨테이너로 띄워 바라보면서 개발을 하게 되고, 데이터 소스인 경우 미리 관리되고 있는 volume 데이터를 로컬에 복제해서 사용한다. 개발이 완료되어 개발 브랜치에 새로운 커밋이 생성되면 CI 환경에 의해 새로운 버전의 도커 이미지로 빌드되어 테스트 환경 관리 시스템의 레지스트리에 저장된다.  

이렇게 구성이 된다면 로컬 개발 환경에서 다른 외부 서비스와 데이터 소스가 일관되게 유지되어 서비스 환경과 무관하게, 개발자가 담당하는 서비스 개발에만 집중할 수 있게 된다.  
만약 하나의 서비스가 여러 개의 하위 서비스로 구성되어 있다면 하위 서비스의 개발 환경을 직접 구성하면서 보내는 시간을 크게 단축할 수 있다.  

사실 이러한 환경이 제대로 구성이 되려면 여러가지 문제가 발생하는데,

1. 모든 서비스의 도커화
2. 테스트용 volume 데이터 관리에 대한 책임 
3. 개별 서비스의 소스코드 프로젝트에 외부 서비스와 데이터 소스를 띄우기 위한 스크립트 필요
4. 3번 스크립트의 유지보수 필요

모든 서비스를 도커화하고 빌드 스크립트를 유지보수 하는 일은 당시 환경에서는 현실적이지 않아 보였다. 윈도우 환경에서는 도커에 대한 지원이 많이 부족했고, 소스코드 외부에서 스크립트를 통해 환경을 구성하는 방식은 테스트 코드 내에서 통합해서 사용하기에는 어려웠기 때문이다.  

# Testcontainers
[Testcontainers](https://www.testcontainers.org)는 테스트 코드 내에서 컨테이너를 생성하고 제어하는 기능을 제공하는 자바 라이브러리이다. .NET, Scala, Python, Nodejs, Go 등 다른 환경을 위한 라이브러리도 작성되었으나, 공식 사이트에 정식으로 공개하지는 않았다.[^2]  

다음과 같은 목적으로 개발되었는데,

* Data access layer integration tests
* Application integration tests
* UI/Acceptance tests

앞서 소개한 아이디어의 테스트 코드에서 통합을 손쉽게 할 수 있을 것으로 보인다.  
다음은 JUnit4를 이용한 테스트 코드이다. `RedisBackedCache` 레디스 클라이언트를 테스트하는 것을 살펴볼 수 있다. 

```java
public class RedisBackedCacheIntTest {
    private RedisBackedCache underTest;

    @Rule
    //(1) 레디스 서버 컨테이너를 생성해서 로컬에 띄우고
    public GenericContainer redis = new GenericContainer<>("redis:5.0.3-alpine")
                                            .withExposedPorts(6379);

    @Before
    public void setUp() {
        //(2) 컨테이너의 IP, port를 가져와서
        String address = redis.getContainerIpAddress();
        Integer port = redis.getFirstMappedPort();

        //(3) `RedisBackedCache` 인스턴스를 생성
        underTest = new RedisBackedCache(address, port);
    }

    @Test
    public void testSimplePutAndGet() {
        //(4) put(), get() 메서드 테스트
        underTest.put("test", "example");

        String retrieved = underTest.get("test");
        assertEquals("example", retrieved);
    }
}
```


[^1]: [Continuous Integration](https://ko.wikipedia.org/wiki/%EC%A7%80%EC%86%8D%EC%A0%81_%ED%86%B5%ED%95%A9). 
[^2]: [GitHub testcontainser organization](https://github.com/testcontainers)을 참고한다.