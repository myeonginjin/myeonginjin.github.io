## 컨트롤러의 공통 관심사 로직을 걷어내 보자 (1편)



서버 개발자라면 API를 개발할 때, 비즈니스 로직에만 집중하고 싶은 욕구가 있습니다. 때문에 스프링 AOP, Lombok 등 다양한 라이브러리들이 개발자들이  반복적이고 지루한 작업들 하지 않을 수 있도록 지원하고 있습니다.<br> <br>
만약, 모든 API에 공통적으로 적용되는 로직(인증, 에러처리 등)을 매 개발마다 작성해주어야 한다면 그 지루함은 배가 될 것입니다.
오늘은 제가 개발하고 있는 플랫폼의 컨트롤러 계층에서 보이는 공통관심사들을 어떻게 모듈화했는지 풀어보겠습니다.


### 📌 배경

* **프로젝트**: PG사 가맹점 관리자용 플랫폼 및 사내 백오피스
* **상황**: 컨트롤러 내에 중복된 인증·무결성 검증 로직이 산재

    * 서명 기반 인증 서비스 객체 호출 로직
    * 서비스 객체의 요청 DTO 생성 및 응답값 처리 로직
* **문제**

    1. **코드 중복**: 컨트롤러마다 7\~10줄씩 반복
    2. **가독성 저하**: 핵심 비즈니스 로직이 인증 코드에 묻혀 관리 어려움
    3. **유지보수 난이도**: 인증 정책 변경 시 모든 핸들러 수정
    4. **테스트 곤란**: 검증 로직과 비즈니스 로직이 뒤섞여 단위 테스트 불가능
       <br>

> **“인증 로직이 핵심 비즈니스 로직보다 더 길다”**
> → **관심사 분리**와 **중앙집중 처리**가 시급했습니다.  <br>

오늘은 이 레거시 컨트롤러의 지루하고 반복적인 API 유효성 검증 로직을 걷어내는 과정을 풀어보겠습니다.<br>
**단, 테이블 구조와 코드는 보안상 예시로 대체하겠습니다.**

---

## 🚩 문제 정의

### 1. 컨트롤러 레거시 코드<br>
**🚨 보안 상 예시 로직으로 대체합니다 🚨**

```java
@PostMapping("/v2/pay")
public String doPay(@Valid ExamParam param, Errors error) {
    try {
    // 파라미터 검증
    BaseTx validateParam = ctx.getBean("validateParamService", BaseTx.class);
    Map errMap = validateParam.executeMap(Map.of("error", error));
    if (!"00".equals(errMap.get("resultCode"))) return toJson(errMap);

    // 서명 확인
    Map dto = new SafeMap(new HashMap<String, Object>());
    dto.put("id", param.getid());
    dto.put("timestamp", param.getTimestamp());
    dto.put("signature", param.getSignature());
    BaseTx validateSignatureService = (BaseTx) ctx.getBean("validateSignatureService");
    errMap = validateSignatureService.executeMap(dto);
    if(!errMap.get("resultCode").equals("00")){ return gson.toJson(errMap.getMap()); }

    // 비즈니스 로직
    BaseTx business = ctx.getBean("payService", BaseTx.class);
    Map result = business.executeMap(toMap(param));

    } catch(Exception e) {
        result.put("resultCode", "99");
    }
    return toJson(result);
}
```

* 동일한 블록이 한 컨들롤러당 10여 개 메서드에 복붙
* 인증 로직 변경 시 **전수 검사** 필요

### 2. 개선 방향

* **인증·무결성 검증**을 **Tomcat 레벨**의 **Servlet Filter**로 중앙화
* **Spring Context** 영역 이전의 컨테이너 입구에서 모든 요청을 **일관되게 제어**
* **컨트롤러**는 **파라미터 바인딩 & 비즈니스 호출**만 담당

---

## 🛠️ 설계 & 의사결정

1. **Filter + Validator** 패턴

    * `AuthFilter`: HTTP 요청마다 등록된 `AuthValidator` 순회
    * `AuthValidator`: `supports()`, `validate()`, `getErrorCd()/getErrorMsg()`
2. **Credentials 객체**

    * `CommonCredentials`, `MerchantCredentials`로 **헤더 파싱** 및 **검증 로직 캡슐화**
3. **시크릿 키 캐싱**

    * `ApiSecretKeyProvider`: `volatile` + `@Scheduled` reload → **Cache-Aside**
4. **에러 전략**

    * 헤더 누락
    * 타임스탬프 오류 (인증 요청 시각, 현 서버 시간으로부터 ±5분)
    * 서명 불일치 (SHA256 해시값 불일치)
    * HTTP 401 응답

---

## 🔍 구현 요약

### 1. FilterConfig

```java
@Bean
public FilterRegistrationBean<AuthFilter> authFilterReg(List<AuthValidator> vals) {
    return new FilterRegistrationBean<>(new AuthFilter(vals)) {{
        addUrlPatterns("/v2/*"); setOrder(2);
    }};
}

// Validator 등록
@Bean AuthValidator commonAuthKeyValidatorReg(ApiSecretKeyProvider p) { 
    return new CommonAuthKeyValidator(p);
}
@Bean AuthValidator merchantAuthKeyValidator(CommonMapper m) {
    return new MerchantAuthKeyValidator(m);
}
```

