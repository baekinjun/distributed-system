# 📚 3장 함수 정의와 호출

___

## 📖 3.1 코틀린에서 컬렉션 만들기

```kotlin
val set = setOf(1, 7, 53) // java.util.LinkedHashSet
val list = listOf(1, 7, 53) // java.util.Arrays$ArrayList
val map = listOf(1 to "one", 7 to "seven") // java.util.LikedHashMap to 는 함수
```

- 코틀린은 표준 자바 컬렉션 클래스를 사용한다.
- 하지만 자바와 달리 코틀린 컬렉션 인터페이스는 읽기 전용이다.
- 자바와 코틀린 사이에 호출을 할때 서로 변환한 필요가 없다.

```kotlin
fun main() {
    val strings = listOf("first", "second")

    strings.last() // second
    strings.shuffled()
    val numbers = setOf(1, 2, 3)
    numbers.sum()
}
```

- 더 많은 기능 사용 가능

___

## 📖 3.2 함수를 호출하기 쉽게 만들기

```kotlin
fun main() {
    val list = listOf(1, 2, 3)
    println(list)
    /**
     * [1,2,3] 자바는 디폴트 toString 이 있다.
     * 커스텀 하게 구현하고 싶을때는?
     */
    println(joinToString(listOf(1, 2, 3), ";", "(", ")")) // 호출 하는 구문이 번잡스러움
    joinToString(listOf(1, 2, 3), separator = ";", prefix = "(", postfix = ")")

}

fun <T> joinToString(
    collection: Collection<T>,
    separator: String,
    prefix: String,
    postfix: String
): String {

    val result = StringBuilder(prefix)
    for ((index, element) in collection.withIndex()) {
        if (index > 0) result.append(separator)
        result.append(element)
    }
    result.append(postfix)
    return result.toString()

}
```

### 🔖 3.2.1 이름 붙인 인자

```kotlin
    joinToString(listOf(1, 2, 3), ";", "(", ")")
```

- 함수의 시그니처를 살펴 보지 않고는 함수 호출 코드 자체가 모호하다.
    - 이런 문제는 특히 boolean 과 같은 flag 값을 전달 할때 실수가 많이 발생

```kotlin
      joinToString(listOf(1, 2, 3), separator = ";", prefix = "(", postfix = ")")
```

- 코틀린은 작성한 함수를 호출할때 함수에 전달하는 인자 중 일부의 이름을 명시할수 있다.
- 전달하는 모든 인자의 이름을 지정할때는 순서도 변경 가능하다.

### 🔖 3.2.2 디폴트 파라미터 값

- 자바 에서는 일부 클래스에서 오버로딩한 메서드가 너무 많아진다는 문제가 자주 발생
    - 파라미터가 다를때마 오버로딩 메서드가 생성됨
    - 중복이라는 결과가 생성
- 코틀린 에서는 함수 선언에서 파라미터의 기본값을 지정할 수 있으므로 이런 오버로드 중 상당수를 피할수 있다.

```kotlin

fun main() {
    val list = listOf(1, 2, 3)
    joinToString(list, ",", "", "")
    joinToString(list)
    joinToString(list, ";")
}

fun <T> joinToString(
    collection: Collection<T>,
    separator: String = ",",
    prefix: String = ",",
    postfix: String = ""
): String {

    val result = StringBuilder(prefix)
    for ((index, element) in collection.withIndex()) {
        if (index > 0) result.append(separator)
        result.append(element)
    }
    result.append(postfix)
    return result.toString()

}
```

- 함수의 디폴트 파라미터 값은 함수를 호출하는 쪽이 아니라 함수 선언 쪽에 인코딩 된다는 사실
    - 어떤 클래스 안에 정의된 함수의 기본값을 바꾸고 그 클래스가 포함된 파일을 재컴파일 하면 그 함수를 호출
      하는 코드 중에 값을 지정 하지 않은 모든 인자는 자동으로 바뀐 기본값을 적용받는다.

### 🔖 3.2.3 정적인 유틸리티 클래스 없애기: 최상위 함수와 프로퍼티

- 자바는 모든 코드를 클래스의 메서드로 작성해야만 한다.
    - 유틸 클래스
        - 특별한 상태나 인스턴스 메서드가 없는 클래스
        - 다양한 정적 메서드를 모아두는 역할
- 코틀린에서는 이런 클래스를 생성할 필요가 없음
    - 함수를 직접 소스 파일의 최상위 수준 , 모든 다른 클래스의 밖에 위치시면된다.
        - 파일의 맨 앞에 정의된 패키지의 멤버 함수이므로 다른 패키지에서 그 함수를 사용하고 싶을때는 그함수가 정의된
          패키지를 임포트 하면되고 불필요한 클래스는 임포트 안해도 됨

