Trouble ID `2020-06-25.reactivex-conversion-overview`

# (내가 보려고 만든) ReactiveX 유형 변환

RxJava 2 에 등장하는 클래스 유형들에 대해 상호 변환이 일어나는 메소드를 모아서 이것으로 그림을 그려 봤음. 가끔은 필요할 때 기억이 안 나서&hellip; 그리고 가끔은 실수로 유형 변환을 할 때가 있어서 외워 두려고 만들었다.

여기에 등장하는 클래스 유형에는 `Observable<T>` 과 그 파생형인 `Single<T>`, `Maybe<T>`, `Completable` 이 있다. Empty, Throw, Never 같은 것들은 ReactiveX 에서는 연산자인 동시에 유형으로 취급해야 하지만, RxJava 2 에서는 이것들까지 유형으로 두지는 않았고 정적 메소드로 기존 클래스 유형을 만드는 팩토리 메소드처럼 쓸 수 있게 만들어 뒀기에 따로 빼 두었다. 그리고 `Subject` 나 `Flowable` 은 굳이 다루지 않았다.

ReactiveX 레퍼런스를 보면 `First`, `Last`, `ElementAt` 같은 것들은 연산자로 소개되고 있는데 이 그림을 보면 RxJava 2 의 경우 이 연산자들이 `Observable<T>` 유형을 `Single<T>` 로 변환하는 기능을 동시에 갖고 있음을 알 수 있다. 또한 `Observable` 을 `Maybe` 와 `Single` 중 어느 쪽으로 변환해야 하는지 가끔 헷갈리게 되기도 하기에, 웬만한 건 그림 그리느라 외워졌지만, 앞으로도 이걸 참고할 일이 가끔 있을 것 같다.

![20200625.reactivex-types-and-conversion-methods-in-rxjava-2.300ppi](<./20200625.reactivex-types-and-conversion-methods-in-rxjava-2.300ppi.png>)

Download (PNG): [300 ppi](<./20200625.reactivex-types-and-conversion-methods-in-rxjava-2.300ppi.png>) / [1200 ppi](<./20200625.reactivex-types-and-conversion-methods-in-rxjava-2.1200ppi.png>) / [3000 ppi](<./20200625.reactivex-types-and-conversion-methods-in-rxjava-2.3000ppi.png>)

이 그림을 그리는 건 잡생각을 아주 많이 하는 과정이기도 했다. `Take` 연산은, 만약 Java 에서 제네릭 유형 인자로 값을 쓸 수 있었다면 이 결과를 `Single` 처럼 별도 유형으로 만들 수 있지 않았을까. `Map` 이나 `Zip` 같은 연산자들은 `Observable<T>` 와 `Observable<U>` 간의 변환이니 그냥 넘어간다 하더라도, `Filter` 는 값에 대한 제약을 만들어 주고 `Delay` 나 `Debounce`/`Throttle`/`Timeout` 같은 건 시간에 대한 제약을 만들어 주는데, 이걸 유용하게 쓸 수 있지 않을까. 의존 유형에서 유형의 일부가 미래에 구체화될 수 있으려면 유형 체계가 어떻게 동작해야 하나, 아니면 린트 레벨에서라도 구분된다면 쓸모있게 만들 수 있지 않을까. 별로 내가 하고 싶지는 않고&hellip;

## ReactiveX 잡설

(Update @ Apr 2021)

ReactiveX 에 대해 찾아 보면 그 유래는 원래 .NET 에서 쓰려고 Microsoft 에서 만든 것이고&hellip; 이런 얘기가 나온다. 개념적으로 들어가면 함수형 반작용적 프로그래밍이라는 (이하 FRP) 프로그래밍 패러다임까지 나오고, 그런 걸 계속 읽다 보면 ReactiveX 라는 게 뭔가 기존에 있던 것과 아주 달라 보일 수 있다. 하지만 대충 기존에 있던 것들과 굉장히 다르게 볼 필요는 없는 것이고, 그럼에도 불구하고 ReactiveX 사용에 조금 주의를 요하는 부분이 생기는데, 잠시 이 부분을 짚어 보자.

원래 Java 나 C\# 의 컬렉션<sup>collection</sup> 내지는 C++ 의 컨테이너<sup>container</sup> 등지에서 쓰는 반복자<sup>iterator</sup>라는 개념이 있다. 이걸 본래 절차적인 문제풀이 과정에서 문법 설탕<sup>syntactic sugar</sup>을 일정하게 만드는 용도로나 사용하는 것이 고작이었다. ReactiveX 는 이것이 비동기적으로 작동할 경우 관찰자 패턴<sup>Observer pattern</sup>과 아주 비슷하다는 점에 착안하여, 제네릭 유형 `Observable<T>` 을 만들고 스케줄러<sup>scheduler</sup>와 람다<sup>lambda</sup>를 도입해 비동기 데이터 흐름을 처리하는 일을 숨쉬듯 쉽게 다룰 수 있게 만들어 둔 것이다. ReactiveX 홈페이지 [reactivex.io](<http://reactivex.io>) 에 보면 실제로 ReactiveX 를 소개하는 모토를 이렇게 적어 두고 있다.

> The Observer pattern done right (옳게 된 관찰자 패턴)

