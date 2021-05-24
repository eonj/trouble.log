Trouble ID `2020-01-25.moshi-kotlin-android-proguard`

# Moshi Kotlin &amp; Android ProGuard (R8)

**대체로 소스 코드는 죽는다.** 소프트웨어 엔드프로덕트의 맥락에서 소스 코드는 죽는 것이 자연의 섭리이기 때문이다. 그런데 소스 코드 상에서 이름을 지어 놓은 것들은 죽어서 이름을 잘 남기는 편이다. 그러나 이렇게 소스 코드가 죽어서도 살아 있을 시절의 흔적이 많이 남으면 엔드프로덕트는 공원이 아니라 공동묘지가 되며&hellip; 소스 코드가 무덤을 기어나와 비즈니스 로직과 서비스 아키텍처과 중요한 데이터에 대해 증언할 수 있게 될 것이다. 그래서 가끔은 빌드 아티팩트에서든 프로비저닝에서든 소스 코드를 비밀로 하는 것이 무시할 수 없는 과제가 된다.

엔드유저에게 나가는 빌드 아티팩트를 만들 때 소스 코드 기밀성을 위해 **난독화<sup>obfuscation</sup>**를 한번 하는 건 Android 앱의 경우에도 예외가 아니다. 물론 요즘은 오픈소스 소프트웨어가 워낙 많아서 자유 소프트웨어 계열이든 아니든 난독화는 하지 않거나 부분적으로만 하는 경우가 많은데, 난독화를 비롯한 후처리 과정 중 어떤 것은 런타임 성능에 도움이 되지만 어떤 것은 런타임 성능을 희생하는 것이 한 가지 이유라 할 수 있겠다.

난독화의 핵심은 소스 코드의 요소들이 죽어서 남긴 것들을 청소하는 것이다. 이는 묘지에서 이름이 적힌 묘비석을 치우는 것과는 조금 다른 일이다. 이름을 아예 제거해 버리는 게 불가능한 경우가 다반사이기 때문이다. 따라서 어떤 이름을 기존과 무관하게 전혀 다른 이름으로 바꾸는 작업이 가해진다. 이를 **심볼 리매핑<sup>symbol remapping</sup>**이라고 하며 이것이 난독화의 기본이다. 그 외에 어떤 핵심 데이터를 (양방향) 암호화한다거나, 실행 코드의 제어 흐름을 바꾼다거나, 엉뚱한 코드를 섞는다거나 하는 잔기술이 포함된다.

그렇다. 사실 난독화는 대부분 잔기술이다. 그 까닭 하나는 이런 난독화 기술들이 근본적으로 소스 코드에서 비밀로 해야 할 부분의 기밀성을 지켜 주는 방법이 아니기 때문이고, 다른 하나는 난독화로 인해 소스 코드의 무결성이 침식되는 면도 있기 때문이다. 난독화는 내가 비용을 치러서 상대방이 비용을 치르게 만드는 창과 방패 싸움이며, 잔기술에 고착되면 어떤 경우에는 내가 어느 수준 이상의 비용을 치르지 못하는 상태에 도달하게 되기도 한다.

## 무슨 일이 있었길래?

`moshi-kotlin` (`moshi-kotlin-reflect`) 과 `moshi-kotlin-codegen`. 잘 쓰던 `moshi-kotlin` 1.8.0 을 제거하고 `moshi-kotlin-codegen` 1.8.0 으로 교체하며 R8 ProGuard 파일에 아래 줄을 추가했다.

```
-keep class com.squareup.moshi.kotlin.reflect.** { *; }
-keepclasswithmembers class * {
    @com.squareup.moshi.* <methods>;
}

-keep @com.squareup.moshi.JsonQualifier interface *

-keepclassmembers @com.squareup.moshi.JsonClass class * extends java.lang.Enum {
    <fields>;
}

-keepnames @com.squareup.moshi.JsonClass class *

-keepclasseswithmembers class **.*JsonAdapter extends com.squareup.moshi.JsonAdapter {
	<init>(...);
	<fields>;
}
```

