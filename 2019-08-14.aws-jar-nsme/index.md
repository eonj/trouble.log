Trouble ID `2019-08-14.aws-jar-nsme`

# AWS Java 런타임에서 NoSuchMethodError 의 원인 찾기

## LOC = 3

- NoSuchMethodError
- JAR 패키지 또는 Java SE 클래스파일 유래 패키지가 런타임의 변화에 따라 또는 빌드 툴체인 러너 변화에 따라 따라 다르게 동작할 때
- 같은 이름 경로의 클래스가 다른 내용으로 정의된 클래스파일이 중복으로 존재할 수 있다.

## LOC &gt; 3

````
19/08/14 13:30:52 WARN TaskSetManager: Lost task 3.0 in stage 0.0 (TID 3, localhost, executor driver): java.lang.NoSuchMethodError: com.amazonaws.http.HttpResponse.getHttpRequest()Lcom/amazonaws/thirdparty/apache/http/client/methods/HttpRequestBase;
````

NoSuchMethodError 라는 희귀한 오류 이야기.

웬만큼 Java 를 오래 쓰는 프로그래머라도 NoClassDefFoundError 나 NoSuchMethodError 를 만날 일은 거의 없다. 많은 사람들이 저것들을 ClassNotFoundException 이나 NoSuchMethodException 과 헷갈린다는 사실이 그에 대한 강력한 증거로 제시될 수 있을 것이다.

ClassNotFoundException 과 NoSuchMethodException 은 reflection 에서 발생하는 예외 유형이라서 reflection code 를 작성해 봤다면 언젠가 이 이름을 써 봤을 것이다. NoClassDefFoundError 의 경우 클래스패스에 클래스파일이나 JAR 파일이 누락되었을 때 발생한다. 그렇다면 NoSuchMethodError 는? 클래스패스에서 알맞은 클래스까지는 찾았는데, 호출하려 하는 메소드가 없다는 뜻이 된다.

이 문제는 특히 AWS Java 런타임에서 돌아가야 하는 ZIP 패키지 빌드에서 발생했기 때문에 독특하게 전개되었다. 같은 빌드 결과물인데 로컬에서 돌리면 잘 동작하고 한번 더 ZIP 파일로 말아서 AWS Lambda 에 올리면 NoSuchMethodError 만 내면서 동작을 하지 않는 것이다. 도대체 왜?

나는 장막을 들추어 미래를 엿보았&hellip; 아니 빌드 결과물을 까서 들여다보았다. 기본적으로 ZIP 패키지 빌드를 만들 수 있는 구조로 빌드 아웃풋을 만들게 되어 있다. `bin/` 폴더와 `lib/` 폴더가 있고, `bin/` 폴더의 엔트리포인트를 실행하면서 `lib/` 폴더의 JAR 파일을 모두 `-cp` 로 끌어다 쓰는 방식이다. ZIP 패키지를 만들면 이 `bin/` 과  `lib/` 을 그대로 ZIP 빌드 안에 넣게 된다.

`lib/` 폴더 안에 있는 `aws-sdk-java-core-1.1.466.jar` 파일을 `unzip` 해서 클래스파일을 얻고, 이 중 `com.amazonaws.http` 패키지의 `HttpResponse` 클래스파일을 `javap` 로 다시 들여다보았다.

```
$ javap -p HttpResponse
Warning: Binary file HttpResponse contains com.amazonaws.http.HttpResponse
Compiled from "HttpResponse.java"
public class com.amazonaws.http.HttpResponse {
  private final com.amazonaws.Request<?> request;
  private final org.apache.http.client.methods.HttpRequestBase httpRequest;
  private java.lang.String statusTest;
  private int statusCode;
  private java.io.InputStream content;
  private java.util.Map<java.lang.String, java.lang.String> headers;
  private org.apache.http.protocol.HttpContext context;
  public com.amazonaws.http.HttpResponse(com.amazonaws.Request<?>, org.apache.http.client.methods.HttpRequestBase);
  public com.amazonaws.http.HttpResponse(com.amazonaws.Request<?>, org.apache.http.client.methods.HttpRequestBase, org.apache.http.protocol.HttpContext);
  public com.amazonaws.Request<?> getRequest();
  public org.apache.http.client.methods.HttpRequestBase getHttpRequest();
  public java.util.Map<java.lang.String, java.lang.String> getHeaders();
  public java.lang.String getHeader(java.lang.String);
  public void addHeader(java.lang.String, java.lang.String);
  public void setContent(java.io.InputStream);
  public void setStatusText(java.lang.String);
  public java.lang.String getStatusText();
  public void setStatusCode(int);
  public int getStatusCode();
  public long getCRC32Checksum();
}
```

문제가 보이는가? 저 위의 NoSuchMethodError 에러 메시지는 `com.amazonaws.http.HttpResponse` 클래스에 대해서 다른 코드가 찾고 있는 `getHttpRequest` 메소드의 반환유형 시그니처가 `Lcom/amazonaws/thirdparty/apache/http/client/methods/HttpRequestBase;` 일 것을 요구하고 있다. 그러나 실제 `aws-sdk-java-core` 에 들어있는 `com.amazonaws.http.HttpResponse` 클래스의 `getHttpRequest` 메소드는 반환유형 시그니처가 `Lorg/apache/http/client/methods/HttpRequestBase;` 이다!

