Trouble ID `2021-04-01.moshi-kapt-test`

# 유닛 테스트 구성에서 Moshi kapt 사용하기

1 년 넘게 JSON 파서로 Moshi 를 사용하는 중이다. 몇 가지 이유로 Moshi 로 JSON 프로세싱을 하는 Android 라이브러리를 하나 만들어서 그대로 배치 작업에도 사용하고 있는데, Android 프로젝트이기 때문에 Gradle 을 사용하고, 이 때문에 엔트리포인트를 임의로 여럿 굴릴 수 있다는 이점으로 유닛 테스트 구성을 사용한다. 커맨드라인에서는 이런 식으로 쓴다.

```
./gradlew --no-daemon :lib:testReleaseUnitTest --tests com.example.lib.JsonTaskTest7.testGenerateAll
```

한참 잘 쓰다가 며칠 전에 왠지 뒷골이 섬뜩함을 느꼈는데, build.gradle 파일을 보니까 이렇게 생겼다.

```
...
dependencies {

    implementation "com.squareup.moshi:moshi:$moshiVersion"
    kapt "com.squareup.moshi:moshi-kotlin-codegen:$moshiVersion"

    ...

    testImplementation "com.squareup.moshi:moshi-kotlin:$moshiVersion"
}
```

왓? 그러니까&hellip; 난 순수 코드젠만 쓰고 있다고 생각했는데 알고 보니 테스트 런타임에서는 반영<sup>reflection</sup> 기반으로 돌아가는 `moshi-kotlin` (`moshi-kotlin-reflect`) 에 의존해서 살고 있었다. `testImplementation` 구문을 지워 봤다. 테스트 구문에서도 자연스럽게 `KotlinJsonAdapterFactory` 참조가 모두 날아간다.

```
java.lang.RuntimeException: Failed to find the generated JsonAdapter class for class ......
```

당연히 어댑터가 없다고 나온다. 시간을 한참 날렸다.

`kapt` 는 Kotlin 어노테이션 프로세서를 동작시키는 Gradle 구성으로, Java 프로젝트의 `annotationProcessor` 와 비슷하며, Maven 의 `provided` 에 대응한다. 그러나 `provided` 와 다른 점이 있다.

`implementation` 구문은 `compile` 이나 `runtime` 처럼 `UnitTest` 에도 전파되는 의존성이지만, `kapt` 는 `main` 소스 코드에만 돌아가는 어노테이션 프로세서 구문이기 때문에, `test` 소스 코드를 위해 `kaptTest` 구문을 써야 한다.

```
kaptTest "com.squareup.moshi:moshi-kotlin-codegen:$moshiVersion"
```

