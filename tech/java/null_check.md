---
title: "자바의 다양한 문자열 null & 공백 체크 방식들"
---

비즈니스 로직에서 null 혹은 공백 체크는 정말 뗄레야 뗄 수 없는 관계입니다.

이 두 값에 따라 특정 행위를 취해야하는 경우, 조건문을 태우기 위해 쓰기도 하지만

자바의 두려움 유발자 NPE을 방지하기 위해서도 쓰입니다.

한번 익혀두면 자신이 선호하는 방식을 선택할 수 있으니 이참에 모두 알아보자 ~

---

# 1.  ==연산

```java
if (name == null || name.isEmpty()) {
		log.error("이름을 입력해주세요");
}
```

가장 직관적이고 간단한 체크 방식.

다만 주의할 점이 있다. 보통 위와 같이 입력 데이터 포맷을 검사하는데 쓰일 수 있는데,

```java
if (name.isEmpty() || name == null) {
		// name이 null일 경우 isEmpty() 호출에서 NPE 발생
}
```

조건문을 위와 같이 잘못 설계할 시 NPE가 발생할 위험이 있다.

isEmpty()는 String, Collection 등이 가지고 있는 인스턴스의 메서드이기에,

name이 이 메서드를 실행 시켜줄 객체의 참조값을 가지고 있지 않다면 JVM레벨에서 예외가 발생한다.

> 비즈니스 로직은 다른 계층에서 기대되는 데이터 타입이 아닌, 엉뚱한 타입을 넘겨줄 때의 대비를 항상 하고 있어야 한다.
그 곳이 클라이언트건 DB건 name에 어떠한 값이 담길 수도 있다고 의심하고, 특정 인스턴스 메서드를
실행시키기 이전에 해당 변수의 데이터 타입을 우선 체크하고 넘어가야한다.
>

---

# 2. isEmpty()

String의 isEmpty()에 대해 더 얘기해보자면, 문자열이 “”인지 확인하는 메서드이다.

정확히는

```java

@Stable
private final byte[] value;

@Override
public boolean isEmpty() {
    return value.length == 0;
}
```

문자열의 길이가 0인지 확인한다. 그런데 위 예시처럼, 필수로 입력해야할 input데이터는 빈문자열 뿐 아니라

해당 문자열이 공백으로만 이루어져 있지는 않은지 확인해야할 필요가 있다.

예를 들어, 누군가 이름으로 “   “으로 공백으로만 이루어진 3글자 데이터를 입력한다면

```java
if (name == null || name.isEmpty()) {
		log.error("이름을 입력해주세요");
}
```

위 검증을 통과해버릴 것이기 때문이다.

---

# 3. String.isBlank()

이럴 때 고려하면 좋은 메서드이다.

```java
public boolean isBlank() {
    return indexOfNonWhitespace() == length();
}

private int indexOfNonWhitespace() {
    return isLatin1() ? StringLatin1.indexOfNonWhitespace(value)
                      : StringUTF16.indexOfNonWhitespace(value);
}

public static int indexOfNonWhitespace(byte[] value) {
    int length = value.length;
    int left = 0;
    while (left < length) {
        char ch = getChar(value, left);
        if (ch != ' ' && ch != '\t' && !CharacterDataLatin1.instance.isWhitespace(ch)) {
            break;
        }
        left++;
    }
    return left;
}

public static int indexOfNonWhitespace(byte[] value) {
    int length = value.length >> 1;
    int left = 0;
    while (left < length) {
        int codepoint = codePointAt(value, left, length);
        if (codepoint != ' ' && codepoint != '\t' && !Character.isWhitespace(codepoint)) {
            break;
        }
        left += Character.charCount(codepoint);
    }
    return left;
}
```

복잡하게 작성되어 있지만, 원리는 단순하다.

indexOfNonWhitespace 메서드는 문자열에서 처음으로 공백이 아닌 문자가 나오는 인덱스를 반환한다.

이 인덱스 추적이 문자열의 길이까지 이어졌다면(left++을 문자열 길이만큼 수행) 해당 문자는 공백으로만 이루어졌으므로 true를 반환한다.

다만, 이 메서드 또한 String의 인스턴스 메서드이므로 마찬가지로 null-safe인 영역 + String 객체 참조값을 가진 경우에만 작성해야한다.

그런데, if문에 조건이 여러개인 것도 싫고

```java
public boolean isInvalidNameData(Object name) {
    if (name == null) {
        log.error("이름을 입력해주세요");
        return true;
    }

    if (!(name instanceof String)) {
        log.error("올바르지 않은 데이터 포맷");
        return true;
    }
    
    String nameStr = (String) name;

    if (nameStr.isBlank()) {
        log.error("공백은 입력할 수 없습니다");
        return true;
    }

    log.info("success");
    return false;
}
```

이런 식으로 비즈니스 로직에서 여러번 유효성 검사하는 것이 보기 싫다면

---

# 4. StringUtils.hasText()

StringUtils 라이브러리의 hasText 메서드가 대안이 될 수 있다.

다만 SpringFramework의 내장 라이브러리이기 때문에 순수 자바로는 사용할 수 없다.

사실 이 메서드는

```java
@Contract("null -> false")
public static boolean hasText(@Nullable String str) {
	return (str != null && !str.isBlank());
}
```

결국 Java의 isBlank를 내부적으로 사용한다. 다만 조건문을 통해 null-safe영역을 만들어 낸뒤 isBlank를 호출하는 것 뿐이다. 이를 활용한다면 위 코드를

```java
public void isInvalidNameData(Object name) {
    if (!(name instanceof String str) || !StringUtils.hasText(str)) {
        log.error("유효하지 않은 이름 입력입니다.");
        return true;
    }

    log.info("success");
    return false;
}
```

더 개선해나갈 수 있을 것이다.