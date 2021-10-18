Trouble ID `2021-06-02.android-project-local-m2-repo`

# Android 프로젝트 로컬 m2 저장소

> **UPDATE 4/27/2021:** We listened to the community and will keep JCenter as a read-only repository indefinitely. Our customers and the community can continue to rely on JCenter as a reliable mirror for Java packages.
>
> **UPDATE 2/28/2021:** To better support the community in this migration, JFrog has extended the JCenter **new package versions submission** deadline through **March 31st 2021**.
>
> ----
>
> jfrog.com, Stephen Chin. *Into the Sunset on May 1st: Bintray, GoCenter, and ChartCenter.* <https://jfrog.com/blog/into-the-sunset-bintray-jcenter-gocenter-and-chartcenter/>

JFrog 의 JCenter 서비스 종료 발표와 함께 Maven 을 사용하는 프로그래머 커뮤니티가 일대 혼란에 빠졌다. 다행히 위에 인용한 대로 JCenter 는 무기한 유지된다는 보충공지가 되긴 했으나&hellip; 한번 오픈소스 생태계에 무상지원을 끊겠다고 한 비용계산법이 또 언제 결론을 달리하게 될지는 모르는 일이기에 실제로 이것이 언제까지 지속될지에 대해서는 의문이 있다. 한편 JCenter 는 예전부터 DDoS 상황에 시달려 왔으며 JCenter 유저들은 서비스 다운을 자주 겪었고, 읽기전용 전환 이후 현재까지도 JCenter 서비스 안정성은 나빠지고 있다.

이 상황에서 JCenter 에 기대어 사는 건 여러 불확정성과 미지위험을 놔두는 셈이 된다. JCenter 를 빠르게 탈출하는 것 외에 적확한 방향은 없겠다고 감히 말할 수 있겠다.

## 대안 찾기

### 풀 미러링

몇 가지 대안이 있을텐데 JCenter 전체를 클론하는 미러를 만드는 풀 미러링 방법이 있다. 이 방법은 결과는 확실하지만 비용이 상당히 많이 들고 비매너로 여겨진다. 그도 그럴 것이 풀 미러링은 주기적으로 저장소의 디렉터리 인덱스 페이지를 재귀적으로 전체 탐색하는 방식으로 무지막지한 부하를 가하기 때문이다! Apache 의 Maven Central 은 공익적 목적으로 사전 협의나 제휴를 거치지 않는다면 프라이빗 풀 미러를 만들려는 접근은 차단해 버리겠다고 공식 문서에서 단언하고 있다.

