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

이제 *alter table* 쿼리로 table default charset 만 바꿔 보자.

```
alter table test_table default charset utf8mb4;
```

결과는 다음과 같다. *show create table* 구문으로 스키마를 보자.

```
show create table test_table;
----
CREATE TABLE `test_table` (
  `test_column` varchar(255) NOT NULL,
  PRIMARY KEY (`test_column`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
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

그래서 왜 MySQL 5.6 때문에 고통스럽냐면 MySQL 5.6 까지 기본 row 형식은 compact 이기 때문이다. 앞에 단서가 있었는데 index key prefix 길이 제한은 row 형식에 따라 달라진다. redundant, compact 인 경우 상한은 767 바이트이며 dynamic, compressed 인 경우 innodb_large_prefix 옵션이 적용되어 상한은 3072 바이트이다 \[3\]\[5\]\[7\]. 그래서 MySQL 에서 index key prefix 길이 제한의 기본값은 767 바이트가 된다.

MySQL 5.7 의 경우 innodb_large_prefix 옵션을 아예 없애 버렸으며 기본 row 형식이 (innodb_default_row_format) dynamic 이고, MySQL 8.0 에서는 아예 innodb_large_prefix 옵션이 없어졌으며 기본 row 형식은 마찬가지로 dynamic 이다.

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

## 참조

\[1\] *MySQL 8.0 Reference Manual.* 10.9.3 The utf8 Character Set (Alias for utf8mb3). <https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8.html>

> **Note**
>
> The utf8mb3 character set is deprecated and you should expect it to be removed in a future MySQL release. Please use utf8mb4 instead. Although utf8 is currently an alias for utf8mb3, at some point utf8 is expected to become a reference to utf8mb4. To avoid ambiguity about the meaning of utf8, consider specifying utf8mb4 explicitly for character set references instead of utf8.

\[2\] Ibid. 8.4.7 Limits on Table Column Count and Row Size. <https://dev.mysql.com/doc/refman/8.0/en/column-count-limit.html>

\[3\] Ibid. 15.22 InnoDB Limits. <https://dev.mysql.com/doc/refman/8.0/en/innodb-limits.html>

\[4\] *MySQL 5.7 Reference Manual.* 8.4.7 Limits on Table Column Count and Row Size. <https://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html>

\[5\] Ibid. 14.23 InnoDB Limits. <https://dev.mysql.com/doc/refman/5.7/en/innodb-limits.html>

\[6\] *MySQL 5.6 Reference Manual.* 8.4.7 Limits on Table Column Count and Row Size. <https://dev.mysql.com/doc/refman/5.6/en/column-count-limit.html>

\[7\] Ibid. 14.22 InnoDB Limits. <https://dev.mysql.com/doc/refman/5.6/en/innodb-limits.html>

\[8\] StackOverflow; answering post \# 45535543 on questioning post \# 24365504. <https://stackoverflow.com/a/45535543/2214758>

\[9\] Database Administrators StackExchange; answering post \# 114870 on questioning post \# 114471. <https://dba.stackexchange.com/a/114870/224173>
