Trouble ID `2019-06-21.wow-much-generic-collection`

Note: 이 글은 Tumblr 로 발행되었으나 (<https://a4.aurynj.net/post/185727107583>), Tumblr 마크다운 에디터의 고질적 문제로부터 도망쳐서 이곳으로 왔습니다.

# 와! 제네릭 컬렉션!

제네릭스<sup>generics</sup>는 정말 많은 프로그래밍 언어에 들어 있는 기능이지만 의외로 제네릭스가 제대로 유형체계<sup>type system</sup>에 들어오게 된 것은 그리 오래 되지 않은 일이다. 클래스 기반 객체지향 언어인 C\#, Java 모두 첫 버전에서는 제네릭스를 지원하지 않았다. 극단적인 안전성을 추구하는 Go에서는 아예 제네릭스 지원이 없으며, Rust는 제네릭스를 지원하지만 struct와 trait/impl을 분리해 버리는 지경에 이르게 되었다. C\+\+에서는 별 문제 없이 템플릿 인자로 온갖 값을 다 넣고 있지 않은가? 대체 뭐가 문제야?

결론부터 말하자면 제네릭 컬렉션 설계가 겁.나.어.렵.기 때문이다. 가끔 우리가 미래지향적 프로그래밍에 도취되다 보면, 가끔 어떤 유형을 만들 때마다 ‘이 클래스는 컨테이너의 일종이니까 제네릭하게 만들어야 하지 않을까?’ 하고 생각하게 되기도 한다. 좋다, 제네릭 클래스를 만든다고 별 문제는 없다. 하지만 제네릭 유형을 진짜로 잘 만든다면 그거야말로 좋겠지? 근데 그게 잘 따져 보면 사실 보통 일이 아니므로, 관심이 있다면 계속 읽어 볼 수 있을 것이다.

그렇다. 이 글은 미래지향적 제네릭스의 주화입마에 빠진 내 경험을 담은 글이다. 그래서 앞으로의 이야기는 Java와 Kotlin으로 진행된다. (수정: 랑데브에서 이 글을 읽고 용어사용에 대해 논의를 함께해 주신 모든 분들께 감사드린다.) (주: 그럼에도 불구하고 다소 읽기 어려울 수 있음.)

## 사람의 모임이면 동물의 모임인 거 아닌가?

Java의 `java.lang` 패키지에 있는 내장 유형<sup>builtin type</sup>인 `String`과 `CharSequence`를 예로 들어 보자. `CharSequence`는 인터페이스이고 (`interface`), `String`은 파이널 클래스인데 (`final class`), `String`이 `CharSequence`를 구현한다 (`implements`). 이를 `String`이 `CharSequence`의 하위 유형<sup>subtype</sup>이라고도 하고 `CharSequence`가 `String`의 상위 유형<sup>supertype</sup>이라고도 한다. 이것을 앞으로 `String <: CharSequence`라고 쓸 것이다.

그리고 Java의 또다른 내장 유형인, `java.util`에 있는 `Set`과 `HashSet`이 있다. `Set`은 인터페이스인데 수학 개념인 집합<sup>set</sup>에서 이름을 빌려 온 것으로, 특정 유형을 인자<sup>parameter</sup>로 해서 그 의 값들의 모임을 만들고 그 안의 값들은 그 모임 안에서 유일하도록 보장하기 위해 만들어진 것이다. `HashSet`은 클래스로 `Set`의 한 구현체이다. 그리고 `Set`은 사실 `Set<T>`라서 `Set<String>`처럼 문자열의 집합임을 명시할 수 있고 `HashSet<String>`은 `Set<String>`의 구현체가 된다.

설명이 길었는데, `String`이면 `CharSequence`이다. 그럼 `Set<String>`은 `Set<CharSequence>`이기도 할까? 답은 **아니다** 이다. 아래 코드는 컴파일 에러가 난다.

```java
import org.junit.Test;

import java.util.HashSet;
import java.util.Set;

public class TypeSystemTest {

    @Test
    public void helloType() {
        final Set<String> stringSet = new HashSet<>();
        final Set<CharSequence> charSequenceSet = stringSet;
    }
}
```

부조리함을 느꼈는가? `T <: U` 여도 `Set<T> <: Set<U>`가 아니다. 모든 `String`인 인스턴스는 동시에 그 자체로 `CharSequence`인 인스턴스이기도 하다. 그러나 `String`의 모임은 `CharSequence`의 모임이 아니다! 대체 왜?

## 배열부터 살펴봅시다: 일해라 유형체계야

어디에서나 만나 볼 수 있는 가장 친숙한 제네릭 유형은 배열이다. 어떤 유형이 있으면 보통 그것의 배열을 만들 수 있다. 정수 유형이 있으면 정수 배열 유형을 만들 수 있고, 문자열 유형이 있으면 문자열 배열 유형을 만들 수 있다.

C\+\+에서는 배열 유형 간에는 변환을 할 수 없다: C 스타일로 강제 변환하거나 포인터 유형으로 `reinterpret_cast`를 쓰지 않는다면. `static_cast`를 쓰지 못하는 이유는, 배열은 본질적으로 값들이 모여 있는 위치가 있는 무언가에 대해 이름을 붙인 것이고, 그래서 값을 `static_cast` 할 수 있더라도, 값에 대한 참조를 역참조했을 때 자동으로 `static_cast`가 적용되지는 않기 때문이다. 그래서 `float` 값을 `static_cast<int>` 하거나 `int` 값을 `static_cast<float>` 할 수는 있지만, `int *` 유형을 `float *` 유형으로 만들려면 `reinterpret_cast<float *>` 해야 한다. (그리고 읽을 때는 UB로 그 동작이 정상일지에 대한 보장은 없다.)

한편 Java에서는 `String[]`을 `CharSequence[]`로 유형배역<sup>typecast</sup>할 수 있는데 대충 생각하면 여기서 전혀 유형 이론적 문제를 느낄 수 없다. `String`의 모임이면 `CharSequence`의 모임이기도 한 거 아닌가? **맞는 말이다, 그 모임의 내용을 바꿀 일이 없다면.** 대체로 배열에 대한 접근은 값들을 늘어 놓은 메모리 위치에 대한 접근과 사실상 같은 것으로 여겨졌기 때문에, 배열을 갖고 있으면 그 배열의 특정 위치에 대한 값을 바꿀 수 있다. 단적인 예시로 다음 Java 테스트 코드는 컴파일 에러를 내지 않는다. 그 대신, 런타임에 `ArrayStoreException`이 날 것이다. (일부 IDE는 린트 에러를 내 주기도 한다.)

```java
import org.junit.Test;

public class TypeSystemTest {

    @Test
    public void helloType() {
        final String[] strings = { "hi" };
        final CharSequence[] charSequences = strings;
        charSequences[0] = new StringBuilder();
        final String s = strings[0];
    }
}
```

이것이야말로 진정한 부조리이며 Java의 유형체계가 똑바로 일을 하지 않고 있다는 명백한 증거이다. 어쩌겠는가, 배열 유형을 다른 배열 유형으로 유형배역하는 Java 코드는 이 세상에 이미 셀 수 없이 많이 존재하며 이제 와서 금지할 수도 없다. 그나마 `Set<String>`을 `Set<CharSequence>`로 마음대로 바꾸지는 못하는 게 다행일 것이다.

## 방법을 찾아봅시다: 함수부터

Java와 같은 JVM 언어이자 함수형 언어인 Scala가 이런 문제를 일찍이 해결하였고 그 방식이 Kotlin에도 일부 적용되었다. Scala에서는 어떤 함수를 사용할 때 그 함수의 인자 유형과 반환 유형을 어떻게 바꾸어 취급해도 되는지에 대해 **공변/반변**<sup>variance</sup>을 이용해 정의한다. 그리고 제네릭 유형 역시 일종의 함수이기 때문에 동일한 논의가 성립한다.

공변/반변 역시 본래 수학 용어이다. 어떤 공간의 대상을 나타낼 때 우리는 어떤 단위가 되는 것을 정의하고 그 단위들을 잘 조합하면 그 공간의 모든 대상을 나타내기에 모자람이 없도록 한다. 이것을 기저<sup>basis</sup>라고 하는데, 예를 들어서 2차원 평면에서는 `x`축과 `y`축을 정의하게 되며 각 축 단위벡터 `(1, 0)`과 `(0, 1)`을 기저로 정할 수 있다.

대체로 어떤 공간이 있다고 해서 기저들<sup>bases</sup>이 유일하게 정해지는 것이 아니다. 2차원 공간의 모든 점들을 나타내기 위해서는 `(1, 1)`과 `(1, -1)`을 기저로 정할 수도 있고 `(-1, -1)`과 `(1, -1)`을 기저로 정할 수도 있다. 기저를 `u = (23810, -19993)`로 정하고 `v = (-37877, 54712)`로 정하면, 본래 `(1, 1)`이었던 어떤 점은 이제 `4409/25972279 u + 14601/181805953 v` 로 표시해야 할 것이다.

이렇게 어떤 공간 상의 모든 대상은 그 공간의 각 기저에 대해 기저 성분을 하나씩 갖고 있다. 예를 들어 벡터의 기저 성분의 크기는 그 기저의 크기 변화와 반대로 움직인다. 반대로 벡터와 같은 선형결합이지만 기저 성분의 크기가 기저의 크기 변화와 함께 움직이는 것이 있는데 이것이 코벡터이다: 기저가 `u`와 `v`에서 `u'`와 `v'`로 변하면 코벡터 `3 u + 2 v`는 `3 u' + 2 v'`가 된다. 우리는 벡터처럼 대상의 기저성분과 기저의 크기 변화 방향이 반대인 것을 **반변**<sup>contravariant</sup>이라고 하고 코벡터를 **공변**<sup>covariant</sup>이라고 한다. 어떤 대상들은 그 기저로 된 성분을 전혀 갖고 있지 않은데 그러면 기저의 변화에도 대상에 변화가 생기지 않을 것이다. 이를 **불변**<sup>invariant</sup>이라고 한다.

프로그래밍 언어에서의 유형 역시 상기한 `<:` 관계를 따라 반순서격자를 이루며, 따라서 **어떤 유형으로 다른 유형을 나타낼 수 있다면 그 사이에도 공변/반변 논의가 성립한다.** 일례로 `String` 유형의 인자를 하나 받아서 다시 `String`을 내놓는 함수가 있다고 하자. 그 함수의 결과값은 상위 유형인 `CharSequence`로 취급할 수 있다. 그러나 그 함수에 `CharSequence` 값을 넣으면 안 된다. 함수 내부에서 `String`에만 정의된 연산에 의존할 수 있기 때문이다. 이때 Scala에서는 `(String) -> String` 함수유형을 `(String) -> CharSequence`로 취급해도 된다고 표현한다(앞으로 모든 함수유형은 Kotlin 꼴로 표시하겠다). 즉 `String <: CharSequence`일 때 `(String) -> String <: (String) -> CharSequence` 이다. 반대로 `(CharSequence) -> CharSequence` 함수에는 `String` 값을 인자로 넣어도 되지만 그 결과를 마음대로 `String`으로 바꿔 쓸 수는 없으므로, `(CharSequence) -> CharSequence <: (String) -> CharSequence`이다.

따라서 **모든 함수유형은 그 인자 유형에 대해서 반변이고 반환 유형에 대해 공변이다.** 즉 임의의 `T, T', U, U'`에 대해 `T' <: T, U <: U' |- (T) -> U <: (T') -> U, T -> U'`이며 `T`와 `U`가 `V <: V'`와 무관하다면 `(T) -> U` 역시 `V <: V'`에 불변이다. 이 이야기는 별 쓸모도 없는데 Scala 관련해서는 여러 글에서 Haskell의 모나드만큼이나 지겹도록 많은 글에서 발견되며 함수형 프로그래밍은 뭔가 대단한 것 같다. 심지어 한국어로 된 글 대부분은 번역어를 공변, 반공변, 무공변이라고 쓰고 있는데, 수학적 근거를 전혀 찾을 수 없어 이해를 한층 어렵게 하는 주범이다. 엄밀히는 공변, 반변, 불변이 맞다. 아무튼 별 거 아니고 우리가 `Set<String>`과 `Set<CharSequence>`에 원하는 바로 그것을 공변/반변이라고 부른다.

## 방법을 찾아봅시다: 클래스 유형은 어떻게?

사람들이 함수형 프로그래밍을 열심히 하지 않는 Kotlin에서는 이것이 의외로 꽤 아무 생각 없이 널리 쓰이고 있다. (이래서 이론 덕후들이 함수형 언어의 가장 큰 적이다.) 일단 함수 객체의 유형부터가 Kotlin에서 공변/반변을 지원하는 방식으로 정의되어 있기 때문이다. Kotlin에서는 제네릭 유형 인자에 `in`/`out`을 붙여서 그 인자에 대해 유형이 공변인지 반변인지 나타낼 수 있게 한다. `<in T>`는 `T`를 인자로 받는다는 뜻으로 반변을 의미하며 `<out T>`는 `T`를 반환한다는 뜻으로 공변을 의미한다. Kotlin 표준 라이브러리에는 `Function<*>`과 `Function1<*, *>`이 각각 `interface Function<out R>`, `interface Function1<in P1, out R>`로 정의되어 있다. 이로써 함수는 반환 유형에 대해 공변이고 인자 유형에 대해 반변으로 정의된다. 그래서 다음 코드는 정상이고, 유형을 배역하는<sup>typecasting</sup> 순서를 뒤집으면 컴파일 에러가 난다.

```kotlin
import org.junit.Test

class TypeSystemTest {

    @Test
    fun helloType() {
        val icf: (CharSequence) -> Int = { it.length }
        val isf: (String) -> Int = icf
    }
}
```

자, 이제 다시 Java의 `Set`으로 돌아가 보자. Java의 `Set`에는 `size()`도 있고 `isEmpty()`도 있고 `contains()`와 `containsAll()`도 있다. 자, 이터레이터도 있지만 `Iterator`에는 `remove()`가 있다. 그리고 `Set`에는 `add()`가 있고 `remove()`가 있고 `clear()`가 있다. Java의 초기 유형체계 설계자들은 대체로 깊은 생각이 없으며 모든 제네릭 유형을 컬렉션으로 취급해서 배열처럼 유형 인자에 대해 공변으로 만들 수도 있었을 것이다. 그러나 **Java의 제네릭 클래스 유형은 기본적으로 유형 인자에 대해 불변이다.** 이는 일견 안타까울 수도 있으나, 다행인 것은 이런 결정이 없었다면 Java 8에 `Consumer<T>`나 `Supplier<T>`나 `Function<T, R>` 등등을 만들 수 있는 일은 없었을 것이다. 그리고 이는 Java 8에서 람다식을 만들면서 `invokedynamic`을 만들어야 했던 결정적인 이유이기도 하다. 람다식의 유형은 맥락 없이도 소극적으로 유추할 수 있어야 하는데 Java 컴파일러는 제네릭 유형의 공변/반변에 대한 이해가 없고 Java 런타임은 아예 제네릭 유형에 대한 이해가 없으며 따라서 그 람다식의 값을 쉽게 다른 곳에 넘길 수 없기 때문이다.

결과적으로 Java에서 `Set<T>`는 `T`에 대해 불변이며 `Set<String>`과 `Set<CharSequence>`는 전혀 무관한 유형이다. 잠깐 짚고 넘어가자면 `T`가 달라지면 `Set<T>`가 전혀 무관한 유형이 되는 걸 불변이라고 할 수 있을까? 다소 웃기긴 하다만 맞는 말이다. 제네릭 유형은 일종의 유형 모임<sup>type class</sup>이며 유형 생성자<sup>type constructor</sup>인데, 유형 해소<sup>type resolution</sup> 과정에서 `Set<>`가 어떤 완결<sup>fully resolved</sup> 유형을 인자로 받아 다른 완결 유형을 내놓는 함수이며 따라서 `Set`은 하나의 유형으로 취급될 수 없다는 뜻이다. Kotlin스러운 느낌으로 대충 말하자면, `Set<T>`가 `T`에 대해 불변이라면 `Set<T>`의 결과가 될 수 있는 유형은 `T`에 대해 `Nothing`이나 다름없을 것이다. 하지만 실제로 `Set<>`의 결과가 될 수 있는 유형을 모으면 `T`에 대해 `Any?` 처럼 된다. 자, `Any?`와 `Nothing`은 Kotlin 유형체계의 반순서격자에서 양 끝에 있는 유형이다 (`verum`과 `falsum`). 그리고 `Any?`인 값을 가지고는, 그 하위 유형인 무언가로 정제해 내지 않으면 아무 것도 할 수 없다 (`Nothing`). `toString()`이라도 하려면 일단 `null`일 가능성부터 제거해야 한다. 그래서, `Set`는 `T`에 대해 불변이다. 실제로 JVM에서도 `Set`의 유형 시그니처를 보면 유형 인자가 날아가고 그냥 `Ljava/lang/Set;`이다. 유형 인자가 있는 유형들을 런타임에 어떻게 다룰 것인가에 대한 문제는 의존적 유형<sup>dependent type</sup> 같은 불가산 문제도 걸려 있고 해서 쉬운 문제가 아니다. 아무튼 `Set`은 여전히 `Set`이고 동시에 아무것도 아니므로 불변이다.

다시 Java로 돌아와서. 그래서 유형 이론적으로 Java의 `Set<T>`는 `T`에 대해 공변이어야 옳을까, 아니면 반변이어야 옳을까? 안타깝게도 이 문제에는 괜찮은 답이 없고 불변으로 두는 것이 옳다. 컬렉션에 너무 빠지지 말고 `AtomicReference`같은 사례를 보자. `AtomicReference<T>`에는 `get()`과 `set()`이 있으며 각각의 유형은 `() -> T`, `(T) -> Unit`이다. 즉 `AtomicReference<T>`는 `T`에 공변인 함수 정의를 포함하며 또 `T`에 반변인 함수 정의를 포함하기도 한다. 따라서 `AtomicReference<T>` 자체는 `T`에 공변일 수도 반변일 수도 없다. 오직 `AtomicReference::get`이 `T`에 공변이고 `AtomicReference::set`이 `T`에 반변이다. 이는 `String[]`을 `CharSequence[]`로 바꿔 읽을 수는 있지만 `String[]`을 `CharSequence[]`로 바꿔 써서는 안 되는 문제와 동일한 모습이다.

그러면 반대로 `T`에 대한 제네릭 유형을 정의할 때, `T` 유형을 받아 내용을 바꾸는 컬렉션이 아니라면 `T`에 공변으로 정의해도 되지 않을까? 그렇게 **Kotlin은 이 문제를 컬렉션 자체에 값 가변성<sup>mutability</sup>을 통제하여 해결하였다.** 거의 모든 것이 참조 기반으로 작동하는 JVM에서 컬렉션의 값 가변성을 통제하는 것은 함수형 프로그래밍에서 장점으로 여기는 고도의 재현가능성<sup>reproducibility</sup>을 확보하는 데에도 중요하지만, 동시에 컬렉션 유형을 공변적으로 만드는 데에도 중요한 역할을 한 것이다.

좀 더 자세히 살펴 보자. Kotlin에서는 기본적으로 `Set <: Collection <: Iterable`이며 이는 Java와 똑같다. 중요한 건 그 다음인데 `Iterable`에는 `iterator()`가 있고 반환 유형이 `Iterator`이며 여기에는 `next()`, `hasNext()`만 있다. 자, `MutableIterator <: Iterator` 이고 `remove()`는 `MutableIterator`에만 있다. 그렇게 `MutableSet <: MutableCollection <: MutableIterable`이 정의되고, `MutableSet`은 `Set`의 하위 유형이기도 하다. 그렇게 `Set`과 `Collection`에는 값을 바꾸지 않는 `isEmpty()`, `contains()`, `iterator(): Iterator`가 정의되며, `MutableSet`과 `MutableCollection`에만 `add()`, `remove()`, `clear()` 그리고 계승 (override) 정의인 `iterator(): MutableIterator`가 정의된다.

상기한 모든 유형은 모두 정의상 `<out T>`이며 `T`에 대해 공변이다 - 당연히 `Mutable`인 것 전부 빼고. `MutableIterable`, `MutableCollection`, 이하 `MutableSet`를 비롯한 `Mutable`인 Kotlin의 컬렉션은 모두 `T`에 불변이다. 이로써 `Set`의 `T`에 대한 공변/반변 문제는 일단락되었다. `T <: T'`이면 `Set<T> <: Set<T'>`이다.

## 해결된 줄 알았지?

이제 좀 복잡한 컬렉션인 `Map`의 사례를 보자. `Map`은 상황이 좀더 복잡한데 Kotlin에서나 Java에서다 일단 `Map`은 `Collection`이 아니며 애초에 `Iterable`도 아니다. `Map` 값을 순회하기 위해서는 Java에서는 `entrySet()`으로 집합으로 바꾸어야 하며 Kotlin에서는 `entries` 속성에 대응한다.

Kotlin에는 `Map`의 경우 역시 하위 유형으로 `MutableMap`이 분리되어 정의되어 있으며 `MutableMap<K, V>`에는 `put()`이 있고 `remove()`가 있고 `clear()`가 있다. 그리고 `Map.Entry<K, V>`가 있는 것처럼 `MutableMap.MutableEntry<K, V>`가 있다. 자, 일단 이 `MutableMap.MutableEntry`에서는 `key`나 `value`가 `val`에서 `var`로 계승되어 있는 것이 아니라 `setValue()`라는 새 메서드가 정의된다. 그리고 놀랍게도 Kotlin의 `MutableMap<K, V>`에는 `Map`의 속성인 `keys`, `values`, `entries`의 계승 정의가 등장하며 `entries`의 유형은 `MutableSet<MutableMap.MutableEntry<K, V>>`이다. 심지어 `keys`의 유형은 `MutableSet<K>`이고 `values`의 유형은 `MutableCollection<V>`이다. `entries` 속성으로 `Set` 유형으로 변환된 값을 받았는데 그게 `MutableSet`이라서 내용을 조작할 수 있는 게 `MutableMap`이라는 말인가? 그럼 그 `MutableSet`에 뭔가를 추가하면 `MutableMap`에도 추가되고, 거기서 뭔가 삭제하면 따라서 삭제되는가? 심지어 `keys`나 `values`에도 추가나 삭제를 할 수 있고 그러면 따라서 `K`/`V` 값만 추가/삭제되어야 하는가? 어떻게 구현하라는 것인지 전혀 이해가 되지 않는 상황이다. ((실 구현은 런타임에 따라 다르다 (`actual`). Kotlin JVM 런타임에서 `kotlin.collections.HashMap`은 Java 런타임에 의존하는 `java.util.HashMap`의 얕은 재추상화 레이어이므로 (`typealias`), `keys`/`values`/`entries`로 `MutableSet` 인스턴스가 나오면 이 내용은 기존 `MutableMap` 인스턴스의 내용과 동일하게 유지되며, 다만 `MutableSet` 에서는 추가가 불가능하다. Kotlin JavaScript 런타임에서 `kotlin.collections.HashMap`은 자체 구현되어 있으나 (`class`), Java 명세와 같은 제약조건으로 동작한다.))

나는 Kotlin의 그럴듯한 유형체계를 만나고 신나서 내 손으로 Guava의 `BiMap`과 `Multimap`과 `Multiset` 비슷한 걸 만들다, 이 상황에 깊은 염증을 느끼고, (그냥 `entries`를 불변 컬렉션으로 정했으면 될텐데 굳이) 새로운 체계를 만들어 보았다. 새로운 체계에서는 값 가변성을 '깊은 값 가변성' (deep mutability, 이하 DM)과 '얕은 값 가변성' (shallow mutability, 이하 SM)이라는 두 단계로 나눈다. 그리고 이를 따르는 `Map`을 `UniqAssoc`로, `Set`을 `Uniq`로 재명명하고 `MultiAssoc`와 `Multi`에 `UniqBiAssoc`, `MultiBiAssoc`까지 이처럼 만들어 보기로 하였다. 이 체계에서 유형 간의 관계는 다음과 같다.

> `DeepMutUniq<T> <: ShalMutUniq<T> <: Uniq<T>`
>
> `DeepMutUniqAssoc<K, V> <: ShalMutUniqAssoc<K, V> <: UniqAssoc<K, V>`

DM인 컬렉션 인스턴스는 값 가변인 다른 컬렉션 인스턴스로 변환할 수 있으며, 변환 중에 SM인 컬렉션으로 바뀐 적이 없다면, 최종 유형의 컬렉션 인스턴스나 그로부터 얻어낸 참조에 대한 변경 연산이 최초의 컬렉션 인스턴스로 전파되어야 할 것이다. 즉 `DeepMutUniqAssoc<K, V>`의 `entries` 속성으로 `DeepMutUniq<DeepMutUniqAssoc.MutEntry<K, V>>` 유형의 값을 얻으면, 그 집합에 `DeepMutUniqAssoc.MutEntry<K, V>` 값을 추가하면 최초의 연관배열에도 그 항목이 추가되고, 집합에서 값을 삭제하면 최초의 연관배열에서도 값이 삭제되도록 해야 한다는 것이다. 한편 `ShalMutUniqAssoc`는 이래야 할 제약을 갖지 않는 것으로 애초에 `entries`의 유형이 `Uniq<UniqAssoc.Entry<K, V>>`으로 정의된다. 다만 얕은 값 불변인 컬렉션에는 `put()`, `remove()`, `clear()` 따위가 정의된다.

이 체계는 내가 `UniqAssoc`를 기존 유형 기반으로 대강 정의하던 시점까지는 꽤 괜찮았다. 아직은 빌드도 잘 된다. (앞으로는 꽤 긴 여정이 되므로, 시행착오를 따라가고 싶지 않다면 전부 건너뛰고 결론부로 바로 가기 바란다.)

```kotlin
interface UniqAssoc<K, out V> {

    val size: Int
    val entries: Set<Entry<K, V>>

    fun isEmpty(): Boolean
    fun get(key: K): V

    interface Entry<K, V> {

        val key: K
        val value: V
    }
}

interface ShalMutUniqAssoc<K, V> : UniqAssoc<K, V> {

    fun put(key: K, value: V)
    fun remove(key: K): V?
    fun remove(key: K, value: V): Boolean
    fun clear()
}

interface DeepMutUniqAssoc<K, V> : ShalMutUniqAssoc<K, V> {

    override val entries: MutableSet<MutEntry<K, V>>

    interface MutEntry<K, V> : UniqAssoc.Entry<K, V> {

        override var value: V
    }
}
```

나는 이 체계에 이미 내재하는 문제를 인지하지 못한 채 끊임없이 해결하고 싶은 문제를 늘려 갔다. 이를테면, `ShalMutUniqAssoc`에 있는 메서드는 모두 컬렉션을 조작하는 것이다. 따라서 컬렉션을 조작하는 인터페이스만 따로 있으면, 그 인터페이스는 반변적으로 사용할 수 있을 것이다! 아, 그리고 `MutEntry`는 `value` 값을 조작할 수 있도록 되어 있으므로 `V`에 대해 반변이어야 한다!

자, 결과를 보자.

```kotlin
interface OutUniqAssoc<K, out V> {

    val size: Int
    val entries: Set<Entry<K, V>>

    fun isEmpty(): Boolean
    fun get(key: K): V

    interface Entry<K, out V> {

        val key: K
        val value: V
    }
}

interface InUniqAssoc<K, in V> {

    fun put(key: K, value: V)
    fun remove(key: K): V?
    fun remove(key: K, value: V): Boolean
    fun clear()
}

interface InOutUniqAssoc<K, V> : InUniqAssoc<K, V>, OutUniqAssoc<K, V>

interface DeepInOutUniqAssoc<K, V> : InOutUniqAssoc<K, V> {

    override val entries: MutableSet<MutEntry<K, V>>

    interface MutEntry<K, out V> : OutUniqAssoc.Entry<K, V> {

        override var value: V
    }
}
```

두 가지 문제가 보이는가? 자애로우신 컴파일러께서는 우선 `InUniqAssoc`의 `fun remove(key: K): V?`를 지적하셨다. `InUniqAssoc`의 `V`는 `in`으로 정의되어 있는데 `out` 위치에 `V` 유형인자를 사용하고 있다는 것이다. 이것은 아무튼 `remove()`의 반환 유형을 제거해서라고 해결할 수 있을 것이다. 두 번째 문제는, `DeepInOutUniqAssoc.MutEntry`의 `V`가 `in`으로 정의되었으나 그 상위 유형인 `OutUniqAssoc.Entry`의 `V`는 `out`으로 정의된 점이다. 나는 이를 해결하기 위해 본질적으로 `InUniqAssoc.InEntry`와 `OutUniqAssoc.OutEntry`를 분리 정의해야 한다고 생각했으며 이를 하위에서 통합하기 위해서는 `UniqAssoc`과 `UniqAssoc.Entry`라는 상위 유형을 만들어야 했다.

통제할 수 없는 수준으로 문제가 커진 코드를 보자.

```kotlin
interface UniqAssoc<K, ??? V> {

    val entries: Set<Entry<K, V>>

    interface Entry<K, ??? V> {

        val key: K
        val value: V
    }
}

interface OutUniqAssoc<K, out V> : UniqAssoc<K, V> {

    override val entries: Set<OutEntry<K, V>>

    fun isEmpty(): Boolean
    fun get(key: K): V

    interface OutEntry<K, out V> : UniqAssoc.Entry<K, V> {

        override val value: V
    }
}

interface InUniqAssoc<K, in V> : UniqAssoc<K, V> {

    fun put(key: K, value: V)
    fun remove(key: K)
    fun remove(key: K, value: V): Boolean
    fun clear()

    interface InEntry<K, in V> : UniqAssoc.Entry<K, V> {

        override var value: V
    }
}

interface InOutUniqAssoc<K, V> : InUniqAssoc<K, V>, OutUniqAssoc<K, V> {

    interface InOutEntry<K, V> : InUniqAssoc.InEntry<K, V>, OutUniqAssoc.OutEntry<K, V> {

        override var value: V
    }
}

interface DeepInOutUniqAssoc<K, V> : InOutUniqAssoc<K, V> {

    override val entries: MutableSet<InOutUniqAssoc.InOutEntry<K, V>>
}
```

`UniqAssoc`과 `UniqAssoc.Entry` 코드에는 `???`라고 물음표를 써 놓았다. 이 부분에 `in`이나 `out` 중 무엇을 넣어도, 심지어 아무것도 넣지 않아서 `V`에 대해 불변을 만들어도 이 코드는 컴파일할 수 없다. 하위 유형이 모두 일정한 공변성이나 반변성을 따르게 할 수 없기 때문이다.  `OutUniqAssoc`은 `out V`이고 `InUniqAssoc`은 `in V`이다. 그래서 `UniqAssoc`은 아예 `V`를 유형 인자로 갖지 않아야 한다. `UniqAssoc`의 값 유형을 `Any`로 바꾸고, `V`가 `Any`의 하위 유형이 아니므로 `Any`를 다시 `Any?`로 바꾸고, 마지막으로 `InEntry`의 `var value`는 값을 넣을 수만 있는 게 아니라 뺄 수도 있으므로 `in V`를 `V`로 바꿔 주고, 그러면 드디어 컴파일이 되는 코드가 나온다.

소개합니다, `In`과 `Out`이 분리된 새 시대의 `Map`: `UniqAssoc`. 아무 짝에도 쓸모 없습니다 (그나마 다시 빌드 가능함).

```kotlin
interface UniqAssoc<K> {

    val entries: Set<Entry<K>>

    interface Entry<K> {

        val key: K
        val value: Any?
    }
}

interface OutUniqAssoc<K, out V> : UniqAssoc<K> {

    override val entries: Set<OutEntry<K, V>>

    fun isEmpty(): Boolean
    fun get(key: K): V

    interface OutEntry<K, out V> : UniqAssoc.Entry<K> {

        override val value: V
    }
}

interface InUniqAssoc<K, in V> : UniqAssoc<K> {

    fun put(key: K, value: V)
    fun remove(key: K)
    fun remove(key: K, value: V): Boolean
    fun clear()

    interface InEntry<K, V> : UniqAssoc.Entry<K> {

        override var value: V
    }
}

interface InOutUniqAssoc<K, V> : InUniqAssoc<K, V>, OutUniqAssoc<K, V> {

    interface InOutEntry<K, V> : InUniqAssoc.InEntry<K, V>, OutUniqAssoc.OutEntry<K, V> {

        override var value: V
    }
}

interface DeepInOutUniqAssoc<K, V> : InOutUniqAssoc<K, V> {

    override val entries: MutableSet<InOutUniqAssoc.InOutEntry<K, V>>
}
```

허망하지 않은가? 컬렉션 유형에서 유형인자의 공변성과 반변성을 엄밀히 하고, 공변인 연산과 반변인 연산을 나누었고, 그것을 하나의 클래스 유형 꼴로 묶으려고 했더니 최상위 연산에서 아예 유형 힌트가 사라졌다.

더 가관인 것은 `Uniq`를 직접 구현하고 `InUniq`와 `OutUniq`를 나누었을 경우이다. 이때 `InUniq` 유형의 `entries`는 `InUniqAssoc`에 정의되어야 한다. 그렇게 하면 **우리는 원하는 것을 절대 얻지 못하게 된다.**

```kotlin
interface Uniq

interface OutUniq<out T>: Uniq {

    val size: Int
    fun isEmpty(): Boolean

    fun contains(value: T): Boolean
}

interface InUniq<in T>: Uniq {

    fun add(value: T)
    fun remove(value: T)
    fun clear()
}

interface InOutUniq<T>: InUniq<T>, OutUniq<T>

interface UniqAssoc<K> {

    val entries: Uniq

    interface Entry<K> {

        val key: K
        val value: Any?
    }
}

interface OutUniqAssoc<K, out V> : UniqAssoc<K> {

    override val entries: OutUniq<OutEntry<K, V>>

    fun isEmpty(): Boolean
    fun get(key: K): V

    interface OutEntry<K, out V> : UniqAssoc.Entry<K> {

        override val value: V
    }
}

interface InUniqAssoc<K, V> : UniqAssoc<K> {

    override val entries: InUniq<InEntry<K, V>>

    fun put(key: K, value: V)
    fun remove(key: K)
    fun remove(key: K, value: V): Boolean
    fun clear()

    interface InEntry<K, V> : UniqAssoc.Entry<K> {

        override var value: V
    }
}

interface InOutUniqAssoc<K, V> : InUniqAssoc<K, V>, OutUniqAssoc<K, V> {

    override val entries: InOutUniq<InOutEntry<K, V>>

    interface InOutEntry<K, V> : InUniqAssoc.InEntry<K, V>, OutUniqAssoc.OutEntry<K, V> {

        override var value: V
    }
}
```

기본적으로 `OutUniq`는 `T`에 대해 공변으로 정의되지만 `OutUniq`에 정의된 `contains()`가 `(T) -> Boolean` 유형으로 `T`가 `in` 위치에 있으며 `T`에 대해 반변이다. 따라서 **이것부터 틀려먹었다.** 게다가 `InUniqAssoc`의 `entries`와 `InEntry`, 그리고 `InEntry`와 `value` 간의 공변/반변을 맞추려면 `InUniqAssoc`을 `V`에 불변으로 정의하는 수밖에 없다. (`V`에 반변으로 만들기 위해 `InUniqAssoc`과 `InEntry`를 만들었는데?) 마지막으로 `InOutUniqAssoc`의 `entries`는 `InOutUniq<InOutEntry<K, V>>` 유형인데 이것은 놀랍게도 `InUniq<InEntry<K, V>>`의 하위 유형이 아니기 때문에 `InUniqAssoc`의 `entries`를 계승하지 않는다. `InOutUniq`가 `T`에 대해 불변이기 때문에!

아직 `keys`와 `values`는 정의하지도 않았다. 그것들이 `InUniqAssoc`에서 알맞게 정의되려면 `replace` 같은 듣도 보도 못한 연산 하나만 정의된 `InplaceInUniq`를 정의해야 하고 `InUniq`의 상위 유형이 되어야 한다.

## 대체 뭐가 진짜 문제인가

`InOutUniq<InOutEntry<K, V>> <: InUniq<InEntry<K, V>>`가 성립하지 못하게 되어 버린 것은 본질적으로 클래스 유형에 공변/반변을 정의한 데 있다. `InUniq`가 `T`에 반변이므로 추이적으로 `InOutUniq<T> <: InUniq<T> <: InUniq<T'>`가 되기 위해서는 `T' <: T`여야 하는데, 우리는 `InOutEntry <: InEntry`로 정의했다. 그럼 순서를 `InEntry <: InOutEntry`로 옮겨 볼까? 순식간에 `InEntry <: InOutEntry <: OutEntry`가 되어, `OutEntry`의 어떤 하위 유형에서는 `value`를 꺼낼 수 없게 되고, 그게 뭔지는 유형체계를 통해 미리 알 수 없기 때문에, `OutEntry`의 존재 의의가 사라진다.

**우리는 흔히 클래스 기반의 객체지향 언어에 대해서 프로토타입 기반의 객체지향 언어보다 강유형<sup>strongly typed</sup>이라고 즐겨 말하고는 한다.** 이는 클래스 기반 언어들이 프로토타입 기반 언어들에 비해 대체로 컴파일 타임에 미리 더 많은 에러 가능성을 유형 오류<sup>type error</sup>로 잡아 주기 때문이다.

그러나 이 말은 **틀렸다.** 클래스는 클래스이고 프로토타입은 프로토타입이다. 다만 클래스는 객체의 생성구조<sup>construct</sup>이기 때문에 런타임에 어떤 객체 인스턴스의 유형 검증을 하기에도 굉장히 쉬운 기준인 반면에, 프로토타입 기반 언어의 코드를 유형 관점에서 미리 검증하기 위해서는 기술적으로 훨씬 많은 어려움이 따르게 된다. 결과적으로 프로토타입 기반 언어에 대한 코드에도 충분한 노력을 들이면 클래스 기반 언어에서 하는 것만큼의 유형 검증을 미리 할 수 있다. 그게 잘 안 되니까 대부분의 객체지향 언어에서는 그런 검증을 포기하고 프로토타입으로 된 값들을 쓰는 코드의 유형 이론적 오류는 그냥 런타임에 터지도록 놔 둘 뿐이다.

사실 진짜 강유형 언어는 클래스를 정의하지 않는 함수형 언어들에 있다. 본래 모든 함수는 하단<sup>bottom</sup> 유형을 (`falsum`) 이용해 정의되었고, 한번 정의한 함수를 여러 번 쓰기 위해서 다형성<sup>polymorphism</sup>이 개발되었다. **다형성 역시 객체지향 프로그래밍의 고유한 장점인 것처럼 널리 알려져 있지만 완벽한 거짓말이다**: 애초에 여러 클래스에 같은 함수가 다르게 작동하도록 정의한 것이 아니라, 한 함수를 여러 완결 유형에 일정하게 작동하도록 정의한 것이다. 그래서 함수형 언어에서 어떤 유형은 그 유형의 값을 받아 연산할 수 있는 모든 함수들 자체로 정의된다. 그렇게 해야 함수가 어떤 유형으로부터 그 유형에 대해 어떤 다른 함수를 쓸 수 있는지를 유추할 수 있기 때문이다.

오늘날 클래스를 불변값처럼 쓰는 사례가 많아지고 있고, 객체지향 언어에서 현실을 객체로 직접 모델링하는 것을 피하여 공유된 변하는 상태<sup>shared mutable state</sup>를 줄이는 것이 여러 설계 패러다임이나 패턴을 통해 권장되고 있다. 그러나 근본적으로 클래스는 모델링의 도구로 태동하였고 상태를 갖도록 설계되었으며 인터페이스 역시 마찬가지이다. 그래서 그 상태를 바꾸는 연산이 클래스나 인터페이스에 정의되며, 하나의 값을 여러 곳에서 참조하게 되면 당연히 그 상태를 바꾸는 연산에도 제약이 만들어져야 한다. 위의 `In`/`Out` 구분이 그것을 유형 체계에 편입하려는 시도를 단적으로 보여 준다.

그래서 안타깝게도 공변/반변을 통해 어떤 클래스 유형에 `In`과 `Out`을 동시에 정의하고 동시에 `InOut`이 잘 작동하도록 하는 방법은 없다. `InOut`이 성립하려면 우리가 값을 쓰고 있는 대상과 값을 읽고 있는 대상이 하나라는 제약이 필요하다. 이는 값을 쓰고 읽는 동작이 순서에 의존하게 만들고 이는 상태에 의존하는 것이다. 그러나 공변/반변을 정확히 정의하기 위해서는 `In` 대상과 `Out` 대상이 하나일 수 없어야 한다. 그러면 우리는 함수에 `In`으로 어떤 값을 넣은 후 `Out`으로 다른 값을 받으면 함수를 통해 새로 만든 정보는 `Out`으로 받은 값에만 포함되어 있고 `In`에 넣은 값에는 섞이지 않았을 것이라고 가정하게 된다. 이것이 함수형 프로그래밍이며 당신이 짜고자 하는 코드가 진정 안전하면서 유형 관점에서도 충분한 일반성(제네릭)을 갖게 하는 가장 확실한 방법이다. 반대로 클래스 기반 객체지향에서 자꾸 유형을 일반화하다 보면 `Entry`에서의 사례와 같이 `In`/`Out` 구분을 할 수 없게 되고 `Uniq`에서의 사례와 같이 유형 인자가 사라지는데 이 과정이 바로 유형 삭제<sup>type erasure</sup>이다. 유형 삭제는 함수형 언어에서는 예외 없이 항상 일어나는 일이다.

Kotlin은 함수형 프로그래밍을 포함하는 언어로 만들어졌지만 사실 그 용법은 대체로 전혀 함수형이 아니다. `In`/`Out`을 만들기 시작했던 이유에서 드러나듯 `MutableMap`같은 값 가변 컬렉션이 있으며, `forEach`는 철저히 순서대로 작동하고, 람다식에서도 외부 값들에 마음대로 접근하고 외부 상태를 조작할 수 있다. 결국 Java와 함께 쉽게 쓰이기 위해서 만들어진 것이고, 그래서 기본적으로 항상 상태에 직접 의존하지만 가끔 그렇지 않은 부분을 명시할 수 있도록 한 것이다.

앞서 언급했듯 Go에는 제네릭스가 없고 Rust의 경우 struct와 trait/impl을 분리했다. Go 2.0은 제네릭스를 포함할 예정으로 알려진 베이퍼웨어이다. 난 Go 2.0이 나올 것이라고 생각하지 않는다. 그리고 Rust는 기본적으로 값 불변 참조<sup>immutable reference</sup>와 이동 의미론<sup>move semantics</sup>을 전제하기 때문에, trait/impl은 정말 피할 수 없는 선택이었다고 생각한다. Kotlin에서 `Set`과 `MutableSet`을 분리한 것처럼, trait/impl을 분리하는 것은 코딩 스타일에도 도움이 되지만 제네릭 유형과 다형성을 잡기 위해서는 필수적이다.

그러면 클래스 유형을 쓰는 Kotlin에는? 미안하지만 **나는 애초에 Kotlin에는 `in`/`out` 따위가 그냥 들어오지 말았어야 한다고 생각한다.** Java처럼 제네릭 메서드의 유형 경계<sup>type bound</sup> 명세법이 있어서 `<U super T>`라든지 `<? extends T>` 식으로 푸는 수준에서 놔뒀어야 한다는 것이다 (이건 현재 Kotlin에서도 `where` 같은 걸 써서 아주 잘 할 수 있다). 클래스 유형에 공변/반변을 정의하면? `Set<T>`는 `T`에 공변이고 `MutableSet<T>`는 `T`에 불변이다: 와, 정말 좋은 아이디어이다. 아까 `OutUniq`의 `contains()` 가 `(T) -> Boolean`이어서 답이 없는 `in`/`out` 충돌이 있었다. 그런데 Kotlin에서는 클래스 유형으로 `Set<out T>`을 정의하고 심지어 확장함수도 아닌 메소드로 `contains()`를 정의했다. Kotlin에서는 이를 어떻게 해결했는지를 붙이고 이 글을 마친다.

```kotlin
public operator fun contains(element: @UnsafeVariance E): Boolean
```