> **Creating Your Own Mirror**
>
> The size of the central repository is [increasing steadily](http://search.maven.org/stats) To save us bandwidth and you time, mirroring the entire central repository is not allowed. (Doing so will get you automatically banned.) Instead, we suggest you setup a [repository manager](https://maven.apache.org/repository-management.html) as a proxy.
>
> If you really want to become an official mirror, contact Sonatype at [MVNCENTRAL](https://issues.sonatype.org/browse/MVNCENTRAL) with your location and we'll work to get you setup.
>
> ----
>
> maven.apache.org. Maven Guide to Mirror Settings. *Using Mirrors for Repositories.* <https://maven.apache.org/guides/mini/guide-mirror-settings.html>

JCenter 는 풀 미러링을 시도하는 사람들로 인해 DDoS 상황에 너무 시달린 나머지 상기 선세팅 공지 후 인덱스 페이지를 제공하지 않고 카탈로그 이하 경로의 파일만 제공한다. 풀 미러링은 사실상 불가능한 셈이다.

풀 미러링은 하지 않기로 했다. 이 과정에서 좀 다른 경험들이 결정에 근거가 되기도 했다.

사내 다른 부서나 자회사에서 운영하는 내부 전용 m2 저장소가 꽤 많은데, 이 중 하나가 이전부터 JCenter 풀 미러를 운영해 왔다. 이 미러를 운영하던 팀에서 운영하던 다른 메인라인 release/snapshot 저장소가 있는데, JCenter 폐쇄 이후 JCenter 미러에만 있는 패키지를 이 메인라인 저장소로부터 JCenter 미러로 리디렉션을 시작했다. JCenter 패키지 기반으로 빌드되던 프로젝트가 워낙 많기에 이들을 위해서 JCenter 가 셧다운되더라도 사내 저장소가 알아서 잘 해 주겠다고 서포트 정책을 세운 셈이다. 그런데 이 정책의 실 구현이 급조되었다 보니 release/snapshot 저장소 전체가 상당히 불안정하게 동작하게 되었고 결국 빌드가 간신히 성공하기는 하지만 빌드 속도는 엉망진창이 되었다. 이러다 보니 결국 JCenter 읽기전용 서비스도 믿지 못하고, 사내 release/snapshot 저장소에서 JCenter 미러가 계속 지원될 것이라고 믿지도 못하는데, (이에 대한 서포트 정책은 해당 부서에서 공식화한 적이 없다) 이게 문제가 특정 형상으로 지속되어 인식을 명확히 공유하는 단계에 이르지는 못하는 이상한 상태가 되었다.

JCenter 풀 미러를 또 인하우스에서 재구축하는 것이 일견 매력적인 방식이나, 이것이 좋은 결과를 얻기 너무 어려운 방식이라는 인식을 얻어, 어렵지만 다른 방식으로 미지위험의 존재를 공론화할 필요성을 느꼈다.

### 프로젝트 로컬 m2 저장소

전혀 방식으로, 프로젝트 로컬 m2 저장소를 만드는 것이 있다. 이것은 외래 패키지의 어떤 바이너리가 사용될지에 대해 바이너리 스냅샷을 프로젝트에 내재시켜 두고, 패키지 참조를 해소하는 전략만 m2 방식을 사용하는 것이다.

이런 식으로 JCenter 유래 외래 패키지의 바이너리 스냅샷을 프로젝트에 내재시키게 되면, JCenter 패키지를 사용한 프로젝트가 JCenter 서비스 다운 시에도 빌드가 될 것을 보장하며, JCenter 의 임의 미러를 사용하는 것에 비해 고의나 과실로 위변조된 패키지가 유입되는 것을 방지해 준다. 한편 프로젝트 로컬 m2 저장소는 미러의 일종이지만 풀 미러와는 다르게 특정 프로젝트에서 사용되는 패키지에 대해서만 미러링을 하는 방식으로 만들어지기 때문에 오히려 m2 저장소 서버에 부하를 줄이고 덤으로 프로젝트 빌드 속도도 개선해 주는 win-win 효과도 있다.

이번에 Android 앱 프로젝트에 대해 프로젝트 로컬 m2 저장소를 만들었다. 순서는 다음과 같다.

- m2-repository Git 저장소 생성 및 파셜 미러 작성
- m2-repository Git 서브모듈 생성
- Gradle 스크립트 작성

다음과 같이 일을 진행하게 되었다.

- 특정 프로젝트에 대해 m2 의존성 중 사내 release/snapshot 저장소에만 있는 것과 JCenter 패키지를 구분해 냄.
- 프로젝트에서 사용 중인 JCenter 패키지에 대해 로컬 미러를 만들어 특정 프로젝트 전용 로컬 m2 저장소 생성.
- 사내 release/snapshot 저장소에서 찾아야 하는 패키지는, 해당 패키지 참조만 사내 저장소에서 해소하도록 하여 상태가 나쁜 사내 m2 사용을 줄임.

## 대안 구성하기

### 빌드스크립트: 프로젝트 로컬 저장소 추가

프로젝트 폴더 내에 `m2-repository` 폴더를 만들고 이 안에 로컬 m2 저장소 구성을 한다. 내 경우에는 이 부분을 Git 서브모듈로 만들어서 여러 프로젝트에서 공유할 수 있도록 했고 필요시 과거 형상의 코드를 다시 빌드해야 할 때 간편하게 사용할 수 있도록 했다.

```
repositories {
    maven {
        url = "file://${project.rootDir.absolutePath}/m2-repository"
        name = "project-local"
    }
}
```

`build.gradle` 에서 `repositories {}` 블럭에 위와 같이 `"project-local"` m2 저장소를 추가한다. 멀티모듈 프로젝트인 경우 최상위 `build.gradle` 파일의 `subprojects {}` 블럭에 (또는 `allprojects {}` 블럭에) 위 내용을 한번 추가하면 되며 `project.rootDir` 대신에 `rootProject.rootDir` 을 사용해야 한다.

### 빌드스크립트: 기존 저장소에서 사용할 패키지 지정

상태 나쁜 저장소에 애초에 특정 패키지에 대해 HEAD 요청을 보내지 않게 하는 방법. 마찬가지로 `repositories {}` 블럭에 이렇게 처리한다.

```
...

maven {
    name "<some-internal-m2-release-repo-name>"
    url "<some-internal-m2-release-repo-url>"
    mavenContent {
        releasesOnly()
        includeGroup("<some-internal-package-group>")
        includeGroup("<another-internal-package-group>")
    }
}

maven {
    name "<some-internal-m2-snapshot-repo-name>"
    url "<some-internal-m2-snapshot-repo-url>"
    mavenContent {
        snapshotsOnly()
        includeGroup("<some-internal-package-group>")
        includeGroup("<another-internal-package-group>")
    }
}
```

`includeGroup()` 구문은 얼라우리스트 역할을 한다. 이로써 꼭 필요한 패키지가 아니면 상태 나쁜 m2 저장소 서버에 HEAD 요청을 했다가 빨리 응답이 돌아오지 않아 기다리는 일이 일어나지 않도록 하였고, JCenter 패키지를 해당 서버에서 조회하지 않음으로써 이후에도 혹시나 JCenter 패키지를 무분별하게 사용할 가능성을 예방하였다.

### 파일시스템: 어떤 파일을 어떤 디렉터리에 제공할까

m2 저장소는 전통적으로 REST 를 엄격히 따르는 HTTP/1.1 서버로 만들어진다. `com.example:sample:1.2.3` (Gradle 표기법) 패키지 참조가 서버에 있다면 Maven POM 파일은 서버의 `/com/example/sample/1.2.3/sample-1.2.3.pom` 경로에서 제공되어야 하며 클라이언트는 이에 대한 HEAD 요청을 수행한다. 그리고 일반적으로 패키지 아티팩트는 JAR 파일로 (존재한다면) `/com/example/sample/1.2.3/sample-1.2.3.jar` 경로에서 제공될 것이다. 그 외에 같은 디렉터리에 `sample-1.2.3-javadoc.jar` 라든지 `sample-1.2.3-sources.jar` 같은 Javadoc JAR 파일, Sources JAR 파일 등이 있을 수 있고, 이들 파일과 함께 존재할 수 있는 것이 `sample-1.2.3.pom.asc` 같은 시그니처 파일 내지는 `sample-1.2.3.pom.md5`, `sample-1.2.3.pom.sha1`, `sample-1.2.3.pom.sha256` 같은 체크섬 파일이다.

HTTP 서버가 디렉터리 인덱스를 제공하는 방법에 대한 규약이 있는 게 아니므로 어떤 패키지에 대해서 어떤 버전들이 존재하는지는 별도로 제공된다. `com.example:sample` 이라면, 이 정보가 제공되는 경로는 `/com/example/sample/maven-metadata.xml` 이다. 이 파일 존재성은 선택적이다.

그 외에 저장소 최상위 디렉터리에 `/source.properties` 나 `/archetype-catalog.xml` 등이 제공될 수 있다.

### 파일시스템: 실제 트리 예시

문제가 된 대표적인 패키지는 JetBrains 에서 만들다 개발 중단한 [Anko](<https://github.com/Kotlin/anko>) 였다. 예전에 다 제거한 줄 알았던 Anko 찌꺼기가 프로젝트 어딘가에 제때 제거되지 않은 채 남아 있었고&hellip; 문제가 발견되었을 때엔 Anko 제거 작업을 하기에 여의치 않았기에 동작을 고정하기 위해 m2 저장소에 커버했다. 프로젝트 의존성에 포함된 패키지 참조는 다음과 같았다.

- `org.jetbrains.anko:anko-commons:0.10.8`
- `org.jetbrains.anko:anko-design:0.10.8`

mvnrepository.com 을 보자. [`org.jetbrains.anko:anko-commons` @ mvnrepository.com](<https://mvnrepository.com/artifact/org.jetbrains.anko/anko-commons>). 실제로는 이들에 `org.jetbrains.anko:commons-base:0.10.8` 과 `org.jetbrains.anko:design-base:0.10.8` 이 추의적 의존성으로 걸려 있었기 때문에 로컬 m2 저장소에 포함해야 하는 패키지는 총 4 종이었다.

파일시스템 트리는 이렇게 생겼다. `maven-metadata.xml` 과 체크섬 파일 등을 모두 제공하는 것을 목표로 삼았고, `0.10.8` 외의 패키지는 굳이 함께 담지 않았다. 체크섬 파일은 원본이 없는 것은 직접 만들었다.

```
+-- org/
 `+-- jetbrains/
   `+-- anko/
     `+-- anko-commons/
      |`+-- 0.10.8/
      | |`+-- anko-commons-0.10.8.aar
      | | +-- anko-commons-0.10.8.aar.md5
      | | +-- anko-commons-0.10.8.aar.sha1
      | | +-- anko-commons-0.10.8.aar.sha256
      | | +-- anko-commons-0.10.8.pom
      | | +-- anko-commons-0.10.8.pom.md5
      | | +-- anko-commons-0.10.8.pom.sha1
      | | +-- anko-commons-0.10.8.pom.sha256
      | | +-- anko-commons-0.10.8-sources.jar
      | | +-- anko-commons-0.10.8-sources.jar.md5
      | | +-- anko-commons-0.10.8-sources.jar.sha1
      | | +-- anko-commons-0.10.8-sources.jar.sha256
      | +-- maven-metadata.xml
      +-- anko-design/
      |`+-- 0.10.8/
      | |`+-- anko-design-0.10.8.aar
      | | +-- anko-design-0.10.8.aar.md5
      | | +-- anko-design-0.10.8.aar.sha1
      | | +-- anko-design-0.10.8.aar.sha256
      | | +-- anko-design-0.10.8.pom
      | | +-- anko-design-0.10.8.pom.md5
      | | +-- anko-design-0.10.8.pom.sha1
      | | +-- anko-design-0.10.8.pom.sha256
      | | +-- anko-design-0.10.8-sources.jar
      | | +-- anko-design-0.10.8-sources.jar.md5
      | | +-- anko-design-0.10.8-sources.jar.sha1
      | | +-- anko-design-0.10.8-sources.jar.sha256
      | +-- maven-metadata.xml
      +-- commons-base/
      |`+-- 0.10.8/
      | |`+-- commons-base-0.10.8.aar
      | | +-- commons-base-0.10.8.aar.md5
      | | +-- commons-base-0.10.8.aar.sha1
      | | +-- commons-base-0.10.8.aar.sha256
      | | +-- commons-base-0.10.8.pom
      | | +-- commons-base-0.10.8.pom.md5
      | | +-- commons-base-0.10.8.pom.sha1
      | | +-- commons-base-0.10.8.pom.sha256
      | | +-- commons-base-0.10.8-sources.jar
      | | +-- commons-base-0.10.8-sources.jar.md5
      | | +-- commons-base-0.10.8-sources.jar.sha1
      | | +-- commons-base-0.10.8-sources.jar.sha256
      | +-- maven-metadata.xml
      +-- design-base/
       `+-- 0.10.8/
        |`+-- design-base-0.10.8.aar
        | +-- design-base-0.10.8.aar.md5
        | +-- design-base-0.10.8.aar.sha1
        | +-- design-base-0.10.8.aar.sha256
        | +-- design-base-0.10.8.pom
        | +-- design-base-0.10.8.pom.md5
        | +-- design-base-0.10.8.pom.sha1
        | +-- design-base-0.10.8.pom.sha256
        | +-- design-base-0.10.8-sources.jar
        | +-- design-base-0.10.8-sources.jar.md5
        | +-- design-base-0.10.8-sources.jar.sha1
        | +-- design-base-0.10.8-sources.jar.sha256
        +-- maven-metadata.xml
```

이 외에 해야 하는 일은 없다. (이미 로컬 m2 저장소치고는 오버킬을 좀 했다.)

## 결과

### 양적 효과

Gradle 작업 수행 시간은 Gradle 대몬 프로세스 상태와 시스템 상태에 따라 현저히 달라지므로 이에 대한 정확한 측정은 어려운 면이 있다. 측정을 정확히 할 수 있었다면 아예 수치를 뽑아 봤을텐데, 빌드 속도라는 게 단일 수치이지만 그 결과를 내는 빌드라는 것이 워낙 복잡한 추계적 다변량 과정이기 때문에 정확히 통제된 통계를 내지는 못했다.

다만 경험적으로, 빌드하는 프로젝트 모듈의 `dependencies` 작업에서는 10%–25% 가량 작업 수행 시간 감소를 관측할 수 있었다. 특히 Gradle 플러그인에서 효과가 컸다. IDEA (Android Studio) 에 Sync project with Gradle 이라는 기능이 있는데 여기에서도 꽤 와닿는 효과를 누릴 수 있었다.

