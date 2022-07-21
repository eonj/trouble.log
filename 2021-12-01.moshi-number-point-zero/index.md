Trouble ID `2021-12-01.moshi-number-point-zero`

# Moshi n.0

```kotlin
val numberJsonAdatperFactory = JsonAdapter.Factory { type: Type, _: MutableSet<out Annotation>, _: Moshi ->
    if (Types.getRawType(type) != Double::class.java) null
    else object : JsonAdapter<Double>() {
        override fun fromJson(reader: JsonReader) = reader.nextDouble()
        override fun toJson(writer: JsonWriter, value: Double?) {
            value ?: return
            if (0.0 == this % 1.0) writer.value(this.toInt())
            else writer.value(this)
        }
    }.nullSafe()
}
```

## JSON 앞담에 입을 떼며

현 시점에서 JSON 입출력 동작의 표준은, 사실상<sup>de facto</sup> ECMAScript 5.1 에 도입된 빌트인 `JSON` 오브젝트에 대한 Node.js 의 동작으로 받아들여지고 있다. 아직도 수많은 JSON 구현체들이 상호운용성<sup>interoperability</sup> 문제에 시달리고 있는 상황에서, JavaScript 의 부분집합을 표방하며 만들어진 “JavaScript Object Notation ” 의 참조 구현을 JavaScript 표준에서 찾고자 하는 것은 자연스러운 일일 것이다.

JSON, 이 넋나간 유사기술표준 욕을 하자면 끝이 없지만, 아무튼 ECMAScript 가 그나마 참조 알고리듬을 제시해 둔 `JSON.parse` 와 `JSON.stringify` 에 비해 상당히 많은 서드파티 구현이 실수(?)하고 있는 부분이 있다. 이 중 하나가 JSON number 표현을 다루는 방법이다.

## Number 나가신다

JSON 에 수치를 기입하는 기본적인 방법은 JSON number 를 그대로 사용해 십진수로 표현하는 것인데, 컴퓨터가 평소 연산에 십진수를 사용하지 않기 때문에 문제가 생긴다. 정수라면 원본 이진수 값을 십진수로 표현하거나 그 역을 수행하는 데에 그나마 별 문제가 없다. 그러나 실수를 유한한 유효숫자로 근사하는 체계인 부동소숫점<sup>floating-point</sup> 값이라면 원본 이진수 값과 십진수 표현 사이의 변환이 아주 골치아픈 수치해석<sup>numerical analysis</sup> 문제가 된다. 특히 JSON 은 현실에서 이 변환을 가장 많이 하게 만드는 이유가 된다. 이 분야의 유명한 알고리듬인 Dragon4 와 Grisu3 가 가장 많이 쓰이는 부분이다.

JSON number 의 데이터 모델은 기본적으로 ECMAScript number 즉 IEEE 754 binary64 (흔히 double 이라고 부르는 그것) 를 상정하고 있지만, 상황이 이렇다 보니 대개의 서드파티 구현은 JSON number 를 가능한 한 정수 유형으로 해석하려고 한다. 즉 소수부가 없으면 그냥 int 같은 유형으로 읽어 버리는 것이다. 모두가 다루기 어려워하는 double 을 사용하지 않는 것, 특히 어떤 다른 구현이 double to string 이나 string to double 을 제대로 할지 적당히 신뢰할 수 없는 상황에서 double 사용을 최대한 회피하는 것은, 어떤 덕목을 실천하게 해 주는 좋은 일일 수 있다.

이런 툴링이 취하는 JSON number 파싱 전략은 다음과 같다. (1) 소수부가 없으면 int 로 읽는다. (1a) 소수부가 없긴 한데 signed 32-bit 범위를 넘는다면, 필요에 따라 BigInt 같은 것을 제공한다. (double 에서 온 값이기 때문에 얼마든지 이럴 수 있다.) (2) 소수부가 있으면 비로소 double 로 읽는다. 특히 2 번이 중요한데 왕복<sup>round-trip</sup>을 보장하기 위해서는 역으로 JSON number 로 표현하는 과정에서도 모든 double 값에 소수부를 써야 하기 때문이다. 그래서 이런 구현에서 int 0 값은 JSON 으로는 `0` 이 되는데 double 0 값은 JSON 으로 보내면 `0.0` 이 된다.

정말 많은 곳에서 이런 동작을 기본으로 하고 있다. 일단 JSON 을 런타임에 어떤 값으로 읽어 낸다. 이 중에 일부만 사용하고 수정한다. 다시 저장하면, 그 필드를 제외한 나머지는 변함이 없어야 할 것이다. 또한 int 값을 저장했으면 나중에 그것을 읽었을 때 int 값으로 나오고, double 값을 저장했으면 double 값으로 나와야 할 것이다.

## Node.js 랑 기싸움이라도 하는 건가

그러나 이런 툴링이 간과하고 있는 것은 이게 JSON 의 근본으로 돌아가는 ECMAScript 런타임과 함께 사용하면 보존적인 왕복 운용이 불가능하다는 사실이다. ECMAScript 에는 int 가 따로 없고 double 에 해당하는 number 만 있기 때문에 double 0 과 같은 값을 JSON number 로는 `0` 으로 변환한다. 이것이 빌트인 `JSON` 오브젝트의 동작이다. 그럼 그게 어떤 툴링에서는 int 로 읽혔다가 `0` 이 될 테니 문제가 없다고 생각할 수 있겠는데, 클래스 유형에 붙는 툴링에서 클래스 유형에 어떤 필드 (Java) 또는 프로퍼티 (Kotlin) 를 double 로 지정했을 때 문제가 된다. 이 경우 JSON number 로 `0` 이더라도 double 로 읽었다 쓰면 `0.0` 이 되고, 다시 Node.js 로 읽었다 쓰면 `0` 이 되어서 JSON 파일에 계속 무의미한 변화가 생기는 무한의 굴레에 빠지게 된다.

