

# Oracle 내장 함수로 문자열을 파싱하고 집합을 만들어보자

최근 PG사 서버개발 업무를 맡으면서, 가맹 점주에게 지급되는 가맹점 관리 시스템에 간단한 기능을 개선하는 개발건을 맡았습니다.

> 거래 내역 조회 시, 조회 조건으로 사용할 수 있는 간편결제(카카오페이 등) 목록을 읽어오는 기능인데요,
> **필수 요건**은 해당 가맹점이 사용하는 간편결제 수단만 읽어오도록 하는 것입니다.

사용하지 않는 간편결제 수단을 조건에 넣어도 결과가 없을 뿐더러, 사용하지 않는 거래 수단을 굳이 보고 싶어하는 점주도 없기 때문입니다.

하지만 대리점(여러 가맹점을 관리하는 사업체)이나 대형법인의 경우, 여러 가맹점의 거래 내역을 한 번에 조회하므로,
**결제 수단 목록은 이 수많은 가맹점 거래 수단의 합집합**이 되어야 했습니다.

바로 이 합집합을 구하는 과정에서, 사용자 경험을 저해하는 속도가 나왔는데요.
오늘은 이 문제의 원인과 해결 과정을 공유해보겠습니다.

**단, 테이블 구조와 코드는 보안상 예시로 대체하겠습니다.**

---

## 🎯 1. 문제의 발단과 배경
### 데이터 구조

* **가맹점 정보** 테이블

    * 가맹점 한 줄당 인증사 코드를 `01:02:03` 식으로 **콜론으로 연결된 문자열**(`간편결제_LST`)로 보관
    * 하나의 그룹 ID(`GID`)에 여러 가맹점(`MID`)이 속할 수 있음

  | MID  | GID | 간편결제_LST       |
    | :--- | :-: |:---------------|
  | A100 |  G1 | `01:02`        |
  | A101 |  G1 | `02:03`        |
  | A102 |  G1 | `01:03:04`     |

### 문제 상황

1. **GID**로 조회 시, 여러 `MID`에서 가져온 결제 수단 문자열을 Java에서 `split`하고 합집합을 구한 뒤, 추가 쿼리로 이름 매핑
2. SQL 호출 2회, Java 파싱·`HashSet` 연산, 네트워크 왕복 비용이 반복되어 대량 가맹점 처리 시 응답이 0.5초 안팎으로 느려짐

**🚨 보안 상 예시 로직으로 대체합니다 🚨**
```java
public List<Map<String,String>> get간편결제ListByGid_Legacy(String gid) {
    // (1) GID로 문자열 리스트 조회 → 1쿼리
    List<String> rawLists = paymentMapper.간편결제수단목록가져오기(gid);

    // (2) Java 문자열 파싱 + HashSet으로 중복 제거
    Set<String> unique = new HashSet<>();
    for (String raw : rawLists) {
        // "01:02:03" → ["01","02","03"]
        String[] parts = raw.split(":");
      ...생략...
    }

    // (3) 코드별 이름 조회 → N쿼리
    List<Map<String,String>> result = new ArrayList<>();
    for (String code : unique) {
      ...생략...
        result.add(Map.of("코드", code, "이름", name));
    }
    return result;
}
```

> **문제점**
>
> * SQL을 두 번 호출하고, Java 파싱이 반복되어 대용량 데이터 처리 시 성능 저하

---

## 🚀 2. 개선 목표

1. SQL 호출 2회 → **1회**로 줄여 왕복 비용 최소화
2. Java 파싱 로직 제거로 CPU·메모리 부하 경감
3. Oracle 내장 함수를 적극 활용해 DB만으로 분리·집계·매핑 처리
4. 목표 응답 시간 **15–20% 단축**

---

## 💡 3. 문제 해결 

### 핵심 SQL 
**🚨 보안 상 예시 쿼리로 대체합니다 🚨**

```xml
<select id="getDistinctItemCodes" parameterType="map" resultType="map">
  SELECT DISTINCT
    -- 1) REGEXP_SUBSTR로 n번째 토큰을 추출 → item_code
    TRIM(REGEXP_SUBSTR(src.code_list, '[^:]+', 1, lvl.COLUMN_VALUE)) AS item_code,
    -- 2) LOOKUP_TABLE과 조인하여 항목명(item_name) 조회
    lk.item_name                                        AS item_name
  FROM (
    -- 원본 문자열 컬럼 가져오기
    SELECT code_list
    FROM source_table
    WHERE pk = #{pk}
      AND code_list IS NOT NULL
  ) src
  -- 3) 계층 쿼리+컬렉션으로 1..토큰수까지 숫자 테이블 생성
  CROSS JOIN TABLE(
    CAST(
      MULTISET(
        SELECT LEVEL
        FROM DUAL
        CONNECT BY LEVEL <= REGEXP_COUNT(src.code_list, ':') + 1
      ) AS SYS.ODCINUMBERLIST
    )
  ) lvl
  -- 4) lookup 테이블과 조인하여 이름 매핑
  JOIN lookup_table lk
    ON lk.category   = 'ITEM'
   AND lk.item_code = TRIM(REGEXP_SUBSTR(src.code_list, '[^:]+', 1, lvl.COLUMN_VALUE))
  -- 5) 중복 제거 + 정렬
  ORDER BY item_code
</select>
```

#### 각 줄별 역할 & 원리

| SQL 부분                                                             | 역할 & 원리                                                  |
| :----------------------------------------------------------------- | :------------------------------------------------------- |
| `REGEXP_COUNT(src.code_list, ':')+1`                               | 문자열 내 `:` 개수 + 1 → 분리할 토큰(LEVEL) 개수 계산                   |
| `CONNECT BY LEVEL <= N`                                            | Oracle 계층형 쿼리로 1\~N 숫자(LEVEL) 생성 → 각 토큰의 인덱스 역할          |
| `MULTISET(SELECT LEVEL...) AS SYS.ODCINUMBERLIST`                  | 숫자 결과를 Oracle 내장 컬렉션 타입으로 변환                             |
| `TABLE(...) lvl`                                                   | 컬렉션을 테이블처럼 풀어내 각 LEVEL 값을 행으로 확장                         |
| `TRIM(REGEXP_SUBSTR(src.code_list, '[^:]+', 1, lvl.COLUMN_VALUE))` | LEVEL번째 토큰(콜론 기준) 추출 → Java `split` 로직 대체                |
| `SELECT DISTINCT`                                                  | 임시 테이블 전체에서 중복된 `item_code` 제거 → 합집합 생성                  |
| `JOIN lookup_table lk ... lk.item_code = ...`                      | 추출한 `item_code`로 lookup\_table에서 `item_name`을 바로 매핑하여 조회 |

* **Legacy**: 2회 쿼리 + Java 파싱 → `446.9 ms`
* **Optimized**: 1회 쿼리 → `366.5 ms` (약 18% 개선)

---

## 결론 & 느낀 점

* **Java 파싱 제거**로 서버 부담 경감
* **SQL 호출 2→1회**로 네트워크 왕복 절감
* **Oracle 내장 기능** 적극 활용으로 단순·강력한 조회 로직 완성

Oracle의 정규식 함수, 계층 쿼리, 컬렉션 타입을 조합하면,
**복잡한 Java 로직 없이도 고급 문자열 처리와 집계를 한 번의 SQL로 해결**할 수 있다는 것을 알게 되었습니다.

감사합니다!