ReactiveX 는 범언어적 표준으로 자리잡았고 FRP 역시 기존의 프로그래밍 패러다임과 공존하게 되었다. 그 과정에서 ReactiveX 에 대한 인식은, FRP 라는 의도에 걸맞게 비동기 데이터를 연속적으로 변환하는 함수 체인을 다루는 프레임워크라기보다는, 비동기적으로 데이터를 제공하는 부분의 인터페이스로 자리잡게 되었다. (이 분야의 대표주자는 [Square](<https://squareup.com/>) 에서 만든 [Retrofit](<https://square.github.io/retrofit/>) 인데, 자사 HTTP 클라이언트 라이브러리인 [OkHttp](<https://square.github.io/okhttp/>) 에 ReactiveX 를 적용해 엄청난 편의를 제공하는 유틸리티 라이브러리이며 Android 에서는 표준 그 자체이다.) 보편적 인식이 이런 구도로 형성되고 있다는 증거는 요즘 ReactiveX 의 대체재로 Java 의 스트림<sup>stream</sup>이나 Kotlin 의 코루틴<sup>coroutines</sup>이 꼽히고 있다는 사실에서도 찾을 수 있다.

한편 이런 현상은 결국 현실에서 비동기 시스템의 핵심 문제가 대체로 시간을 정밀하게 다루는 것보다는 데이터를 다른 데이터로 변환하는 데 평소보다 많이 드는 시간에 있다는 사실을 시사하기도 한다. 일례로 C\# 과 Python 에는 발생자<sup>generator</sup>를 작성할 때 사용하는 `yield` 키워드가 있어서 루틴의 실행이 중단되었다 재개될 수 있는 경우를 정의해 두고 있었고, 특히 C\# 에서는 Unity 에서 사용하는 `StartCoroutine` 이라는 유사 코루틴 컨벤션이 일찍이 C\# 반복자 메소드 반환 유형인 `IEnumerable`/`IEnumerator` 중 `IEnumerator` 를 기반으로 하고 있었기 때문에 이 사실이 좀 더 알기 쉬운 형태로 존재하는 편이다. 시간이 드는 부분 간에는 선후 관계가 있고, 이 관계를 세어 나가는 것은 반복자로 간주할 수 있으며, 데이터의 생산자와 소비자를 비동기적으로 처리해도 된다고 스레드를 분리하기 쉽게 해 둔 것이 ReactiveX 이다. 그래서 ReactiveX 는 일부 비동기 문제에서 Unity 코루틴과 (그리고 JavaScript 의 Promise 와) 비슷한 도구가 된다.

이런 관점에서, ReactiveX 에서 아쉬운 점으로 흔히 성능을 꼽는데, 이게 ReactiveX 가 편하지만 성능이 좋지 않다는 식으로 와전되어 집단적 오해를 조장하고 있는 것은 개인적으로 안타까운 일이다. ReactiveX 는 특정한 데이터 생산자로부터 어떤 유형의 값이 반복적으로 발생할 때 이를 소비하는 방법으로 고안되었고, 함수형의 느긋한 평가법 관점을 도입하여 스케줄러를 도입하고 스레드를 분리할 수 있게 한 것이다. 즉 ReactiveX 는 자료가 충분히 많이 준비되어 있을 때 이를 처리하는 데 있어 스레드를 효율적으로 쓰는 방법을 제공하는 일환으로 부작용을 회피하는 함수형의 패턴을 권고하는 것이 아니다. 이 방식으로 만들어진 것은 명백히 Java 스트림이나 Kotlin 코루틴이다.

물론 ReactiveX 에서도 `FlatMap` 연산자를 써서 병렬 처리를 하는 코드를 만들 수 있다. `FlatMap` 은 본래 컬렉션의 각 항목을 컬렉션으로 사상해서 이 결과를 다시 하나의 컬렉션으로 합치는 연산인데, ReactiveX 에서는 `Observable<T>` 에 대해 `(T) -> Observable<U>` 함수를 `FlatMap` 연산자에 적용하면 `Observable<U>` 가 되는 방식으로 작동한다. `Observable<T>` 에 스케줄러를 적용하면 단일 스레드를 사용하겠지만, `Observable<U>` 에 스케줄러를 적용하면 `(T) -> U` 작업이 모두 다른 스레드를 사용할 것이다 ([SO Question \#35425832](<https://stackoverflow.com/questions/35425832/rxjava-and-parallel-execution-of-observer-code>)). 그러나 이건 보통 우리가 원하는 게 아니다. 우리는 스레드를 몇 개만 굴려서 이 스레드들이 알아서 이미 쌓여 있는 일감을 가져가도록 하고 싶은 것이다.

ReactiveX 의 성능에 아쉬운 점을 구체적으로 하자면 ReactiveX 는 성능을 부스팅하는 도구가 되지 못한다고 써야 할 것이다. ReactiveX 의 디자인으로 잘 풀 수 있는 문제에서 ReactiveX 는 여전히 멀쩡한 성능을 낸다. 이건 진짜 잡설인데, 나는 발생자로 만든 유사 코루틴도 코루틴이라고 부르면 안된다고 생각하는 편이고, `yield` 조차 변변찮은 Kotlin 코루틴의 스레드풀과 함수형 태스크 러너를 코루틴이라고 부르는 건 거의 사기라고 느끼며, 이런 식으로 ReactiveX 에 대한 오해까지 꾸준히 퍼지는 상황을 달갑게 볼 수가 없다. 물론 Kotlin 코루틴은 정말 괜찮은 도구이고 함수형 프로그래밍의 장점을 ReactiveX 와 정반대에서 실체화해 준다. 그런데 요즘의 오해는 ReactiveX 를 쓰는 사람들의 불안을 자극하고 Kotlin 코루틴이 그 인기에 편승하는 식으로 작동하니, Kotlin 코루틴이 별 하자 없이 ReactiveX 를 대체할 수 있는 것처럼 말하는 사람이 생겨날까 조금 걱정이다.
