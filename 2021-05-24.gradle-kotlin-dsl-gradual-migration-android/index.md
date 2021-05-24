Trouble ID `2021-05-24.gradle-kotlin-dsl-gradual-migration-android`

# Android 프로젝트, Gradle Groovy DSL 에서 Gradle Kotlin DSL 로 점진적 이전하기

Android SDK 툴체인 생태계에 Gradle Kotlin DSL 이 들어온지도 거의 1 년이 다 되어 가는데 우리 프로젝트는 아직도 Gradle Groovy DSL 을 쓰고 있다. 이유는 터무니없다. Gradle Kotlin API 가 Gradle Groovy API 보다 커졌기 때문이다. 할일이 훨씬 커진 것이다.

> ![0.nonsense](./0.nonsense.jpg)
> 
> (c) 이토 준지 (伊藤潤二; Junji Ito) 1998. 이토준지 공포만화 콜렉션 9권 "소이치의 즐거운 일기" (首幻想, 伊藤潤二恐怖マンガCollection 第9巻; *Hallucinations*, *The Junji Ito Horror Comic Collection* Vol. 9; 번역 시공사 1999).

사실 엔터프라이즈 환경에서 빌드 툴체인을 갈아 버리는 것은 꽤 부담스러운 일이다. 특히 Android 개발의 경우 그 부담이 좀 크다. Gradle 플러그인이 거의 모든 일을 다 해 주고 있고, 결과물을 들여다 보는 절차 역시 까다로워서 검증 절차도 조금 느슨하기 때문이다. 툴체인의 급격한 변화는 프로그래머가 예상하지 못하는 결과를 가져올 리스크가 있다.

한편 Gradle 빌드스크립트의 DSL 프레임워크를 교체하는 것은 사실 그렇게까지 드라마틱한 변경은 아닌데, 런타임인 Gradle 이나 플러그인들의 동작 방식을 전혀 건드리지 않기 때문이다. 그러나 이 경우 DSL 프레임워크 교체 작업에 대한 불안의 원인은 평가 가능한 위험 assessable risk 보다는 불확정성 uncertainty 에 가깝다. 우리는 Gradle 빌드스크립트가 동작하는 API 나 내부 구조에 대해서 자세히 모르기 때문에 Groovy 와 Kotlin 코드 간에 차이가 없다고 확신할 수 없다. Gradle 의 Groovy API 와 Kotlin API 가 조금 다르고 두 DSL 런타임의 동작도 조금 다르기 때문에 이 문제는 더 복잡해진다. ABI 레벨에서라도 빌드스크립트의 불변을 보장할 수 있냐면, 빌드스크립트는 클래스파일로 컴파일되는 것도 아니라서 그런 작업은 불가능하다.

이런 노답시추에이션에 대해 대강 두 가지의 판이한 전략을 제시할 수 있다. 하나는 그냥 이 작업이 어떤 전문성 있는 장인의 영역임을 인정하는 것이다. 나머지 사람들은 그냥 장인정신을 믿고 결과물의 정교함에 감동하면 된다. 장인이 없는데 결과물을 얻어야 한다면 어거지로 뭔가 만들 수는 있겠지. 그 산출물의 품질에 대해서 의심을 거둘 수 있는 날은 오지 않을 것이다. 남은 하나의 전략은 그 장인의 작업을 모방하는 것이다. 장인의 작업 과정을 꼼꼼히 따라한다면 단시간 내에 (장인이 되지는 못하더라도) 장인과 거의 같은 결과를 낼 수는 있다. 이런 전략에 보다 구체적인 절차를 제시하는 것이 점진적 방법론이고, 오늘의 난리로그는 Groovy Kotlin DSL 을 점진적으로 도입해 나가는 방법에 대한 것이다.

## \*.gradle 파일에서 \*.gradle.kts 파일로

Gradle Kotlin DSL 이 무엇인지 어렵지는 않지만 굳이 길게 설명해 보자.

원래 Gradle 빌드스크립트는 Gradle 런타임에서 Groovy 코드를 돌리는 것이고, Gradle API 가 잘 설계되어 Groovy 코드를 Gradle 전용으로 설계한 DSL 마냥 미려하게 쓸 수 있어서 이걸 Gradle Groovy DSL 이라고 부른다. JetBrains 에서 Kotlin 을 만들 때 Scala 의 유형 체계를 조금 모방하면서 문법은 Groovy 처럼 조금 친숙한 느낌을 주도록 했는데, 덕분에 AndroidX 의 Jetpack Compose 같은 것도 나오고 Gradle 에는 Gradle Kotlin DSL 이 나올 수 있었다.

