Trouble ID `2023-02-23.ktlint-why-disabling-filename-and-enum-entry-name-case`

# ktlint 에서 “File name” 과 “Enum entry” 항목을 불능 처리하게 된 이유

이번 편의 솔루션은 제목에 제시되어 있다. 나는 [ktlint](<https://pinterest.github.io/ktlint/>) 에서 다음 두 항목을 ignore 처리하기로 정했다.

- File name `filename` <del><https://pinterest.github.io/ktlint/rules/standard/#file-name></del>
  - <https://pinterest.github.io/ktlint/0.48.0/rules/standard/#file-name> (&larr; Update @ May 2023)

- Enum entry `enum-entry-name-case` <del><https://pinterest.github.io/ktlint/rules/standard/#enum-entry></del>
  - <https://pinterest.github.io/ktlint/0.48.0/rules/standard/#enum-entry> (&larr; Update @ May 2023)


ktlint 설정에는 `.editorconfig` 파일을 이용한다. ktlint 0.48.x 까지는 `disabled_rules` 를 이용할 수 있다.

```
[*.{kt,kts}]
disabled_rules = filename,enum-entry-name-case
```

> (Update @ May 2021)
>
> ktlint 0.49 부터는 `disabled_rules` 가 삭제되고 ktlint 0.48 에 도입된 `ktlint_standard_<rule-id>` 를 이용할 수 있다.
>
> ```
> [*.{kt,kts}]
> ktlint_standard_filename = disabled
> ktlint_standard_enum-entry-name-case = disabled
> ```

자, 오늘의 주인공인 린터에 대한 이야기는&hellip; 이 글을 과거와 미래라는 배경부터 말하는 걸로 시작해 보려고 했는데 무슨 수를 써도 재미가 없다. 그냥 현존하는 개별 규칙의 예외 사례나 설명해 보겠다.

## filename

ktlint 문서는 `filename` 규칙을 다음과 같이 설명하고 있다.

> **File name**
>
> Files containing only one toplevel domain should be named according to that element.
>
> ----
>
> <del><https://pinterest.github.io/ktlint/rules/standard#file-name></del>
>
> <https://pinterest.github.io/ktlint/0.48.0/rules/standard/#file-name> (&larr; Update @ May 2021)

예를 들어서, 이런 최상위 선언/정의 식별자는 파스칼케이스<sup>PascalCase; Pascal case</sup>를 따르는 것으로 (요즘은 대문자 캐멀케이스<sup>upper camel case</sup>라고도 부르는 그것으로) 정해져 있는데, Kotlin 파일명은 `.kt` 확장자를 갖되 확장자를 제외한 부분은 식별자를 그대로 따르게 되어 있다.

```kotlin
// AbstractPersonnel.kt
class AbstractPersonnel {
    // ...
}
```

이는 파스칼케이스를 따르는 식별자를 갖는 모든 최상위 선언/정의에 적용된다. 즉 `class` 나 그 파생인 `sealed class`, `data class`, `enum class` 뿐만 아니라 `interface` 와 `object` 에도 적용되는 규칙인 것이다.

다만 여기에서 예외가 생기는데, 바로 최상위 함수<sup>top-level function; TLF</sup>. 이들은 함수이기 때문에 메소드와 같은 (소문자) 캐멀케이스<sup>camelCase; lower camel case</sup>가 식별자 규칙으로 적용된다. 따라서 최상위 요소의 이름을 파일명으로 사용하는 규칙을 일괄 적용하게 된다면 다음과 같은 파일명을 사용하게 된다.

```kotlin
// createPersonnel.kt
fun createPersonnel(serial: String): AbstractPersonnel {
    // ...
}
```

이는 기계적 일관성을 잃지 않으면서 규칙을 설립한 목적도 잃지 않는 것으로 보인다. 이 방식은 `fun` 과 함께 최상위 속성<sup>top-level property; TLP</sup>, 즉 `val`/`var` 도 포함하게 된다.

ktlint 는 이 경우 `filename` 규칙을 어기는 것으로 판단한다. ktlint 가 파일명을 규율하는 제 1의 규칙은 파스칼케이스를 따라야 한다는 것이기 때문이다.

최상위 선언/정의가 파스칼케이스를 따르지 않는 경우에 대해 제대로 규율하고 있는 코딩 스타일 가이드는 없다시피 하다. JetBrains 의 Kotlin 공식 코딩 컨벤션 문서에는 소스 파일명에 대해 이렇게 기재되어 있다.

> **Source file names**
>
> If a Kotlin file contains a single class or interface (potentially with related top-level declarations), its name should be the same as the name of the class, with the `.kt` extension appended. It applies to all types of classes and interfaces. If a file contains multiple classes, or only top-level declarations, choose a name describing what the file contains, and name the file accordingly. Use [upper camel case](https://en.wikipedia.org/wiki/Camel_case) with an uppercase first letter (also known as Pascal case), for example, `ProcessDeclarations.kt`.
>
> The name of the file should describe what the code in the file does. Therefore, you should avoid using meaningless words such as `Util` in file names.
>
> ----
>
> <https://kotlinlang.org/docs/coding-conventions.html#source-file-names>

Kotlin stdlib 은 클래스 이름이 없어도 대문자로 시작되는 `Arrays.kt`, `Sets.kt`, `Collections.kt` 등을 사용하고 있다.

- `Collections.kt` <https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/collections/Collections.kt>

별로 좋은 선례로 보이지는 않는다. 한편 `fun Foo.bar()` 같은 확장함수의 경우 `FooExt`  같은 파일명을 사용하는 것이 관례로 되어 있으며, 이는 비슷한 유형체계를 제시하는 Swift 에서 `String+Extensions.swift` 같은 파일명을 사용하는 것과 비슷한 정책으로 여겨지고 있다.

- SO Question 33636280: *Is there a recommended naming convention for files that contain only functions/extension methods?* <https://stackoverflow.com/questions/33636280/is-there-a-recommended-naming-convention-for-files-that-contain-only-functions-e>

대개는 파일명을 정하는 데에 참고해야 할 정보를 많이 제시할 뿐이다. 파일명에 항상 파스칼케이스를 사용하라는 규칙을 명확히 게시해 놓은 곳으로는 Google 의 Android 개발자 홈페이지에 있는 Kotlin 스타일 가이드가 있다.

> **Naming**
>
> If a source file contains only a single top-level class, the file name should reflect the case-sensitive name plus the `.kt` extension. Otherwise, if a source file contains multiple top-level declarations, choose a name that describes the contents of the file, apply PascalCase, and append the `.kt` extension.
>
> ```kotlin
> // MyClass.kt
> class MyClass { }
> ```
>
> ```kotlin
> // Bar.kt
> class Bar { }
> fun Runnable.toBar(): Bar = // …
> ```
>
> ```kotlin
> // Map.kt
> fun <T, O> Set<T>.map(func: (T) -> O): List<O> = // …
> fun <T, O> List<T>.map(func: (T) -> O): List<O> = // …
> ```
>
> ----
>
> Android Developers &mdash; Kotlin Guides. *Kotlin Style Guide.* <https://developer.android.com/kotlin/style-guide?hl=en#naming>

파일의 TLD 가 클래스 비슷한 것이 아닌 경우에 대해 이렇게 어떤 질서도 찾을 수 없는 현상의 원인은 Kotlin 생태계의 절대적 주류가 Kotlin/JVM 이라는 점에서 찾을 수 있을 것 같다. Kotlin/JVM 에서는 Java 코드로 작성된 구성요소<sup>component</sup>와의 상호운용성<sup>interoperability</sup>을 무시할 수 없고 내부적으로는 Kotlin 문법을 따르며 장점을 취할지언정 외부적으로는 TLD 가 클래스여야 한다는 Java 어법을 버릴 수 없기 때문이다. 대안으로 `@file:JvmName` 이라거나 파일명 기반의 클래스명 생성 (`foo.kt` &rarr; `FooKt`) 같은 방법이 있긴 하지만, 굳이 복잡한 컴파일러 툴체인 기능에 기대면서 `static` 인 것들을 만들어내느니, 객체지향이라는 (몇 안 되는) 근본을 따라 다른 도구들과의 연계성을 잃지 않고자 하는 경로의존적/심리적 요인들도 한몫 한다 (예: 현 시점 기준, 단위 시험<sup>unit testing</sup>에서 특정 단위의 동작만 검증하기 위해서는 대상 단위가 의존성 주입<sup>dependency injection; DI</sup>이 가능한 설계여야 하며 의존성이 모의<sup>mock</sup>할 수 있는 클래스나 인터페이스일 필요가 있는데, TLF 에 직접 의존성이 발생한다면 구현이 구현에 의존하는 것이므로 이것이 불가능하다).

한편 우리 팀은 이전부터 자신있게 Kotlin 을 쓰며 확장이나 TLF, TLP 를 자주 사용하는 편이고, 내가 파일명을 결정하는 원칙을 대강 다음과 같이 제시해서 ktlint 도입 이전부터 꽤 오랜 실전 검증을 거쳐 왔다.

- `class`, `interface`, `object` 의 경우 파스칼케이스를 따르고, 한 파일에 하나의 (묵시적<sup>implicit</sup> `public`) TLD 만 허용하며 이를 파일명으로 그대로 사용해야 한다. `object` 내부에서만 사용하는 값이 있다면 `private val` 로 처리한다. `class` 내부에서만 사용하는 값이 있다면 `companion object` 를 개설한다. 또한 이들에 대해 `private` TLD 는 만들지 않고, `public` TLD 내에 중첩<sup>nest</sup> 처리한다.
  - Kotlin 에서 `public` 은 `public get()` 같은 경우가 아니면 기본으로 적용되는 묵시적 가시성<sup>visibility</sup>이지만 `private` 와의 구분을 위해 기재한다.
- TLF, TLP 의 경우 (소문자) 캐멀케이스를 따르고, 한 파일에 하나의 (묵시적) `public` TLD 만 작성하는 것을 권장하며 이 경우 이를 파일명으로 그대로 사용해야 한다. 이 `public` TLF 내부에서만 사용하는 값이 있다면 `private` TLP 로 처리해도 좋다.
  - 다만 이와 같은 `private` TLP 의 개설은 Java 기준 `static` 메서드 내부에 작성하는 것보다 `static {}` 블럭에 작성하는 것이 유리할 만큼의 성능적 이점이 있는 경우에 한함. 즉 `Int` (JVM `int`/`Integer`) 값이나 `String` 값은 바이트코드에서 아예 명령어의 일부가 되거나 JVM 에서 상수풀<sup>constant pool</sup>에 배정되므로 만약 `public` 으로 도처에서 참조할 이름이 필요한 게 아니라면 굳이 TLP 로 작성할 필요가 없다. 한편 어떤 값을 활용해 `"0+${count.toString()}".toRegex()` 따위로 `Regex` 유형의 인스턴스를 만들어 두는 `private` TLP 가 있다면 의존성 문제로 `count` 도 `private` TLP 처리하게 될 것이다. `getInstance()` 같은 처리보다는 이게 낫다고 본다.
- `class`, `interface`, `object` 에 확장함수/확장속성을 개설하는 경우 이름은 리시버<sup>receiver</sup> (`this`) 유형을 응용해 다음과 같이 한다.
  - 단일 확장함수 파일인 경우: `fun Epic.meme()` &rarr; `Epic$meme.kt`.
  - `Observable` 의 스레드 관련 확장함수 모음인 경우: `Observable$threadings.kt`.
  - 그 외: `fun Foo.bar()`, `fun Foo.baz()` &rarr; `Foo$extensions.kt`.
  - 달러 기호<sup>dollar sign</sup>는 JVM 에서 이름에 사용해도 무방하며 `Foo$extensions.kt` 와 `Foo+Extensions.swift` 를 비슷하게 가져갈 수 있다.
- 다수의 TLF, TLP 를 모아 둔 파일인 경우: `functions.kt`, `properties.kt`. 그 외 `extensions.kt`, `values.kt`, `utils.kt` 등이 쓰이기도 함.
  - 이는 엄밀히 말해서 원칙 위반이지만 간혹 클래스 수나 클래스 생성자 수, 메서드 수가 너무 많아져 성능 저하가 생기는 사고를 막기 위해 이처럼 독특한 기법이 허용된다. 다만 `@file:JvmMultifileClass`, `@file:JvmName` 을 이용하면 TLF 를 각각의 파일로 나눠 정의하면서 클래스 수와 클래스 생성자 수의 범람은 막을 수 있다. 또 다른 이슈는 TLF 나 확장함수가 더러 사용되는 경우 함수형 프로그래밍의 영향으로 개별 TLF 의 역할이 적어지고 코드 길이가 짧아지는 경향이 생기므로, LOC 대비 파일 수 비율이 너무 커지는 것이 작업자의 코드 가독성과 빌드 속도에 오히려 악영향을 준다는 부분. 여러 TLF 를 모아 둔 파일은 Haskell 로 치면 인지적으로 모듈 (`module`) 같은 역할을 하는데 ([Prelude](<https://github.com/ghc/ghc/blob/master/libraries/base/Prelude.hs>)), JVM 동작과 Kotlin TLF `import` 특성상 모듈의 효과는 패키지 분류를 따라가야 하므로 `functions.kt` 같은 파일명을 하나 정해 두는 것은 일정한 이점을 갖는다.
- 요약. TLD 가 `class`, `interface`, `object` 또는 이들의 확장(들)인 경우 파일명은 파스칼케이스를 따른다. 아닌 경우 파일명은 (소문자) 캐멀케이스를 따른다.

다시 확인하자면<sup>To recall,</sup> ktlint 는 파일명에 쓰인 (소문자) 캐멀케이스를 전면으로 거부한다. 사람들을 설득해 이걸 고칠 수 있을까? 아마 당장 너른 동의를 얻어내기는 어려울 것이다. 누군가의 생각에는, 오히려 기존의 단순명료한 철칙을 따르는 사람들에게 복잡한 제안으로 혼란을 준다거나, 어떤 도구든 다른 도구들과 최적으로 어울러 써야 한다는 덕목을 해치고 적정 대안을 제시하지 않는 것이 될 수 있다. 나는 내가 아웃라이어인 것을 인정하고 `filename` 을 불능 처리했다.

(Update @ May 2021)

ktlint 문서에서 `filename` 의 설명이 바뀌었다. 적어도 현재의 동작을 기존 설명보다는 잘 말해 주는 것 같다.

> ```diff
> ## File name
> 
> -Files containing only one toplevel domain should be named according to that element.
> +A file containing only one visible (e.g. non-private) class, and visible declarations related to that class only, should be named according to that element. The same applies if the file does not contain a visible class but exactly one type alias or one object declaration. Otherwise, the PascalCase notation should be used.
> 
> Rule id: `filename`
> ```
>
> ----
>
> <https://github.com/pinterest/ktlint/commit/566eb8a8b28973592f3a1dfa2c9c7f4f41276360#diff-c2c60646b8a539cda9b3e374e3875d17d308b4a19b86a5ef86be1642ca2db0c5L82-R187>

> **File name**
>
> A file containing only one visible (e.g. non-private) class, and visible declarations related to that class only, should be named according to that element. The same applies if the file does not contain a visible class but exactly one type alias or one object declaration. Otherwise, the PascalCase notation should be used.
>
> ----
>
> <https://pinterest.github.io/ktlint/0.49.0/rules/standard/#file-name>

## enum-entry-name-case

이 부분은 `filename` 보다 새 규칙 자체는 간단한데 그 채택 원리는 조금 심오하다. 나는 Kotlin `enum class` 의 항목을 (소문자) 캐멀케이스를 따라 작성하고 있다.

```kotlin
enum class TernaryLogic {
    yes,
    no,
    neitherYesNorNo,
    /* end of enum */;
}
```

(^ 여담: 항상 줄을 내리고 `/* end of enum */;` 로 끝내는 내부 규칙이 함께 적용되어 있다.)

### 상수를 만들려거든 고함을 치세요

주류 규칙은 이를 금지한다. 본래 Java 에서는 같은 열거<sup>enumeration</sup> 유형인 `enum` 의 항목에 이름을 붙일 때 언더스코어<sup>underscore</sup>로 구분해 대문자만으로 작성하는 관습이 있다. 이렇게.

```java
public enum TernaryLogic {
    YES,
    NO,
    NEITHER_YES_NOR_NO,
    /* end of enum */;
}
```

이 대문자의 전통이 Kotlin 에서도 비슷하게 이어진다. 상기한 Kotlin 공식 코딩 컨벤션 문서는 `enum class` 의 항목에 대해 같은 꼴의 이름짓기를 권고하고 있다.

> **Property names**
>
> Names of constants (properties marked with `const`, or top-level or object `val` properties with no custom `get` function that hold deeply immutable data) should use uppercase underscore-separated ([screaming snake case](https://en.wikipedia.org/wiki/Snake_case)) names:
>
> ```kotlin
> const val MAX_COUNT = 8
> val USER_NAME_FIELD = "UserName"
> ```
>
> Names of top-level or object properties which hold objects with behavior or mutable data should use camel case names:
>
> ```kotlin
> val mutableCollection: MutableSet<String> = HashSet()
> ```
>
> Names of properties holding references to singleton objects can use the same naming style as `object` declarations:
>
> ```kotlin
> val PersonComparator: Comparator<Person> = /*...*/
> ```
>
> For enum constants, it's OK to use either uppercase underscore-separated names ([screaming snake case](https://en.wikipedia.org/wiki/Snake_case)) (`enum class Color { RED, GREEN }`) or upper camel case names, depending on the usage.
>
> ----
>
> <https://kotlinlang.org/docs/coding-conventions.html#property-names>

공식 코딩 컨벤션에서는 그나마 대문자 캐멀케이스를 허용하고 있으나, ktlint 는 `enum-entry-name-case` 에서 열거 유형의 항목을 언더스코어로 구분해 대문자만으로 작성하는 것을 강제하고 있다.

> **Enum entry**
>
> Enum entry names should be uppercase underscore-separated names.
>
> ----
>
> <del><https://pinterest.github.io/ktlint/rules/standard#enum-entry></del>
>
> <https://pinterest.github.io/ktlint/0.48.0/rules/standard/#enum-entry> (&larr; Update @ May 2021)

보다시피 이런 모양의 이름짓기는 스크리밍 스네이크케이스<sup>screaming snake case</sup>라고 한다. 얼마 전까지는 저 스네이크케이스조차 `under_score`, `underscore_separated` 라고 대강 불렀는데 (`hyphen-ated` 처럼, 엄밀히는 언더스코어는 케이싱이 아니니 사실 오히려 옳긴 함), 요즘 들어 특히 그럴듯하게 대소문자 구분을 명시하는 이름이 생긴 것이다. 그런데 그 사이에 있었던 이름이 있다. 바로 매크로케이스<sup>macro case; MACRO\_CASE</sup>이다. 매크로란 무엇인가? 매크로라는 이름이 붙은 이 세상의 모든 것들을 여기에 다룰 수는 없지만, 이 부분에서는 잠시 줄이고, 매크로케이스라는 이름의 배경이 된 조립 언어<sup>assembly language</sup>와 C 에서 그것의 역사를 짚어 보도록 하자.

### 매크로케이스의 유래와 쓸모

매크로<sup>macro</sup>. 원래 마이크로<sup>micro</sup>의 반대되는 말로, 거시경제학<sup>macroeconomics</sup>과 미시경제학<sup>microeconomics</sup>을 대조할 때에도 쓰는 바로 그 크고 작다는 뜻이다.

우리는 전통적으로 프로그램을 작성할 때 보조저장장치<sup>secondary storage</sup>에 저장<sup>store</sup>해 두었다가 프로그램을 실행해야 하는 시점에 주기억장치<sup>primary memory</sup>에 이를 적재<sup>load</sup>해 사용하는 편인데, 중앙처리장치<sup>CPU; central processing unit</sup>가 실제로 주기억장치에 있는 프로그램을 실행할 때에도 마찬가지로 프로그램을 중앙처리장치 내부의 레지스터<sup>register</sup>로 적재하는 과정이 이루어진다. 이 레지스터는 우리가 알고 있는 기명 레지스터<sup>named -</sup>뿐만 아니라 무명 레지스터<sup>unnamed -</sup>를 포함하는 개념으로, 웬만하면 우리가 자세히 알 필요도 없고 알아서 도움 될 개념도 아니지만, 0 과 1 의 연쇄로 구성되어 있는 프로그램 부호, 즉 코드<sup>code</sup>는 주기억장치에서는 명령어<sup>instruction</sup> 단위로 존재하지만 레지스터에 적재되면서 명령어보다 더 작은 단위로 분해된다. 이것이 마이크로코드<sup>microcode</sup> 즉 풀이하자면 미소부호이다.

이 반대 개념이 매크로코드<sup>macrocode</sup> 즉 거대부호, 줄여서 매크로이다. 명령어가 사람과 CPU 사이의 규약이라면, 마이크로코드는 CPU 의 효율을 위해 (= 성능을 향상시키려는 목적으로) 명령어를 그보다 더 작은 수준의 일 여러 개로 쪼개는 것이고, 매크로코드는 사람의 효율을 위해 (= 편의를 향상시키려는 목적으로) 명령어 여럿을 묶어 명령어 수준보다 더 큰 일을 하게 만들어 둔 것이다. 다만 마이크로코드가 CPU 내부에만 존재하듯, 매크로코드 역시 사람이 알고 있는 수준에만 존재한다. 사람은 매크로코드를 이용해 프로그램을 작성하더라도, 실제로 프로그램이 주기억장치에 올라가 있게 되면 실행되는 코드 부분은 명령어 단위로만 이루어져 있게 된다.

물론 이와 같은 구분은 이제 꽤나 낡아빠진 것이 되었다. 애초에 이 글에서 다루고 있는 것이 Java, Kotlin, Kotlin/JVM 이런 것들인 현 시점의 사정을 고려해 보자. 프로그램이 자신이 실행해야 하는 계산을 기술한 코드를 자신이 다룰 수 있는 추상<sup>abstract</sup> 자료로 들고 있다가, 적시에<sup>just in time</sup> 컴파일해서 명령어로 만들어 내고, 자신의 실행 성능을 측정하고 분석해서, 열점<sup>hotspot</sup>을 추정해 지목하고, 필요에 따라 자신의 일부분을 또 스스로 수정하는 능력 정도는, 이제 너무나 당연한 것이 되어 있다. 그러나, 이런 환경이 근본적으로 프로그래밍 언어라는 개념의 연장선상에 있는 만큼, 그 근본인 매크로의 정신 역시 두 갈래 정도로 구분되어 오늘날까지 살아남아 있다.

하나는 반복적으로 수행해야 하는 단순한 일련의 컴퓨터 작업으로 확장되는<sup>expanded</sup> 컴퓨터 프로그램 기능 전반을 매크로라는 단어가 의미하게 된 것이다. 컴퓨터 역사 초기에 비해 훨씬 다양한 배경의 사람들이 컴퓨터를 &lsquo;직접&rsquo; 사용할 수 있게 되면서, 사람이 직접 수행하는 것보다 컴퓨터에게 시켜서 훨씬 효율적으로 수행할 수 있는 작업 역시 매우 다양해졌는데, 여러 사람들의 일상에서 경쟁적이거나 소모적인 여러 순간들에 끼어드는 자동화 도구들이 전부 매크로라는 이름을 얻게 되었다. 귀성열차나 콘서트 티켓을 예매하는 매크로, 수강신청 매크로, 엑셀 매크로, 포토샵 매크로 등이 이에 해당하며, 다방면에서 깊은 관심을 얻고 있다.

다른 하나는, 좀더 직접적인 영향으로, 컴퓨터 프로그래밍에서 사람이 작성한 후에는 마찬가지로 &lsquo;확장되어&rsquo; 사라지는, 사람이 인식하는 영역 내지는 상대적으로 프로그램 외부에만 존재하는 이름을 매크로케이스로 작성하게 된 것이다. 당연하지만 대문자로 된 매크로케이스는 텍스트 전반에서 확 튀기 때문에 사실 전반적으로 가독성을 향상시키지는 않는다. 따라서, 컴퓨터로 텍스트를 작성하면서 대소문자를 구분할 수 있게 된 이래로, 매크로케이스는 일반적인 소스 코드를 작성하는 목적으로는 거의 사용되지 않게 되었다. 그러나, 텍스트 전반의 대소문자나 소문자로 된 나머지로부터 대문자로만 된 부분을 골라내는 일은 인간에게나 기계에게나 아주 유리하다는 부분에서, 외려 매크로케이스가 살아남게 된다. 이를테면, 소스 코드에 `sed` 나 `awk` 같은 기본적인 텍스트 처리 프로그램을 적용할 때에 대체될 부분을 매크로케이스로 작성할 수 있고, 이것이 살아남아 C 컴파일러인 `cc` 의 한 단계로 전처리기<sup>preprocessor</sup>인 `cpp` 가 다루는 부분을 매크로케이스로 작성한다 (그래서 전처리기 지시자<sup>- directive</sup>인 `#define` 으로 처리하는 부분을 매크로라고도 한다). 또한 OS 가 프로세스에 제공하는 환경 변수<sup>environment variable</sup>는 셸 스크립트<sup>shell script</sup>에서 그 이름과 값 그대로를 사용할 수 있는데, POSIX 에서는 환경 변수의 이름을 매크로케이스로 정함으로써, 셸 스크립트에서 환경 변수와 그 외의 일반적인 변수가 자연히 구분되게 하였다.

그리고 이로써 매크로케이스로 되어 있는 이름은, 프로그램의 동작 시간에 비해 상대적으로 외부에 정의되어 있어 프로그램이 스스로 변경할 수 없는 값을 (그리고 무의미한 이름을, 어쩌면 사라지는 이름을) 나타내는 것으로 인식되어, 그 이후의 프로그래밍 언어에서도 소스 코드의 특정한 부분에 매크로케이스를 사용하는 전통이 이어진다. 예를 들어 C# 에는 `const` 에 매크로케이스를 사용하는 관습이 있고 ([*const (C# Reference)* @ Microsoft Learn](<https://learn.microsoft.com/ko-kr/dotnet/csharp/language-reference/keywords/const>); 한편 `static readonly` 에는 매크로케이스가 쓰이지 않으며), Java 개발 환경은 `static final` 에 매크로케이스를 사용하도록 권고한다 ([*Google Java Style Guide* &sect; 5.2.4 Constant names](<https://google.github.io/styleguide/javaguide.html#s5.2.4-constant-names>)). Python 에서는 전역 변수와 전역 상수가 구분되지 않으므로 상수에 매크로케이스를 사용하여 사용 상의 유의를 꾀한다 ([PEP 8 – *Style Guide for Python Code* &sect; Constants](https://peps.python.org/pep-0008/#constants)).

### 문제제기

그래서, 매크로케이스는 코드가 변경할 수 없는 고정된 값을 나타낸다는 실체를 나타내고 있다는 부분에서 현실적인 효용이 있는 컨벤션인가? 내 대답은 이렇다: 아니올시다, 대관절 매크로케이스를 왜 여태 쓰고 있는지조차 모르겠소.

```c
#include <stdio.h>

int get_num_magic() {
    int const num_magic = 3;
    return num_magic;
}

int main()
{
    printf("Hello, %d World(s)!\n", get_num_magic());
    return 0;
}
```

위에 보인 C 예시 코드에는 `get_num_magic` 이라는 함수가 있다. `main` 에서는 `get_num_magic` 을 호출하고, 그 함수는 `num_magic` 이라는 변수 이름이 가리키는 주기억장치 지점에서 3 을 찾아서, 그 값을 반환하면, 이 값이 출력될 것이다. 맞지?

아니지. 이 전제는 이제 완전히 틀렸다. 대부분의 경우 아주 기본적인 최적화를 거치면 `num_magic` 이라는 이름이 가리키는 주기억장치 지점은 아예 존재하지도 않는다. 옛날에는 C 에 저장소 분류 지정자<sup>storage class specifier</sup>라는 것이 있었다. `auto`, `register`, `static`, `extern` 으로 구분되는 저장소 분류법은, `auto int const num_magic;` 을 선언하면 주기억장치 지점 할당이 정의되고, `extern int const num_magic;` 을 선언하면 그 정의를 외부에서 찾아야 하는 방식으로 동작했다. 그러나 주기억장치 지점을 직접 정의할 것인지는 모듈 간의 경계에서는 아주 본질적이며 중요할지 몰라도 내부에서는 중요하게 다루고 싶지 않은 문제이고, 머지않아 코드 상의 요소에 대한 이런 종류의 분류는 사람이 수행한 것을 존중하는 것보다 기계가 적극적으로 개입하는 것이 더 나은 결과를 만들어 낼 수 있다는 관점이 힘을 얻는다. 나는 이 당시에 어쩌다 이 정책결정이 되었는지 아주 정확히 알지는 않으나, 어법에 비해 보조적인 분류였던 `auto` 와 `register` 는 그런 역사적 맥락에서 결국 C 언어 표준으로부터 배제된다.

요즘은 컴파일러가 일을 더 많이 하게 만드는 편이다. 실행 시간에 주기억장치에서 레지스터로 바로 적재되는 상수 값이 있다고 해 보자. 이 값을 만드는 데에 계산이 필요하다면, 원래대로라면 매크로로 계산식이 전개되고 특정한 컴파일러의 최적화 규칙을 전제하여 하나의 `register` 를 지정해 이름을 붙여야 할 것이며, 그 값은 실행 파일의 `.rodata` 내지는 `.rdata` 구역에서 실행 시간을 기다릴 것이다. 요즘은? `constexpr` 을 쓴다. 이 지정자는 C++11 에서 최초로 시작되어 12 년만에 지구를 한 바퀴 돌아 C23 에 도착할 예정인데, `constexpr` 은 `constexpr int num_magic = ...;` 처럼 써서 `num_magic` 이라는 이름이 연산을 위해 기억될 필요조차 없게 해 주고, `get_num_magic` 의 실행 시간 중에 어쩌면 오염될 수도 있는 주기억장치 지점 할당이 아예 존재하지 않도록 하는 모종의 강제성을 높이는 효과를 갖는다.

```assembly
movl %eax, $3
ret
```

이와 같은 변화는 근본적으로 호출 스택<sup>call stack</sup>을 갖는 함수 자체가 &mdash; 위 예시에서는 아예 `get_num_magic` 이 통째로 &mdash; 아예 인라인<sup>inline</sup>되어 없어질 수도 있고, `constexpr` 이 아닌 나머지 영역에서는 이에 대한 통제권을 컴파일러가 갖게 되었으며 인간은 웬만하면 굳이 그 결과에 대한 세부사항으로 머리를 싸매지 않아도 된다는 사회적 합의에서 도출된 것이다. 물론 그 과정에서 최적화하지 못할 거라 간주하고 그냥 놓아 준 또 다른 세부사항들에 대한 아쉬움; 이를테면 우리가 항상 복사<sup>copy</sup>에 너무 큰 비용을 지불해 왔다는 반성, 멀티스레드로 작성된 프로그램들에서 꾸준히 벌어지는 실수들, 하지만 포기할 수 없는 성능&hellip; 요컨대 여태 무시하던 세부사항으로 정교하게 큰 퍼즐을 맞추기 위한 좋은 도구가 필요하다는 신념 등에서 Rust 가 태어나고 성장해 오긴 했다만. 그 미래에 대한 개인적 소회는 따로 글의 마지막에서 논해 보고자 한다.

요약하자면, 코드 상에서 어떤 이름지어진 대상이 그 코드가 변경할 수 있는 값 같은 것인지, 외부에서 주입된 토큰으로 확장될 매크로나, 아니면 일종의 환경 변수같은 것인지, 외래 변경의 여지가 있는 주기억장치에 존재하긴 하는지, 이런 부분들은 이미 사람의 손을 떠난지 오래되었다. 생성구조의 이름은 파스칼케이스로, 생성된 개체들이나 그 개체에 적용하는 연산의 이름은 소문자 캐멀케이스를 (내지는 소문자 스네이크케이스를) 따른다는 원칙은 이름짓기에서 다른 무엇보다도 견고하게 유지되어 왔고 , 아주 특정한 범주에 있는 경우에만 스크리밍 스네이크케이스를 써야만 하는 규칙은 사실상 효용이 없다. 예를 들어 Java 에서는 통상 `static final` 값에는 스크리밍 스네이크케이스를 쓰지만 지역<sup>local</sup> `final` 값에는 이를 적용하지 않는데, 실질적으로 개발자인 내가 읽고자 하는 값이 쓸 수는 없는 `final` 인지 아닌지의 판단은 IDE 가 제공하는 코드 하이라이터<sup>code highlighter</sup>와 자동 완성<sup>auto completion</sup>, 문법 오류 시각화 등을 통해 시각적으로 수행하게 된다. 이름짓기의 규칙은 개발자 인간들이 자신을 위해 만드는 것인데, 효과적이지 않고 거추장스럽다면 규칙을 폐기하는 것이 옳지 않겠는가?

비슷한 논리로 진작에 폐기된 규칙이 있는데 바로 헝가리인 표기법<sup>Hungarian notation</sup>이다. 요즘 `TextView` 유형의 참조 값 변수에 이름을 붙인다면 `nameTextView` 가 되거나 더 짧게는 `name` 이 될텐데, 헝가리인 표기법으로는 `tvName` 이 된다. 헝가리인의 이름은 동양인처럼 성명 순으로 되어 있기 때문에 붙은 이름이다. 헝가리인 표기법은 소문자 캐멀케이스로 된 모든 이름의 첫 마디에 그 유형을 나타내는 약어를 소문자로 표기하도록 하는데, 지역 변수가 만들어질 모든 유형에 대해 미리 약어를 정해야 하며 그것이 충분히 간결하고 서로 겹치거나 혼동되지 않아 사람이 보고 즉각 구분해 낼 수 있어야 한다는 전제를 유지하도록 한다. 뭐라고&hellip;? 당연하게도 이 규칙은 이미 죽었는데, 흥미로운 점은 헝가리인 표기법 계열 규칙에 컴파일 시간에 정해지는 상수 이름를 `k` 로 시작하도록 하는 부분이 있어 이쪽에서는 매크로케이스를 진작에 거의 전부 걷어냈다는 사실이다.

```cpp
enum class TerneryLogic : unsigned {
    kYes,
    kNo,
    kNynn,
};
```

즉 유추하자면 C 시기에 발달한 스크리밍 스네이크케이스는 헝가리인 표기법으로도 대체할 수 있는 시각적 정보만을 제공하는 규칙이라는 것데, 헝가리인 표기법이 불필요해졌다면 스크리밍 스네이크케이스가 필요할 일도 더더욱 없을 것이다. 다만 이름에 유형 이름을 포함하는 방법 자체가 전부 틀렸다는 말은 아니다. IDE 의 도움이 없던 시절에는 `int32_t int32_result`, `size_t sz_res`, `HWND hWnd` 같은 이름짓기를 해 왔고, 이 규칙은 아직도 코드 상에서 명시적 유형 변환이 잦을 때 자체적으로 지역변수 간에 구분되는 이름공간<sup>namespace</sup>을 부여하는 데에 유용하기도 한데 (예: `int32_result` 와 `size_result` 가 한 블럭 내에 모두 있는 경우), 만약 간혹 전역적으로 이름공간이 없는 것이 문제가 된다면 이 규칙을 계속 따르는 것도 꽤 괜찮은 방법이 될 수 있다.

```c
enum ternarylogic_t {
    ternarylogic_yes,
    ternarylogic_no,
    ternarylogic_nynn,
};
```

### 결론

그래서 스크리밍 스네이크케이스는? 결론은 그냥 버리자는 것이다. 난 이제 거의 어느 언어에든 파스칼케이스와 소문자 캐멀케이스만 있어도 충분하다고 본다 (소문자 스네이크케이스를 쓰는 경우가 아니라면). 특히 JavaScript 와 같이 자유도가 높은 언어에서는 하나의 이름이 가리키는 것이 `object` 이면서 동시에 `[[Call]]` 이고 `[[Construct]]` 일 수도 있는 일이 더러 있는데, Kotlin 에서도 `object` 인 것은 하나의 이름이 생성구조인 동시에 값이기도 한 방식으로 비슷한 일이 벌어지고 있다는 것을 생각하면, 케이싱을 달리해 이름의 목적을 구분하는 것은 앞으로 점점 더 실효가 희박하고 관념 간에 경계선을 긋기 모호한 일이 될 가능성이 높다. 이 미래는 C# 에서 속성과 메서드 이름에 파스칼케이스를 쓰도록 하던 시절에 예견된 것이다.

대문자의 연쇄와 언더스코어를 아예 걷어낸다면 가변 폭 폰트로 읽는 코드의 가독성에도 일대 혁신이 찾아올 것이고 프로그래밍 언어가 더욱 사람에게 가깝게 다가오는 움직임에도 동력을 보태게 될 것이다. INSTANCE 라고? 너무 대문자라 잘 안 들리는데~, 그거 그냥 instance 라고 써도 되지 않을까?

Kotlin 에서 `enum` 항목뿐만 아니라 `const` 에도 소문자 캐멀케이스를 쓸 것을 추천해 본다.

----

이하 붙임 1 과 붙임 2 는 본문을 작성하던 2 월부터 얼개를 잡아 둔 내용이지만, 사실 자체보다는 해설에 치중한 글의 특성을 고려하여, 아래 구역은 나중에 더한 것으로 일러 둔다.

## 붙임 1/2: ENIAC, 조립 언어<sup>assembly language</sup>, 매크로코드<sup>macrocode</sup>의 태동.

(Update @ Aug 2023)

요즘과 같은 컴퓨터 프로그래밍의 역사는 존 폰 노이만<sup>John von Neumann</sup>, 존 모클리<sup>John Mauchly</sup> 등이 참여한 것으로 유명한 ENIAC 시기에 시작한다. 동시대의 ABC, Colossus, Z3 모두 전쟁을 거치면서 만들어진 것들인데, 이 중 에니악은 탄도 문제<sup>ballistics problem</sup>를 풀려고 만들어진 것으로 유명하다. (발사체 운동 문제<sup>projectile motion problem</sup>이라고도 한다.)

요즘은 온갖 미사일에 꽤 좋은 컴퓨터를 넣어서, 탱크를 따라가고 가속해서 벙커를 터뜨리고 플레어를 무시해서 전투기를 요격하고 우주에 다녀오고 뭐 이런 일들이 벌어지는데, 원래 포탄은 소총탄처럼 장약<sup>propelling charge</sup>의 폭발력으로 추진되어 포신을 떠나면 중력과 공기저항을 받으며 목표 탄착지점에 도달하고 작약<sup>shaped charge</sup>이 터지는 게 끝이다. 그러나 뭐 수십 킬로미터짜리 대포가 정말로 수백 미터짜리 소총과 동등한 것도 아니고, 사람의 감각으로 어림해서는 목표물을 절대 맞힐 수 없기 때문에, 사표<sup>firing table</sup>라는 걸 작성해서 거리와 고도차에 풍향, 풍속, 온습도, 좌표와 방위 영향이 전부 들어간 문서를 야전교범<sup>field manual</sup>으로 만들어 전장에서 아주 마르고 닳도록 사용하게 된다. 특히 WWII 는 전차들의 기동전이었기 때문에, 포병<sup>artillery</sup>이 WWI 시기에 화망으로 화력지원을 하던 수준이 아니라 기갑<sup>armored</sup>이라는 주요 전략자산으로 변모해 활약하게 되었고, 이에 따라 무기체계 전반에 일대 혁신이 일어나는데&hellip;.

문제는 이 사표를 작성하는 게 꽤 복잡한 지식노동을 단순반복해야 하는 고된 일이었다는 사실이다. 그 당시의 계산은 기껏해야 전기식 계산기<sup>electric calculator</sup>를 쓰는 것으로, 조금 큰 수의 산술<sup>arithmetic</sup>이나 되는, 말하자면 전기로 돌아가는 주판같은 것이다 (찰스 배비지<sup>Charles Babbage</sup>가 고안한 차분 기관<sup>difference engine</sup>과 해석 기관<sup>analytical engine</sup>이 이미 19 세기에 고안되었으나 만들어진 적은 없고, 자동화는 대량 입력을 위한 펀치 카드<sup>punched card</sup>로 이루어지는 수준이었다). 계산원<sup>computer</sup>들이 계획<sup>program</sup>을 따라 이 기계를 다루며, 계획에는 수치가 특정 기준을 만족할 때까지 계산을 반복하거나 기준을 만족한 수치로 다른 일을 해야 하는 일이 여러 단계로 제시되어 있다. 그리고 이런 체계를 통해 예측자-수정자 방법<sup>predictor-corrector method</sup> 같은 것까지 개입시켜 미분방정식을 왕창 계산해서 사표를 완성하는 것이다. 그런데 많은 무기들이 신규 개발되면서 화력이 좋고 사거리가 길어지긴 하는데, 목표 지점이 멀어질수록 기존의 모형<sup>model</sup>에는 포함되지 않은 코리올리 효과<sup>Coriolis effect</sup> 같은 것이 끼어들어 사표 작성 자체가 불가능해지고, 기껏 개발 시작한 좋은 화력을 전혀 써먹을 수가 없는 일이 벌어지며, 전쟁 중이니 적국은 이를 달성해서 압도적인 전력차로 패전할 수도 있겠다는 집단적 공포까지 더해진다.

따라서 실제 장비를 통해 얻은 수치들을 분할해 일부는 훈련 집합<sup>training set</sup>이라고 부르고 나머지는 시험 집합<sup>test set</sup>이라고 불러서, 훈련 집합에 모형을 회귀<sup>regress</sup>시키고 이것이 시험 집합에도 충분한 정확도를 갖는지를 판정하는 식으로, 사표를 작성하는 모형 자체가 적절한지를 검증하며 모형을 개선해야 하는 일이 시도된다. 그런데 이게, 지금 보면 말이야 쉽지만, 당시 기준으로는 상기한 계산원들의 고된 지식노동 반복작업 분량을 수십에서 수백 배쯤 뻥튀기하는 끔찍한 일이 벌어지는 것이다. 인간들에게 피로도가 누적되면 인적 오류<sup>human error</sup>가 산발하는 것은 피할 수 없는 일이고, 교육은 아주 오래 걸리며, 무기체계를 발전시키는 것이 비싸고 어려운 일에서 그냥 불가능한 일이 되어 버린다. 이런 배경에서 계산의 절차<sup>process; procedure</sup> 자체를 자동화하고자 탄생한 것이 ENIAC 이다.

이후 이와 같은 전산화를 통한 질적 개선은 미국의 성공도식이 되어, 현실 복잡계의 모든 것을 촘촘히 계량해서 그것을 설명하는 다변량 해석<sup>multivariate analysis</sup> 모형을 찾아내고 조작적으로 접근하려는 뚜렷한 기조, 예를 들어 미국 심리학, 로보틱스, 자동번역, 오늘날의 ML 기반 AI 같은 것들이 발달하는 데에 DoD 예산이 주력으로 투입되는 연구개발 문화에 지대한 영향을 주었으며, 이것이 제조업 강국이었던 미국에서 기초과학과 공학 인프라에 대한 투자가 되는 추세로 연결되어 IT 분야에서는 인터넷과 C 프로그래밍 언어 같은 것들이 미국에서 최초로 만들어지며 산업계로 이어져서는 글로벌 IT 공룡들의 탄생까지도 이어지는 (하지만 이런 고도의 지성화에 상반되게 노동시장의 상품 질을 올리는 보편 교육 정책에는 별 관심이 없어서 사회 문제까지 생기는) 어디에도 없고 오직 미국에만 있는 특유의 산업구조를 만드는 데에도 한몫 하게 된다.

아무튼 다시 ENIAC 과 매크로로 돌아가면, ENIAC 에도 ENIAC 부호화 체계<sup>ENIAC coding system</sup>이라는 프로그래밍 언어가 있었다. 그러나 이건 우리가 알고 있는 프로그래밍 언어가 아니라 말 그대로 계획, 즉 업무 계획을 계산원들에게 전달하기 위해, 수행가능한 단계들로 이루어진 유한절차로 알고리듬<sup>algorithm</sup>을 만들어 둔, 오늘날의 유사코드<sup>pseudocode</sup>와 비슷한 것이었다 (이는 Z3 에 쓰인 플란칼퀼<sup>Plankalkül</sup>도 비슷하며, 이 분야는 정형 명세 언어<sup>formal specification language</sup>로 발전한다). 그러나 현실 속의 ENIAC 은 상당히 지루한 진공관 덩어리였고, 산술과 비교와 반복을 수행하는 회로는 미리 구성되어 있지만 그걸 어떻게 조립<sup>assemble</sup>해서 프로그램을 동작시킬지를 구성하는 것은 인간의 일이었다. 당연히, 얼마 후 이 부분에서도 개념과 구현을 분리해서 양쪽 모두를 명확히 하고 그 변환까지도 자동화하려는 시도가 있었으니, 이 중 구현부의 조립 언어가 니모닉<sup>mnemonic</sup>을 활용한 조립 언어이다 (그리고 개념부의 수학적 요구사항만 기술하는 언어가 해스켈 커리<sup>Haskell Curry</sup>의 커리 표기법<sup>Curry notation</sup>이다).

최초의 조립 언어에는

TBW

## 붙임 2/2: ChatGPT, 프롬프트 엔지니어링<sup>prompt engineering</sup>, 노코드<sup>NoCode</sup>의 부상.

(Update @ Oct 2023)

TBW