출처: <https://www.techwasti.com/progurad-for-android-kotlin-104a1169fdcd/>

사실 잘 쓰고 있었던 건 아니다. release 빌드에서 `moshi-kotlin` 의 `KotlinJsonAdapterFactory` 와 기타등등이 꽤 말썽을 부려서 앱 크래시를 냈다. 분명히 Moshi 와 관련 있는 클래스와 메서드와 프로퍼티에 대고 무차별로 `@Keep` 을 달아 뒀는데 소용이 없었다. (이 부분은 아직도 미스테리이다&hellip; 안타깝게도 제대로 파고들어서 빈틈을 찾아낼 시간이 없었다.)

여기에서 끝이 아니고, `moshi-kotlin-codegen` 의 경우, Kotlin 컴파일러가 Jetifier 를 만나 문제를 일으키므로, `gradle.properties` 를 만져야 한다. 아래 라인이 있는 경우,

```
android.enableJetifier=true
```

이렇게 생긴 라인을 추가한다.

```
android.jetifier.blacklist=kotlin-compiler-embeddable-.*\\.jar
```

출처: <https://github.com/square/moshi/issues/804>

## Android ProGuard (R8)

Android 빌드 툴체인은 원래 Guardsquare 사의 [ProGuard](<https://www.guardsquare.com/proguard>) 라는 제품을 난독화 도구로 채용했다 ([ProGuard @ GH](<https://github.com/Guardsquare/proguard>)). ProGuard 는 Java 쪽에서 꽤 유명한 난독화 표준 도구이고 한동안 ProGuard 는 Android 에서도 잘 쓰였다.

2017 년부터 Android SDK 에서 빌드 파이프라인을 개선하면서 ProGuard 가 R8 에 자리를 내주게 된다. AAPT 가 AAPT2 로, DX 가 D8 로 교체되는 대규모 변화 속에서, 비슷한 시기에 사람들에게 널리 퍼진 인식이 있었다: 빌드 툴은 서로 입출력만 명확히 알고 나머지에는 관심 끄는 툴체인인 것보다, 각자의 내부 동작에 대해 상호 공유가 이루어지는 툴세트인 것이 더 나은 결과를 주는 방식이라는 것이다. 그렇게 2018 년에 D8 은 Java 8 디슈가링 프로세스에 포함되었고 2019 년에는 R8 가 등장해 ProGuard 를 대체하면서 D8 을 포함하는 단일 도구가 되었다.

원래 ProGuard 가 어떤 도구인지를 조금 설명해야 할 것 같다. ProGuard 는 Java 클래스파일에서 심볼 리매핑을 하는 도구이고 JAR 파일의 크기를 획기적으로 줄여 준다고 알려져 있다. 그런데 ProGuard 는 코드 축소<sup>code shrinking</sup>도 하고 최적화<sup>optimize</sup>도 한다.

둘은 좀 다르다. 예를 들어, **최적화**는 `1 + 1` 처럼 결과가 정해진 덧셈을 수행하는 코드를 날리고 아예 `2` 라는 결과 값으로 대체해 버리는 과정이라고 할 수 있다. 이 과정에서 도달할 수 없는 코드<sup>unreachable code</sup> 같은 것들이 사라진다. 예를 들어, `if (true)` 의 `else` 블럭이 제거될 수 있을 것이다. 마찬가지로 `c = a + b` 라는 덧셈을 수행하는 코드가 있으나 결과값인 `c` 가 아무데서도 쓰이지 않는다면 최적화 이후에 이 덧셈 연산이 사라질 수 있게 된다.

`c = a + b` 에서 `a` 와 `b`, `c` 가 같은 클래스 유형 `X` 의 인스턴스이고, 이 외에 `X` 의 인스턴스가 없는 경우를 생각해 보자. 최적화를 거쳐 `c` 는 아무데서도 참조되지 않아 제거되었고, `=` 와 `+` 역시 제거된다. `a` 와 `b` 가 이 식 외에 사용되지 않았다면, `a` 와 `b` 역시 최적화 과정에서 제거해 버릴 수 있다. (`a` 와 `b` 의 생성자<sup>constructor</sup>가 부작용<sup>side effect</sup>을 수반하지 않는 경우에 한정된다.) 이제 `X` 의 모든 인스턴스가 제거되었으므로, 클래스 유형 `X` 은 아무에서도 참조되지 않는다. 코드 간의 의존성 그래프에서 `X` 에 도달할 수 있는 경로가 없으므로 `X` 자체를 저세상으로 보낼 수 있게 되는데, 이것이 **코드 축소**이다. 의존성 그래프에서 끊어진 가지 너머의 노드를 제거하는 것이기 때문에 이것을 나무 털기<sup>tree shaking</sup>라고도 부른다.

이제 R8 를 보자. R8 는 코드 간의 나무 털기에 한 술 더 떠서 이미지나 XML 같은 리소스도 코드에서 참조되는 것만 남긴다. 이것을 **리소스 축소<sup>resource shrinking</sup>**라고 부른다. 엔터프라이즈 레벨에서 리소스 관리는 엄격하게 이루어지기 힘들기 때문에 이런 기능이 빌드 툴에 들어 있는 건 실용성이 대단한 일이다. 그런데 뭐랄까, 결과가 멀쩡하면 당연히 좋은 일이겠으나, 결과가 나쁘면 영문도 모르고 앱 크래시까지 나오는 엉망진창이 된다. 꽤 많은 앱에서, 앱 빌드에 번들해서 릴리스하는 리소스가 실제로 앱에서 사용되지만 앱 소스 코드에서 직접 참조되지 않는 사례가 발견된다. 이 경우 해당 리소스를 이용하는 UI 구성은 서버에서 받은 데이터 따위를 이용해서 리소스를 간접 참조하는 방식으로 진행된다.

그래서 음&hellip; 상당히 이상하고 아름다운 룰이 따라붙게 되는데&hellip; 바로 느슨한 참조 점검<sup>loose reference check</sup>.

>### 엄격한 참조 확인 사용
>
>일반적으로 리소스 축소기는 리소스의 사용 여부를 정확하게 판별할 수 있습니다. 그러나 코드가 `Resources.getIdentifier()`를 호출하거나 임의 라이브러리가 호출을 실행하는 경우(예: AppCompat 라이브러리가 호출을 실행), 이 코드는 동적으로 생성된 문자열을 기반으로 리소스 이름을 찾습니다. 이렇게 하면 리소스 축소기는  기본적으로 방어적인 동작을 하며 매칭 이름 형식을 가진 모든 리소스를 잠재적으로 사용 중이며 삭제할 수 없는 리소스로 표시합니다.
>
>예를 들어, 다음 코드는 `img_` 접두사가 있는 모든 리소스를 사용되는 리소스로 표시합니다.
>
>```kotlin
>val name = String.format("img_%1d", angle + 1)
>val res = resources.getIdentifier(name, "drawable", packageName)
>```
>
>리소스 축소기는 또한 다양한 `res/raw/` 리소스와 코드에서 모든 문자열 상수를 찾고 `file:///android_res/drawable//ic_plus_anim_016.png`와 유사한 형식의 리소스 URL을 찾습니다. 이 리소스와 유사한 문자열을 찾았거나 이와 같은 URL을 구성하는 데 사용될 수 있는 것처럼 보이는 문자열을 찾은 경우, 상응하는 리소스가 삭제되지 않습니다.
>
>다음은 기본적으로 사용되는 안전 축소 모드의 예입니다. 그러나 '나중에 후회하는 것보다는 더 안전한' 처리를 사용 중지하고 리소스 축소기가 확실히 사용되는 리소스만 유지하도록 지정할 수 있습니다. 이를 위해서는 다음과 같이 `keep.xml` 파일에서 `shrinkMode`를 `strict`로 설정합니다.
>
>```xml
><?xml version="1.0" encoding="utf-8"?>
><resources xmlns:tools="http://schemas.android.com/tools"
>    tools:shrinkMode="strict" />
>```
>
>위에서 보는 바와 같이 엄격한 축소 모드를 사용하고 코드가 동적으로 생성된 문자열이 있는 리소스를 참조한다면 `tools:keep` 속성을 사용하여 이 리소스를 반드시 수동으로 유지해야 합니다.

출처: Android Developers – Android Studio User Guide. *Shrink, obfuscate, and optimize your app.* <https://developer.android.com/studio/build/shrink-code>

코드 축소나 리소스 축소에 난독화가 개입하면 문제는 더욱 어려워진다. 난독화는 기본적으로 심볼 리매핑을 수행하는데, 이 과정에서 심볼 참조를 누락해서 일부 심볼을 리매핑하지 않는다면 마찬가지로 앱 크래시를 유발할 수 있게 된다. 반영<sup>reflection</sup>을 사용하는 코드는 우리 생각보다 꽤 많은 곳에 이미 침투해 있고, 코드 축소나 리소스 축소 과정에서 인식된 느슨한 심볼 참조가 난독화 과정에서 검토되는 심볼 참조와 긴밀하게 연계되지 않는다면, 이런 비극적 결과는 거의 필연이다.

## ProGuard 파일

이 부분을 회피하기 위해 ProGuard 시절부터 있던 것이 ProGuard 파일이다. ProGuard 파일의 기본은 코드 축소나 심볼 리매핑 등에 예외를 지정하는 구문으로 되어 있다. 아래와 같은 것들이 대표적이다.

* `-keep`
* `-keepclassmembers`
* `-keepclasseswithmembers`

전체적으로는 `-dontshrink`, `-dontoptimize`, `-dontobfuscate` 등 ProGuard 기능 일부를 끌 수 있게 해 주기도 한다.

Android 빌드스크립트인`build.gradle` 에서 ProGuard 파일은 `proguardFiles` 구문을 이용해 지정된다. Android 프로젝트를 처음 생성하면 이 `proguardFiles` 구문에 `getDefaultProguardFile("proguard-android.txt")` 이 기본 제공된다. 이 내용을 먼저 살펴보자.

* 앞서 설명한, 특히 골치아픈 리소스 참조 부분을 코드 축소와 심볼 리매핑으로부터 보호한다.

  ```
  -keepclassmembers class **.R$* {
      public static <fields>;
  }
  ```

* `Parcelable` 인터페이스를 구현하는 클래스가 포함해야 하는 `CREATOR` 싱글턴 인스턴스를 보호한다.

  ```
  -keepclassmembers class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator CREATOR;
  }
  ```

* 레이아웃 XML 에 등장하는 속성<sup>attribute</sup> 값인 `android:onClick` 의 동작을 보존한다.

  ```
  # We want to keep methods in Activity that could be used in the XML attribute onClick
  -keepclassmembers class * extends android.app.Activity {
     public void *(android.view.View);
  }
  ```

* `@Keep` 어노테이션이 실제로 동작하도록 한다.

  ```
  # Understand the @Keep support annotation.
  -keep class android.support.annotation.Keep
  -keep class androidx.annotation.Keep
  
  -keep @android.support.annotation.Keep class * {*;}
  -keep @androidx.annotation.Keep class * {*;}
  
  -keepclasseswithmembers class * {
      @android.support.annotation.Keep <methods>;
  }
  
  -keepclasseswithmembers class * {
      @androidx.annotation.Keep <methods>;
  }
  
  -keepclasseswithmembers class * {
      @android.support.annotation.Keep <fields>;
  }
  
  -keepclasseswithmembers class * {
      @androidx.annotation.Keep <fields>;
  }
  
  -keepclasseswithmembers class * {
      @android.support.annotation.Keep <init>(...);
  }
  
  -keepclasseswithmembers class * {
      @androidx.annotation.Keep <init>(...);
  }
  ```

* 기타 등등

* `proguard-android.txt` 와 `proguard-android-optimize.txt` 의 차이. `proguard-android-optimize.txt` 에만 이런 부분이 삽입된다.

  ```
  -optimizations !code/simplification/arithmetic,!code/simplification/cast,!field/*,!class/merging/*
  -optimizationpasses 5
  -allowaccessmodification
  ```

이 파일은 원래 Android SDK 에서 SDK tools 의 일부로 `tools/proguard/proguard-android.txt` 에 제공되는 ProGuard 파일이었다. 그러나 만약 이 부분에 문제가 생긴다고 하면, SDK tools 보다는 SDK build-tools 가 SDK platforms 에 알맞게 동작하지 않은 버그이므로, `tools` 가 아니라&hellip; Gradle 빌드스크립트를 담당하는 `com.android.build.tools:gradle` 에서 매번 생성하고 있다. 뭐라고?

사실이다. 위 내용은 Android Gradle Plugin JAR 파일의 `com/android/build/gradle/proguard-common.txt` 와 `com/android/build/gradle/proguard-optimizations.txt` 에서 가져왔다. `tools/proguard/proguard-android.txt` 는 더이상 업데이트되지 않고 이 파일은 AndroidX `@Keep` 어노테이션에 대한 이해가 없다.

## 예측 불가능한 빌드

R8 가 어떻게 동작할지 예측할 수 있는 사람이 있을까? 현실적으로 그건 그냥 불가능한 일이다. R8 는 각 단위가 어떻게 동작해야 하는지에 대한 명세를 제공하지 않고, AGP 의 Gradle task 역시 블랙박스인 것은 마찬가지이다. R8 에 Java 8 라이브러리 디슈가링이 추가되면서 R8 는 더 거대한 L8 라는 프로젝트가 되었다. 2019 년 12 월에 [Jake Wharton 이 블로그에서 L8 을 소개했고](<https://jakewharton.com/d8-library-desugaring/>), 이는 Android Gradle Plugin v4 에 포함된다. (Update: 2020 년 4 월에 `coreLibraryDesugaringEnabled` 옵션이 추가된 [Android Gradle Plugin 4.0.0 안정 릴리스가 발표되었다](<https://developer.android.com/studio/releases/gradle-plugin#4-0-0>).)

Android Studio (IDEA Android Support Plugin) 버전과 AGP 버전 그리고 build-tools 버전은 함께 상승하며, 이젠 패치 버전이 어떤 빌드 버그를 어떻게 해결했는지 알기도 어렵다. [AOSP](<https://android.googlesource.com/>) platform/tools 와 [AOSP Gerrit](<https://android-review.googlesource.com>) 을 주시하고 산다면 어느 정도 가능하겠지만&hellip; 그 변경을 추적하며 사소한 사실들을 알고 있는 게 우리의 문제를 해결해 줄까? 

물론 L8 가 사고를 치는 일은 거의 없다. L8 의 개별 단위는 우리에게 노출되지 않을 뿐 실제로는 상당히 많은 단위 시험<sup>unit test</sup>을 거쳐 만들어진다. 아니라고? 차라리 그렇다고 믿고 넘어가자. ㅠㅠ

현 시점에서 L8 과 같은 거대하고 불투명한 시스템이 전체적으로 일을 잘 해 줬다고 간주할 수 있으려면 결국 빌드 결과물이 정상 작동하는지 직접 돌려 보는 방법까지 동원해야 한다. 그 과정에서 어느 정도는 Android 의 계측 시험<sup>instrumented test</sup>이 제 역할을 해 준다만&hellip; 혹시 다들 계측 시험은 성실하게 작성해 두셨는지?

이런 불확정성에 대한 막연한 두려움은 결국 일선 개발자들이 AGP 버전과 SDK build-tools 버전을 잘 올리지 못하는 결과로도 이어지게 되었고, 그 결과로 compileSdkVersion 도 올리지 못하고 targetSdkVersion 도 올리지 못하는 연쇄작용이 일어나게 된다. 물론 Android 는 OS 버전업에 따라 알림 서비스<sup>notification service</sup>를 도입해서 보이지 않는 곳에서 코드가 돌아가는 일을 없애 주기도 했고, 꾸준히 사용자를 위한 프라이버시 필터링 도구를 제공하고 있다. 엔드유저 단말의 연산능력과 정보를 악용하려는 앱은 실존한다. 미쳐 돌아가는 세상에서 꽤 중요한 일이다. 그런데 빌드 툴체인 자체의 불확정성이 OS 업데이트를 따라가는 일을 상당히 어렵게 만들고 있으니, Play 스토어와 Google Android OS 의 targetSdkVersion 하한 상승 정책이 정말 엔드유저의 프라이버시를 보장하기 위한 것인지, 아니면 사실상 빌드 툴체인의 불확정성을 극복하게 만드는 것인지 조금 헷갈린다.

사실 빌드 툴체인의 예측가능성에 대한 논의는 Android 앱에 국한되는 것이 아니다. 패키지 빌드 단위의 재현가능성을 다루는 그 유명한 [Reproducible Builds](<https://reproducible-builds.org/>) 프로젝트가 Debian 프로젝트에서 창안된 것이 2013 년이고, Google 내부 도구인 Blaze 가 [Bazel](<https://bazel.build/>) 로 오픈소스되어 나온 것이 2015 년이다. (Bazel 의 분산 빌드 서버가 전체 빌드 과정을 가속할 수 있는 핵심 원리가 바로 빌드의 재현가능성이다.) AOSP 역시 최상위 빌드 시스템으로 Bazel 을 채택하였고, Bazel 이 정상 동작하려면 빌드 툴체인의 모든 유닛이 Bazel 확장으로 작성되어 있어야 한다. 그래서 실제로는 어떻냐면, Bazel 로 건너가기 위해 Bazel 과 비슷한 [Soong](<https://android.googlesource.com/platform/build/soong/+/master>) 이라는 것을 사용하고, 아직 Android.mk 가 많이 남아 있으며, build.gradle 도 곳곳에 살아 있다 (무려 platform/tools/base 가 아직도 build.gradle 을 사용한다). 내부사정이 이해되는 대목이다.

## 무슨 말을 하려고 했더라

Moshi 때문에 R8 ProGuard 파일을 수정하다가 여기까지 왔다. L8 (D8/R8) 의 거대하고 불투명한 동작, ProGuard 파일까지 관리하는 AGP, 이 둘의 동작에 기대어 대응할 수 있는 OS API level. 평범한 Android 앱 프로그래머의 입장에서 이런 문제들이 응집되어 ProGuard 파일을 직접 편집해야 하는 지점에서 나타난다. 웬만하면 앱은 정상 빌드되고 정상 작동할 거다; 최적화와 난독화 계통을 거치지 않는다는 가정 하에 말이지. Android 개발 도구는 Kotlin 을 공식 지원하고 있고, 특히 Android Platform API 보다는 AndroidX API 에서 Jetpack Compose 처럼 Kotlin 의존성이 필수인 부분들이 세일즈 포인트가 되고 있으나, 정상 빌드를 만들려면 우선 ProGuard 파일에서 Kotlin 대응부터 직접 해야 한다.

```
-keep class kotlin.Metadata { *; }
```

이렇게 작성된 ProGuard 파일의 연원은 점차 잊혀지며 이런 라인들은 L8 의 거대하고 불투명한 동작에 맞서 보수적으로 복붙되어 끝없이 늘어난다. 그걸로 끝나면 좋은데 악마같은 디테일이 더 나온다. Android 라이브러리 프로젝트에서 `consumerProguardFiles` 에 지정한 ProGuard 파일은 Maven 저장소에 JAR 파일 대신 업로드되는 AAR 파일에 `proguard.txt` 로 포함되어 있다. 이 `proguard.txt` 는 거역할 수 없는 질서로 작용한다. 다음, R8 는 `-keep` 처리되어 있는 클래스들을 일종의 진입점<sup>entrypoint</sup>으로 인식하고, multidex 빌드에서 진입점 클래스들은 가능한 한 많이 main DEX 파일에 지정된다. 이후 D8 가 Java 8 람다 디슈가링을 한다. 보수적인 라이브러리 프로젝트 메인테이너의 AAR 파일에서 유입된 수많은 `-keep` 구문이 아주 많은 R8 진입점 클래스를 만들어 내고, D8 의 Java 8 람다 디슈가링 결과가 R8 의 예측과 어긋나면, main DEX 파일의 method count 가 DEX 파일 한계를 넘어서, 정말 말도 안 되게도 multidex 프로젝트에서 이런 에러를 만난다.

```
Too many method references: 65571; max is 65536
```

(Update: 이 문제는 이후 Android 빌드 툴체인 최신 버전에서 수정되었다. 재발할 가능성은 있다.)

솔직히 빌드 재현가능성까지는 바라지도 않는다. 그래도 ProGuard 파일은 어떻게 안 될까?

ProGuard 파일에 명세되는 내용은 대체로 유형 체계적 제약에 해당한다고 간주할 수 있다. ProGuard 를 이용해 빌드된 JAR 바이너리에 나무 털기나 심볼 리매핑같은 과정을 적용할 때, Java (그리고 Kotlin/JVM) 프로그램이 런타임에 정상이도록 하는 기본 원리는 그 프로그램이 컴파일타임에 정상이도록 하는 것이다. 이는 JAR 파일을 클래스패스로 지정해서 컴파일러가 소스 코드마냥 참조하도록 할 수 있기 때문이기도 하며, 컴파일타임에 프로그램의 정합성<sup>consistency</sup>을 점검했다면 런타임에도 그 프로그램의 정합성이 유지되는 것이 일반적인 일이기 때문이다. 그러나 어떤 클래스, 메서드, 필드의 이름과 존재를 유지해 주는 것은 컴파일러가 알 수 없는 수준의 프로그램 정합성을 보존하기도 하며, 이는 컴파일타임에 유형 체계로 점검해 내는 프로그램의 정합성과 본질적으로 다를 것이 없다. 즉 ProGuard 파일은 기존 패키지에 보조적인 유형 체계를 이용해 유형 힌트를 도입하는 역할을 하는 셈이다.

개인적으로는 [Definitely Typed](<https://definitelytyped.org/>) 같은 방식이 맞다고 본다. TypeScript 의 경우에는 유형 체계가 처음부터 기존 생태계를 건드리지 않고 보조적인 역할만을 하도록 만들어져 있는데, TypeScript 팀에서 수많은 프로젝트들의 d.ts 파일만 관리하는  프로젝트를 운영하고 있으며 ([Definitely Typed @ GH](<https://github.com/DefinitelyTyped/DefinitelyTyped>)) 이 파일들이 사실상 TypeScript 컴파일러의 tier one 서포트 대상이다. 이론적으로야 새로운 유형체계를 도입한 언어로 코드가 재작성되어 세상을 지배하면 좋겠으나, 이런 결과가 정말 달성되더라도 그게 지독한 악연이 되지 않으려면 여러 가지 가정이 더 필요하다. 안타깝지만 패키지의 유형 체계적 제약에 대해서는 생산자보다 소비자가 더 정확히 알고 있을 가능성이 크고, 소비자에게는 선택권이 필요한 일이 생긴다.
