Trouble ID `2023-05-26.android-studio-cannot-resolve-class-space`

# Android Studio &ldquo;Cannot resolve class Space&rdquo;

```diff
-    <Space
+    <android.widget.Space
```

Android Studio Flamingo 2022.2.1 Patch 2 를 사용하고 있는데 뭔가 이상했다. 문득 열어 본 layout XML 파일의 코드 뷰에서 Space 태그네임이 빨간색 글자로 표시되는 것이다.

이는 IDE 에서만 Space 클래스를 찾지 못하는 현상으로 빌드에는 전혀 문제가 없기 때문에 그대로 둬도 큰 문제는 없다.

**교정하는 방법은 간단하다. 위와 같이 패키지 경로를 포함한 FQN<sup>fully qualified name</sup> 을 사용한다.**

이 부분에 대해 AS 의 내부 동작을 직접 추적해 보지는 않았으나, 다른 클래스 참조 해소가 이름만으로 정상 작동하는데 Space 만 빠진 원인은 명확히 추정할 수 있다.

- **내부에 `<Space ... />` 를 포함하는 렌더링 테스트가 없음.**
- layout XML 의 태그명이 FQN 이 아닐 때 `app:` 속성<sup>attribute</sup> 때문에 편집기<sup>editor</sup>가 참조 해소를 Android 쪽으로가 아닌 AndroidX 쪽으로 하고 있음. 이는 `app:` 속성까지 반영해서 디자인 프리뷰를 그려야 하기 때문이다.
  - 예시: `android.widget.TextView` &rarr; `androidx.appcompat.widget.AppCompatTextView`
  - Space 의 compat 구현은 `androidx.legacy.widget.Space` 인데 이쪽으로 참조 해소가 되지 않으므로 빨간색 글자로 표시되는 것.
- 현재 IDE 는 디자인 프리뷰에서 참조 해소되지 않은 요소의 영역을 width, height 만 참고해서 설정하고 그 안은 비워 두는데, Space 가 정상으로 그려진 결과가 이와 같으므로, 디자인 프리뷰 사용에 큰 문제는 없음.
- 상술했듯 빌드 문제도 없음.
- 이 문제에 신경쓰는 사람은 딱히 없을 것 같다.

----

**Space 의 연원.** 예전엔 `<Space />` 를 사용하는 것이 그럭저럭 적절한 기술결정이었다. LinearLayout 과 RelativeLayout 을 계속 쌓아 나가던 시기에 여백 조정을 위해 margin 보다는 visibility, height 를 사용하도록 한 것이다.

