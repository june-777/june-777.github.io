---
layout: single
title:  "MySQL 에서의 문자열 (char, varchar)"
toc: true
toc_sticky: true
---

### 개요
Ludo 스터디 매칭 팀 프로젝트에서 데이터베이스 물리적 모델링을 진행하며, 자료형에 대한 이해가 필요하다 느껴졌다. 특히, `char`와 `varchar`의 차이에 대해 명확히 인지하고 적용할 필요가 있었다.

<br>

---
### char 과 varchar의 공통점

MySQL에서 char와 varchar 괄호 안의 수는 `문자의 수`를 의미한다. byte 수를 의미하는 것이 아니다.
- **char(10)와 varchar(10) 모두 문자의 수 1 ~ 10개의 문자열을 저장**할 수 있다.  
	char(10)는 문자의 수가 10개인 문자열만 저장할 수 있는 것이 아니다. 
- 두 자료형 모두 문자열의 개수가 10개를 초과하면 오류가 발생한다.

``` sql
CREATE TABLE char_test (
	col1 INT NOT NULL,
    col2 CHAR(10) NOT NULL,
    col3 DATETIME NOT NULL
    );
```

```sql
INSERT INTO char_test(col1, col2, col3) VALUES(1, 'ABCD', '2011-06-27 11:02:11');
INSERT INTO char_test(col1, col2, col3) VALUES(1, 'ABCDEF', '2011-06-27 11:02:11');
INSERT INTO char_test(col1, col2, col3) VALUES(1, 'ABCDEFGHIJ', '2011-06-27 11:02:11');

INSERT INTO char_test(col1, col2, col3) VALUES(1, 'ABCDEFGHIJK', '2011-06-27 11:02:11');
>> Error Code: 1406. Data too long for column 'col2' at row 1
```

<br>

```sql
CREATE TABLE varchar_test (
	col1 INT NOT NULL,
    col2 VARCHAR(10) NOT NULL,
    col3 DATETIME NOT NULL
    );
```

```sql
INSERT INTO varchar_test(col1, col2, col3) VALUES(1, 'ABCD', '2011-06-27 11:02:11');
INSERT INTO varchar_test(col1, col2, col3) VALUES(1, 'ABCDEF', '2011-06-27 11:02:11');
INSERT INTO varchar_test(col1, col2, col3) VALUES(1, 'ABCDEFGHIJ', '2011-06-27 11:02:11');

INSERT INTO varchar_test(col1, col2, col3) VALUES(1, 'ABCDEFGHIJK', '2011-06-27 11:02:11');
>> Error Code: 1406. Data too long for column 'col2' at row 1
```


<br>

### char과 varchar의 차이점
status 를 컬럼으로 갖는 레코드가 있다고 해보자. status의 값은 ACCEPTED, UNCHECKED 이다.  
status 컬럼이 각각 varchar, char 인 경우를 비교해보자.  

#### varchar

status 컬럼이 ACCEPTED인 레코드가 있을 때,  
```sql
INSERT INTO study_applicant_lnk(.., status, ..)
VALUES(.., 'ACCEPTED', ..);
```

<br>

![image](https://github.com/june-777/june-777.github.io/assets/68291395/a18ae0df-55d2-4d19-868e-6385aee44931)

<br> <br>

status 컬럼의 값을 UNCHECKED **(현재 크기보다 더 큰 값)** 으로 변경하면, **레코드 자체를 다른 공간으로 옮겨서 (Row migration) 저장** 해야 한다.
```sql
UPDATE study_applicant_lnk
SET status = 'UNCHECKED'
WHERE ~~ ;
```

<br>

![image](https://github.com/june-777/june-777.github.io/assets/68291395/07c07c24-a147-4d28-bc03-6ab0de748793)

<br> <br>

#### char

하지만 같은 예시에서 char 의 경우, 물리적 저장 공간이 준비되어 있으므로 변경되는 칼럼의 값만 업데이트하면 된다.

```sql
INSERT INTO study_applicant_lnk(.., status, ..)
VALUES(.., 'ACCEPTED', ..);
```

![image](https://github.com/june-777/june-777.github.io/assets/68291395/fd6fda8f-2295-4b80-b7ad-3d3cd5d0ba5e)

<br>

```sql
UPDATE study_applicant_lnk
SET status = 'UNCHECKED'
WHERE ~~ ;
```

![image](https://github.com/june-777/june-777.github.io/assets/68291395/88059372-9119-46a2-a7d1-2b10884f5d33)


<br> <br>

정리하면,
- char(N), varchar(N) 모두 1 ~ N개의 문자를 저장할 수 있다.
- 차이점은 물리적인 데이터 저장 방식에 있다.
- char(N)는 N 바이트 수만큼 저장공간을 할당한다.
- varchar(N)는 저장되는 문자열 + 2~3 바이트 수만큼 저장공간을 할당한다.
	- 저장된 문자열보다 큰 문자열로 update할 경우 레코드 자체를 다른 공간으로 이동시켜야 하므로 비용이 발생한다.

- **칼럼의 값이 자주 변경되고, 저장되는 문자열의 길이가 비슷**하다면 -> char를 사용하자.
- **변경이 잦지 않다면** -> varchar를 사용하자.


<br>


---

그렇다면 현재 진행 중인 스터디 매칭 팀 프로젝트에선 어떻게 적용할 수 있을까? 다음과 같은 비즈니스 요구사항이 산출된 상황이다.

- **요구사항1:** 스터디 지원자는 여러 상태가 있다.
  
	스터디 방장이 지원자의 신청 요청을 확인하지 않은 경우 `UNCHCKED` 상태이다.

	스터디 방장이 지원자의 신청 요청을 수락한 경우 `ACCEPTED` 상태이다.

	스터디 방장이 지원자의 신청 요청을 거절한 경우 `REJECTED` 상태이다.

	스터디 방장이 스터디 모집 글을 삭제한 경우 `REMOVED` 상태이다.

	스터디 지원자가 스터디 신청을 취소한 경우 `CANCLLED` 상태이다.

<br>

- **요구사항 2:** 스터디는 여러 상태가 있다.
  
	스터디 게시글이 막 작성된 경우, 모집중인 경우 `RECRUITING` 상태이다.

	스터디가 진행중인 경우 `PROGRESS` 상태이다.

	스터디가 완료된 경우 `COMPLETED` 상태이다.

	스터디 방장은 스터디의 상태를 변경할 수 있다.

스터디 지원자와 스터디 상태는 각각 여러 상태가 존재한다. 또한 스터디 상태는 방장이 마음대로 바꿀 수 있는 요구사항이 있다.  
**상태에 대한 update query가 빈번할 것으로 예측되고 문자열의 길이 또한 비슷** 하다.


즉 스터디 지원자 및 스터디 상태에 대하여, 현재 비즈니스 요구사항에 만족하는 자료형은 `char`가 적합할 것이다.




---
**Reference**  
Real MySQL 15.1 문자열(CHAR와 VARCHAR)
