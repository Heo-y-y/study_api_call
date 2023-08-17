# 알고리즘 API를 이용한 OpenFeign을 적용해보기

## 학습 설명

테스트 프로젝트의 알고리즘 API를 **Spring Cloud openFeign**으로 받아 알고리즘을 풀고, 테스트 코드까지 작성해보는 학습을 진행했다.

- GitHub: <https://github.com/yet-another-study-group/study_api_call>

## OpenFeign이란?
- Code: <https://github.com/Heo-y-y/study_api_call/tree/main/hyy/src/main/java/com/study/hyy>

OpenFeign은 Netflix에 의해 처음 만들어진 Declarative(선언적인) HTTP Client 도구로써, 외부 API 호출을 쉽게할 수 있도록 도와준다. 

OpenFeignd의 변화

`Netflix OSS → Spring Cloud Netflix → Open Feign → Spring Cloud Open Feign`

장점

- 인터페이스와 어노테이션 기반으로 작성할 코드가 줄어든다.
- 익숙한 Spring MVC 어노테이션으로 개발이 가능하다.
- 다른 Spring Cloud 기술들(Eureka, Circuit Breaker, LoadBalancer) 과의 통합이 쉽다.

단점 & 한계

- 기본 Http Client가 Http2를 지원하지 않는다(Http Client에 대한 추가 설정이 필요함).
- 공식적으로 Reactive 모델을 지원하지 않는다(비공식 오픈소스 라이브러리로 사용 가능함).
- 경우에 따라 애플리케이션이 실행될 대 초기화 에러가 발생할 수 있다(Object Provider로 대응이 필요함).
- 테스트 도구를 제공하지 않는다(별도의 설정 파일을 작성하여 대응이 필요함).

---

## OpenFeign 적용

1. **의존성 추가**

```java
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	compileOnly 'org.projectlombok:lombok'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	implementation group: 'org.springframework.cloud', name: 'spring-cloud-starter-openfeign', version: '3.1.3'
	implementation 'org.apache.commons:commons-math3:3.6.1'
}
```

여기서 주의점은 Spring Cloud는 Spring Boot 버전과 호환되는 버전을 사용해야 한다.