이는 MarginLayoutParams 를 조작하는 것보다 LayoutParams 를 조작하는 것이, Kotlin 코드와 layout XML 코드 간에 분산되어 있지만 상호 검증이 안 되는 정보가 줄어드는 유형 안전<sup>type safety</sup> 관점에서 좋은 방식이기 때문이다. 반대로 Kotlin 코드 상의 논리<sup>logic</sup> 조건에 따라 LayoutParams 를 MarginLayoutParams 로 유형 배역<sup>type cast</sup>하는 코드가 선택적으로 실행된다면, 기능 시험 시나리오<sup>feature test scenario</sup>에서 놓치고 프로덕션으로 나갈 수 있는 버그의 가능성이 증가한다. 한편 태그 추가로 인해 XML 나무<sup>tree</sup> 너비<sup>width</sup>가 증가하면서 UI 렌더링 시간을 소요하는 문제는 LinearLayout/RelativeLayout 중첩으로 인해 나무 깊이<sup>depth</sup>가 높아서 생기는 문제에 비해서는 애교 수준이었고, 이미 관습적으로 사용되던 `<View />` 는 무의미하게 성능 잡아먹는 괴물이었다 ([`android.view.View`](<https://developer.android.com/reference/android/view/View>) 는 [`android.view.ViewGroup`](<https://developer.android.com/reference/android/view/ViewGroup>) 의 상위 클래스<sup>superclass</sup>이기도 하다). 이런 이유로 [`android.widget.Space`](<https://developer.android.com/reference/android/widget/Space>) 는 `<View />` 를 회피하고 성능을 향상시키는 컨벤션을 정착시키겠다는 선의까지 가세되어 API level 14 에 들어오고,  Android Support v4 Library 에도 추가된다 (`android.support.v4.widget.Space`). 이것이 2011 년에 있었던 일이다.

상황은 아주 천천히 변한다. Android 의 체감 성능이 iOS 에 비해 낮았던 부분이 실제 약점이 되어 Android 자체가 존재의 위기를 맞는다. 단말 성능이 좋아지고 CPU 클럭이 올라가도 발열만 심해지고 스로틀링으로 오히려 앱 UI 프레임레이트<sup>framerate</sup>가 더 떨어지는 현상들이 나타났기 때문이다. Android 에서는 앱을 아무리 잘 짜도 UI 가 매끄럽게 나오지 않는다는 OS 책임론의 공방이 끊이지 않았다.

이에 대응해 Android 는 ICS 이후 Project Butter 를 발표하며 말 그대로 참기름 바른 듯 매끄러운<sup>buttery smooth</sup> 모습을 보여주겠음을 표방한다. 그렇게 Jelly Bean 과 KitKat 을 거쳐, 수직동기화 시간성<sup>vsync timing</sup>, 트리플 버퍼링<sup>triple buffering</sup>, 터치 지연 감소<sup>reduced touch latency</sup>, CPU 인풋 부스트<sup>- input boost</sup> 등 온갖 부문에서 몸비틀기를 선보이면서, Android 는 적어도 게임기로도 크게 손색은 없는 수준에 올라서게 된다. 이때 만들어진 것이 [`android.view.Choreographer`](<https://developer.android.com/reference/android/view/Choreographer>) (since 16) 이다. 여기에 고도의 노력이 집약되면서, 프로파일링<sup>profiling</sup>을 통해 성능을 계량하고 대량의 데이터로 앱 UI 가 아직도 느려터진 원인이 무엇인지에 대해 질적 분석까지도 할 수 있게 될 근거가 열린다.

OS 수준에서 해 줄 수 있는 걸 모두 해 줬으니 이제 개별 앱 개발자가 새 시대의 표준을 받는 차례가 되는 걸 피할 수 없다. 그 동안 Android 생태계에는 복잡다단한 레이아웃 기반으로 작성한 커스텀 뷰 클래스<sup>custom view class</sup>가 난립하고 있었고, `fill_parent` 같은 어중간한 편의성으로 데이터 변경 때마다 뷰의 크기가 변하며 measure&ndash;layout&ndash;draw 구간에서 같은 코드가 수십 번 도는 것이 예삿일이었으며, 그 일이 뷰 위계<sup>view hierarchy</sup>를 위아래로 오가며 반복되는 동시에 클래스 기반 다형성<sup>class based polymorphism</sup>의 단점까지 뒤집어쓰느라 성능은 극악으로 치달아 있었다.

**다음 타자인 ConstraintLayout 이 나타난다.** 여태껏 레이아웃 문제의 요구사항들은 뷰 위계 내에 층층이 숨겨져 있었고 뷰 구성요소 인스턴스들이 이걸 각자 풀 수 있는 만큼만 풀며 나머지는 OOP 스타일로 대리<sup>delegate</sup>되느라 measure&ndash;layout 구간의 계산이 무의미하게 반복되었는데, ConstraintLayout 에 뷰를 전부 펼쳐 두고 레이아웃 문제를 ConstraintSet 으로 집중해 한번에 풀도록 하면 이를 획기적으로 줄일 수 있었던 것이다. 이후 ConstraintLayout 이 발달하면서 margin 계열 속성<sup>attribute</sup>으로 할 수 있는 것이 훨씬 많아지게 되었고 (`layout_goneMarginEnd` 같은 것), 이젠 XML 나무 깊이와 너비를 모두 최소화하는 것이 layout XML 작성의 기본 덕목이 되었기에, Space 는 그냥 구시대의 유물에 불과하게 되었다.

ConstraintLayout 의 효능은 실존했을 뿐만 아니라 프로파일러 기반으로 보기 좋게 입증되었다. 30 fps 를 프레임 드랍<sup>frame drop</sup> 없이 꼭 맞추려면 기존의 깊은 계층을 걷어내고 ConstraintLayout 을 써야 한다는 공식이 굳어졌고, 이는 60 fps 가 표준이며 iOS 는 적응적 24&ndash;120 fps 의 ProMotion 으로 앞서 달려나가고 있는 2023 년에도 견고하다.

----

위에서 등장한 Android Support 라이브러리와 ConstraintLayout 등은 현재 **AndroidX (Android Jetpack)** 로 성장했고, 이 중 AppCompat 라이브러리에서 유래한 후방 호환성<sup>backward compatibility</sup> 지원 분야에서는 Android 최신 API level 에 추가된 기능들을 하위 OS 버전에서도 고르게 사용할 수 있도록 Fragment, Activity, 애니메이션과 드로어블 등 광범위하게 코딩 과정 전반을 보조하고 있다. 예를 들어 Android Support v7 라이브러리에서 시작해서 지금도 AppCompat 라이브러리에 남아 있는 `ContextCompat` 과 `ResourcesCompat`, `AppCompatResources` 는 신규 API 를 구버전 OS 에서도 쓸 수 있게 지원할 뿐만 아니라 Kotlin 소스 코드에서 Android 프레임워크 API 를 사용하지 않도록 린트<sup>lint</sup> 규칙을 포함하고 있다. UI 요소에는 `AppCompatImageView`, `AppCompatTextView`, `AppCompatButton` 같은 것들이 구비되어, `AppCompatActivity` context 에서 `LayoutInflater` 가 layout XML 을 실제 객체로 만들 때 프레임워크 클래스 참조를 이런 AppCompat 위젯 클래스 참조로 자동으로 대체함으로써 compileSdk 와 minSdk 의 간극을 메워 준다.

**Space 만 빼고.** Android Support v4 라이브러리의 Space 는 현재 형식상 AndroidX Legacy 라이브러리에 포함되어 있으나 `@Deprecated` 표시되어 있다. 이 라이브러리의 `minSdk` 값은 14 이고, Space 가 API level 14 에서부터 정의되어 프레임워크에 있으므로, 실상 무용지물인 셈이다. 이 클래스가 라이브러리에서 아예 삭제되지 않은 유일한 이유는 Android Support 라이브러리 참조를 남기면 안 되기 때문이다. Jetifier 가 Android Support 라이브러리의 클래스 참조를 AndroidX 라이브러리의 클래스 참조로 일괄 변환할 때, `android.support.v4.widget.Space` 도 존재한다면 `androidx.legacy.widget.Space` 로 변환해 주게 될 것이다. 하지만 `android.widget.Space` 는 AppCompat 에 의해 변환되지 않는다.

이 규칙은 Jetifier 에만 존재하고 IDE 에는 누락되어 있다. 그저 `<Space />` 에서 `Space` 가 빨간색 글자로 표시될 뿐이다. 적당한 메시지, 예를 들어, ConstraintLayout 으로의 전반적인 이전을 권유한다거나 하는 제시되지 않고 있다.

**`<Space />` 가 존재한다는 사실은 다른 어떤 단서조차 없이 그 자체를 코드 냄새<sup>code smell</sup>로 취급해도 무방하다.** 정말로. 기술론적<sup>technological</sup> 맥락에서 이렇게 납작하게 다뤄도 무방하며 거의 엄밀한 지표를 얻을 수 있는 건 상당히 드물고 귀한 일이다. 세상 거의 모든 기술적<sup>technical</sup> 실체가 다른 기술<sup>technique</sup>에 각자 그 존재를 의존하며, 경쟁적으로 가치를 입증하거나 그렇지 못하면 소멸하는 것이 현실이기 때문이다. 도태된 기술은 흔적조차 남지 않는 경우가 많고, 도태될 것 같은 기술은 제거하기 힘들기에, 코드 냄새같은 지표를 정하는 작업은 세상 모든 패션의 옷에서 도안을 틀렸거나 마감이 잘못된 옷을 골라내는 것과 같이 대체로 엄밀하지 못하며 상당히 자주 고도의 이론을 요구하는 일이다. 자, 이 아주 단순명료하게 잘못되었으며 완벽하게 도태될 요건을 갖췄지만 세상에 남아 있는 `<Space />` 는 그럴 필요가 없다. **그리고 AS 에서 `<Space />` 는 간편하게도 아예 그 존재가 지워져 있다.**

바로 이 부분에서, 솔직히, 이제 와서 Space 가 고작 layout XML 코드 편집기 규칙에서 누락되어 있다는 사실을 두고 이렇게 긴 글을 쓰는 것이 꽤나 어리석어 보일 수 있다. 그러나 이 문제에 내가 소프트웨어 개발 문화라고 부르고 싶은 것의 일부분이 투영되어 있으므로 조금 의견을 남겨 보고자 한다.

우리가 기술자들<sup>technicians</sup> 사회에서 적잖이 볼 수 있는 현상으로, 이런 것들을 다루는 상황을 굉장히 불편하고 어색해하는 사람들의 모습이 있다. 그도 그럴 것이, Space 라고? 이제 “그건 그냥 틀린 거잖아.” 자, 일상은 해야만 하는 일로 이루어져 있고, 우린 그 일을 하며 살아야 한다는 부분은 외면할 수 없다. 특히 소프트웨어 엔지니어링에서는 번성과 도태의 이터레이션이 유독 빠르기에, 기술의 실체가 사람의 노하우로 개인들의 과업수행 역량에 강결합되어 있다는 것을 굳이 지목하는 이야기는 간혹 아주 금기시되는 경우까지도 볼 수 있다. **그럴 시간에 그걸 자동화하라고.** 소프트웨어의 상당량은 다른 소프트웨어 위에 쌓아올려져 있기에 생산기반을 재구축하는 데에 물질적 비용이 거의 들지 않는 편이며, 고도로 발달한 기술론에서 생산체계를 따라서 숙련공이 기예를 발휘하는 인본주의적 미학은 그 어떤 산업 분야에서보다도 금방 도태되고 만다. 섬세하고 감각적인 소수 인간보다는, 융통성이 없더라도 늙지도 않고 교육 비용도 없어서 규모가변성이 좋은 기계의 대량생산에 다수 수요와 공급을 맞춰 “파이 자체를 늘리는” 일이 성공하면, 그 시장규모를 선점하여 번성한 참여자는 기존의 나머지 시장 참여자를 압도하고 필요에 따라 이들을 아예 매수해 버릴 수도 있게 된다. 컴퓨터와 인터넷을 기반으로 하는 산업계는 현 시점에 특히 이 흐름을 선도하는 핵심에 서 있다. 일개 기술자들이 무슨 이의를 제기할 수 있겠나. 기존 기술 기반에서 장인정신으로 정교하게 좋은 것을 조금씩 만들어내는 인간 중심의 예술은, **명품시장에 편입되기라도 하지 않는다면** 세월의 흐름 속에서 **식자공 선배님들을 따라** 퍽 거친 신세를 겪게 될 것이다.

그럼에도 불구하고 나는 이 문화적인 부분을 지목하며 이쪽에 공동체적 목표설정<sup>goalsetting</sup>의 정책으로써 보다 장기적으로 우월하며<sup>dominant</sup> 진화적으로 안정하기도 한 전략<sup>ESS</sup>의 방향성이 있음을 주장하려는 것이다.

**고성장 시대의 모든 것은 꿈을 먹고 자란다.** 첨단기술이 고부가가치 산업으로 사회에서 빛을 보고, 산업 구조가 개편되며 새로운 사업들이 자리를 잡을 때, 고성장 시대의 흐름에 올라타서 잉여가치를 이윤으로 남겨 자산으로 확보할 수 있다는 꿈은 (그게 생존권이든 구매력이든 예외는 없음), 인간 집단의 의식을 숙주로 삼는 전염병처럼 빠르게 퍼져 나간다. 그러나 &ldquo;모두 부자 되세요&rdquo; 라는 캐치프레이즈의 공허함만큼, 구조적으로 이 꿈이 모두에게 달성가능하지 않을 것임은 너무도 자명한데, 다른 한편 그 꿈은 마냥 자유 의지와 선택의 영역에 있진 않다. 이 같은 시대에 인간 노동의 일부 형태는 이미 필연적으로 연속성의 미래를 잃기 때문이다. 그래서, **사실 이 시기의 꿈과 공포는 엄밀히 구분되지 않는다.** 세계사적으로는, 산업 혁명의 표현형이 국가의 도시화율에 한정되지 않고, 기술력에 기반한 제국의 식민지 팽창이나 전쟁이 나타나기도 해 왔다. 이런 경험의 일련은 집단무의식의 영역에 성공도식이나 실패도식으로 자리잡아, 문화적 유전자<sup>gene</sup>인 밈<sup>meme</sup>으로 민족&middot;문화권 또는 국가의 정체성을 수십 년 넘게 형성해 왔다 (인문학과 기초과학에 대한 정책적 선행 투자 성향 역시 이에 포함됨). 하여간에 꿈과 공포의 치열한 경쟁은 첨단산업의 생산능력에 따라 언젠가 한계를 맞고, **머지않아 저성장 시대가 찾아오면 각자의 꿈은 심판대에 올라 지극히 냉혹한 판결을 받는다.** 그 결과로 생겨나는 도시빈민은 현실 사회문제이다. 이제 도시는 더 이상 단순히 희망을 섭취하고 절망을 배설할 수 없다. 시스템은 이들을 다시 생산체계에 편입하려고 노력할 시대정신적 책무를 부여받는다. 그렇기에 저성장 시대에는 서로를 인정하고 공존을 시도하며 고성장 시대의 성취가 낙수효과적으로 내부에 잘 흐르도록 하는 방법에 대해 논의가 발생해야 하는 것이다. 그것이 결여된다면 청소년 세대는 중장년 세대의 질서를 더 이상 떠받치는 것을 거부하고 혁명을 일으키거나 이민을 가 버리거나 그냥 무기력하게 적당히 대충 살 것이고, 도시빈민의 문제는 더 큰 현실 사회문제가 되기 때문이다.

소프트웨어 개발에서도 구조적으로 비슷한 일이 벌어진다.

과거의 실패를 단죄하는 것은 쉽고 명쾌하다.

과거의 실패를 문제의 본질을 외면할 수 있게 하는 상품들

TODO: Dalvik 의 옛날 HotSpot VM Parallel GC 같은 더블 STW GC 와 ART 에서 개선된 CMS GC

TODO: 어차피 120 fps 할 거라면 Jetpack Compose

