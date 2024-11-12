---
title: "MySQL char와 varchar의 차이"
date: 2024-01-05 12:00:00
categories:
- 데이터베이스
tags:
- 데이터베이스
- 데이터 모델링
---

## 개요
최근 `Ludo` 팀 프로젝트에서 데이터베이스 물리적 모델링을 진행하며, `char`와 `varchar`의 차이에 대해 인지할 팔요가 있어 글을 기록합니다.

<br>


## char, varchar 공통점
괄호 안의 수는 **저장 가능한 문자의 최대 개수**를 의미합니다. `char(N)`, `varchar(N)`은 문자의 개수가 1 ~ N개인 문자열을 저장할 수 있습니다. N보다 긴 문자열을 저장하면 오류가 발생합니다. byte 개수를 의미하는 것이 아니므로 혼동하지 않도록 주의해야 합니다. `char(N)`은 문자의 길이가 N개인 문자열만 저장할 수 있는 것이 아닙니다.


``` sql
-- DDL
CREATE TABLE char_test (
	col1 CHAR(10) NOT NULL
);

-- DML
-- success
INSERT INTO char_test(col1) VALUES('1234');
INSERT INTO char_test(col1) VALUES('123456');
INSERT INTO char_test(col1) VALUES('1234567890');

-- fail
INSERT INTO char_test(col1) VALUES('12345678901');
>> Error Code: 1406. Data too long for column 'col1' at row 1
```

<br>

```sql
-- DDL
CREATE TABLE varchar_test (
	col1 VARCHAR(10) NOT NULL
);

-- DML
-- success
INSERT INTO char_test(col1) VALUES('1234');
INSERT INTO char_test(col1) VALUES('123456');
INSERT INTO char_test(col1) VALUES('1234567890');

-- fail
INSERT INTO char_test(col1) VALUES('12345678901');
>> Error Code: 1406. Data too long for column 'col1' at row 1
```

<br>

## char, varchar 차이점
물리적인 데이터 공간에 **데이터를 저장하는 방식**에 차이가 있습니다. `char(N)`은 저장하려는 문자열과 상관없이, **`N바이트 수`만큼 저장 공간을 할당**해둡니다. 따라서 컬럼의 값이 자주 변경되었을 때 유리할 수 있습니다. 최대 길이 N만큼 물리적 저장 공간을 할당해두었기 때문입니다. 하지만 저장 가능한 문자열 최대 길이인 N보다 훨씬 작은 길이의 문자열을 저장하는 경우 저장 공간의 낭비가 발생할 수 있습니다.

```sql
insert into tableA(column1, status, column3)
values(.., 'ACCEPTED', ..);
```

<img src = "https://github.com/user-attachments/assets/f0ae571b-3693-41fe-a5c3-88f4e49475a5">

```sql
update tableA
set status = 'UNCHECKED'
where ..;
```

<img src = "https://github.com/user-attachments/assets/5e47cbe6-0af4-4004-92d1-df91423fa845">

<br>

`varchar(N)`은 **`저장하려는 문자열` + `문자열 길이(2 ~ 3바이트)`만큼 저장공간을 할당**합니다. 필요한 만큼만 저장하므로 저장 공간을 효율적으로 사용할 수 있는 장점이 있습니다. 하지만 기존에 저장된 문자열의 길이보다 더 긴 문자열로 컬럼을 수정하는 경우, 레코드 자체를 다른 공간으로 옮기는 과정인 `Row migration` 비용이 발생하는 단점이 있습니다.

```sql
insert into tableA(column1, status, column3)
values(.., 'ACCEPTED', ..);
```

<img src = "https://github.com/user-attachments/assets/31ae5cec-5549-4c2b-ae45-66275b4f904e">

```sql
update tableA
set status = 'UNCHECKED'
where ..;
```

<img src = "https://github.com/user-attachments/assets/6bb10487-b193-4a65-8d68-ca399bb78699">


<br>

## char, varchar 판단 기준
정리하면 컬럼의 값이 자주 변경되는 경우 → `char` 자료형을, 컬럼의 값 변경이 많지 않은 경우 → `varchar` 자료형을 고려하면 되겠습니다.

앞서 살펴본 이론을 실제 비즈니스에 적용해보겠습니다. `Ludo` 프로젝트의 비즈니스 요구사항은 아래와 같습니다.

```
요구사항1: 스터디 지원자는 여러 상태가 있다.
- 스터디 방장이 지원자의 신청 요청을 확인하지 않음: `UNCHCKED` 상태
- 스터디 방장이 지원자의 신청 요청을 수락: ACCEPTED 상태
- 스터디 방장이 지원자의 신청 요청을 거절: REJECTED 상태
- 스터디 방장이 스터디 모집 글을 삭제: REMOVED 상태
- 스터디 지원자가 스터디 신청을 취소: CANCLLED 상태

요구사항2: 스터디는 여러 상태가 있다.
- 스터디원 모집중: RECRUITING 상태
- 스터디 진행중: PROGRESS 상태
- 스터디 완료: COMPLETED 상태
- 스터디 방장은 스터디의 상태를 변경할 수 있다.
```

`스터디 지원자`와 `스터디`는 여러 상태가 존재합니다. 또한 스터디 상태는 방장이 마음대로 바꿀 수 있습니다. 즉 **상태에 대한 수정이 빈번할 수 있는 요구사항**이므로, `char` 자료형을 채택하는 것이 합리적이겠습니다.

<br>

---
**Reference**  
Real MySQL: 15.1 문자열(CHAR와 VARCHAR)