[**Spring Cloud 문서**](https://spring.io/projects/spring-cloud)를 참고하기 바란다.

2. **OpenFeign 활성화**

OpenFeign을 활성화하려면 `@EnableFeignClients` 어노테이션을 붙여주면 된다.

기본적으로는 main 클래스에 붙여준다.

```java
@SpringBootApplication
@EnableFeignClients
public class HyyApplication {

	public static void main(String[] args) {
		SpringApplication.run(HyyApplication.class, args);
	}
}
```

여기서 궁금한게 있다 왜 main 클래스에 적용을 할까?

@EnableFeignClients 어노테이션은 하위 클래스를 탐색하면서 @FeignClient를 찾아 구현체를 생성하는 역할을 한다.

위 글을 보면 알겠지만 main 클래스에 적용하면 하위 클래스를 다 탐색하기 때문에 기본적으로 main 클래스에 적용하는 것 같다.

하지만 **main 클래스에 @EnableFeignClients 어노테이션을 붙여주는 것은 SpringBoot가 제공하는 테스트에 영향을 줄 수 있다.** 그러므로 별도의 **Config** 파일로 만들어주는 것이 좋다.

별도의 파일로 설정할 경우 feign 인터페이스들의 위치를 반드시 지정해주어야 한다. componentScan 처럼 baseBackes로 지정해주거나, clients로 직접 클래스들을 지정해줘도 된다.

일반적으로 아래와 같이 패키지로 지정한다.

```java
@Configuration
@EnableFeignClients("com.study.hyy")
class OpenFeignConfig {

}
```

3. **Feign Client 구현**
- 적용할 외부 API 코드

```kotlin
@RestController
@RequestMapping("/api")
class TestController(
    private val testService: TestService
) {

    @GetMapping( "/cal/{x}")
    fun calculateQuadraticEquation(@PathVariable x: Int) = testService.calQuadraticEquation(x)

}
```

API 호출을 수행할 클라이언트는 인터페이스에 ``@FeignClient`` 어노테이션을 설정한다. **name에는 클라이언트의 이름을 설정**하고, **url에는 호출할 api의 주소를 설정**한다.

```java
@FeignClient(name = "YkFeignClient", url="http://localhost:8081")
public interface YkFeignClient {
    @GetMapping(value = "/api/cal/{x}")
    int getValue(@PathVariable("x") int x);
}
```

4. **FeinÇlient config 정보 설정**

Config 클래슬르 생성하고 필요한 각 설정 정보를 아래와 같이 셋팅 가능하다.

✅ Config 클래스를 따로 생성하지 않아도 아래 Bean들은 기본적으로 제공된다.

```java
Spring Cloud OpenFeign provides the following beans by default for feign (BeanType beanName: ClassName):
Decoder feignDecoder: ResponseEntityDecoder (which wraps a SpringDecoder)
Encoder feignEncoder: SpringEncoder
Logger feignLogger: Slf4jLogger
MicrometerCapability micrometerCapability: If feign-micrometer is on the classpath and MeterRegistry is available
CachingCapability cachingCapability: If @EnableCaching annotation is used. Can be disabled via feign.cache.enabled.
Contract feignContract: SpringMvcContract
Feign.Builder feignBuilder: FeignCircuitBreaker.Builder
Client feignClient: If Spring Cloud LoadBalancer is on the classpath, FeignBlockingLoadBalancerClient is used. If none of them is on the classpath, the default feign client is used.
```

❎ 아래 bean들은 기본으로 제공되지 않고 필요시 config 클래스에 생성해줘야 한다.

```java
Logger.Level
Retryer
ErrorDecoder
Request.Options
Collection<RequestInterceptor>
SetterFactory
QueryMapEncoder
Capability (MicrometerCapability and CachingCapability are provided by default)
```

---


# Mockito를 적용한 단위 테스트
- Code: <https://github.com/Heo-y-y/study_api_call/blob/main/hyy/src/test/java/com/study/hyy/service/FeignServiceTest.java>

## Mockito란?

Mock을 지원하는 프레임워크로, Spring Boot Test에서 사용하는 JUnit 위에서 동작하며 Mock 객체를 만들고 관리하며 검증할 수 있는 방법을 제공해주는 라이브러리다.

즉 Mockito는 개발자가 동작을 직접 제어할 수 있는 **가짜 객체**를 지원하는 테스트 프레임워크이다.

### Test Double이란?

테스트할 때 실제 객체의 역할을 대신 해주는 객체이다.

즉 테스트하려는 객체와 연관된 객체를 사용하기 어렵고 모호할 때 대신 해주는 객체를 Test Double이라고 한다.

**테스트 더블의 종류**

- Dummy
    - 가장 기본적인 테스트 더블이다.
    - 인스턴스화 된 객체가 필요하지만 기능은 필요하지 않은 경우에 사용한다.
    - Dummy 객체의 메서드가 호출되었을 때 정상 동작은 보장하지 않는다.
    - 객체는 전달되지만 사용되지 않는 객체이다.
    
    정리하면 인스턴스화된 객체가 필요해서 구현한 가짜 객체일 뿐이고, 생성된 Dummy 객체는 정상적인 동작을 보장하지 않는다.
    
- Fake
    - 복잡한 로직이나 객체 내부에서 필요로 하는 다른 외부 객체들의 동작을 단순화하여 구현한 객체이다.
    - 동작의 구현을 가지고 있지만 실제 프로덕션에는 적합하지 않은 객체이다.
    
    정리하면 동작은 하지만 실제 사용되는 객체처럼 정교하게 동작은 하지 않는 객체이다.
    
- Stub
    - Dummy 객체가 실제로 동작하는 것 처럼 보이게 만들어 놓은 객체이다.
    - 인터페이스 또는 기본 클래스가 최소한으로 구현된 상태이다.
    - 테스트에서 호출된 요청에 대해 미리 준비해둔 결과를 제공한다.
    
    정리하면 테스트를 위해 프로그래밍된 내용에 대해서만 준비된 결과를 제공하는 객체이다.
    
- Spy
    - Stub의 역할을 가지면서 호출된 내용에 대해 약간의 정보를 기록한다.
    - 테스트 더블로 구현된 객체에 자기 자신이 호출 되었을 때 확인이 필요한 부분을 기록하도록 구현한다.
    - 실제 객체처럼 동작시킬 수도 있고, 필요한 부분에 대해서는 Sutb으로 만들어서 동작을 지정할 수도 있다.
    
    정리하면 실제 객체로도 사용 가능하고, Stub 객체로도 활용할 수 있으며 필요한 경우 특정 메서드가 제대로 호출되었는지 여부를 확인할 수 있다.
    
- Mock
    - 호출에 대한 기대를 명세하고 내용에 따라 동작하도록 프로그래밍된 객체이다.

---

## 필요한 Dependency

SpringBoot를 사용하면 Gradle에 Spring-boot-starter-test를 추가하게 된다.

그안에 mockito-core, mockito-junit-jupiter 패키지가 포함되어있다.

<img width="471" alt="스크린샷 2023-06-28 오전 12 12 25" src="https://github.com/Heo-y-y/chicken/assets/112863029/34241020-eb84-4416-956d-f047c9d182ae">

---

## Mockito를 적용한 Service 클래스 테스트

아래 테스트 코드를 클래스 단위로 테스트 해보겠다.

```java
@Service
@RequiredArgsConstructor
public class FeignService {
    private final YkFeignClient ykFeignClient;

    public ABResponseDto calculateAB() {
        int x1 = 14;
        int result1 = ykFeignClient.getValue(x1);
        int x2 = 40;
        int result2 = ykFeignClient.getValue(x2);
        double[][] coefficients = {{x1 * x1, x1}, {x2 * x2, x2}};
        double[] constants = {result1, result2};
        RealMatrix matrix = MatrixUtils.createRealMatrix(coefficients);
        QRDecomposition solver = new QRDecomposition(matrix);
        RealVector constantsVector = MatrixUtils.createRealVector(constants);

        if (!solver.getSolver().isNonSingular())
            throw new IllegalArgumentException("방정식의 시스템에 유일한 해가 없습니다.");
        RealVector solution = solver.getSolver().solve(constantsVector);
        int a = (int) Math.round(solution.getEntry(0));
        int b = (int) Math.round(solution.getEntry(1));
        return new ABResponseDto(a, b);
    }
```

### 단위 테스트(Unit Test) 작성 준비

우선 JUniit5와 Mockito를 연동하기 위해서는 ``@ExtendWith(MockitoExtension.class)``를 적용해야 한다.

```java
@ExtendWith(MockitoExtension.class)
class FeignServiceTest {
}
```

그 다음 의존성 주입을 해주어야 한다. 먼저 테스트 대상인 FeignService에는 **가짜 객체 주입을 위한 @InjectMocks**를 붙여주고, YkFeignClient에는 **가짜 객체 생성을 위해 @Mock** 어노테이션을 붙여준다.

```java
@ExtendWith(MockitoExtension.class)
class FeignServiceTest {
    @InjectMocks
    FeignService feignService;

    @Mock
    YkFeignClient ykFeignClient;
}
```

### FeignService 성공 테스트

```java
@ExtendWith(MockitoExtension.class)
class FeignServiceTest {
    @InjectMocks
    FeignService feignService;

    @Mock
    YkFeignClient ykFeignClient;

    @Test
    @DisplayName("과제1 calculateAB() 테스트코드")
    void successCalculateAB() {
        given(ykFeignClient.getValue(14)).willReturn(2912);
        given(ykFeignClient.getValue(40)).willReturn(22880);

        ABResponseDto abResponseDto = feignService.calculateAB();

        then(ykFeignClient).should(times(1)).getValue(14);
        then(ykFeignClient).should(times(1)).getValue(40);
        then(ykFeignClient).shouldHaveNoMoreInteractions();

        assertThat(abResponseDto.getA()).isEqualTo(14);
        assertThat(abResponseDto.getB()).isEqualTo(12);
    }
}
```

여기서는 ``BDDMocktio``를 사용했다.

Mockito에서 BDDMockito라는 패키지를 추가하면 사용가능하고, 기능적으로는 별 차이가 없고 **given/when/then**을 지키면서 쓸 수 있도록 틀을 바꿔주는 라이브러리다.
