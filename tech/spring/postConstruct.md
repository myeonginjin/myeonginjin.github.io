# 정적 메서드에서 스프링 빈을 사용할 수 있을까?

스프링에서는 대부분의 의존성이 `@Autowired`, `@Value`, `@Inject` 등을 통해

**인스턴스 기반으로 주입**됩니다. 하지만 간혹 **static 메서드나 static 필드에서
스프링 빈이 필요**한 경우가 생깁니다.

흔히 유틸 클래스의 메서드는 static으로 선언한 경험이 있을 겁니다. 유틸 메서드들은 보통 값 변환, 간단한 계산, 포맷 변형 등의 작업만 수행하지 내부의 무언가 상태를 저장하거나 관리하지 않습니다. 따라서 굳이 객체를 만들 필요가 없습니다. 애플리케이션 전역에서 클래스 단위로 직접 호출되어 간단한 역할만 수행하는 것이 더 효율적이기 때문입니다.

하지만 이러한 정적 영역에서는 스프링 빈을 사용할 수 없습니다. 간단한 예시를 통해 왜 사용할 수 없는지, 그럼 어떻게 해결할 수 있는지 알아보겠습니다.

## 🧐 왜 사용할 수 없을까?

- static 필드는 **클래스 로딩 시점에 JVM 메모리에 올라감**
  (즉, Spring 컨테이너가 생성되기 전)
- `@Autowired`는 Spring이 **빈을 생성한 이후**에 의존성을 주입
- Mybatis의 @Mapper 또한 동일하게 스프링빈 등록 시점에서야 메모리에 로드됨
- 그래서 static 필드는 `@Autowired`의 의존성 주입 대상이 될 수 없음 → 순서상 주입 불가

한 줄로 말해서, 정적 필드가 자바 클래스 로더에 의해 인스턴스화 될 때 아직 Spring Context는 로드되지 않았기 때문에 Spring 프레임워크에 의해 관리되는 @AutoWired가 동작하지 않는 것이다.

그럼 해결책은 무엇이 있을까요 ?

아래 예시처럼  `static` 메서드 안에서 DB나 설정 값을 조회해야 할 때가 있을 수 있습니다.

이때 선택할 수 있는 해결책 중 하나는 바로 `@PostConstruct`를 이용한 **정적 필드 초기화 방식**입니다.

---

## ✅ 예제: `@PostConstruct`를 활용한 정적 필드 초기화

```java
@Component
public class ExamUtil {

    @Autowired
    CommonMapper commonMapper;

    private static CommonMapper mapper;

    @PostConstruct
    private void initStaticDao() {
        mapper = this.commonMapper;
    }

    public static String getInfo(String str) {
        try {
            return mapper.getResultCodeAndMsg(str);
        } catch (Exception e) {
            return "쿼리오류";
        }
    }
}

```

---

## ✅ 이 구조는 왜 필요할까?
- `@Autowired`는 **인스턴스 필드**에만 작동합니다.
- 하지만 `getInfo()`는 **static 메서드**이므로 인스턴스 필드인 `commonMapper`에 직접 접근할 수 없습니다.
- 따라서 인스턴스가 생성된 이후, `@PostConstruct`를 이용해 주입된 `commonMapper`를
  정적 필드 `mapper`에 복사합니다.

---

## 🔄 스프링 빈 라이프사이클과의 관계

Spring에서의 객체 생성 순서는 다음과 같습니다:

1. 클래스 로딩
2. 객체 생성 (`new ExamUtil()`)
3. 의존성 주입 (`@Autowired`)
4. **초기화 단계** → `@PostConstruct` 호출
5. 빈 등록 완료

> @PostConstruct는 주입이 완료된 직후 실행되므로
>
>
> 이 시점에서 정적 필드에 안전하게 값을 할당할 수 있습니다.
>

---

## ✅ 언제 이런 구조를 쓰게 될까?

- 정적 유틸 클래스에서 DB 조회나 설정 정보 접근이 필요할 때
- 예외 메시지 포맷팅, 코드 매핑 등 공통 처리를 static으로 구현해놓은 경우
- 레거시 코드와의 통합에서 어쩔 수 없이 static 메서드를 써야 하는 경우

---

## ⚖️ 장점과 단점

| 항목 | 장점 | 단점 |
| --- | --- | --- |
| `@PostConstruct` + static 필드 | static에서도 DI 가능 | 구조가 우회적이고 테스트 어려움 |
| 인스턴스 기반 설계 | DI 원칙 준수, 테스트 편리 | 기존 static 구조 변경 필요 |
| `ApplicationContext.getBean()` 직접 호출 | 어디서나 빈 사용 가능 | Spring 강결합, 유연성 하락 |

---

## 🚫 대안은 없을까?

### 1. **정적 구조를 없애고 인스턴스 기반으로 리팩토링**

가장 이상적인 방법입니다.

DI가 가능하고, 테스트도 쉬워지고, 유지보수도 편해집니다.

### 2. **ApplicationContext 사용**

정적 메서드 내에서 아래처럼 사용 가능하지만,

DI 원칙 위배이고 코드가 스프링에 강하게 의존하게 됩니다.

```java
CommonMapper mapper = ApplicationContextProvider.getBean(CommonMapper.class);

```

> ApplicationContextAware, BeanFactory, ContextHolder 등을 구현해야 함
>

### 3. **구조 개선**

- static 메서드가 필요한 이유가 없다면, 인스턴스 메서드로 변경
- 필요한 메서드를 빈으로 추출해 직접 주입

---

## ✅ 결론

- `@PostConstruct`는 **의존성 주입 이후 초기화**에 사용할 수 있는 스프링 생명주기 어노테이션입니다.
- static 메서드에서 DI가 필요한 상황에서는 정적 필드 초기화를 위해 자주 사용됩니다.
- 하지만 이 방식은 우회적이며 테스트/유지보수에 불리할 수 있습니다.
- 가능한 경우 구조를 개선하거나, static 대신 인스턴스를 사용하는 쪽이 좋습니다.

---

무엇보다 이러한 구조는 스프링 프레임워크 내에서 개발을 하면서, Spring의 제어를 벗어난 방법을 사용하는 것이니 가급적 사용을 자제하는 것이 좋을 수 있습니다.