Trouble ID `2025-04-09.an-octet-aspect-to-ietf-rfc-9110`

# 옥텟 규칙으로 본 IETF RFC 9110 &ldquo;HTTP Semantics&rdquo;

1 비트<sup>bit</sup>는 크기 2인 집합에서 자유롭게 하나를 추출한 결과를 표현할 수 있는 정보량의 단위이고, 이진수<sup>binary number</sup> 기수법으로 된 수 표현을 구성하는 이진 자리숫자<sup>binary digit</sup>이기도 하다. 디지털 컴퓨터는 그 시작부터 전기신호의 있음과 없음을 1과 0으로 봤기에 비트라는 단위를 쓰는 건 자연스러웠다. 컴퓨터 시스템에서 그걸 몇 개 묶은 것을 일정한 연산과 입출력 단위로 삼아서, 한입에 덥석 &lsquo;깨물&rsquo; 했다고 해서 바이트<sup>byte</sup>라고 부른 것은 꽤 오래 된 일이지만, 그게 8 비트라는 뜻으로 상식 선에서 고정된 것은 한참 나중 일이다. 몇 바이트 정도를 또 묶어서 메모리 주소 크기를 워드<sup>word</sup>라고 (32 비트 시스템 &rarr; 1 워드 = 32 비트) 부른다거나, 4 비트는 한입의 절반쯤 &lsquo;깔짝&rsquo; 했다고 니블<sup>nibble</sup>이라고 부르는 방법이 널리 퍼진 건 또 그 다음이다. 여기에서 영어에서 bit 라는 단어가 bite 에서 왔다는 어원까지 가면 뭐 좀 재밌는 얘기가 될 것이다.

컴퓨터 네트워크에 대한 연구는 그보다 오래됐다. 아니, 애초에 TCP/IP 의 대성공 때문에 &lsquo;1 바이트 = 8 비트&rsquo; 인 세상이 된 셈이다. 바이트나 워드가 6 비트, 12 비트, 36 비트, 60 비트 정도 하는 메인프레임이 사실상 증발했기 때문이다. 사람이 메인프레임 앞에서 입출력 형식에 맞는 변환에 개입해야 한다는 비용을 도대체 어떤 사회에서 감당할 수 있겠는가? TCP/IP 가 만든 세상에서는, 단대단 전용회선을 손으로 바꿔 꽂아야 한다는 회선 교환<sup>circuit-switching</sup> 없이, 전송해야 하는 내용을 한 회선에서 받으면 잠시 대기시켜 뒀다가 가능할 때 상대편 회선으로 축적 전달<sup>store-and-forward</sup>하면 되는데 말이다. 이게 메시지 교환<sup>message switching</sup>에서 발전한 패킷 교환<sup>packet switching</sup>의 기본 개념이자, 패킷의 기술론이 사회에 자리잡게 된 이유였던 것이다. (그리고 OSI 모형에서 엄격하게는 L3 PDU 만 패킷이라고 부르는 까닭이기도 하다.)

그래서 TCP/IP 가 모두를 위한 표준 중간 계층에서 쓸 단위로 길이 8 짜리 비트열을 먼저 제시했고, 나중에 그게 1 바이트가 됐다는 배경 설명이었는데, 쓸데없이 좀 길게 풀어 봤다. 아무튼 이런 이유로 이쪽 규격서에서는 길이 8 짜리 비트열을 옥텟<sup>octet</sup>이라고 부르는 관행이 있다. HTTP 는 TCP/IP 위에서 작동하고, TCP/IP 이래로 대부분의 선배 프로토콜들이 그랬듯 &lsquo;1 바이트 = 8 비트&rsquo; 라는 등식의 덕을 많이 본 편이다. 하지만 딱 거기까지. 그래서 그 8 비트가 의미하는 바가 뭔데? 이걸 까 보면 웬만하면 ASCII 텍스트이고 가끔 안 그런 다국어 값도 들어가기 때문에, 2025 년의 상식이라면 그냥 UTF-8 이라고 생각할 수 있겠지만, 만약 그렇다면 이 글감이 머리채 잡혀 세상에 나올 일 자체가 없었을 것이다. 이제 오늘의 주제로 들어갈 때다.

<center style="letter-spacing: .2em;">&ndash;&middot;&ndash;&middot;</center>

