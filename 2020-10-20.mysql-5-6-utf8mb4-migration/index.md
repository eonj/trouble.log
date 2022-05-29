Trouble ID `2020-10-20.mysql-5-6-utf8mb4-migration`

# MySQL 5.6 `utf8mb4` 마이그레이션

최근에 우리 부서에서 내부 시스템을 통해 모바일 앱에 보내는 푸시 알림을 등록하던 중에 이상한 에러를 만나게 되었다. 그리고 이 내부 시스템의 서버는 내가 최근에 서스테이닝 롤을 받은 것으로&hellip; 생판 모르는 시스템에서 내가 그 문제 원인을 찾아야 했다. ㅠㅠ

내부 시스템은 SQL 에러를 그냥 `window.alert` 로 보여주고 있었다. 대강 이런 에러 메시지였다.

```
Incorrect string value: '...' for column '...' at row 1
```

## 원인 찾기

푸시 알림을 등록할 때 포함한 이모지가 문제였다. 예를 들어 `📚` (U+1F4DA BOOKS) 같은 값이 입력되었는데 이는 운좋게 메시지에 포함된 바이트 시퀀스가 `\xf0\x9f\x93\x9a` 로 시작했기 때문에 알 수 있었다고 할 것이다.

```
$ echo -ne '\xf0\x9f\x93\x9a'
📚
```

그럼 왜 `Incorrect string value` 같은 에러 메시지가 났냐면, 이 서버가 쓰는 MySQL 5.6 table 의 charset 으로 `utf8mb4` 가 아니라 `utf8` 이 지정되어 있었기 때문이다.

UTF-8 은 가변 길이 바이트 시퀀스로 1 개의 코드포인트를 나타내는 유니코드 인코딩으로, 1 개의 코드포인트가 최대 4 바이트를 점유할 수 있다.

MySQL 은 문자열 (string of characters) 자료형에 적용되는 charset 에서 **`utf8` 과 `utf8mb4` 을 구분한다.** `utf8` (`utf8mb3`) 은 유니코드에 BMP 만 있고 평면 개념조차 없던 시절에 만들어진 자료 유형이라서, 유니코드 코드포인트 하나에 대해 UTF-8 인코딩 결과로 최대 3 바이트가 나올 것이라는 가정을 유지한다. `utf8mb4` 는 UTF-8 인코딩 결과로 코드포인트 당 최대 4 바이트가 나올 수 있음을 전제하고 만들어졌다. (MySQL 8.0 까지도 `utf8` 은 `utf8mb3` 의 alias 인데, 조만간 이후 버전에서 `utf8mb3` 자체가 삭제되고 `utf8` 을 `utf8mb4` 의 alias 로 만든다고 한다. \[1\] 그게 언제일지는 모르겠다&hellip;)