Kotlin 은 Groovy 에 비해 유형 추론 기능이 강력하기 때문에 빌드스크립트에서도 오타나 유형 오류를 조기에 탐지하고 교정하는 데에 IDE 의 도움을 받을 수 있다는 장점이 있다. 그리고 특히 타깃 시스템이 JVM 이거나 Android 같은 유사 JVM 인 경우, 소스 코드와 빌드스크립트를 Kotlin/JVM 이라는 같은 프레임워크로 코딩하는 것은 상당한 이점이다. 내가 아는 한 이 정도로 부트스트래핑이 되는 사례는 autohell 과 Rakefile 그리고 webpack.config.js 정도밖에 없었다. (autohell 이 왜 빌드스크립트 부트스트래핑인가? GNU Autotools 를 쓰면 `./configure` 과정에서 특정 C 함수가 존재하는지 체크하기 위해 C 파일을 컴파일하고 링크해 보기까지 하므로, Unix-like 시스템 전체를 C 코드를 빌드하기 위해 C 코드를 빌드해서 돌리는 거대한 툴체인으로 보고 거기에 Bash 나 Makefile 이나 m4 같은 편의 요소가 조금 붙은 걸로 취급해야 한다. 한편 컴파일타임 전체를 부트스트래핑하려는 시도는 C, C\#, Rust 정도가 성공한 상황이다.)

얘기가 많이 샜다. 아무튼 Gradle 빌드스크립트에 Gradle Kotlin DSL 을 도입하는 과정은 다음과 같다.

1. 파일 이름을 \*.gradle 에서 \*.gradle.kts 로 바꾼다.
2. 코드의 Groovy 문법을 Kotlin 문법으로 바꾼다.

참 쉽죠? (That easy.)

> ![1.bob-ross](./1.bob-ross.webp)
>
> You have to have dark in order to have light. Gotta have opposites, light and dark and dark and light, continually in painting. If you have light on light, you have nothing. If you have dark on dark you also have nothing. It’s like in life. Gotta have a little sadness once in a while so you know when the good times come. I'm waiting on the good times now.
>
> ― Bob Ross

물론 밥 로스의 그림만큼이나 마냥 쉽지는 않다. 붓질 하나씩 함께해 보자.

* 프로젝트 준비
* 미리보기
* `settings.gradle.kts`
* `plugins {}` 블럭 도입하기
* 루트 `build.gradle.kts`
* `app/build.gradle.kts`
* `buildSrc` 도입하기
* 결과

## 프로젝트 준비

최근에 Android Studio 의 Stable 채널 버전이 4.2 까지 올라왔다. 그런데 Android Studio 4.2 에서도 Gradle 빌드스크립트는 Gradle Kotlin DSL 로 제공되지 않고 Gradle Groovy DSL 로 제공되고 있다. 이게 좋은 샘플이다.

Android Studio 4.2 에서 (또는 패치버전에서) 새 프로젝트를 만들어 새 프로젝트부터 Gradle Kotlin DSL 도입을 시작해 보는 것을 권한다. 아무리 Gradle Kotlin DSL 이 좋다 한들 기존 프로젝트의 빌드스크립트를 모두 이전할 때까지 결과를 보지 못하는 건 고통스러운 일이 될 수 있기 때문이다. 새 프로젝트로 조금씩 진행하면서 Gradle Kotlin DSL 을 실제로 동작시켜 보고, 또다시 더 큰 규모의 Gradle Groovy DSL 스크립트를 Gradle Kotlin DSL 스크립트로 변환해 보고, 이런 식으로 Gradle Kotlin DSL 과 친해지기 바란다. 그러고 나서 실제 프로젝트에 Gradle Kotlin DSL 을 적용할 용기를 얻었을 때 연습했던 순서대로 실전을 수행해 볼 수 있을 것이다. 실전에서 특정 단계를 넘지 못하고 깨지더라도 그 전까지는 진행했을테니, 괜찮다, 잠시 연습으로 돌아오자.

Android Studio 4.2.1 에서 생성한 새 Android 앱 프로젝트 템플릿의 빌드스크립트이다.

`settings.gradle`:

```groovy
rootProject.name = "My Application"
include ':app'
```

`build.gradle`:

```groovy
// Top-level build file where you can add configuration options common to all sub-projects/modules.
buildscript {
    ext.kotlin_version = "1.5.0"
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.2.1"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
        jcenter() // Warning: this repository is going to shut down soon
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

`app/build.gradle`:

```groovy
plugins {
    id 'com.android.application'
    id 'kotlin-android'
}

android {
    compileSdkVersion 30
    buildToolsVersion "30.0.3"

    defaultConfig {
        applicationId "net.aurynj.app.myapplication"
        minSdkVersion 24
        targetSdkVersion 30
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
}

dependencies {

    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    implementation 'androidx.core:core-ktx:1.5.0'
    implementation 'androidx.appcompat:appcompat:1.3.0'
    implementation 'com.google.android.material:material:1.2.1'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
}
```

시작하기 전에 이 기초 작업만 하고 넘어가자.

* `lib/*.jar` 에 대한 `implementation` 의존성을 추가한다.

  ```groovy
  implementation fileTree(dir: 'libs', include: ['*.jar'])
  ```

* `testImplementation` 의존성 `junit:junit:4.+` 를 `junit:junit:4.12` 로 바꿔 준다.

* `release.keystore` 파일을 준비하고 위 `app/build.gradle` 의 `buildTypes {}` 블럭을 다음과 같이 바꾼다. `signingConfigs.release` 에 쓰인 값들은 Gradle 빌드 옵션으로 제공할 것이다. 빌드 옵션이 익숙하지 않다면 이 이름들을 잠시 루트 프로젝트의 `ext` 에 넣어 두고 따라와도 좋다. `ext` 도 익숙하지 않다면&hellip; 이 작업을 하기에는 숙련도가 적절하지 못하므로 우선 Gradle 의 개념들에 익숙해지길 권한다.

  ```groovy
  signingConfigs {
      release {
          storeFile file(releaseKeystorePath)
          storePassword releaseKeystorePass
          keyAlias releaseKeyAlias
          keyPassword releaseKeyPass
      }
  }
  
  buildTypes {
      release {
          signingConfig signingConfigs.release
          minifyEnabled false
          proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
      }
  }
  ```

## `settings.gradle.kts`

웬만하면 Gradle Groovy DSL 을 쓰면서 `settings.gradle` 파일을 건드릴 일은 없다. 아래에 `plugins {}` 블럭을 도입하는 과정에서 `settings.gradle` 파일에 필요한 내용을 써야 하는데, 이걸 일단 Gradle Groovy DSL 로 작성한 후에 또 `settings.gradle.kts` 로 변환하면 이중고가 되므로, 먼저 여기에 Gradle Kotlin DSL 을 도입한다.

별 거 없다. `settings.gradle` 파일은 보통 이렇게 생겼다.

```groovy
rootProject.name = 'My Application'
include ':app'
```

저 `.name =` 부분은 Gradle 인터페이스 유형 `ProjectDescriptor` 의 `void setName(String name);` 메서드를 사용하는 것인데, 같은 유형에 `String getName();` 메서드가 있으므로 `name` 이 Kotlin 에서도 `var` 프로퍼티로 간주되어 정상 작동한다. 이후에 Gradle Kotlin DSL 변환 과정에서 배정 (`=`) 연산자가 없어 문제가 되는 부분들은 Kotlin 변환 규칙에 따라 `var` 프로퍼티가 되지 못한 것들이다. Groovy 에서는 설정자 setter 만 있어도 배정 연산자를 쓸 수 있지만, Kotlin 에서는 설정자 setter 와 획득자 getter 가 모두 있어야 `var` 프로퍼티가 되어 배정 연산자를 쓸 수 있기 때문이다.

`include` 는 메서드 콜로, Groovy 에서는 클로저 인자가 없고 일반 인자만 있다면 괄호 없이 메서드를 호출할 수 있지만, Kotlin 에서는 그런 건 허용되지 않는다. 그래서 괄호를 쳐 주고, 마지막으로 문자열 리터럴을 모두 큰따옴표로 바꿔 주면, 이 파일은 `settings.gradle.kts` 가 된다.

```kotlin
rootProject.name = "My Application"
include(":app")
```

## `plugins {}` 블럭 도입하기

|                                  | Gradle Groovy DSL                           | Gradle Kolin DSL                                             |
| -------------------------------- | ------------------------------------------- | ------------------------------------------------------------ |
| `plugins {}`                     | `plugins { id 'com.android.application' }` | `plugins { id("com.android.application") }`                  |
| `buildscript {} + apply plugin:` | `apply plugin: 'com.android.application'`   | `apply(mapOf("plugin" to "com.android.application"))`<br />`apply { plugin("com.android.application") }` |

Gradle Kotlin DSL 와는 하등 상관없는 `plugins {}` 블럭을 도입하는 것부터 시작한다. Gradle Kotlin DSL 에서도 기존과 똑같은 `apply` 구문으로 플러그인을 적용할 수 있는데, 플러그인 적용을 `plugins {}` 블럭으로 옮기는 것부터 시작해 보자는 것이다. 이 부분은 Gradle Kotlin DSL 의 실효를 보기 위함이다. Gradle 플러그인을 `apply` 문법으로 적용하지 않고 `plugins {}` 블럭으로 적용하면 플러그인 적용에 멱등성 idempotence 을 부여할 수 있기 때문이다. Gradle User Guide 의 [*Using Gradle Plugins*](<https://docs.gradle.org/current/userguide/plugins.html>) 문서를 보면 이런 얘기가 있다.

> *Applying* a plugin means actually executing the plugin’s [`Plugin.apply(T)`](https://docs.gradle.org/current/javadoc/org/gradle/api/Plugin.html#apply-T-) on the Project you want to enhance with the plugin. Applying plugins is *idempotent*. That is, you can safely apply any plugin multiple times without side effects.

Gradle Kotlin DSL 빌드스크립트는 매립형 Kotlin 컴파일러를 (또는 Kotlin 스크립트 컴파일러를) 사용한다 (`kotlin-{scripting-}compiler-embeddable`). IDE 상에서 Kotlin 유형의 도움을 받으려면 이 스크립트 컴파일러가 끊임없이 돌아야 한다. 스크립트 컴파일러의 개별 스텝을 빠르게 하려면 빌드스크립트의 변경된 부분만 증분 컴파일해도 되어야 하며, Gradle 자체뿐만 아니라 각 플러그인도 한번 로드해서 적용한 후에 다시 적용하지 않아도 다시 적용한 것과 같은 결과를 얻을 수 있어야 한다. 이것이 Gradle 에서 말하는 플러그인 적용의 멱등성이다. 플러그인 적용에 멱등성이 없다면 Gradle Kotiln DSL 을 이용할 때 Kotlin 에 약간의 도움을 받을 수 있더라도 결국 Gradle Groovy DSL 에서처럼 매번 빌드스크립트 전체를 다시 로드해서 (Gradle 의 Init, IDEA 의 Sync Project) 그 결과를 확인해야 한다. Gradle Kotlin DSL 에 익숙하지 않은 상태에서 이런 방식으로 작업하는 건 꽤나 고역이 될 만하므로 `plugins {}` 블럭을 도입해 두는 것을 최우선으로 한다.

### Android 플러그인과 `pluginManagement {}` 블럭

`buildscript {}` 블럭에는 별 제약이 없지만 멱등성을 위해 `plugins {}` 블럭은 꽤 많은 제약을 안고 있다. 그래서 `build.gradle` 파일에서 `plugins {}` 블럭의 위치는 본래 최상위였던 `buildscript {}` 블럭보다도 더 위에 있을 것으로 정해져 있다.

`project.extra` (`ext`) 같은 것은 여전히 `buildscript {}` 에 정의해야 하지만, 그 외에 `buildscript {}` 블럭 하위의 `dependencies {}` 블럭에 `classpath` 선언되어 있는 것들을 `plugins {}` 블럭으로 옮기는 것이 `plugins {}` 블럭 도입의 핵심 절차이다.

원래 `apply plugins:` 문법은 `buildscript {}` 블럭 아래에 있으므로 `buildscript {}` 블럭의 영향을 받는데, 플러그인을 `plugins {}` 블럭에 선언하면 순서상 `buildscript {}` 블럭의 효과가 빠지므로 문제가 생긴다. `buildscript {}` 블럭 하위의 `repositories {}` 블럭과 `dependencies {}` 블럭에는 어떤 Maven 저장소의 어떤 아티팩트를 사용할지에 대한 정보가 담겨 있기 때문이다. Android Studio 가 만들어 주는 Android 앱 프로젝트나 Android 라이브러리 프로젝트는 보통 이 `buildscript {}` 블럭을 루트 `build.gradle` 에 두기 때문에 사람들이 이를 충분히 신경쓰지 못하게 되는데 사실 얘네는 한 몸이다.

```groovy
buildscript {
    ext {
        androidGradlePluginVersion = "4.2.1"
    }
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    dependencies {
        classpath "com.android.build.tools:gradle:$androidGradlePluginVersion"
    }
}

apply plugin: 'com.android.application'
```

이 위에 `plugins {}` 블럭을 쓴다고 해 보자. 당연히 `buildscript {}` 블럭에 선언한 것은 무용지물이다.

좀더 구체적으로 들여다보자. 아무런 부가 설정이 없다면, `plugins {}` 블럭에 쓰이는 플러그인 이름은 다음 약속된 규칙을 따르게 되어 있다.

- 구문이 `id("com.android.application") version 4.2.1` 이라면
  - group = `com.android.application`
    - namespace = `com.android`
    - name = `application`
  - module = `com.android.application.gradle.plugin`
  - version = `4.2.1`
  - [Gradle Plugin Portal](<https://plugins.gradle.org/>) 에서 `com.android.application:com.android.application.gradle.plugin:4.2.1` 을 찾아서 사용
- 구문이 `id("com.android.application")` 이라면
  - 나머지는 상동
  - Gradle 코어 플러그인으로 판단, version = (Gradle 버전)
  - Gradle 버전이 6.7.1 이라면, Gradle 런타임 의존성에 포함되어 있을 `com.android.application:com.android.application.gradle.plugin:6.7.1` 을 사용

문제가 뭔지 알 것이다. Android 앱 프로젝트용 Gradle 플러그인 아티팩트는 Google Maven 저장소에 있는 `com.android.build.tools:gradle` 이지 Gradle Plugin Portal 에 있는 `com.android.application:...` 이 아니다. 이걸 정정해 주기 위해서 `settings.gradle.kts` 에 `pluginManagement {}` 블럭을 추가한다.

```kotlin
rootProject.name = "My Application"

pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    resolutionStrategy {
        eachPlugin {
            val namespace = requested.id.namespace
            val name = requested.id.name
            if (namespace == "com.android") {
                useModule("com.android.tools.build:gradle:${requested.version}")
            }
        }
    }
}

include(":app")
```

루트 `build.gradle` 의 `buildscript {}` 블럭은 이제 `project.extra` 외에 아무 쓸모가 없다. 심지어 이 값은 `buildscript {}` 에 있으므로 `plugins {}` 에서 쓸 수 없다. 그럼 여기에 정의한 AGP 버전 `androidGradlePluginVersion` 은 어떻게 되는가?

개별 프로젝트 `build.gradle` 에서 다음과 같이 지정한다.

```kotlin
plugins {
    id 'com.android.application' version '4.2.1'
}
```

본래 루트 `build.gradle` 에서 버전을 통합 관리하던 것이 빠지는 걸 상당한 단점으로 느낄 수 있다. 여기에 대해서는 `buildSrc` 를 도입하는 것이 Android 쪽에서 정석으로 알려져 있는데, 닭 잡자고 소 잡는 칼 가져오는 꼴이다! `buildSrc` 를 도입할 일이 아니다. 루트 `gradle.properties` 에 다음과 같이 쓴다.

```
androidGradlePluginVersion=4.2.1
```

이 이름은 Gradle Groovy DSL 의 경우 `project.extra` 처럼 그대로 값으로 쓸 수 있다. Gradle Kotlin DSL 의 경우, Kotlin `by` 를 이용해 Gradle 인터페이스 유형 Settings 에서 이름으로 꺼내 올 수 있다. `settings.gradle.kts` 의 `pluginManagement {}` 에서 다음과 같이 플러그인 통합 버전을 지정한다.

```kotlin
pluginManagement {
    val androidGradlePluginVersion: String by settings
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    plugins {
        id("com.android.application") version androidGradlePluginVersion
        id("com.android.library") version androidGradlePluginVersion
    }
}
```

최종 버전이다. `pluginManagement {}` 블럭 하위의 `resolutionStrategy {}` 블럭을 뺐는데, AGP 4.2.0 부터는 이 블럭이 있어도 되고 없어도 된다. 4.1.x 까지는 Google Maven 저장소에 `com.android.build.tools:gradle` 만 올라와 있었지만, 4.2.x 부터는 이것이 `com.android.application:com.android.application.gradle.plugin` 과 공존하고 있다. (이쪽이 기존 패키지 이름을 의존성으로 참조하고, 알맹이는 없다.) 따라서 현재 4.0.x 나 4.1.x 를 사용하는 경우 `resolutionStrategy {}` 블럭을 넣어서 `plugins {}` 블럭 도입을 우선 마치고, Gradle Kotlin DSL 을 도입한 후에, 4.2.x 를 사용하게 되면 `resolutionStrategy {}` 를 빼는 게 맞다.

개별 프로젝트 모듈 `build.gradle` 의 `plugins {}` 구문에서 `version` 이 빠지고, 결과는 다음과 같다. `com.android.library` 인 경우에도 동일한 결과를 얻는다.

```groovy
plugins {
    id 'com.android.application'
}
```

현재 Android Studio Beta 채널 버전인 2020.3 Arctic Fox 는 내부적으로 AGP 버전을 7.0.0 으로 지정해 두었다. Gradle 버전과 맞추려고 하는 것이 아닐까 생각이 든다. 이렇게 될 경우, `settings.gradle` 에서도 `version` 구문이 불필요하게 된다.

### Google Services 플러그인

Google Services Gradle Plugin 은 Android 앱 프로젝트에서 가장 많은 사람들을 골치아프게 만든 Gradle 플러그인일 것이다. Google Services 패키지 의존성이 플러그인 버전과 맞지 않으면 플러그인이 빌드를 실패시켜 버리기 때문이다.

> Execution failed for task ':processDebugGoogleServices'.
>
> \> Please fix the version conflict either by updating the version of the google-services plugin (information about the latest version is available at https://bintray.com/android/android-tools/com.google.gms.google-services/) or updating the version of com.google.android.gms to ?.

`plugins {}` 블럭 이전 문법에서, Google Services 플러그인은 `buildscript {}` 에서 `classpath "com.google.gms:google-services:..."` 로 지정하고 `apply plugin: 'com.google.gms.google-services'` 로 프로젝트에 적용한다. 그런데 이걸 적용하고 나서 빌드 에러가 나면 이 `apply` 구문을 `build.gradle` 빌드스크립트의 맨 마지막 줄로 보내라는 게 거의 정석처럼 굳어져 있다. 도대체 이런 괴상한 방법을 누가 찾아서 이렇게 널리 퍼뜨렸는지 신기할 따름.

위 오류 메시지는 `com.google.gms:google-services` 의 Google Services 플러그인이 낸 것이다. Google Services 플러그인은 빌드 과정에서 `google-services.json` 파일을 체크하면서 동시에 `com.google.android.gms:play-services-location` 이나 `com.google.firebase:firebase-core` 같은 패키지가 이 `google-services.json` 파일을 정상적으로 읽을 수 있는지도 체크한다. 플러그인이 JSON 파일을 읽을 수 있지만 앱에 번들된 패키지가 JSON 파일을 읽을 수 없다면 이 앱은 런타임에 크래시를 낼 수도 있다! 따라서 Google Services 플러그인은 `com.google.android.gms` 패키지나 `com.google.firebase` 패키지의 버전이 플러그인 버전과 맞는지 체크하게 되고, 앱 크래시를 예방하는 목적에서 빌드를 실패시킨다.

그런데 Google Services 플러그인을 적용하는 `apply` 구문이 `build.gradle` 파일의 최상부가 아니라 맨 마지막에 기재되면 Google Services 플러그인이 패키지 버전 점검을 하지 못하게 되고, 빌드를 억지로 성공시킨다. 결국 앞서 말한 플러그인 멱등성이 위반되는 동작을 아주 적극적으로 사용하고 있는 셈인데&hellip; 대체 왜? 제발 빌드가 실패한다면 오류 메시지를 읽고 시키는 대로 하는 것부터 시작해 보자.

이 순서로 작업한다.

1. `build.gradle` 의 `apply plugin: 'com.google.gms.google-services'` 를 파일 위쪽으로 이동시킨다. (`plugins {}` 블럭이나 `buildscript {}` 블럭보다는 아래에 둔다.)
1. 빌드가 실패한다면 문제가 되는 패키지 의존성을 업데이트해서 빌드를 성공시킨다.
1. 상기 `com.android.application` 또는 `com.android.library` 처럼, `'com.google.gms.google-services'` 를 `plugins {}` 블럭으로 옮기고, `pluginManagement {}` 블럭 하위의 `resolutionStrategy {}` 블럭에 알맞은 패키지 참조 해소 규칙을 추가한다.
1. 빌드 성공을 재확인한다.

## 루트 `build.gradle.kts`

루트 `build.gradle` 을 `build.gradle.kts` 로 바꿀 차례이다. 마찬가지로 대부분의 경우 여기에서 할 일은 별로 없다. 전체적으로 이런 일을 해 주면 좋다.

- `buildscript {}` 에 `project.extra` 값을 지정한 경우

  - Gradle Groovy DSL:

    ```groovy
    ext {
        someLibVersion = '1.2.3'
        anotherLibVersion = '4.5.6'
    }
    ```

    Gradle Kotlin DSL:

    ```kotlin
    extra.run {
        this["someLibVersion"] = "1.2.3"
        this["anotherLibVersion"] = "4.5.6"
    }
    ```

  - 개별 프로젝트 모듈 빌드스크립트에서 플러그인을 모두 `plugins {}` 블럭으로 옮기고 `buildscript {}` 블럭을 모두 제거했다면, 루트 빌드스크립트에서 이 위치의 `repositories {}` 블럭과 `dependencies {}` 블럭은 이제 아무 효과가 없으므로 지워도 좋다.

  - 이 extra 사용이 좀 불편하게 느껴지기 때문에 `buildSrc` 를 도입하는 것이 널리 알려져 있는데, 다시 말하지만, 이걸 위해서라면 하지 말자.

  - 이 값들을 gradle.properties 에 모두 보내 버린다면, 이상적으로는 루트 빌드스크립트에서 `buildscript {}` 블럭을 완전히 제거할 수 있게 된다. 이 내용은 목록을 마치고 다룬다.

- `subprojects {}` 또는 `allprojects {}` 의 `repositories {}` 블럭에 임의 Maven 저장소를 지정한 경우

  - Gradle Groovy DSL:

    ```groovy
    maven {
        url 'https://maven.google.com'
    }
    ```

    Gradle Kotlin DSL:

    ```kotlin
    maven(url = "https://maven.google.com")
    ```

  - Gradle Groovy DSL:

    ```groovy
    maven {
        name 'Google\'s Maven Repository'
        url 'https://maven.google.com'
    }
    ```

    Gradle Kotlin DSL:

    ```kotlin
    maven(url = "https://maven.google.com") {
        name = "Google's Maven Repository"
    }
    ```

  - 이 부분의 Groovy 클로저와 Kotlin 람다에 적용되는 리시버는 Gradle 인터페이스 유형 `MavenArtifactRepository` 의 인스턴스이다. `MavenArtifactRepository` 에는 `url` 의 획득자로 `URI getUrl();` 메서드가 있고 설정자로 `void setUrl(URI url);` 메서드와 `void setUrl(Object url);` 메서드가 있는데, Groovy 에서는 알아서 `void setUrl(Object url);` 을 사용하여 `url = '...'` 을 쓸 수 있으나, Kotlin 에서는 `url` 이 `var` 프로퍼티가 되려면 획득자와 유형이 일치하는 `void setUrl(URI url);` 를 설정자로 사용해야 하므로, `url = "..."` 을 쓸 수 없다. 굳이 `url` 프로퍼티를 사용하고자 한다면 `url = java.net.URI("...")` 이 되어야 할 것이다.

- `clean`

  - Gradle Groovy DSL:

    ```groovy
    task clean(type: Delete) {
        delete rootProject.buildDir
    }
    ```
    
    Gradle Kotlin DSL:
    
    ````kotlin
    tasks.register<Delete>("clean") {
        delete(rootProject.buildDir)
    }
    ````
    
  - `tasks.getByName<Delete>("clean")` 을 사용할 수 없다. 루트 프로젝트에는 어떤 플러그인도 적용되어 있지 않고, `clean` task 역시 존재하지 않기 때문이다. 실제로 명령줄에 다음과 같이 입력하면 루트 `build` 폴더만 사라진다.
  
    ```
    $ ./gradlew --no-daemon :clean
    ```
  
    명령줄에 다음과 같이 입력하면 모든 프로젝트 모듈의 `clean` task 를 실행하게 된다. 이것이 IDE 의 Clean Project 와 동등한 동작이다.
  
    ```
    $ ./gradlew --no-daemon clean
    ```
  
- 그 외에, 기본 유형이 아닌 것들은 유형을 명시해 줘야 한다. Groovy 클로저는 Kotlin 람다로 바뀐다.

  - Gradle Groovy DSL:

    ```groovy
    ext {
        dependencyClosure = {
            // ...
            // may need rehydration
        }
    }
    ```

    Gradle Kotlin DSL:

    ```kotlin
    extra.run {
        val dependencyClosure: ExternalModuleDependency.() -> Unit = {
            // ...
        }
        this["dependencyClosure"] = dependencyClosure
    }
    ```

    or, Gradle Kotlin DSL:

    ```kotlin
    extra.run {
        this["dependencyClosure"] = Action<ExternalModuleDependency> {
            // ...
        }
    }
    ```

루트 `build.gradle` 역시 `build.gradle` 이기 때문에, `gradle.properties` 나 `local.properties` 파일에 기재되어 있거나 명령줄 옵션 `-P` 로 전달되는 Gradle 프로퍼티 값을 다루는 방법을 짚고 넘어가겠다. `settings.gradle.kts` 의 경우 다음과 같이 `Settings` 로부터 가져왔다.

```kotlin
val something: String by settings // getSettings()
```

`build.gradle.kts` 의 경우 안타깝지만 이렇게 장황하게 써야 한다. `rootProject.extra` 가져오듯.

```kotlin
val something = properties["something"] as String // getProperties()
```

&hellip; 농담이다. 이런 안타까운 일은 없다.

```kotlin
val something: String by project // getProject()
```

정말 미친 편의성 아닌가!? 이제 버전은 `gradle.properties` 파일로 옮겨 두자. 웬만하면 `buildscript {}` 블럭을 없앨 수 있다.

그래도 `ext` 를 꼭 쓰고 싶다면, `extra` 값도 `by` 로 꺼내 쓸 수 있다.

```kotlin
val something: String by extra // .extra; getExtensions() |> getExtraProperties()
```

## `app/build.gradle.kts`

가장 골치아픈 부분이다. 사실 이 부분에는 정답이 없고 하나씩 해 보는 게 맞다. 다행인 점은, 빌드스크립트의 모든 부분에 `plugin {}` 블럭처럼 멱등성을 부여해야 한다거나 이런 건 없다는 사실이다. Gradle Kotlin DSL 을 쓰든 Gradle Groovy DSL 을 쓰든 Gradle 빌드스크립트는 Gradle 의 근본 디자인을 따라 위에서 아래로 실행되는 절차적 맥락에서 해석된다.

### `android {}` — `defaultConfig {}`

가장 기본이 되는 부분부터 보자.

Gradle Groovy DSL:

```groovy
android {
    compileSdkVersion 30
    buildToolsVersion '30.0.3'
    defaultConfig {
        applicationId 'net.aurynj.app.myapplication'
        minSdkVersion 24
        targetSdkVersion 30
        versionCode 1
        versionName '1.0'
        testInstrumentationRunner 'androidx.test.runner.AndroidJunitRunner'
    }
    // ...
}
```

Gradle Kolin DSL:

```kotlin
android {
    compileSdkVersion(30)
    buildToolsVersion("30.0.3")
    defaultConfig {
        applicationId("net.aurynj.app.myapplication")
        minSdkVersion(24)
        targetSdkVersion(30)
        versionCode(1)
        versionName("1.0")
        testInstrumentationRunner("androidx.test.runner.AndroidJUnitRunner")
    }
    // ...
}
```

여기에서 `buildToolsVersion`, `applicationId`, `versionCode`, `versionName`, `testInstrumentationRunner` 는 배정 연산자가 작동하므로 이렇게도 쓸 수 있다.

```kotlin
buildToolsVersion = "30.0.3"
```

음&hellip; 이게 되는 것과 안 되는 것들의 구분을 당장 구체적으로 아는 게 그렇게 중요할까? Android Gradle Plugin 은 꾸준히 업데이트되고 있고, 기존 Android 앱 프로젝트나 Android 라이브러리 프로젝트의 빌드스크립트 모델에 대한 확장 유형을 제공하고 있다. `com.android.build.tools:builder-model:4.2.1` 은 Sources JAR 파일을 제공하는데 이걸 열어 보면 상당부분이 Kotlin 으로 작성되어 있음을 알 수 있다. 앞으로 AGP 가 Kotlin 으로 작성된 부분의 비율은 더 커질 것이고, 기존에 배정 연산자가 동작하지 않던 이름이었던 것은 머지않아 AGP 업데이트와 함께 배정 연산자가 동작하는 이름으로 바뀔 수도 있다. 그러니 이쪽 디테일에는 신경을 끄고 전부 괄호를 쳐 보자. 적어도 AGP 4.1.2 에서는 전부 함수로 되어 있긴 하다. AGP 버전에 따라 함수로 되어 있지 않은 것이 있다면&hellip; 건투를 빈다. 적어도 함수와 `var` 프로퍼티 중 하나는 동작할 것이다.

### `android {}` — `buildConfigField`

다음은 BuildConfig 필드이다. `buildConfigField` 로 정의한 값은 `BuildConfig.java` 에 `public static final` 값으로 나타나기 때문에 꽤 요긴하게 쓰인다. 흔히 이렇게 바꾸라고 알려져 있다.

Gradle Groovy DSL:

```groovy
buildConfigField 'String', 'APP_NAME_RAW', '"My Application"'
```

Gradle Kotlin DSL:

```kotlin
buildConfigField("String", "APP_NAME_RAW", "\"My Application\"")
```

몇 안 되는 굉장히 짜증스러운 부분이다. Kotlin 에서는 `Char` 유형 리터럴만 작은따옴표로 만들 수 있고 `String` 유형 리터럴은 큰따옴표로만 만들 수 있다. 그러면 이걸 계속 이렇게 써야 하냐면, 이거 삼중따옴표를 이용해 이렇게도 쓸 수 있다.

```kotlin
buildConfigField("String", "APP_NAME_RAW", """"My Application"""")
```

### `android {}` — `buildTypes`

Gradle Groovy DSL:

```groovy
buildTypes {
    release {
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```

Gradle Kotlin DSL:

```kotlin
buildTypes {
    getByName("release") {
        minifyEnabled(false)
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
    }
}
```

`buildTypes` 의 인자는 `Action<in NamedDomainObjectContainer<BuildType>>` 이다. Gradle 인터페이스 유형 `NamedDomainObjectContainer<T>` 의 상위 유형으로 `NamedDomainObjectSet<T>`, `NamedDomainObjectCollection<T>`, `DomainObjectCollection<T>` 등이 이어지는데, `NamedDomainObjectCollection<T>` 에 있는 메소드가 바로 다음과 같은 것들이다.

- `T getByName(String name) throws UnknownDomainObjectException;`
- `T getByName(String name, Closure configureClosure) throws UnknownDomainObjectException;`
- `T getByName(String name, Action<? super T> configureAction) throws UnknownDomainObjectException;`

이 중 두번째가 Gradle Groovy DSL 에서 `release { /* ... */ }` 를 가능하게 하는 부분이다. 세 번째가 `getByName("release") { /* ... */ }` 에서 실행된다. 따라서 `getByName("release") { /* ... */ }` 은 `getByName("release").run { /* ... */ }` 과 동등하다.

위 예시에서 `proguardFiles` 구문은 첫 인자를 다음 줄로 내려 뒀다. Groovy 의 경우, 함수 호출을 여러 줄에 걸쳐 쓰려면 적어도 첫번째 인자는 함수 이름과 같은 줄에 쓰고 그 줄을 쉼표로 끝내야 한다는 규칙이 있다. 아니면 괄호를 치게 되는데, Gradle Groovy DSL 에서 괄호는 아주 예외적으로 쓰이기에 상당히 거슬리므로 이걸 피하고 싶어지는 느낌이 있다. 그런데 Kotlin 에서는 함수 호출에 항상 괄호를 쳐야 하므로 오히려 이런 걸 자유자재로 할 수 있게 된다. 컬럼이 긴 빌드스크립트는 아무도 읽고 싶어하지 않는다는 사실을 생각하면, 이런 특성은 Gradle Kotlin DSL 이 유지보수성 좋은 빌드스크립트를 작성하는 데에 도움이 된다고 해석할 수도 있다.

`minifyEnabled(false)` 는 `isMinifyEnabled = false` 로도 쓸 수 있다.

### `android {}` — `signingConfigs`

`signingConfigs` 블럭은 여러 이름의 `SigningConfig` 를 만들고 나중에 이름으로 이 중 하나를 찾아 `signingConfig` 로 지정하는 데에 사용된다. `signingConfigs` 의 경우, `getByName` 이 아니라 `create` 를 사용한다. 호스트는 마찬가지로 `NamedDomainObjectContainer<T>` 이다.

Gradle Groovy DSL:

```groovy
signingConfigs {
    release {
        storeFile file(releaseKeystorePath)
        storePassword releaseKeystorePass
        keyAlias releaseKeyAlias
        keyPassword releaseKeyPass
    }
}
```

Gradle Kotlin DSL:

```kotlin
signingConfigs {
    create("release") {
        storeFile(file(releaseKeystorePath))
        storePassword(releaseKeystorePass)
        keyAlias(releaseKeyAlias)
        keyPassword(releaseKeyPass)
    }
}
```

위 코드는 `releaseKeystorePath`, `releaseKeystorePass`, `releaseKeyAlias`, `releaseKeyPass` 가 위에서 언급한 방식으로 `local.properties` 나 `-P` 옵션으로 주입된 값일 것으로 가정하고 작성되었다.

사용법은 다음과 같다.

Gradle Groovy DSL:

```groovy
signingConfig signingConfigs.release
```

Gradle Kotlin DSL:

```kotlin
signingConfig = signingConfigs.getByName("release")
```

or, Gradle Kotlin DSL:

```kotlin
setSigningConfig(signingConfigs.getByName("release"))
```

### `android {}` — `variantFilter`, `applicationVariants`

`variantFilter` 와 `applicationVariants` 는 많은 빌드스크립트의 분량을 늘려 주는 주범이다. 특히 `applicationVariants` 에 `tasks.register` 같은 task 정의가 있다면 이건 꼭 필요한 일이겠지만 가끔 그렇지 않은 경우도 있다. 어쩌겠는가&hellip; 일단 기존 빌드스크립트와 동일한 동작을 맞춰 주는 것이 중요한 일이다.

Gradle Groovy DSL:

```groovy
variantFilter { variantFilter ->
    def names = variantFilter.flavors*.name
    // ...
}
```

Gradle Kotiln DSL:

```kotlin
variantFilter {
    val names = flavors.map { it.name }
    // ...
}
```

Groovy 에서 가장 유명한 스프레드 연산자를 Kotlin 에서 없애는 방법이다. Groovy 에서 `foo*.bar` 라고 쓰면 `foo.collect { it.bar }` 와 같은 의미가 되는데, Kotlin 의 경우 `map` 이 `Iterable<T>` 와 `(T) -> U` 를 받아 `List<U>` 값을 반환하므로, `map` 을 사용하면 된다.

트레일링 람다를 사용하기 싫다면 이렇게 할 수 있다. `ProductFlavor` 를 임포트하고, 이렇게 쓴다.

```kotlin
val names = flavors.map(ProductFlavor::getName)
```

(`import` 구문은 `plugins {}` 블럭보다도 위에 있어야 한다.)

다음은 `applicationVariants` 이다.

Gradle Groovy DSL:

```groovy
applicationVariants.all { variant ->
    def flavorName = variant.getFlavorName()
    // ...
}
```

Gradle Kotlin DSL:

```kotlin
applicationVariants.all { // or forEach, whatever
    val flavorName = flavorName
    // ...
}
```

`all` 은 앞서 등장한 Gradle 인터페이스 `DomainObjectCollection<T>` 에 있는 메소드이다. Java 에 java.util.function 이 도입된지 얼마 안 되었고 Gradle 이 하위 버전 JDK 에서도 동작해야 하다 보니 Gradle 에 `Collection<T>` 의 하위 유형으로 `DomainObjectCollection<T>` 을 정의해 두고 `void all(Closure action);` 같은 것을 넣어 둔 것이다.

Kotlin 에서는 `void all(Action<? super T> action);` 메소드가 사용되는데 이거 쓰지 않고 `forEach` 써도 무관하다. `applicationVariants` 를 따라가다 보면 상위유형이 `Iterable<ApplicationVariant>` 이기 때문이다. 그러나 `all` 을 사용하면 람다를 `Action` 으로 넘기기 때문에 `ApplicationVariant` 인스턴스로 람다의 리시버로 들어오므로 좀더 편하게 코딩할 수 있다.

이쯤 해서 굳이 Groovy 클로저를 Kotlin 에서 만드는 방법을 알아보자. 스크립트 언어는 Kotlin 이지만 클래스패스 의존성으로 Groovy 구현이 들어있으므로 Groovy 클로저도 만들 수 있다. `ApplicationVariant` 를 임포트하고, 이렇게 쓴다.

```kotlin
applicationVariants.all(closureOf<ApplicationVariant> {
    val flavorName = flavorName
    // ...
})
```

이 부분에서 헤매는 사람이 꽤 많은데 Kotlin 컴파일러는 오히려 Groovy `Closure` 와 Gradle `Action` 사이에서 헤매지 않는 편이다. Gradle 인터페이스 유형 `Action` 은 어노테이션 `@HasImplicitReceiver` 를 달고 있고 단일 메서드 `void execute(T t);` 만 갖고 있는 함수형 인터페이스이다. Kotlin 컴파일러는 함수형 인터페이스로 가는 SAM 변환도 수행하고 `@HasImplicitReceiver` 어노테이션에 대한 이해도 있기 때문에, 사람이 굳이 `Action { /* ... */ }` 라고 SAM 변환 힌트를 줄 필요는 없다. (그게 안 됐다면 `buildTypes`, `signingConfigs` 에서 람다도 마음대로 못 썼을 것이다.)

### `tasks.register` or `tasks.create`

`tasks` (`getTasks()`) 는 `TaskContainer` 유형으로 Gradle task 리스트에 접근하고 task 를 만드는 등의 목적으로 쓰인다. 옛날 API 인 `create(...)` 는 호출하는 즉시 `Task` 를 생성하고 `Task` 하위 유형 인스턴스를 즉시 제공하지만 Gradle v5 API 인 `register(...)` 는 `TaskProvider` 를 만들어서 `dependsOn` 같은 참조에 쓸 수 있게 하되 `Task` 는 즉시 생성하지 않는다는 차이가 있고&hellip; 자세한 차이는 Gradle User Guide 의 [*Task Configuration Avoidance*](<https://docs.gradle.org/current/userguide/task_configuration_avoidance.html>) 문서에 잘 나와 있다.

> **How do I defer task creation?**
>
> Effective task configuration avoidance requires build authors to change instances of [`TaskContainer.create(java.lang.String)`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskContainer.html#create-java.lang.String-) to [`TaskContainer.register(java.lang.String)`](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskContainer.html#register-java.lang.String-).
>
> Older versions of Gradle only support the `create(…)` API. The `create(…)` API eagerly creates and configures tasks when it is called and should be avoided.
>
> Using `register(…)` alone may not be enough to avoid all  task configuration completely. You may need to change other code that  configures tasks by name or by type, as explained in the following  sections.

`TaskContainer` 메소드 `create` 와 `register`, 그리고 상위 유형 `TaskCollection<Task>` 의 메소드 `withType` 과 `named` 에 대해서, Kotlin `KClass<T>` 를 응용한 제네릭 확장 메서드가 제공된다.

Gradle Groovy DSL:

```groovy
tasks.create('t', TaskT) {
    // ...
}

tasks.register('u', TaskU) {
    // ...
}

tasks.withType(TaskV) {
    // ...
}

tasks.named('w', TaskW) {
    // ...
}
```

Gradle Kotlin DSL:

```kotlin
tasks.create<TaskT>('t') {
    // ...
}

tasks.register<TaskU>('u') {
    // ...
}

tasks.withType<TaskV> {
    // ...
}

tasks.named<TaskW>("w") {
    // ...
}
```

깜찍하다. `create` 와 `register` 는 `TaskContainer` 의 또다른 상위유형인 `PolymorphicDomainObjectContainer<Task>` 의 메소드인데 여기에 대해서는 `creating` 과 `registering` 이라는 추가 확장이 제공되어서 이 내부 구현이 `by` 를 지원하고 task 이름을 생략할 수 있게 해 준다.

```kotlin
val awesome: Copy by tasks.creating(Copy::class) {
    // ....
}

val moreAwesome: TaskProvider<Copy> by tasks.registering(Copy::class) {
    // ....
}
```

유형 선언은 임의로 생략해도 좋다. Gradle API 에서 `T (T extends Task)` 와 `TaskProvider<? extends Task>` 는 동등하다.

```kotlin
val awesome by tasks.creating(Copy::class) {
    // ....
}

val moreAwesome by tasks.registering(Copy::class) {
    // ....
}
```

`::class` 표기가 불편하다면 제네릭 유형 인자를 받는 `TaskContainer` 확장을 만들어 두는 것도 한 가지 방법이다. 이 경우 전체적인 코드 형상은 `tasks.withType` 과 거의 똑같게 된다. `by` 를 보고 프로퍼티 이름을 보존해야 한다는 사실을 기억해 둬야 할 것이다.

### `dependencies {}`

가장 기본적인 `libs/*.jar` 부터 보자. Groovy 문법을 털어내는 마지막 부분이다.

Gradle Groovy DSL:

```groovy
implementation fileTree(dir: 'libs', include: ['*.jar'])
```

Gradle Kotlin DSL:

```kotlin
implementation(fileTree("dir" to "libs", "include" to arrayOf("*.jar")))
```

다음은 `exclude` 이다.

Gradle Groovy DSL:

```groovy
implementation('some-group:some-module:some-version') {
    exclude group: 'another-group', module: 'another-module'
}
```

Gradle Kotlin DSL:

```kotlin
implementation("some-group:some-module:some-version") {
    exclude(mapOf("group" to "another-group", "module" to "another-module"))
}
```

or, Gradle Kotlin DSL (simpler):

```kotlin
implementation("some-group:some-module:some-version") {
    exclude(group = "another-group", module = "another-module")
}
```

`ModuleDependency` 에 대해서는 `exclude` 의 이름 있는 인자 버전이 추가로 제공된다.

그 외에 이런 대응 규칙이 있다.

- Gradle Groovy DSL:

  ```groovy
  apply from: 'common.gradle'
  ```

  Gradle Kotlin DSL:

  ```kotlin
  apply(from = "common.gradle")
  ```

  모든 스크립트의 언어를 통일할 필요는 없기 때문에 `apply from:` 은 일단 놔두고 저쪽 스크립트도 Gradle Groovy DSL 인 채로 둔다. 이 내용은 다음 절에서 다룬다.

## `buildSrc` 도입하기

이쯤 했으면 개별 프로젝트 모듈의 `build.gradle.kts` 를 만드는 작업도 끝난다. 마지막이 바로 `buildSrc` 이다. 많은 경우 `buildSrc` 에 버전같은 공통 상수를 보관하는 용법을 추천하는데 이 구조는 다음과 같다.

```
+-- app/
|`+-- src/main/java/...
| +-- build.gradle.kts
+-- buildSrc/
|`+-- build.gradle.kts
| +-- src/main/java/Version.kt
+-- build.gradle.kts
+-- gradle.properties
+-- gradlew
+-- local.properties
+-- settings.gradle.kts
```

`buildSrc/build.gradle.kts`:

```kotlin
plugins {
    `kotlin-dsl`
}

```

`buildSrc/src/main/java/Version.kt`:

```kotlin
object Version {
    const val androidGradlePluginVersion = "4.2.1"
    // ...
}
```

분명히 말할 수 있는 건, `buildSrc` 는 `extra` 값을 상술한 `by extra` 없이 깔끔하게 쓰자고 도입하기에는 득보다 실이 너무 크다. `buildSrc` 의 정체는 바로 빌드스크립트의 클래스패스로, `buildSrc` 에 있는 코드는 Gradle 런타임에 포함되어 빌드스크립트에서 바로 참조 가능한 코드가 되는 것이다. 즉 Gradle 플러그인 전체를 구성하지 않고 Gradle 플러그인과 비슷한 것을 작성하는 간편한 도구로 `buildSrc` 가 제공되는 셈이다.

그냥 생각해도 공통 상수를 여러 모듈에서 쓰도록 보관하겠다고 플러그인을 만든다고? 싶은데 실제 동작은 더욱 심각하다. `buildSrc` 는 애초에 플러그인이 아니라서 상술한 멱등성 보장이 되지 않고, 따라서 `buildSrc` 의 코드를 수정하면 수정된 클래스를 로드하기 위해 기존 클래스를 내려야 하므로 Gradle daemon 을 아예 못쓰게 된다. `buildSrc` 를 자주 수정하면 Gradle 은 계속 `buildSrc` 를 컴파일하고 task 의존성 그래프를 처음부터 다시 그려야 하며 플러그인도 매번 다시 로드해야 한다. 이럴 거면 왜 `plugins {}` 블럭을 썼을까? 공통 상수 관리에는 웬만하면 Gradle 프로퍼티 값을 쓰기 바란다.

`buildSrc` 를 작성하는 것 자체는 전혀 어렵지 않은 일이니 설명은 생략한다. `buildSrc` 를 작성하는 일은 최소로 하는 것이 좋다고 보나, 이런 경우에는 재사용이 자주 될 것이므로 `buildSrc` 를 만들면 도움이 될 것이다.

* `extra` 없이 `by` 로 `Int` 나 `Double` 값을 빠르게 만드는 `by` 구현 만들기. `extra` 는 대입한 값을 그대로 보존하고 있으며 유형도 보존되지만 Gradle 프로퍼티는 모두 문자열이기 때문에 Gradle 프로퍼티를 `Int` 나 `Double` 로 변환해 주는 Kotlin 대리 프로퍼티 확장을 만들면 쓸모있을 것이다. 최대한 대강 만들어 보자면 이런 식으로.

  `buildSrc/src/main/java/Extensions.kt`:

  ```kotlin
  import org.gradle.api.Project
  import kotlin.reflect.KProperty
  
  val Project.integers get() = DelegatedIntegersForProjectFromProperties.of(this)
  
  data class DelegatedIntegersForProjectFromProperties
  constructor(
      val project: Project
  ) {
      companion object {
          fun of(project: Project): DelegatedIntegersForProjectFromProperties {
              return DelegatedIntegersForProjectFromProperties(project)
          }
      }
  
      operator fun getValue(receiver: Any?, property: KProperty<*>): Int {
          val raw = project.properties[property.name] as String
          return raw.toInt()
      }
  }
  ```

* 상술한 `creating`/`registering` 의 제네릭 유형 인자 버전 만들기 등.

* **`apply` 구문 대체하기.** `apply` 구문은 `apply plugin:` 뿐만 아니라 `apply from:` 도 지원하는데, `apply from:` 의 경우 로드되는 스크립트에서 로드하는 스크립트 쪽의 맥락을 미리 알 수 없으므로, 로드되는 스크립트의 내용이 로드하는 스크립트의 맥락에 의존했다면 로드되는 스크립트를 Gradle Kotlin DSL 로 작성하기에는 무리가 있다. 많은 경우 로드되는 스크립트 쪽에 `android {}` 블럭이 포함되어 있으므로 이 문제를 겪을 수 있다. 이 경우 로드되는 스크립트는 Gradle Groovy DSL 로 놔둬도 되지만, Gradle Kotlin DSL 로 바꿔 보는 걸 도전과제로 삼고자 한다면, 로드되는 스크립트의 일부를 재사용 가능하게 만들어 `buildSrc` 로 옮기는 것이 한 가지 방법이 된다. 다만 `buildSrc` 의 빌드 결과는 `clean` 시에 함께 지워지기 때문에 이런 해법을 완성했다면 장기적으로는 Gradle 플러그인으로 만들어 버리는 것이 가장 깔끔한 방법이다.

## 결과

최초의 `settings.gradle`, `build.gradle`, `app/build.gradle` 이 어떻게 바뀌었는지 보자.

`gradle.properties`:

```
androidGradlePluginVersion=4.2.1
kotlinVersion=1.5.0
releaseKeystorePath=
releaseKeystorePass=
releaseKeyAlias=
releaseKeyPass=
```

`settings.gradle.kts`:

```kotlin
rootProject.name = "My Application"

pluginManagement {
    val androidGradlePluginVersion: String by settings
    val kotlinVersion: String by settings
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    plugins {
        id("com.android.application") version androidGradlePluginVersion
        kotlin("android") version kotlinVersion
    }
}

include(":app")
```

`build.gradle.kts`:

```kotlin
// Top-level build file where you can add configuration options common to all sub-projects/modules.

subprojects {
    repositories {
        google()
        mavenCentral()
    }
}

tasks.register<Delete>("clean") {
    delete(rootProject.buildDir)
}
```

`app/build.gradle.kts`:

```kotlin
plugins {
    id("com.android.application")
    kotlin("android")
}

val kotlinVersion: String by project
val releaseKeystorePath: String by project
val releaseKeystorePass: String by project
val releaseKeyAlias: String by project
val releaseKeyPass: String by project

android {
    compileSdkVersion(30)
    buildToolsVersion("30.0.3")
    defaultConfig {
        applicationId("net.aurynj.app.myapplication")
        minSdkVersion(24)
        targetSdkVersion(30)
        versionCode(1)
        versionName("1.0")

        testInstrumentationRunner("androidx.test.runner.AndroidJUnitRunner")
    }

    signingConfigs {
        create("release") {
            storeFile(file(releaseKeystorePath))
            storePassword(releaseKeystorePass)
            keyAlias(releaseKeyAlias)
            keyPassword(releaseKeyPass)
        }
    }

    buildTypes {
        getByName("release") {
            setSigningConfig(signingConfigs.getByName("release"))
            minifyEnabled(false)
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = "1.8"
    }
}

dependencies {
    implementation(fileTree("dir" to "libs", "include" to arrayOf("*.jar")))
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8:$kotlinVersion")
    implementation("androidx.core:core-ktx:1.5.0")
    implementation("com.google.android.material:material:1.2.1")
    testImplementation("junit:junit:4.12")
    androidTestImplementation("androidx.test.ext:junit:1.1.2")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.3.0")
}
```

## 히히 이제 도망 못 가!

축하한다! 생각보다 길고 험난한 여정이었다.

Gradle Kotlin DSL 은 한번 도입하고 마무리하는 일이 아니고 앞으로 끝나지 않을 일의 시작일지도 모른다. 특히 마지막에 등장한 `apply from:` 으로 로드되는 스크립트에 `android {}` 블럭이 있는 경우를 생각하면, 전체 빌드스크립트를 Gradle Kotlin DSL 로 유지하는 데에는 꽤 많은 의지가 필요할 것이라 생각한다.

다시 한번 짚어 두자면, 빌드스크립트와 소스코드가 언어와 런타임 모두를 공유하는 부트스트래핑은 프로그래밍 언어 전체를 둘러 보더라도 그 사례가 그리 많지 않다. Kotlin/JVM 은 지금 그 대열에 합류하는 도전의 길을 걷고 있다.

다른 말로 이곳은 약간 미지의 세계이기도 하다. 오늘날에도 빌드스크립트 엔지니어링은 소스코드가 개입하는 영역과는 상당히 별개 도메인으로 간주되며 빌드스크립트의 런타임과 컴파일타임에 대한 지식은 상당히 낮은 접근성에 놓여 있는 편이었다. 그러나 Gradle Kotlin DSL 은 소스코드와 빌드스크립트의 경계를 허물어 줬고, 무엇을 할 수 있을지 찾는 것은 우리 몫이다. 바꿔 말하자면, Gradle Kotlin DSL 을 도입함으로써 빌드 프로세스에 대고 무엇이든 한번 해 볼 수 있는 가능성이 좀더 열리는 것이다.

거창한 얘기를 하고자 한 건 아니다. 다만 이 가능성이 다 결실로 완성되기 전에 내가 뭐라도 해 봐야 재밌지 않겠나. 혹시 재밌는 일이 생기면 나도 껴 달라는 부탁을 드린다.