빌드스크립트를 보면 이런 의존성 선언들이 있는데 문제가 되는 건 이 두 개였다. `lib/` 폴더를 보면 `aws-sdk-java-bundle-1.1.271.jar` 와 `aws-sdk-java-core-1.1.466.jar` 가 동시에 있었기 때문이다.

- `com.amazonaws:aws-sdk-java-bundle:1.11.271`
- `com.redislabs:spark-redis:2.4.0`
  - `com.amazonaws:aws-sdk-java-dynamodb:1.1.466`
    - `com.amazonaws:aws-sdk-java-core:1.1.466`

`aws-sdk-java-bundle` 은 `aws-sdk-java-core` 를 사용하지만 서드파티 라이브러리 의존성에 대해 relocate 처리하는 작업을 거친 것이다. (Maven Shade Plugin 을 사용한다.) 그래서 똑같은 `com.amazonaws.http.HttpResponse` 클래스가 있지만, `aws-sdk-java-core` 는 `org.apache.httpcomponents:httpclient` 에 Maven 의존성이 있고, `aws-sdk-java-bundle` 은 이 라이브러리의 `org.apache.http` 패키지를 `com.amazonaws.thidparty.apache.http` 패키지로 relocate 하여, 원래 보유한 클래스처럼 Maven 의존성 없이 갖고 있게 된다.

무슨 짓을 해도 중복된 클래스에 대해서는 `aws-sdk-java-core` 보다는 `aws-sdk-java-bundle` 쪽 클래스파일이 사용되어 NoSuchMethodError 가 발생했고, 결국 빌드스크립트에서 추이적 의존성으로 들어온 `aws-sdk-java-core` 를 제외하는 방식으로 해결하게 되었다. 이 방식은 빌드 툴마다 다르므로 굳이 기입하지 않는다.

## LOC = 3 (?)

AWS 에 업로드하는 ZIP 패키지 빌드 파일의 구성은 파일시스템의 구성과 비슷한 것이고, AWS Java 런타임은 어떤 Java 런타임에서든 발생할 수 있는 문제를 똑같이 갖고 있을 뿐이다. AWS 문서는 클래스패스에 있는 JAR 파일을 알파벳순으로 로드하여 먼저 로드한 것을 우선으로 사용한다고 명시해 놓기까지 했다. <https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/java-package.html>

도대체 런타임에 NoSuchMethodError 가 나는 코드가 왜 빌드된단 말인가? Java 에서 클래스 중복과 메소드 오버로딩 유형 매칭 문제는 유형체계에서 보는 부분이다. Java 의 유형체계는 제네릭스 등 웬만한 것에 대해 런타임보다 컴파일타임에 더 빡빡하게 검증하는 컴파일러를 갖고 있지만, 오직 이 부분에 대해서는 예외이다. 클래스패스에 찾는 클래스가 없으면 NoClassDefFoundError 가 발생하지만, 찾는 클래스가 여러 개라면 그냥 조용히 하나를 쓴다니? JVM 구현체를 포함하는 Java 런타임은 대부분 이렇게 구현되어 있다. 이는 클래스파일 바이너리를 만드는 컴파일타임에, 소스패스의 소스코드에 대해서는 그냥 넘어가지 않는 문제이다. 컴파일러에 JAR 파일을 똑같이 클래스패스로 넘기면 마찬가지로 클래스 중복 문제는 조용히 묻히게 된다. 

나는 이렇게 대강 넘어가는 부분을 Java SE 명세가 소프트웨어 결함을 지속적으로 유발하는 부분이라 생각한다. Java 런타임이 지정된 클래스패스에서 유형 오류를 일으킬지 미리 점검하는 도구를 미리 갖추는 것은 OpenJDK 와 SDK 들의 덕목이 되어야 한다. JAR 아카이브 파일과 파일시스템의 존재는 실재하는 대상이고 거의 모든 시스템에서 표준이 되었으며 클래스파일은 유형 오류를 잡을 수 있는 정보를 보존하고 있기 때문에, 이 자체는 크게 어려운 문제가 아니다. 현재 Java 언어 명세와 JVM 명세로 나뉜 Java SE 명세 구조에서는 의무사항으로 존재하기 어려운 내용이지만, 적어도 미래의 어떤 언어는 이런 부분에 결함을 예방하는 명세를 갖추기를 바란다. 물론 이런 내용이 자꾸 명세에 들어가면 명세가 아주 장황해지고 과거의 문제를 없애 버릴 대안을 오히려 고착시키는 효과도 있는데, 이에 대해서는 어느 쪽이 맞다고 딱 잘라 말하기 어려운 면이 있다. 적어도 C++ 와 W3C 문서들은 그렇게 하고 있는데, 명세를 읽지 않는 실무자가 너무 많은 분야의 대표주자이기 때문이다.