```kotlin
package strings

fun <T> joinToString(): String {
    return ""
}
```

- JVM 에서는 클래스 안에 들어있는 코드만을 실행 할수 있기 때문에 이 파일을 컴파일 하는 과정에서 새로운 클래스가 생성

```java
package strings

public class Kt {
    public static String joinToString() {
    }
}
```

- 코틀린 컴파일러가 생성하는 클래스의 이름이 최상위 함수가 들어있던 코틀린 소스 파일의 이름과 대응
- 코틀린 파일의 모든 최상위 함수는 이 클래스의 정적인 메서드가 된다.

#### 최상위 프로퍼티

```kotlin
    var opCount = 0
val UNIX_LINE_SEPARATOR = "\n"
const val UNIX_LINE_SEPARATOR = "\n" // public static final 로 노출 시키고 싶으면 const 추가
```

---

## 📖 3.3 메서드를 다른 클래스에 추가: 확장 함수와 확장 프로퍼티

기존 코드와 코틀린 코드를 자연스럽게 통합하는 것은 코틀린의 핵심 목표중 하나다.

- 확장 함수
    - 어떤 클래스의 멤버 메서드인 것처럼 호출할 수 있지만 그 클래스의 밖에 선언된 함수

```kotlin
package strings

fun String.lastChar(): Char = this.get(this.length - 1)
fun main() {
    println("kotiln".lastChar())
}
```

- 수신 객체 타입
    - 추가 함수 이름 앞에 그 함수가 확장할 클래스의 이름을 덧붙이는것
    - String
- 수신 객체
    - 확장 함수 호출시 호출하는 대상값(객체)
    - this


- 다른 JVM 언어로 작성된 클래스도 확장할수 있고 final로 상속을 할수 없게 선언된 경우에도 문제가 되지 않음
- 자바 클래스로 컴파일된 클래스 파일이 있는 한 그 클래스에 원하는 대로 확장을 추가할수 있다.

```kotlin
package strings

fun String.lastChar(): Char = get(length - 1) // this 생략 가능 
```

- 확장 함수 내부에서는 일반적인 인스턴스 메서드의 내부와 마찬가지로 수신객체의 메서드나 프로퍼티 바로 사용 가능
- 캡슐화를 깨지 않는다!
- 클래스 내부에서만 사용할수 있는 private, protected 멤버를 사용할수 없음

### 🔖 3.3.1 임포트와 확장 함수

- 다른 클래스나 함수와 마찬가지로 함수를 임포트 해야 한다.
    - 이름 충돌을 막기위함
- 코틀린 문법상 반드시 짧은 이름을 사용해야 한다.
- 임포트할 때 이름을 바꾸는 것이 확장 함수의 이름 충돌을 해결하는 유일한 방법

### 🔖 3.3.2 자바에서 확장 함수 호출

- 확장 함수는 수신 객체를 첫 번째 인자로 받는 정적 메서드다.
    - 다른 어댑터 객체나 실행 시점 부가 비용이 발생하지 않는다.
- 자바에서는 단지 정적 메서드를 호출하면서 첫번째 인자로 수신 객체를 넘기면된다.

```java
char c=StringUtilKt.lastChar("Java");
```

### 🔖 3.3.3 확장 함수로 유틸리티 함수 정의

```kotlin

fun main() {
    val list = listOf(1, 2, 3)
    println(list.joinToString(" ")) //클래스의 멤버처럼 호출 가능 
}

fun <T> Collection<T>.joinToString(
    separator: String = ",",
    prefix: String = ",",
    postfix: String = ""
): String {

    val result = StringBuilder(prefix)
    for ((index, element) in this.withIndex()) {
        if (index > 0) result.append(separator)
        result.append(element)
    }
    result.append(postfix)
    return result.toString()
}
```

- 확장 함수는 단지 정적 메서드 호출에 대한 문법적 편의에 클래스가 아닌 더 구체적인 타입을 수신 객체 타입으로 지정 할수도 있다.

```kotlin

fun main() {
    val list = listOf(1, 2, 3)
    println(listOf("1", "2", "3").join(" ")) //클래스의 멤버처럼 호출 가능 
    println(list.join(" ")) //문자열이 아닌 다른 타입의 객체로 이뤄진 리스트에 대해 호출 불가능 
}

fun Collection<String>.join(
    separator: String = ",",
    prefix: String = ",",
    postfix: String = ""
) = joinToString(separator, prefix, postfix)
```

- 확장 함수는 정적 메서드와 같은 특성을 가지므로 확장 함수를 하위 클래스에서 오버라이드 할수 없다.

### 🔖 3.3.4 확장 함수는 오버라이드 할수 없다.