### 2. AuthFilter

```java
@RequiredArgsConstructor
public class AuthFilter implements Filter {
  private final List<AuthValidator> validators;

  @Override
  public void doFilter(ServletRequest rq, ServletResponse rs, FilterChain chain)
        throws IOException, ServletException {
    HttpServletRequest  req = (HttpServletRequest) rq;
    HttpServletResponse res = (HttpServletResponse) rs;

    for (AuthValidator v : validators) {
        if (v.supports(request) == false) {
            continue;
        }
        if (v. validate(request) == false) {
            setErrorResponse(response, v);
            return;  // 체인 중단
        }
    }
    chain.doFilter(req, res); //체인 통과
  }
}
```

### 3. Credentials 캡슐화

```java
public class CommonCredentials {
  private final String timestamp, signature;
  private final ApiSecretKeyProvider keyProvider;
  private static final long DRIFT = 5*60;

  public boolean hasMissingRequiredHeaders() { … }
  public boolean isInvalidTimestamp() { return between(LocalDateTime.now(), timestamp) > DRIFT }

  public boolean isInvalidSignature() {
    String s = keyProvider.getSecretKey().orElse("");
    return !SHA256.encoding(s+timestamp).equals(signature);
  }
}
```



### 4. Validator 구현 예

```java
@Component @RequiredArgsConstructor
public class CommonAuthKeyValidator implements AuthValidator {
  private final ApiSecretKeyProvider provider;
  private String errorCd;

  @Override public boolean supports(HttpServletRequest r) {
    return r.getRequestURI().startsWith("/v2/pay");
  }
  @Override public boolean validate(HttpServletRequest r) {
    var creds = new CommonCredentials(
                  r.getHeader("X-Request-Timestamp"),
                  r.getHeader("X-Api-Signature"),
                  provider);
    if (creds.hasMissingRequiredHeaders())     {errorCd="01"; return false;}
    if (creds.isInvalidTimestamp()) {errorCd="02"; return false;}
    if (creds.isInvalidSignature()) {errorCd="03"; return false;}
    return true;
  }
  // getErrorCd(), getErrorMsg()…
}
```


### 5. 개선된 컨트롤러

```java
@PostMapping("/v2/pay")
public String doPay(@Valid ExamParam param, Errors error) {
    try {
    // 파라미터 검증
    BaseTx validateParam = ctx.getBean("validateParamService", BaseTx.class);
    Map errMap = validateParam.executeMap(Map.of("error", error));
    if (!"00".equals(errMap.get("resultCode"))) return toJson(errMap);

    // 비즈니스 로직
    BaseTx business = ctx.getBean("payService", BaseTx.class);
    Map result = business.executeMap(toMap(param));

    } catch(Exception e) {
        result.put("resultCode", "99");
    }
    return toJson(result);
}
```

---

## 🎉 개선 효과

| 구분       | 기존       | 개선 후                                |
| -------- | -------- | ----------------------------------- |
| 컨트롤러 길이  | 500줄 내외     | 400줄 내외                              |
| 인증 코드 중복 | 10곳 이상   | 0곳 (Filter로 일원화)                    |
| DB 키 조회  | 요청당 N회   | 캐시 + 주기적 갱신 1회/주                    |
| 응답 시간    | 평균 120ms | 평균 100ms (–16%)                      |
| 테스트 커버리지 | low      | 95%↑ (Credentials·Validator 단위 테스트) |
| 유지보수 비용  | 높음       | 낮음 (정책 변경 시 Validator만 수정)          |

---

## 🔮 남은 과제 & 다음 단계

1. **파라미터 검증(Spring AOP 전환)**

    * `validateParamService` 호출 블록을 `@Valid` + `@ControllerAdvice`로 완전 대체
2. **전역 예외 처리**

    * `@RestControllerAdvice`로 `try-catch` 전면 제거
3. **모니터링 & 알람**

    * Prometheus/Grafana 연동: 인증 실패율, 레이턴시 수집
4. **보안 고도화**

    * Nonce 기반 2차 Replay 방지
    * RSA 전자서명 지원

---

> **“컨트롤러의 깔끔함은 곧 개발 생산성과 서비스 신뢰성의 기본”**

이글이 **인증·검증 로직의 일관된 플랫폼화**를 고민하시는 분들께 실용적인 레퍼런스가 되길 바랍니다. <br>

컨트롤러의 공통 관심사 로직을 걷어내 보자 2편에서는 [🔮 남은 과제 & 다음 단계] 중 한가지를 수행하려고 합니다. <br>
최종 목표는 컨트롤러가 파라미터 바인딩 & 비즈니스 호출의 역할만 하도록 하여, 숨 막히는 현재의 컨트롤러 계층의 가독성을 높이는 것입니다.