음&hellip; 바로 Moshi 가 이 문제를 똑같이 일으킨다. 솔직히 나는 JSON number 의 데이터 모델은 ECMAScript number 로 보고 (-0, ±Infinity 와 NaN 제외), 근본적으로 읽을 때 항상 double 로 변환해야 하며, int 는 JSON 으로 보낼 수 없도록 하고, double 0 은 `0.0` 이 아니라 `0` 으로 써야 한다고 생각하는 편이다. 그러나 정말 이게 정책이 된다면 대부분의 사람들은 double to int or v.v. 를 직접 해야 하는 상황을 아주 불편하게 여기고 그냥 쉽게 쉽게 쓸 수 있는 다른 툴링을 찾을 것이다.

아니면 다른 대안으로, 어떤 JSON number 에 소수부가 없으면 int 로 해석해도 좋고, double 은 JSON number 로 보내면 소수부를 만들지만, 클래스 유형의 필드나 프로퍼티가 double 로 되어 있다면 1 의 정수배 값은 JSON number 로도 소수부 없이 표현되도록 변환하는 것이 기본동작이 될 수는 없을까 생각이 든다. 그러나 이 역시 사람들이 자연스럽게 생각하는 것이, double 값을 저장했던 것은 이후 클래스 유형 정의 없이도 툴링의 기본동작으로 double 값으로 읽어낼 수 있어야 한다는 요구사항이라면, 아마 내가 말한 건 이해하기도 어렵고 기본 동작이 되기에는 저항이 있겠다는 생각이 든다.

이렇게 골치아픈 JSON 의 실체를 똑바로 다루도록 툴링이 개발자들을 좀 독려했으면 좋겠다만, 세상엔 서버의 Java 코드에서 int 로 생각한 값을 웹브라우저의 JavaScript 코드에서 number 로 적당히 다루고 싶어하는 사람들을 위한 도구가 자리를 잡는다. 수학 연구도 아니고 생업인데 매번 깊은 생각을 하게 강요하는 도구라면 아주 질색이지 않겠는가? JSON 은 그렇게 우리의 일상이 되었다.

## 맺음

아무튼 맨 위의 Kotlin 코드는 Moshi 의 `JsonAdapter.Factory` 구현을 하나 작성한 것으로, 모든 Kotlin `Double` 값에 대해 1 의 정수배 값은 JSON number 로는 소수부 없는 표현으로 변환되도록 해서 Node.js 와의 상호운용에서 왕복 동작을 갖추는 방법이다. Kotlin `Int` 의 범위를 넘지 않는 값이라면, 잘 작동할 것이다. 그 외의 경우는 피하면 된다. JSON number 로 변환될 정수 값은 웬만하면 signed 32-bit 범위를 지키는 것이 다른 구현을 고려하는 방법이기도 하다.

참고로 위 코드는 클래스 유형 없이 `Map` 을 사용하는 경우에도 마찬가지로 동작하니, JSON object 가 `Map` 으로 파싱되도록 하는 코드가 있다면 오히려 왕복 동작을 희생하는 셈이 된다. 나는 `Any` 와 `moshi-kotlin` (`kotlin-reflect` 기반) 을 아예 배제하고 `@JsonClass` 가 붙은 클래스 유형을 만들고 있으니 이 단점은 회피할 수 있게 된다.

## Appendix A. Moshi `IllegalStateException: Nesting Problem.`

```
java.lang.IllegalStateException: Nesting Problem.
```

가끔 이런 일이 생긴다.

- 내가 뭔가 잘못했다.
- 이 코드는 내 잘못을 알아서 적당히 봐 줄 생각이 없다.
- 코드가 런타임에 오류 메시지를 하나 남기고 뻗는데 그로부터 내가 뭘 잘못했는지 알 수 있는 게 하나도 없다.

이번에는 Moshi 님을 눈치껏 모셔 드리고 왔다. 뭘 잘못했는가 하면,

```kotlin
@JsonClass(generateAdapter = true)
data class T
constructor(
    val someField: U?,
)
```

위와 같이 JSON 파싱을 위해 데이터 유형을 Kotiln data class 로 정의하고 Moshi 어댑터를 컴파일타임에 생성한다. `T` 클래스는 `object` 의 하위 유형을 파싱하고, `someField` 필드는 필수 값이 아니며, 이 필드의 파싱은 `U` 클래스에 맡겨진다.

`U`  유형에 대한 커스텀 어댑터를 등록했을 때 문제가 생긴다. `someField` 필드 자체의 탈/직렬화<sup>de/serialization</sup>가 모두 `U` 유형에 대한 커스텀 어댑터에 맡겨지기 때문이다. 이는 `U` 유형 자리의 값에 대한 탈/직렬화를 `U` 유형에 대한 커스텀 어댑터에 맡긴다는 말과는 다르다. 즉 `someField: U?` 같은 사례가 있다면, `U` 유형에 대한 커스텀 어댑터는 `U?` 유형을 다룰 줄 알아야 한다.

`EnumJsonAdapter` 라든지 `PolymorphicJsonAdapterFactory` 에도 같은 문제가 적용된다. 방법은 맨 위의 코드와 같이, 커스텀 어댑터에 이 구문을 추가하는 것이다. Moshi 소스 코드를 보면 기본 어댑터는 모두 이게 적용되어 있다.

```
.nullSafe()
```