- 확장 함수는 클래스의 일부가 아니다.
- 이름과 파라미터가 완전히 같은 확장 함수를 기반 클래스와 하위 클래스에 대해 정의는 할수 있지만 실제 호출될 함수는
  확장 함수를 호출할 때 수신 객체로 지정한 변수의 컴파일 시점의 타입에 의해 결정되지, 실행 시간에 그변수에 저장된 객체의 타입에
  의해 결정되지 않는다.
- 코틀린은 호출될 확장 함수를 정적으로 결정하기 때문이다.

### 🔖 3.3.5 확장 프로퍼티

```kotlin
val String.lastChar: Char
    get() = get(length - 1)
```

- 확장 프로퍼티도 단지 프로퍼티에 수신 객체 클래스가 추가된 것

```kotlin
val String.lastChar: Char
    get() = get(length - 1)
    set(value: Char) {
        this.setCharAt(length - 1, value)
    }
```

- 자바에서 사용하고 싶으면 명시적으로 get,set 을 붙여야 한다.

---

## 📖 3.4 컬렉션 처리 : 가변 길이 인자, 중위 함수 호출, 라이브러리 지원

- vararg
    - 호출 시 인자 개수가 달라질 수 있는 함수를 정의 할 수 있다.
- 중위(infix)함수 호출
    - 인자가 하나뿐인 메서드를 간편하게 호출
- 구조 분해 선언(destructuring delaration)
    - 복합적인 값을 분해해서 여러 변수에 나눠 담을 수 있다.

### 🔖 3.4.1 자바 컬렉션 API 확장

```kotlin
fun main() {
    val strings: List<String> = listOf("first", "second", "fourteenth")
    strings.last() //last , max 모두 확장 함수로 정의 되고 항상 코틀린 파일에 디폴트로 임포트 된다.
    val numbers = Collection<Int> = setOf(1, 14, 2)
    numbers.sum()

    fun <T> List<T>.last(): T {/**/
    }
    fun Collection<Int>.max(): Int {}
}
```

### 🔖 3.4.2 가변 인자 함수: 인자의 개수가 달라질 수 있는 함수 정의

```kotlin
fun main() {
    val list = listOf(1, 2, 3)
}

fun listOf<T>(vararg values: T): List<T> {} // 표준라이브러리 구현
```

- 가변 길이 인자 사용
    - 원하는 개수만큼 값을 넣어줌
    - 자바는 ... , 코틀린은 vararg 를 사용


- 이미 배열에 들어있는 원소를 가변 길이 인자로 넘길 때도 코틀린과 자바 구문이 다름
- 자바 : 배열을 그냥 넘김
- 코틀린 : 배열을 명시적으로 풀어 각 원소가 인자로 전달
- 이런 기능을 스프레드 연산이라고 한다.
    - 배열 앞에 * 만 붙이면 됨

```kotlin
fun main(args: Array<String>) {
    val list = listOf("args:", *args) // 스프레드 연산자가 내용을 펼쳐줌 
}
```

### 🔖 3.4.3 쌍(튜플) 다루기 : 중위 호출과 구조 분해 선언

```kotlin
val map = mapOf(1 to "one", 7 to "seven") // to 중위 호출
```

- 중위 호출 시에는 수신 객체 뒤에 메서드 이름을 위치시키고 그 뒤에 유일한 메서드 인자를 넣는다.
    - 인자가 하나뿐인 일반 메서드나 확장 함수에만 중위 호출을 사용할수 있음
    - 함수를 중위 호출에 사용하게 허용 하고 싶으면 infix 변경자를 함수 선언 앞에 추가 해줘야 한다.

```kotlin
    infix fun Any.to(other: Any) = Pair(this, other)
val (number, name) = 1 to "one" // 구조 분해 선언
```

