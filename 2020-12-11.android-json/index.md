Trouble ID `2020-12-11.android-json`

# Android 에서 JSON 파싱이 이상해졌음

## tl;dr:

When:

```
JSONObject["key_here"] is not a string.
```

Don't:

```
-keep class org.json.** { *; }
```

Do:

```
implementation('some-group:some-module:some-version') {
    exclude group: 'org.json', module: 'json'
}
```

## how:

시작은 Android 앱에서 JSON 파싱 관련해 이런 오류 로그를 만난 것이었다.

```
JSONObject["key_here"] is not a string.
```

여기에서 `JSONObject` 는 `org.json.JSONObject` 이다. `JSONObject` 에 대해서 `getString` 메소드를 호출했는데, 값이 string 유형이 아니라고 실행시간 예외가 발생한 것이다. JSON object 값에서 해당 필드의 값은 number 유형으로 되어 있었다. `{ "key_here": 1 }` 라는 느낌으로.

이 오류의 아주 괴상한 점은 **오류가 릴리스 빌드에서만 나타난다**는 것이다. 즉 디버그 빌드에서 `JSONObject` 의 `getString` 은 정수값을 알아서 문자열로 바꿔서 읽어 주는 동작을 했다. 난독화를 거친 릴리스 빌드는 그렇지 못했다.

스토어에 나갈 앱에 난독화를 안 하면 이런 문제는 피하겠지만 애석하게도 거대한 비즈니스 요구사항께서는 그걸 허락하시지 않는다.

어떤 블로그에는 이렇게 써 있다. **Proguard (R8) 파일에 아래 라인을 추가해라.**

```
-keep class org.json.** { *; }
```

<https://puch-android.tistory.com/2>

음&hellip; 좋다. `org.json` 패키지를 통째로 Proguard Keep 처리해 버린다면 해결되겠지. 그리고 실제로 해결된다.

그러나 지난 버전에서는 릴리스 빌드에서 정상 동작하던 코드가 왜 이제는 디버그 빌드에서만 정상 동작하는지 알아야 하지 않겠는가?

정답은 다음과 같다. **빌드스크립트를 수정해라.** (Groovy Gradle DSL 기준)

```
implementation('some-group:some-module:some-version') {
    exclude group: 'org.json', module: 'json'
}
```

Kotlin Gradle DSL에서는&hellip; 원래 `exclude(mapOf("group" to "org.json", "module" to "json"))` 처럼 썼던 걸로 기억하는데, 아마 지금은 DSL 버전이 좋아져서 `exclude(group = "org.json", module = "json")` 처럼 쓸 수 있게 됐을 거다.

그럼 어떤 라이브러리에서 추이적 의존성으로 들어온 `org.json:json` 을 찾을 수 있나? Gradle 에는 `dependencies` 작업이 있다. 릴리스 빌드의 의존성만 보려면 `--configuration` 옵션을 준다.

```
$ ./gradlew --no-daemon :app:dependencies --configuration releaseCompileClasspath
```

우리의 경우에는 최근에 도입한 Emoji 라이브러리가 문제가 됐다.

```
...
+--- com.vdurmont.emoji-java:5.1.1
     \--- org.json:json:20170516
...
```

## WHY

요즘 JSON 처리를 기본 제공하지 않는 곳이 있을까? 수 년 전까지는 데이터를 어떤 형식으로 적재/보관하고 또한 어떤 형식을 기대해서 넘길 것인지에 대해 많은 싸움이 있었다. 전통의 XML 이 있었지만, XML 얘기는 생략해도 될 것 같다. Protocol Buffer 나 MessagePack 같은 바이너리 대안은 프론트엔드로 넘어오면 별 장점을 발휘하지 못했다. 다들 갑론을박하던 MongoDB 의 BSON 이나 PostgreSQL 의 JSONB 도 마찬가지였다. 결국 JSON 이 최후의 승자로 남았다.

Android 는 API level 1 부터 프레임워크 라이브러리의 일부로 (무려 libcore 에서) `org.json.*` 패키지에 JSON 런타임 라이브러리를 기본 제공하고 있다. 이 패키지 컨벤션은 Sean Leary \[stleary\] 의 JSON-java 에서 가져온 것인데, 패키지를 따서 org.json 이라는 별명으로 더 많이 알려져 있다. 이 라이브러리가 Java 에서 JSON 처리의 표준이다. 그래서, 요즘은 대부분의 라이브러리가 마찬가지로 JSON 데이터를 직접 생산/소비할 수 있도록 `org.json:json` 에 의존성을 두고 있다.

