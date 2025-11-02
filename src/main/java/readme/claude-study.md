# 프로젝트 개요
프로젝트 명: 책 스터디 정리
목적 : 30가지 패턴으로 배우는 분산 시스템 설계와 구현기법 정리하여 readme 만들기 

---

## 정리 구조

1. 제목 챕터 정리
- x장으로 내가 작성
- 제목 에 관한 내용도 정리 할거야
- cladue : #을 붙이고 📚 와 함께 제목 작성후 아래칸은 --- 으로 단락 바꿈 

아래는 예시 
# 📚 3장 함수 정의와 호출
- 함수 정의에관한 호출을 정리한다.
 ---

2. 소제목 챕터 정리
- 3.1 및 소제목으 로 내가 작성 
- claude : ## 을 붙아고 📖 와 함께 소제목 작성 
  - 이후 해당 소제목에 관한 내용을 내가 정리 
- 정리하다가 단이 나오게 되면 내가 3.1.1 로 작성 
- claude : ### 을 붙이고 3.1.1 🔖와 함께 단제목으로 작성 
  - 단에 관한 내용 정리 
- 해당 소제목이 끝나면 --- 으로 마무리 

아래는 예시

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
---

## ⚙️ 특별한 규칙
- 내용 정리 시에는 88자 기준으로 한칸 엔터
- 내용 정리는 - 이거를 붙이고 정리 
  - 예시 - 코틀린은 작성한 함수를 호출할때 함수에 전달하는 인자 중 일부의 이름을 명시할수 있다.
- 해당 내용과 관련있는 부분은 한칸 엔터후 들여쓰기 하여 - 를 붙여줘 관련있는 부분은 내가 한칸 엔터후 space 두번붙일게 
  - 아래 예시
    - 자바 에서는 일부 클래스에서 오버로딩한 메서드가 너무 많아진다는 문제가 자주 발생
      - 파라미터가 다를때마 오버로딩 메서드가 생성됨
      - 중복이라는 결과가 생성

- 코드가 나오면  ```{프로그래밍 언어 이름} ``` 으로 묶어줘 프로그래밍 언어이름은 코드 앞에 내가 !와같이 써줄거야
  - 코드는 내가 !java @@@ {코드} @@@ 이런식으로 써줄게 그럼 너가 ```{프로그래밍 언어 이름} ``` 반환
  - 코드는 구글 포맷터 기준대로 맞춰줘
  - 예시 !java system.out.println()
- 이미지는 내가 너한테 찾아달라고 할거고 ![Image](https://github.com/user-attachments/assets/497082ac-3253-4055-a70f-7b685bda1d86) 이런식으로 넣어달라고 할꺼야
- github에 커밋하여서 관리할거야