![Image](https://github.com/user-attachments/assets/497082ac-3253-4055-a70f-7b685bda1d86)

- 구조 분해는 순서 쌍에만 한정되지 않는다.

```kotlin
for ((index, element) in collection.withIndex()) {
    println("$index : $element")
}
```

- 루프에서도 구조 분해 선언을 활용 할수 있다.

---

## 📖 3.5 문자열과 정규식 다루기

- 코틀린 문자열은 자바 문자열과 똑같다.
    - 코틀린 코드가 만들어낸 문자열을 자바 메서드에 넘겨도 문제가 없다.
    - 특별한 변환도 필요 없고 자바 문자열을 감싸는 별도의 래퍼 객체도 생기지 않는다.

### 🔖 3.5.1 문자열 나누기

- split 메서드
    - 자바에서는 정규식을 구분 문자열로 받는다.
        - 정규식에 따라 문자열을 나누기 때문이다.
    - 코틀린에서는 여러가지 다른 조합의 파라미터를 받는 확장 함수를 제공
        - 정규식 이나 일반 텍스트 둘다 받을수 있다.

```kotlin
fun main() {
    println("12.345-A.A".split("\\.|-".toRegex())) // 정규식을 명시적으로 만듬 
    println("12.345-A.A".split(".", "-")) // 일반 텍스트를 넘겨도 같은 결과를 받는다.
}
```

### 🔖 3.5.2 정규식 과 3중 따옴표로 묶은 문자열

```kotlin
fun parsePath(path: String) {
    val directory = path.substringBeforeLast("/")
    val fullName = path.substringAfterLast("/")
    val fileName = fullName.substringBeforeLast(".")
    val extension = fullName.substringAfterLast(".")
}// 정규식을 사용하지 않고 문자열을 파싱할수 있다. 확장 함수 사용

fun parsePathRegex(path: String) {
    val regex = """(.+)/(.+)\.(.+)""".toRegex()
    val matchResult = regex.matchEntire(path)
    if (matchResult != null) {
        val (directory, filename, extension) = matchResult.destructured
    }
} /*
 3중 따옴표 문자열을 사용해 정규식을 사용
 3중 따옴표 문자열에서는  \. 로 사용, 일반 문자열에서는  \\. 사용
*/
```

### 🔖 3.5.3 여러 줄 3중 따옴표 문자열

- 문자열을 이스케이프 하기 위해서만 사용하지 않는다.
- java 17 에서 나온것과 같다.
- test case 작성시 json 형태를 묶을때 용이

---

## 📖 3.6 코드 깔끔하게 다듬기: 로컬 함수와 확장 
- 반복하지 말라
  - 좋은 코드의 중요한 특징중 하나
- 많은 경우 메서드 추출 리팩터링을 적용해서 긴 메서드를 부분부분 나눠 각 부분을 재활용 
  - 하지만 이렇게 하면 클래스 안에 작은 메서드가 많아 지고 각 메서드 사이의 관계를 파악하기 힘들어서 코드를 이행하기 더
  어려울 수도 있다.
- 코틀린은 함수에서 추출한 함수를 원래의 함수에 내포 시킬수 있다.

```kotlin // 중복코드 예시
class User(val id: Int, val name: String, val address: String)

fun saveUserDuplicate(user: User) {
    if (user.name.isEmpty()) {
        throw IllegalArgumentException("error")
    }
    if (user.address.isEmpty()) {
        throw IllegalArgumentException("error")
    }
}

fun main() {
    saveUserDuplicate(User(1, "", ""))
}
```
- 검증 코드가 중복된다.
- 검증 코드를 로컬 함수로 분리하면 중복을 없애는 동시에 코드 구조를 깔끔하게 유지할수 있다.
```kotlin
class User(val id: Int, val name: String, val address: String)
fun saveUserLocalMethod(user: User) {
    fun validate(
        user: User,
        value: String,
        fieldName: String
    ) {
        if (value.isEmpty()) {
            throw IllegalArgumentException("error")
        }
    }
    validate(user, user.name, "Name")
    validate(user, user.address, "Address")
}
```
- 검증 중복 로직은 사라 졌지만 User 객체를 일일이 로컬 함수에게 전달해야한다.
- 로컬 함수는 자신이 속한 바깥 함수의 모든 파라미터와 변수를 사용할 수 있다. 
```kotlin
class User(val id: Int, val name: String, val address: String)
fun saveUser(user: User) {
    fun validate(value: String, fieldName: String) {
        if(value.isEmpty()){
            throw IllegalArgumentException("error")
        }
    }
    validate(user.name,"Name")
    validate(user.address, "Address")
}
```
- 로컬 함수 에서 바깥 함수의 파라미터 접근하기

```kotlin
class User(val id: Int, val name: String, val address: String)
fun User.validateBeforeSave() {
    fun validate(value: String, fieldName: String) {
        if (value.isEmpty()) {
            throw IllegalArgumentException("error")
        }
    }
    validate(name, "Name")// 유저의 프로퍼티를 직접 사용할수 있음 
    validate(address, "Address")
}

fun saveUser(user: User) {
    user.validateBeforeSave()
}
```
- 한 객체만을 다루면서 객체의 비공개 데이터를 다룰 필요가 없는 함수는 확장 함수로 만들면 객체.멤버 처럼 수신객체를 
지정 하지 않고도 공개된 멤버 프로퍼티나 메서드에 접근 할수 있다.
- 확장 함수를 로컬 함수로 정의할 수도 있다.
  - User.validateBeforeSave를 saveUser 내부에 로컬 함수로 넣을수 있다.
#### 내포된 함수의 깊이가 깊어지면 코드 읽기가 상당히 어려워 진다.
- 일반적으로는 한단계만 내포시키라고 권장한다.