**IETF RFC 9110 은 흔히 REST 의미론이라고 하는 부분&mdash;즉 HTTP 요청의 메서드<sup>method</sup>가 어떤 자원<sup>resource</sup>을 조회하거나 어떤 식별자에 상응하는 자원에 영향을 주는 범위와 방식, 그 결과로 반환되는 응답의 상태 부호<sup>status code</sup> 같은 것들&mdash;을 촘촘히 다룬다.** 이런 작업은 HTTP/2 가 너무나도 당연하게 쓰이게 된 시기에 와서 더 중요해졌는데, HTTP/1.1 과 HTTP/2 가 문법적 외관만 다르고 **동등한 의미론을 지니는** 규격이라는 사실이, **규범적<sup>normative</sup> 조항들에 정의된 것들로부터 도출되도록** 할 필요가 있었기 때문이다. 이런 포괄적인 작업이 기존에 수행되었던 것은 2014 년에 즈음해 있었던 일로 그 성과는 [7230 *Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing*](<https://datatracker.ietf.org/doc/html/rfc7230>), [7231 *Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content*](<https://datatracker.ietf.org/doc/html/rfc7231>) 에서 7232 *Conditional Requests*, 7233 *Range Requests*, 7234 *Caching*, 7235 *Authentication* 까지 이어지는 일련의 IETF RFC 로 발행되었던 바 있다. 2022 년 6 월에는 [9110 *HTTP Semantics*](<https://datatracker.ietf.org/doc/html/rfc9110>), [9111 *HTTP Caching*](<https://datatracker.ietf.org/doc/html/rfc9111>), 그리고 [9112 *HTTP/1.1*](<https://datatracker.ietf.org/doc/html/rfc9112>) 이라는 IETF RFC 가 발행되고 최종적으로 IETF 표준 트랙에서 IS (Internet Standard) 지위를 인정받아서 현재에 이르고 있다. 한편 [9113 *HTTP/2*](<https://datatracker.ietf.org/doc/html/rfc9113>) 의 표준 트랙 지위는 아직 PS (Proposed Standard) 이다. 당연한 일이다.

한편 이렇게, 어떤 문법적 외관을 따를지&mdash;즉 TCP 나 TLS 같은 접속지향적<sup>connection-oriented</sup> 기반을 이용해 Telnet 처럼 텍스트 라인을 주고받을지, 또는 그 위에서 한번 더 추상화된 스트림<sup>stream</sup>을 만들지 같은 것들&mdash;에 무관하게, HTTP 의미론을 다루는 **IETF RFC 9110 에서 규정하고 있는 것은 또 하나 있다.** 그것은 바로, **HTTP 의미론을 구현하고 있는 HTTP/1.1 이나 HTTP/2 같은 문법적 환경이 어떤 확장이나 보충적인 값에** (`text/plain` 이든 `application/octet-stream` 이든, 또는 `X-` 로 시작하는 헤더 같은 것들이든) **충분히 열려 있도록 정의되어 있을 때, 이것들에 다시 한번 어떤 적절한 제약을 적용하여, HTTP 의미론에서 필요한 값을 얻어내기에 충분한 것으로 만드는 것이다.** 이는 단순히 &ldquo;어떤 값<sup>value</sup>/행동<sup>action</sup>은 어떤 기능적 의미를 갖고, 이는 어떤 선행조건에서 유효하며 어떤 후행조건을 보장한다.&rdquo; 같은 체계적 규범조항과는 구분된다. 왜냐면, 제약된 의미론과 문법적으로 제약된 도메인 간의 관계는 필연적이지 않기 때문이다. 예를 들어 Section 10.1.5 를 보자.

> **[10.1.5.](<https://datatracker.ietf.org/doc/html/rfc9110#section-10.1.5>) User-Agent**
>
> The "User-Agent" header field contains information about the user agent originating the request, which is often used by servers to help identify the scope of reported interoperability problems, to work around or tailor responses to avoid particular user agent limitations, and for analytics regarding browser or operating system use. *A user agent **SHOULD** send a User-Agent header field in each request unless specifically configured not to do so.*
>
> ```abnf9110
> User-Agent = product *( RWS ( product / comment ) )
> ```
>
> The User-Agent field value consists of one or more product identifiers, each followed by zero or more comments (<u>Section 5.6.5</u>), which together identify the user agent software and its significant subproducts. By convention, the product identifiers are listed in decreasing order of their significance for identifying the user agent software. Each product identifier consists of a name and optional version.
>
> ```abnf9110
> product         = token ["/" product-version]
> product-version = token
> ```
>
> A sender **SHOULD** limit generated product identifiers to what is necessary to identify the product; a sender **MUST NOT** generate advertising or other nonessential information within the product identifier. A sender **SHOULD NOT** generate information in <u>product-version</u> that is not a version identifier (i.e., successive versions of the same product name ought to differ only in the product-version portion of the product identifier).
>
> Example:
>
> ```http-message
> User-Agent: CERN-LineMode/2.15 libwww/2.17b3
> ```
>
> A user agent **SHOULD NOT** generate a User-Agent header field containing needlessly fine-grained detail and **SHOULD** limit the addition of subproducts by third parties. Overly long and detailed User-Agent field values increase request latency and the risk of a user being identified against their wishes ("fingerprinting").
>
> Likewise, implementations are encouraged not to use the product tokens of other implementations in order to declare compatibility with them, as this circumvents the purpose of the field. If a user agent masquerades as a different user agent, recipients can assume that the user intentionally desires to see responses tailored for that identified user agent, even if they might not work as well for the actual user agent being used.
>
> (*emphasis* mine, <u>hyperlinks</u> removed.)

미안하지만 여기에서 **의미론적으로 규범성을 갖는 문장은 하나뿐이다.**

> *A user agent **SHOULD** send a User-Agent header field in each request &hellip;*

나머지는 `User-Agent: Mozilla/5.0 (compatible already, also marvelous and wishfully the world best; libjpeg/6b @ 1998-03-27) MyNewProduct/v0.2025.9999-release` 이런 짓 좀 UA 문자열에다가 하지 말라는 얘기다. 그러나 만약 적법한 User-Agent 문자열을 직접 만들어야 한다면 우리는 뭘 할 수 있겠는가?

물론 ABNF 문법을 잘 보면 될 일이다. 하지만 치트시트 하나 깔아 둬서 나쁠 거 없잖아?

<center style="letter-spacing: .2em;">&ndash;&ndash;&middot;&ndash;</center>

이어지는 내용으로는 그냥 이렇게까지 친절할 필요가 있나 싶은 표를 준비했다. 이런 내용들을 이해하는 데에는 통합된 시스템의 일관성에 대한 심미안 같은 게 필요하지 않다.

- US-ASCII, VCHAR, obs-text
- token, tchar
- comment, ctext
- quoted-string, qdtext
- opaque-tag, etagc

## US-ASCII, VCHAR, obs-text

일단 US-ASCII 부터. 굳이 설명하지 않아도 명확하다.

|          |     `_0` |     `_1` |     `_2` |     `_3` |     `_4` |     `_5` |     `_6` |     `_7` |     `_8` |     `_9` |     `_A` |     `_B` |     `_C` |     `_D` |     `_E` |     `_F` |
|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|
| **`0_`** | &#x2400; | &#x2401; | &#x2402; | &#x2403; | &#x2404; | &#x2405; | &#x2406; | &#x2407; | &#x2408; | &#x2409; | &#x240A; | &#x240B; | &#x240C; | &#x240D; | &#x240E; | &#x240F; |
| **`1_`** | &#x2410; | &#x2411; | &#x2412; | &#x2413; | &#x2414; | &#x2415; | &#x2416; | &#x2417; | &#x2418; | &#x2419; | &#x241A; | &#x241B; | &#x241C; | &#x241D; | &#x241E; | &#x241F; |
| **`2_`** | &#x2420; |      `!` |      `"` |      `#` |      `$` |      `%` |      `&` |      `'` |      `(` |      `)` |      `*` |      `+` |      `,` |      `-` |      `.` |      `/` |
| **`3_`** |      `0` |      `1` |      `2` |      `3` |      `4` |      `5` |      `6` |      `7` |      `8` |      `9` |      `:` |      `;` |      `<` |      `=` |      `>` |      `?` |
| **`4_`** |      `@` |      `A` |      `B` |      `C` |      `D` |      `E` |      `F` |      `G` |      `H` |      `I` |      `J` |      `K` |      `L` |      `M` |      `N` |      `O` |
| **`5_`** |      `P` |      `Q` |      `R` |      `S` |      `T` |      `U` |      `V` |      `W` |      `X` |      `Y` |      `Z` |      `[` |      `\` |      `]` |      `^` |      `_` |
| **`6_`** |  `` ` `` |      `a` |      `b` |      `c` |      `d` |      `e` |      `f` |      `g` |      `h` |      `i` |      `j` |      `k` |      `l` |      `m` |      `n` |      `o` |
| **`7_`** |      `p` |      `q` |      `r` |      `s` |      `t` |      `u` |      `v` |      `w` |      `x` |      `y` |      `z` |      `{` |      `|` |      `}` |      `~` | &#x2421; |

이 중 당연히 VCHAR 는 그래픽 문자 구역에서 SP, DEL 을 제외한 `%x21-7E` 를 부르는 이름이다. SP 는 원래 isprint(3) 를 통과할 수 있는 값이지만 제어 문자라고 취급해도 되고; 한편 SP 와 CR, LF, HT, VT, FF 는 제어 문자이만 isspace(3) 를 통과한다는 (그리고 간혹 isprint(3) 를 통과하거나 통과하지 못한다는) 특수한 지위가 있다는 사실은 짚어 두겠다.

|          |     `_0` |     `_1` |     `_2` |     `_3` |     `_4` |     `_5` |     `_6` |     `_7` |     `_8` |     `_9` |     `_A` |     `_B` |     `_C` |     `_D` |     `_E` |     `_F` |
|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|
| **`0_`** | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; |
| **`1_`** | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; | &#x274E; |
| **`2_`** | &#x274E; |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`3_`** |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`4_`** |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`5_`** |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`6_`** |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`7_`** |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          | &#x274E; |

그리고 obs-text 는 MSB 가 1 인 옥텟 전부를 의미한다. US-ASCII 를 따르는 옥텟의 MSB 는 0 이다.

| (a valid code in its encoding) | all allowed octet |
| ------------------------------ | ----------------- |
| in `00-7F`                     | US-ASCII          |
| in `80-FF`                     | obs-char          |

즉 HTTP 의미론 규격서에서는 적어도 헤더나 멀티파트 바운더리 같은 문자열 값의 각 옥텟은 항상 US-ASCII 범위 안에 있으면 좋겠다고 하고 있는 셈인데; 그런 한편 규격서에서는, 이 규칙을 엄격히 따르는 서브셋 또는 이 규칙을 명확히 고정할 수 있는 방법을 정의하고 있지는 않고, 그 외의 도메인에 대한 해석 규칙 같은 것을 정의하고 있지도 않다. 그저 이 코드 값 구간이 구태<sup>obsolete</sup>한 영역이라고 표시해 뒀을 뿐이다. 사람들에게 전자의 도구가 필요했다면 OSI L6 표현계층 프로토콜이 분리되어 있었을 것이다. 그렇지 않았기에, 대부분의 인터넷 인구는 예전에는 후자의 도구를 ISO 8859-1 이라고 생각했고 요즘은 UTF-8 이라고 생각하고 사는 중이다.

동아시아의 경우, GR&times;GR 로 94&times;94 문자 집합을 쓰는 EUC-CN/-JP/-KR 같은 건 이 규격에서도 문제를 만들지 않는다. 이 인코딩을 로케일로 사용하는 서버와 클라이언트는 아직도 어딘가에 살아 있는데, 이 인코딩으로 된 멀티바이트 문자가 요청이나 응답 헤더에 포함되어도 괜찮다. 한편 GR&times;GR 외에 GR&times;GL 같은 것도 나오는 Shift\_JIS 의 경우 (첫 바이트가 홀수이면 둘째 바이트가 GL로 내려올 수 있다), GL 이 항상 US-ASCII 로 해석되어야 하는 HTTP 의미론에서 후술할 VCHAR 의 서브셋 문자로 된 문자열 조건을 둘째 바이트가 위반할 수 있다. 따라서 Shift\_JIS 를 로케일로 쓰는 서버나 클라이언트가 만드는 요청이나 응답 헤더는 부적법할 수 있다.

&ldquo;obs-&rdquo; 로 시작하는 ABNF 규칙 이름이 구태함을 표시하는 방법이라는 서술은 Section 2.1 에서 볼 수 있다.

> **[2.1.](https://datatracker.ietf.org/doc/html/rfc9110#section-2.1) Syntax Notation**
>
> This specification uses the Augmented Backus-Naur Form (ABNF) notation of \[<u>RFC5234</u>\], extended with the notation for case-sensitivity in strings defined in \[<u>RFC7405</u>\].
>
> It also uses a list extension, defined in <u>Section 5.6.1</u>, that allows for compact definition of comma-separated lists using a "#" operator (similar to how the "*" operator indicates repetition). <u>Appendix A</u> shows the collected grammar with all list operators expanded to standard ABNF notation.
>
> *As a convention, ABNF rule names prefixed with "obs-" denote obsolete grammar rules that appear for historical reasons.*
>
> (&hellip; omitted below; *emphasis* mine, <u>hyperlinks</u> removed.)

VCHAR 과 obs-text 의 합집합은 특별한 규칙을 요하지 않는 필드 값에 쓰이도록 하고 있다. 이 경우 텍스트 구분자는 가장 기초적인 텍스트 구분자인 SP 와 HTAB 이다. CR 이나 LF 는 필드 값에 쓰일 수 없다. 이는 HTTP/1.1 에서는 문법에 의해 주어지는 자연스러운 규칙이지만 HTTP/2 같은 바이너리 프로토콜 문법에서도 같은 규칙을 갖도록 의미론적 제약에도 포함되었다. 필드 값은 Section 5.5 에서 정의하고 있다.

> **[5.5.](https://datatracker.ietf.org/doc/html/rfc9110#section-5.5) Field Values**
>
> HTTP field values consist of a sequence of characters in a format defined by the field's grammar. Each field's grammar is usually defined using ABNF (\[<u>RFC5234</u>\]).
>
> ```abnf9110
> field-value    = *field-content
> field-content  = field-vchar
>                     [ 1*( SP / HTAB / field-vchar ) field-vchar ]
> field-vchar    = VCHAR / obs-text
> obs-text       = %x80-FF
> ```
>
> A field value does not include leading or trailing whitespace. When a specific version of HTTP allows such whitespace to appear in a message, a field parsing implementation **MUST** exclude such whitespace prior to evaluating the field value.
>
> Field values are usually constrained to the range of US-ASCII characters \[<u>USASCII</u>\]. Fields needing a greater range of characters can use an encoding, such as the one defined in \[<u>RFC8187</u>\]. Historically, HTTP allowed field content with text in the ISO-8859-1 charset \[<u>ISO-8859-1</u>\], supporting other charsets only through use of \[<u>RFC2047</u>\] encoding. Specifications for newly defined fields **SHOULD** limit their values to visible US-ASCII octets (VCHAR), SP, and HTAB. A recipient **SHOULD** treat other allowed octets in field content (i.e., <u>obs-text</u>) as opaque data.
>
> Field values containing CR, LF, or NUL characters are invalid and dangerous, due to the varying ways that implementations might parse and interpret those characters; a recipient of CR, LF, or NUL within a field value **MUST** either reject the message or replace each of those characters with SP before further processing or forwarding of that message. Field values containing other CTL characters are also invalid; however, recipients **MAY** retain such characters for the sake of robustness when they appear within a safe context (e.g., an application-specific quoted string that will not be processed by any downstream HTTP parser).
>
> (&hellip; omitted below; <u>hyperlinks</u> removed.)

## token, tchar

토큰은 토큰 문자가 하나 이상 연속된 것이고, 토큰 문자는 VCHAR 의 서브셋이다. VCHAR 중 일부는 토큰 관점에서 구분자로 분류되었다. 토큰은 Section 5.6.2 에 정의되어 있다.

> **[5.6.2.](https://datatracker.ietf.org/doc/html/rfc9110#section-5.6.2) Tokens**
>
> Tokens are short textual identifiers that do not include whitespace or delimiters.
>
> ```abnf9110
> token          = 1*tchar
> tchar          = "!" / "#" / "$" / "%" / "&" / "'" / "*"
>                   / "+" / "-" / "." / "^" / "_" / "`" / "|" / "~"
>                   / DIGIT / ALPHA
>                   ; any VCHAR, except delimiters
> ```
>
> Many HTTP field values are defined using common syntax components, separated by whitespace or specific delimiting characters. Delimiters are chosen from the set of US-ASCII visual characters not allowed in a <u>token</u> (DQUOTE and "(),/:;<=>?@[\]{}").
>
> (<u>hyperlinks</u> removed.)

아래는 이미 VCHAR 인 것 중 tchar 인 특수문자와 tchar 가 아닌 특수문자를 구분하는 표이다. isalnum(3) 을 통과하는 것들은 너무 당연해서 표에 넣지 않았다.

|          | `_0`     | `_1`     | `_2`     | `_3`     | `_4`     | `_5`     | `_6`     | `_7`     | `_8`     | `_9`     | `_A`     | `_B`     | `_C`     | `_D`     | `_E`     | `_F`     |
| -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- | -------- |
| **`0_`** |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
|          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`1_`** |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
|          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`2_`** |          | `!`      | `"`      | `#`      | `$`      | `%`      | `&`      | `'`      | `(`      | `)`      | `*`      | `+`      | `,`      | `-`      | `.`      | `/`      |
|          |          | &#x2B55; | &#x274C; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x274C; | &#x274C; | &#x2B55; | &#x2B55; | &#x274C; | &#x2B55; | &#x2B55; | &#x274C; |
| **`3_`** | `0`      | `1`      | `2`      | `3`      | `4`      | `5`      | `6`      | `7`      | `8`      | `9`      | `:`      | `;`      | `<`      | `=`      | `>`      | `?`      |
|          |          |          |          |          |          |          |          |          |          |          | &#x274C; | &#x274C; | &#x274C; | &#x274C; | &#x274C; | &#x274C; |
| **`4_`** | `@`      | `A`      | `B`      | `C`      | `D`      | `E`      | `F`      | `G`      | `H`      | `I`      | `J`      | `K`      | `L`      | `M`      | `N`      | `O`      |
|          | &#x274C; |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`5_`** | `P`      | `Q`      | `R`      | `S`      | `T`      | `U`      | `V`      | `W`      | `X`      | `Y`      | `Z`      | `[`      | `\`      | `]`      | `^`      | `_`      |
|          |          |          |          |          |          |          |          |          |          |          |          | &#x274C; | &#x274C; | &#x274C; | &#x2B55; | &#x2B55; |
| **`6_`** | `` ` ``  | `a`      | `b`      | `c`      | `d`      | `e`      | `f`      | `g`      | `h`      | `i`      | `j`      | `k`      | `l`      | `m`      | `n`      | `o`      |
|          | &#x2B55; |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`7_`** | `p`      | `q`      | `r`      | `s`      | `t`      | `u`      | `v`      | `w`      | `x`      | `y`      | `z`      | `{`      | `|`      | `}`      | `~`      |          |
|          |          |          |          |          |          |          |          |          |          |          |          | &#x274C; | &#x2B55; | &#x274C; | &#x2B55; |          |

## comment, ctext

코멘트는 괄호 안에 들어간다. 위에서 UA 문자열을 구성하는 것이 토큰과 코멘트였다. 예를 들어 `Mozilla/5.0 (compatible)` 에서 `Mozilla/5.0` 이 product 이고 `(compatible)` 이 comment 가 된다 (그리고 RWS 가 있으니 공백이 들어가야 한다). 그런데 tchar 는 VCHAR 의 서브셋이지만 ctext 는 VCHAR 의 서브셋이 아니고 obs-text 도 포함한다. 이는 토큰에 비해 코멘트에서 쓸 수 있는 문자가 많다는 뜻이다. 코멘트에는 공백도 넣을 수 있다. Section 5.6.5 를 보자.

> **[5.6.5.](https://datatracker.ietf.org/doc/html/rfc9110#section-5.6.5) Comments**
>
> Comments can be included in some HTTP fields by surrounding the comment text with parentheses. Comments are only allowed in fields containing "comment" as part of their field value definition.
>
> ```abnf9110
> comment        = "(" *( ctext / quoted-pair / comment ) ")"
> ctext          = HTAB / SP / %x21-27 / %x2A-5B / %x5D-7E / obs-text
> ```

표로 ctext 를 (그 중 obs-text 가 아닌 것을) 나타내면 이와 같다.

|          |     `_0` |     `_1` |     `_2` |     `_3` |     `_4` |     `_5` |     `_6` |     `_7` |     `_8` |     `_9` |     `_A` |     `_B` |     `_C` |     `_D` |     `_E` |     `_F` |
|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|
| **`0_`** |          |          |          |          |          |          |          |          |          | &#x2409; |          |          |          |          |          |          |
|          |          |          |          |          |          |          |          |          |          | &#x2705; |          |          |          |          |          |          |
| **`1_`** |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
|          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`2_`** | &#x2420; |      `!` |      `"` |      `#` |      `$` |      `%` |      `&` |      `'` |      `(` |      `)` |      `*` |      `+` |      `,` |      `-` |      `.` |      `/` |
|          | &#x2705; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x274C; | &#x274C; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; |
| **`3_`** |      `0` |      `1` |      `2` |      `3` |      `4` |      `5` |      `6` |      `7` |      `8` |      `9` |      `:` |      `;` |      `<` |      `=` |      `>` |      `?` |
|          |          |          |          |          |          |          |          |          |          |          | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; |
| **`4_`** |      `@` |      `A` |      `B` |      `C` |      `D` |      `E` |      `F` |      `G` |      `H` |      `I` |      `J` |      `K` |      `L` |      `M` |      `N` |      `O` |
|          | &#x2B55; |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`5_`** |      `P` |      `Q` |      `R` |      `S` |      `T` |      `U` |      `V` |      `W` |      `X` |      `Y` |      `Z` |      `[` |      `\` |      `]` |      `^` |      `_` |
|          |          |          |          |          |          |          |          |          |          |          |          | &#x2B55; | &#x274C; | &#x2B55; | &#x2B55; | &#x2B55; |
| **`6_`** |  `` ` `` |      `a` |      `b` |      `c` |      `d` |      `e` |      `f` |      `g` |      `h` |      `i` |      `j` |      `k` |      `l` |      `m` |      `n` |      `o` |
|          | &#x2B55; |          |          |          |          |          |          |          |          |          |          |          |          |          |          |          |
| **`7_`** |      `p` |      `q` |      `r` |      `s` |      `t` |      `u` |      `v` |      `w` |      `x` |      `y` |      `z` |      `{` |      `|` |      `}` |      `~` |          |
|          |          |          |          |          |          |          |          |          |          |          |          | &#x2B55; | &#x2B55; | &#x2B55; | &#x2B55; |          |

여는 괄호, 닫는 괄호, 리버스 솔리더스 빼고 아무 거나 써도 된다! 그럼 괄호는 한 겹밖에 안 되나? 당연히 그런 거 아니고, `comment` ABNF 식을 보면 괄호 안에 `comment` 를 또 넣어도 된다는 걸 알 수 있다.

User-Agent 가 정말 출발하기 좋다. 대체로 token 정도는 쓰고, Accept 는 parameter 를 더 쓰는 정도이며, Range 는 간단하다. 위에 quoted-pair 라는 게 있었는데, 이것만 짚고 가자. 그러면 남은 건 ETag 정도다.

## quoted-string, qdtext

> **[5.6.4.](https://datatracker.ietf.org/doc/html/rfc9110#section-5.6.4) Quoted Strings**
>
> A string of text is parsed as a single value if it is quoted using double-quote marks.
>
> ```abnf9110
> quoted-string  = DQUOTE *( qdtext / quoted-pair ) DQUOTE
> qdtext         = HTAB / SP / %x21 / %x23-5B / %x5D-7E / obs-text
> ```
>
> The backslash octet ("\\") can be used as a single-octet quoting mechanism within quoted-string and comment constructs. Recipients that process the value of a quoted-string **MUST** handle a quoted-pair as if it were replaced by the octet following the backslash.
>
> ```abnf9110
> quoted-pair    = "\" ( HTAB / SP / VCHAR / obs-text )
> ```
>
> A sender **SHOULD NOT** generate a quoted-pair in a quoted-string except where necessary to quote DQUOTE and backslash octets occurring within that string. A sender **SHOULD NOT** generate a quoted-pair in a comment except where necessary to quote parentheses \["(" and ")"\] and backslash octets occurring within that comment.

이쯤 왔으면 qdtext 와 ctext 가 정말 똑같이 생겼다는 걸 알 것이다. 큰따옴표 안에서 들어갈 수 있는 값 중 빠지는 값은 딱 `"` 과 `\` 밖에 없다. 큰따옴표는 열고 닫는 짝이 맞아야 하니 빠진다. 그리고 이게 마지막으로 어이없이 웃긴 부분인데 quoted-pair 가 일종의 탈출열<sup>escape sequence</sup>이라서 quoted-string 의 따옴표 사이에 `\"` 를 넣거나 comment 의 열고 닫는 괄호 사이에 `\(` 나 `\)` 를 넣는 용도로 쓸 수 있다는 사실이다. 리버스 솔리더스를 단독으로 쓸 수 없을 뿐. 그러면 리버스 솔리더스를 표현하고 싶으면? 당연히 `\\` 하면 된다. 반대로 말하면, comment 나 quoted-string 에 `\` 가 단독으로 들어갈 수 없고 그 뒤 글자를 잡아먹는다.

## opaque-tag, etagc

마지막으로 entity-tag. 응답 헤더 ETag 에서 쓰고, 그리고 요청에서는 GET 요청의 If-Match 나 If-None-Match 헤더에 쓰는 값이다. `W/` 로 시작하면 약한 개체 태그, 그렇지 않으면 강한 개체 태그이다. 이 둘의 의미론적 구분에 대해서는 Section 8.8.1 에서 다루는데; 쉽게 말하면 강한 검증자가 주어진다면 클라이언트가 HEAD 로 ETag 만 가져가서 임의로 판단해도 되지만, 약한 검증자가 주어진다면 그게 안되고 GET 으로 서버에 판단을 넘겨야 한다. 옥텟 규칙은 이렇다.

> **[8.8.3.](https://datatracker.ietf.org/doc/html/rfc9110#section-8.8.3) ETag**
>
> The "ETag" field in a response provides the current entity tag for the <u>selected representation</u>, as determined at the conclusion of handling the request. An entity tag is an opaque validator for differentiating between multiple representations of the same resource, regardless of whether those multiple representations are due to resource state changes over time, content negotiation resulting in multiple representations being valid at the same time, or both. An entity tag consists of an opaque quoted string, possibly prefixed by a weakness indicator.
>
> ```abnf9110
> ETag       = entity-tag
> 
> entity-tag = [ weak ] opaque-tag
> weak       = %s"W/"
> opaque-tag = DQUOTE *etagc DQUOTE
> etagc      = %x21 / %x23-7E / obs-text
>         ; VCHAR except double quotes, plus obs-text
> ```
>
> **Note:** Previously, opaque-tag was defined to be a quoted-string (\[<u>RFC2616</u>\], <u>Section 3.11</u>); thus, some recipients might perform backslash unescaping. *Servers therefore ought to avoid backslash characters in entity tags.*
>
> (&hellip; omitted below; *emphasis* mine, <u>hyperlinks</u> removed.)

가장 제약이 적다. VCHAR 중에는 쓸 수 있는 게 가장 많다. 큰따옴표로 묶어야 하고, 큰따옴표만 없으면 된다 (공백이나 탭은 안된다). 다만 리버스 솔리더스는 웬만하면 피하면 좋은데, 어떤 구현체는 이를 다음 옥텟과 함께 quoted-pair 로 해석할 수 있기 때문이다. 다만 본문은 이 지점에서 &ldquo;ought to&rdquo; 를 소문자로 사용함으로써 비규범적 문장으로 만들어 이에 대해 규정하기를 회피하고 있다. 이는 어딘가에 있는 또 다른 구현체가 리버스 솔리더스를 quoted-pair 가 아닌 한 글자로 써서 entity-tag 에 넣는 사례가 있고, 심지어 너무 많아서 표준 규칙의 호환성을 희생해서라도 이쪽을 존중해 주는 방향으로 표준 규격서가 대응했음을 시사한다.

표준에서 정한 바는 없으나, opaque-tag 의 이상적인 값에 대한 ABNF 문법은 `DQUOTE *( DIGIT / ALPHA ) DQUOTE` 라고 보면 된다. 아니면 명확한 통제 범위 내에서 bencode 처럼 특수문자 몇 가지만 들어간다든가. d5:hello10:globeworld4:list8:integral3:map6:stringe 뭐 이런 거 있잖은가.

<center style="letter-spacing: .2em;">&ndash;&middot;&middot;</center>

위에서 살펴본 부분들은 사실 파서를 짜지 않는 이상 볼 일 없는 디테일이다. 빌더를 제공하는 라이브러리가 넘쳐나는데 MDN 문서도 아니고 누가 굳이 IETF RFC 원문에서 ABNF 를 보겠냐고. 그런데 가끔은 이런 값이 문제를 일으키고, 그래서 그 전후에 이런 부분에 관해 논할 일이 생긴다.

정형화된 언어에서, 구문론은 아주 고도화된 규칙으로 정확성을 보장하고 자유도를 남기며, 의미론은 그 위에서 우아하게 자유로운 표현의 가능성을 구사한다. 그러나 그 연결<sup>binding</sup>은 종종, 지극히 단순한 형상을 이리저리 나열해 유래를 알 수 없는 변주를 반복할 뿐인 지루한 모양이 된다. 그러나 참고로 이런 형상들은 물질로 치자면 확산되는 것과 비슷한 경향이 있기 때문에, 서로 간섭하지 않도록 하려면 수학적인 분석이 필요하고, 이에 대한 철학은 다분히 언어철학적이며 전혀 낭만적이지 않다. 기껏해야 앞서 언급한 통합된 시스템의 일관성에 대한 심미안 같은 게 좀 필요할 뿐이다. 근데 그게 없이 느낌적인 느낌만으로 알아서 시스템이 통합되는 건 불가능에 가깝다. 기술론을 지어 올리는 과정에서 이런 양상은 간혹 상당한 이해충돌을 형성하며 대체로 임의적인 의사결정을 요한다.

이런 부분을 하나하나 읽는 것도 (적어도 당분간은 여전히) 인간 엔지니어가 할 줄 알아야 하는 일이라는 점을 말하고 싶었다. 끝.