U+1F4DA BOOKS 는 SMP 에 있는 코드포인트이기 때문에 UTF-8 로 인코드하면 위에 옮긴 것처럼 `F0 9F 93 9A` 가 된다. MySQL 은 UTF-8 4 바이트 시퀀스를 적법한 `utf8` 문자열 값으로 받아들이지 않기 때문에, SQL *insert*/*update* 쿼리가 varchar(n) 같은 문자열 column 에 저런 문자열 값을 지정할 경우 자연스럽게 쿼리가 실패하게 된다. 우리가 다뤄야 하는 문자열 값이 UTF-8 바이트 시퀀스가 4 바이트를 점유하는 코드포인트를 포함할 수도 있다면 적어도 column 에 charset `utf8mb4` 가 지정되어 있어야 한다.

## 고통의 시작

자 그럼 *alter table* 쿼리로 table charset 을 `utf8mb4` 로 바꾸면 되겠지? 이제 MySQL 5.6 때문에 구체적으로 고통받는 시간이 찾아오게 된다. 그리고 왜 `utf8mb3` 가 아직도 레거시로 오래오래 남아있는지에 대한 이야기를 할 차례이다.

MySQL 에서 한 row 내의 값은 blob 이나 text 를 제외하면 모두 직렬화되어 저장되기 때문에, 이를 합산한 **row 크기**가 일정 한계 범위 내에 있을 것으로 가정한다. MySQL 은 row 크기 한계로 65535 바이트를 고집하고 있다 \[2\]\[4\]\[6\]. 이는 MySQL 의 스토리지 엔진인 InnoDB 가 제시하는 한계는 아니고 그냥 MySQL 이 미리 성능을 예측할 수 있는 범위 내에서 동작하도록 한 것이다. 또 다른 제약은 InnoDB 가 제시하는 것인데, **index key prefix 길이** 역시 일정 한계 범위 내에 있어야 한다. 이 값은 column 의 prefix 를 index 로 사용할 때 (또는 full-column index 일 때) 각 index 의 크기에 대한 것이며 구체적으로 row 형식에 따라 달라진다. MySQL 5.6 에서 기본값은 767 바이트이다.

MySQL 의 문자열 자료 유형인 varchar(n) 역시 바이트 수준에서 일정 길이를 점유하여 row 크기와 index key prefix 크기에 기여하기에 이 한계 범위를 계산하는 데에 중요한 변수가 된다. varchar(n) 은 지정된 문자 체계에서 최대 n 개의 문자를 저장할 수 있는 문자열 자료 유형이다. 그런데 유니코드의 경우 정확히 문자 개념에 대응하는 문자소군 (grapheme cluster) 을 데이터베이스에서 다뤄 줄 수는 없으므로 그냥 n 개의 유니코드 코드포인트를 의미하게 된다. (다른 인코딩에서도 제어 문자를 넣을 수 있는 점과 유사하다.)

즉 `utf8mb3` 에서 varchar(n) 으로 지정된 column 은 row 크기에 최대 3 n 바이트만큼 기여하고, varchar(n) 으로 지정된 값을 포함한 index 는 index key prefix 길이에 최대 3 n 바이트만큼 기여한다. `utf8mb4` 에서 varchar(n) 의 기여분은 최대 4 n 바이트가 된다.

그럼 column charset 이나 table charset 을 `utf8mb3` 에서 `utf8mb4` 로 바꾸려 할 때 무슨 일이 벌어질까? 예를 들어 varchar(255) 인 column 이 있다면? 아래 DDL을 보자.

```
create table test_table(test_column varchar(255) primary key) charset utf8mb3;
```

실행 결과는 다음과 같다. *show create table* 구문으로 스키마를 보자.

```
show create table test_table;
----
CREATE TABLE `test_table` (
  `test_column` varchar(255) NOT NULL,
  PRIMARY KEY (`test_column`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
----
```

이제 *alter table* 쿼리로 table default charset 만 바꿔 보자.

```
alter table test_table default charset utf8mb4;
```

결과는? 다시 한번 *show create table* 구문을 사용한다.

```
show create table test_table;
----
CREATE TABLE `test_table` (
  `test_column` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  PRIMARY KEY (`test_column`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
----
```

*alter table* 구문으로 table default charset 만 바꾸는 것은 column 인코딩을 바꾸지 못한다. column charset 을 바꾸려면 *alter table modify* 쿼리를 사용하거나 MySQL 의 *alter table convert to* 구문을 사용해야 한다.

```
alter table test_table modify column test_column varchar(255) charset utf8mb4;
/* or */
alter table test_table convert to charset utf8mb4;
----
Specified key was too long; max key length is 767 bytes
----
```

255 × 3 = 765 ≤ 767 이지만, 255 × 4 = 1020 &gt; 767 이기 때문에 row 크기 한계 제약에 의해 *alter table* 구문 자체가 실패해 버렸다. 그래서 이런 경우 column 자료 유형을 미리 바꿔서, column charset 이 `utf8mb4` 가 되더라도 row 크기나 index key prefix 길이에 대한 column 자료 유형의 최대 기여분이 각각의 제약을 넘지 않도록 조정해야 한다.

## 고통의 연속

그래서 왜 MySQL 5.6 때문에 고통스럽냐면 MySQL 5.6 까지 기본 row 형식은 compact 이기 때문이다. 앞에 단서가 있었는데 index key prefix 길이 제한은 row 형식에 따라 달라진다. redundant, compact 인 경우 상한은 767 바이트이며 dynamic, compressed 인 경우 innodb_large_prefix 옵션이 적용되어 상한은 3072 바이트이다 \[7\]. 그래서 MySQL 5.6 에서 index key prefix 길이 제한의 기본값은 767 바이트가 된다.

MySQL 5.7 의 경우 innodb_large_prefix 옵션 사용을 삼가도록 하였으며 기본 row 형식이 (innodb_default_row_format) dynamic 이고 \[5\], MySQL 8.0 에서는 아예 innodb_large_prefix 옵션을 아예 없애 버렸으며 기본 row 형식은 마찬가지로 dynamic 이다 \[3\]\[10\].

나는 2020 년에 index key prefix 길이 제한이 767 바이트인 낡은 DBMS 때문에 대단치도 않은 DB 스키마에 이상한 고통을 받고 있는 것이다. 게다가 하필이면 `utf8mb3` 같은 낡아빠진 인코딩 때문에 말이다. 유니코드가 UCS-2 범위인 BMP 를 초월할 것으로 여겨져 그 대안으로 UTF-16 이 만들어진 것이 벌써 1996 년 7 월이다. 스토리지 절약한다고 `euckr` 쓰던 시대도 아니고 도대체 누가 `utf8mb3` 로 테이블을 만든단 말인가?

분노는 접어 두고, 그 제한이 3072 바이트라면 또 index 를 얼마나 길게 만들었을지 모르는 일이다. 그렇기 때문에 실질적인 대안을 알아보도록 하겠다.

첫째로, 현재 생성된 테이블에 대해 row 크기와 index key prefix 길이의 최댓값을 보는 방법. 이것부터가 상당히 난관인데 information_schema 의 tables 뷰를 보면 도움이 되는 값이 하나도 없기 때문이다. data_length? 그거 가짜다. SO 에 보면 data_length, index_length 로 뭔가 하려고 하는 답변이 있는데 \[8\] 말도 안 되는 내용이다. 그럼 *show create table* 구문을 돌려서 결과를 보면 어떨까? 여느 DBMS 가 그렇듯 MySQL 역시 *create table* 구문에서부터 커스텀인 부분이 아주 많고 자료 유형에도 MySQL 특이적인 것이 많아서 *create table* 구문을 제대로 이용하는 건 불가능에 가깝다. DBA SX 에 보면 *create table* 구문에 *create database* 구문을 보고 information_schema 에서 얻은 정보까지 추가해서 최대 row 크기를 계산하는 Perl 스크립트를 짜 놓은 사람이 있긴 하다 \[9\].

둘째로, 나는 그냥 이게 정답인 것 같은데, 그냥 기존 MySQL 서버에 *create database* 구문을 이용하든 다른 MySQL 서버를 이용하든 해서 DB 를 하나 새로 파는 것이다. 여기에 기존 데이터베이스 전체를 옮겨서 마이그레이션을 해라 뭐 이런 말이 아니고, 물론 MySQL 5.7 이나 MySQL 8.0 아니면 탈-MySQL 이나 그런 걸 정답으로 본다면 마이그레이션으로 읽겠지만 그 얘기가 아니다. 일단 *create database* 시점에 default charset 을 `utf8mb4` 로 지정한다. 그리고 기존 DB 에서 *show create table* 구문으로 *create table* 구문을 긁어낸다. 여기에서 table default charset 이나 column charset 이 `utf8mb3` 로 지정된 것이 있는지 확인해서 이걸 제거한 후에, 새 DB 에서 *create table* 구문을 수행한다. 기존 DB 를 사용하지 않기 때문에 table 이름이나 column 이름, index 이름 같은 게 겹칠 것을 걱정할 일 없이 거의 원문 그대로 *create table* 구문을 돌릴 수 있다. 우선 이 결과를 보고 `utf8mb4` 로 *alter table modify* 할 수 없는 스키마를 골라내서, 하나씩 고쳐 가면서 charset 변경 계획을 온전히 수립할 수 있다.

## 덤으로, 잘못 알려진 것

유의할 사실 하나는 **multicolumn index 에서 index key prefix 길이 제한이 전체 column prefix 길이를 합산한 것에 대해 적용되는 것은 아니라는** 점이다. index key prefix 길이 제한이 767 일 때 아래 DDL 은 정상 실행된다.

```
create table test_table(test_column_a varchar(255), test_column_b varchar(255), unique key a_concat_b(test_column_a(190), test_column_b(190)));
```

아래 DDL 은 정상 실행되지 않는다.

```
create table test_table(test_column_a varchar(255), test_column_b varchar(255), unique key a_concat_b(test_column_a(191), test_column_b(191)));
```

multicolumn index 의 경우 전체 길이를 합산한 것에 대해 길이 제한이 발생한다는 오해가 꽤 널리 퍼져 있는데, `Specified key was too long; max key length is 767 bytes` 이라는 오류 메시지가 이런 오해를 조장하는 면이 있다. multicolumn index 는 실제로는 column prefix key 각각을 index 로 만드는 것처럼 구현되기 때문에 수치적 제한은 prefix 에 대한 것이다. index key prefix 길이 제한이라는 말에서 보다시피, 이 제한은 index 를 만들 때 사용되는 column prefix (또는 full-column) 각각에 대해 적용되는 것으로, index (key) 전체에 대해 적용되는 것이 아니다.

## 다시, 고통의 원인을 지목하기

MySQL 이라는 RDBMS 가 꼭 어떤 필드에 정상 UTF-8 문자열만 (그게 `utf8mb3` 든 `utf8mb4` 든) 담고 있다고 보장할 필요가 있었을까? 이것은 정답이 뭐라고 확신하기 상당히 어려운 문제이다. 이론적으로야 RDBMS 의 일차적 역할은 unique 나 primary key, foreign key 와 같은 관계 대수적 제약을 유지하는 것이고, 그 위에서 join 이나 aggregation 등 관계 대수적 연산을 고속으로 수행할 수 있는지가 RDBMS 의 성능을 판단하는 기준이 된다. 그렇다면 UTF-8 인코딩 같은 것은 그냥 DBMS 를 사용하는 외부 시스템에서 점검하면 될 사항이 되며, 어차피 유니코드 정상 문자소열인지는 외부에서 판단할 수밖에 없는 부분이다. 그러나 DBMS 는 결국 주기억장치에만 담아 둘 수 없는 규모의 자료를 효율적으로 다루기 위해 사용되기 때문에, 문자열 패턴 매칭 따위의 계산이 DBMS 에 내장되어 있는 것은 일정한 이점을 주며, 이를 위해서는 RDBMS 가 레코드를 단순히 직렬화된 바이트 배열로 다루는 것에서 끝나지 않고 탈직렬화 이후의 의미론에 대해서도 어느 정도 알고 있어야 한다는 결론을 얻는다.

이건 유형과 검증에 관한 문제이다. **프로그래밍에서, 강한 유형 체계는 우리가 적게 일하고도 더 값진 결과를 얻을 수 있도록 돕는 도구가 된다.** 프로그래밍은 결국 논리를 따라 가는 것인데, 논리적 구조는 대체로 선행 조건을 많이 도입했을 때 자동으로 더 많은 후행 조건이 그에 따라서 성립되도록 해 주며 이 위에서 우리는 더욱 강력해짐을 누릴 수 있게 된다. 좋은 유형 체계는 코드의 구조에서 발생하는 결론이 자연스럽게 우리가 코드로 구성해야 하는 프로그램에서 필요한 전제들로 흐르도록 해 준다. 유형 체계는 치명적인 오동작을 탐지해 주기도 하고, 입출력의 규모에 따라 필요한 연산 성능을 예측할 수 있게 해 주기도 한다.

그런데 이번 사례는, 조금 (?) 일반화하자면, **유형 체계가 도리어 사람을 괴롭히는 경우**에 속한다고 할 수 있겠다. 유형 체계는 논리적으로 올바른 흐름을 기술한 코드와 그렇지 않은 것을 구별해 주는 역할을 한다. 그런데 우리가 유형 체계의 도움으로 여러 일을 손쉽게 하는 데에만 익숙해지면 간혹 유형 체계의 본질이 무엇인지 잊게 되기도 한다. 유형 체계를 이용하는 과정은 우리의 코드 조각이 동작하기 이전에 외부 환경이 만족할 제약을 추가하는 것으로부터 출발한다는 사실 말이다.

괜찮은 유형 체계가 마련되어 있지 않은 언어로 프로그래밍하는 건 굉장한 고역이다. 그러나 희한하게도 **성숙한 엔터프라이즈 애플리케이션은 꽤 많은 경우 오히려 유형 체계를 역행하는 방식으로 작성된다.** 예를 들어, 어떤 값이 분명히 정수이고 정수 연산을 하지만 자료를 교환하는 형식에서는 굳이 문자열로 만든다거나 하는 것이다. 이런 접근은 정수 오버플로 문제를 회피할 수 있게 되는 방법이 된다. 부호 있는 32 비트 정수를 0 에서부터 세어 나가면 22 억을 만나지 못하고 21 억 부근에서 별안간 음수가 되며 이것이 널리 알려져 있는 정수 오버플로이다. 정수 값을 문자열로 교환한다면 이런 문제가 생겼을 때 이중으로 검증할 수 있는 방법이 될 수 있다. 그러나 비단 이것만이 유형 체계를 역행하는 코드가 만들어지는 이유는 아니다. 낡은 유형 체계에 의존하는 코드는 정상 입력에 대해서도 예상될 법한 결과를 출력하기보다 대뜸 유형 오류를 내게 된다. 그리고 **대체로 이런 코드의 동작을 정상화하는 과정은 문제가 되는 유형의 정의를 조금 수정해 새 정의를 도입하는 것으로 끝나지 않는다.** 어떤 유형이 기존에 제공하던 제약을 잃으면, 그 유형의 이전 정의에 의존하는 코드 역시 유형 체계 정합성에 문제를 일으키는 일이 생기고, 이런 식으로 **수정이 연쇄적으로 전파되어 소프트웨어 시스템 전체에 변경의 영향을 주게 된다.** 이는 변경 범위와 연계된 비용, 예컨대 QA 등의 프로세스에 의한 시스템 검증 비용 따위가 급증하는 현상을 초래한다.

**우리는 종종 컴파일러님의 비위를 맞춰 가며 코드를 작성한다.** 유형 체계는 우리가 적게 생각하고도 올바른 코드를 만들어 내도록 도와 주는 도구들의 핵심 기술이다. 그래서 프로그래밍 언어와 구현체들은 주로 이 부분에 기능의 방점을 찍는 경향이 있다. 그러나 어떤 경우에는, 유형 체계가 너무 엄격하게 작동하지 않는 것이 소프트웨어 엔지니어에게 도움이 되는 일이다. 많은 경우에 소프트웨어 시스템은 그냥 그 자리에 있는 것만으로 가치를 발휘한다. 코드 작성 당시에 가정한 정상 입력에 대해서만 출력을 낼 수 있는 프로그램을 만들고, 가정이 잘못되었다면 프로그램을 다시 작성해서, 항상 예측가능하게 동작하는 소프트웨어 시스템의 형상을 유지한다&hellip; 아 너무 좋다. 그러나 유형 체계가 검증해 줄 수 있는 것에는 항상 한계가 있고 그 외의 검증은 돈과 시간이 든다. 이런 얘기도 있지. 우리 모두가 사실상 프로그래밍 언어 개발자가 되어, 요구사항을 유형 체계로 변환하고, 유형 체계의 건전성을 증명하여, 입출력이 항상 유형 체계를 따르는 것만으로 우리 코드가 올바른 프로그램임을 자동 증명한다&hellip; 이 역시 정말 행복한 얘기이다. 하지만 냉정하게 얘기해 보자. **많은 경우에, 소프트웨어 시스템은 그냥 그 자리에 무사히 있어 준다면 조금은 덜 예측가능하게 동작해도 괜찮고, 출력이 부정확할 땐 기존 시스템을 수정하지 않는 방식으로 그 문제를 보완해도 치명적이지 않다.** 유형 체계의 장점을 온전히 누리겠다고 현재의 요구사항에 아주 꼭 맞춘 유형들을 도입한다면, 머지않아 코드를 수정하게 되고, 그 코드를 수정하는 것은 인간이며, 여기에도 휴먼 에러가 개입할 구석이 있다. 요구사항을 유형 체계로 변환하면, 어떤 종류의 요구사항은 애초에 건전성을 증명할 수 없는 꼴의 유형 체계를 만들어 낼 수 있고, 상당한 노력에도 불구하고 유의미한 것을 전혀 얻어 내지 못하게 될 공산이 적지 않다. 그래서 컴파일러님의 비위를 맞추는 식으로 코드를 작성하는 것은, 인간 지능의 존엄성을 잃는다거나 그런 종류의 문제가 아니고, 그냥 좀 게으르게 구는 것이다.

이는 스키마를 통해 MySQL 이 문자열 인코딩 `utf8mb3` 를 보장하도록 할 수 있는 부분에 대해서도 해당되는 이야기이다. 문자열 유형이 정상 인코딩을 보장해야 문자열 연산이 정상일 것을 보장할 수 있으니, 인코딩을 지정하도록 하긴 했는데, 그게 미래에 `utf8mb4` 데이터를 아예 넣지도 못하게 만드는 결과를 낳기 때문이다. 사실 다른 RDBMS 라고 상황이 더 나은 건 아니다, 더 심하면 심했지. 그렇다고 아예 schemaless 한 NoSQL 쪽이 정답이냐면, 난 절대 그렇게 생각하지 않는다. 이 부분에 대해서 나는 점진적 유형화를 핵심 기술로 채택하는 적절한 중간 지점이 있을 거라 생각하는데, 그게 스토리지 엔진 등 DBMS 프레임워크 위에서 각자 알아서 지어 올리는 커스텀 DBMS 가 될지, 아니면 SQL 이후의 스크립트 표준을 제시하는 DBMS 가 될지, 결말은 두고 봐야 될 일이다.

## 참조

\[1\] *MySQL 8.0 Reference Manual.* 10.9.3 The utf8 Character Set (Alias for utf8mb3). <https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8.html>

> **Note**
>
> The `utf8mb3` character set is deprecated and you should expect it to be removed in a future MySQL release. Please use `utf8mb4` instead. Although `utf8` is currently an alias for `utf8mb3`, at some point `utf8` is expected to become a reference to `utf8mb4`. To avoid ambiguity about the meaning of `utf8`, consider specifying `utf8mb4` explicitly for character set references instead of `utf8`.

\[2\] Ibid. 8.4.7 Limits on Table Column Count and Row Size. <https://dev.mysql.com/doc/refman/8.0/en/column-count-limit.html>

\[3\] Ibid. 15.22 InnoDB Limits. <https://dev.mysql.com/doc/refman/8.0/en/innodb-limits.html>

> - The index key prefix length limit is 3072 bytes for InnoDB tables that use `DYNAMIC` or `COMPRESSED` row format.
>
>   The index key prefix length limit is 767 bytes for `InnoDB` tables that use the `REDUNDANT` or `COMPACT` row format. For example, you might hit this limit with a [column prefix](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_column_prefix) index of more than 191 characters on a `TEXT` or `VARCHAR` column, assuming a `utf8mb4` character set and the maximum of 4 bytes for each character.


\[4\] *MySQL 5.7 Reference Manual.* 8.4.7 Limits on Table Column Count and Row Size. <https://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html>

\[5\] Ibid. 14.23 InnoDB Limits. <https://dev.mysql.com/doc/refman/5.7/en/innodb-limits.html>

> - If [`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix) is enabled (the default), the index key prefix limit is 3072 bytes for `InnoDB` tables that use the `DYNAMIC` or `COMPRESSED` row format. If [`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix) is disabled, the index key prefix limit is 767 bytes for tables of any row format.
>
>   [`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix) is deprecated; expect it to be removed in a future MySQL release. [`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_large_prefix) was introduced in MySQL 5.5 to disable large index key prefixes for compatibility with earlier versions of `InnoDB` that do not support large index key prefixes.

\[6\] *MySQL 5.6 Reference Manual.* 8.4.7 Limits on Table Column Count and Row Size. <https://dev.mysql.com/doc/refman/5.6/en/column-count-limit.html>

\[7\] Ibid. 14.22 InnoDB Limits. <https://dev.mysql.com/doc/refman/5.6/en/innodb-limits.html>

> - By default, the index key prefix length limit is 767 bytes. See [Section 13.1.13, “CREATE INDEX Statement”](https://dev.mysql.com/doc/refman/5.6/en/create-index.html). For example, you might hit this limit with a [column prefix](https://dev.mysql.com/doc/refman/5.6/en/glossary.html#glos_column_prefix) index of more than 255 characters on a`TEXT` or `VARCHAR` column,assuming a `utf8mb3` character set and the maximum of 3 bytes for each character. When the [`innodb_large_prefix`](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_large_prefix) configuration option is enabled, the index key prefix length limit is raised to 3072 bytes for `InnoDB` tables that use the `DYNAMIC` or `COMPRESSED` row format.

\[8\] StackOverflow; answering post \# 45535543 on questioning post \# 24365504. <https://stackoverflow.com/a/45535543/2214758>

\[9\] Database Administrators StackExchange; answering post \# 114870 on questioning post \# 114471. <https://dba.stackexchange.com/a/114870/224173>

\[10\] *MySQL 8.0 Release Notes.* Changes in MySQL 8.0.0 (2016-09-12, Development Milestone). <https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-0.html>

> - ***Important Change; InnoDB:*** The following `InnoDB` file format configuration options were deprecated in MySQL 5.7.7 and are now removed:
>
>   - `innodb_file_format`
>   - `innodb_file_format_check`
>   - `innodb_file_format_max`
>   - `innodb_large_prefix`
>
>   File format configuration options were necessary for creating tables compatible with earlier versions of `InnoDB` in MySQL 5.1. Now that MySQL 5.1 has reached the end of its product lifecycle, these options are no longer required.
>
>   The `FILE_FORMAT` column was removed from the [`INNODB_SYS_TABLES`](https://dev.mysql.com/doc/refman/5.7/en/information-schema-innodb-sys-tables-table.html) and [`INNODB_SYS_TABLESPACES`](https://dev.mysql.com/doc/refman/5.7/en/information-schema-innodb-sys-tablespaces-table.html) Information Schema tables.