패키지명과 클래스명만 같다는 점에 주목할 필요가 있다. Android 런타임의 `org.json.*` 클래스는 org.json 에서 가져온 것이 아니고 org.json 의 명세를 보고 새로 클린룸 구현된 것이다.

실제로 AOSP platform/libcore 를 보면 `JSONObject.java` 파일이 추가된 커밋케시지 첫 줄은 다음과 같다.

```
A cleanroom implementation of the org.json API.
```

<https://android.googlesource.com/platform/libcore/+/51a095f0bc7aadfcc7e6b3873b97c050c523d102>

이 Git 저장소에서 `JSONObject.java` 도 잠깐 살펴 보자. `android-10.0.0-r47` 태그까지의 기록이다. 중간에 조금 생략했다. (tag 94b6ad5367; object 2b72e0b848)

```
30ba19d Add nullability annotations to JSONObject. by Pete Gillin · 2 years, 1 month ago
3b0456b Add @UnsupportedAppUsage to non-ojluni classes by Paul Duffin · 2 years, 2 months ago
        ...
1483c41 Javadocs for JSONObject. by Jesse Wilson · 11 years ago
014aa1a Adding an Apache-licensed implementation of org.json by Jesse Wilson · 11 years ago[Renamed from json/src/rewrite/java/org/json/JSONObject.java]
51a095f A cleanroom implementation of the org.json API. by Jesse Wilson · 11 years ago
```

<https://android.googlesource.com/platform/libcore/+log/refs/heads/master/json/src/main/java/org/json/JSONObject.java>

Android libcore 의 `org.json.*` 구현은 문서화된 API 이기도 하기 때문에 다른 구현들과 함께 Zygote 에서 런타임으로 주입될 것이고 난독화를 거치지도 않는다. org.json 을 포함해서 앱을 빌드하면, Android 런타임에는 org.json 과 libcore 양쪽에서 온 `org.json.*` 클래스가 존재하게 된다. 난독화를 거치지 않으면 이 둘 중 하나가 실행될 것이다. 난독화를 거치면 실제로는 앱에 포함된 org.json 라이브러리만 난독화를 겪게 되므로 org.json 라이브러리가 dispatch 될 것이다.

구현을 보면 Android libcore 의 `JSONObject#getString` 은 JSON 원본 값이 number 이더라도 string 으로 변환해 주는 타입 강제를 수행한다. 그러나 org.json 의 경우, 그런 타입 강제가 `JSONObject#getString` 에는 없고 `JSONObject#getInt` 에는 있다.

이 두 가지 사실이 만나면 앱의 릴리스 빌드에서만 저런 예외를 만나는 것을 예상할 수 있게 된다.

org.json 라이브러리의 동작을 정상으로 간주하고 싶을 수도 있다. Android 런타임이 제공하는 `org.json.*` 구현은 사용자 OS 버전에 따라 동작이 달라지기 때문에, 라이브러리 버전을 이용해 코드의 동작을 정확히 통제하고 싶다면 이런 전략은 유효할 때가 있다. 그러나 클래스 리네이밍이 없다면, 한 런타임에 둘 또는 그 이상의 `org.json.JSONObject` 클래스가 존재할 것이다. 이것은 별로 바람직한 일이 아니다. 두 라이브러리의 동작이 정확히 같다는 보장도 없으니 빌드 결과물을 들여다 보지 않으면 동작을 예측하기 어렵게 될 것이다.

나는 이 건에 있어서는 이런 기술결정을 지지할 수 없다. 내 기준에서 이 라이브러리는 Android 앱 빌드에 포함되기에 부적절하다.

아무튼 Maven 추이적 의존성을 대충 쓰는 게 이럴 때 문제가 되는데, 나도 모르는 사이에 `org.json:json` 라이브러리가 추가되어 `org.json.*` 클래스패스가 오염되고 기존에 Android 런타임 구현을 쓰던 우리 코드가 자동으로 외래 라이브러리 구현을 쓰도록 변해 버렸기 때문이다.

앞으로는 의존성 트리를 분석해서 라이브러리 의존성 트리가 요구사항을 제대로 만족하는지를 주기적으로 검사할 방법이 필요할 것 같다.
