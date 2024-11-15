---
title: "Java HashMap"
date: 2024-05-14 12:00:00
categories:
- 자료구조
tags:
- 자료구조
---

## Hash 기본 개념 정리
### Hash Table
`<Key, Value>` 쌍으로 이루어진 데이터를 ***어디에***, ***어떻게*** 저장해야 ***효율적***으로 관리할 수 있을까요?  
`Key` 값에 대응하는 ***유니크***한 값을 만들고 그 값에 해당하는 위치에 데이터를 저장하면, O(1)만에 데이터에 접근할 수 있을 것입니다. 즉, 해시 테이블은 `<Key, Value>` 값들을 ***가능한 한*** 유일한 위치에 저장하는 공간을 말합니다.

<img src = "https://github.com/user-attachments/assets/2187cc32-2119-4ef0-813e-784bcdab382b" width = 400>

<br>

### Hash Function
핵심은 `Key` 값을 기반으로 만든 ***유니크***한 값이 데이터의 저장 위치를 결정한다는 것입니다. 그리고 유니크한 값을 만드는 방법이 바로 **Hash Function**입니다. 다시 말해, 해시 함수는 **`<Key, Value>` 데이터의 저장 위치를 결정하는 함수**입니다.  

<img src = "https://github.com/user-attachments/assets/3ddd6055-bade-4014-857a-c3d4671c532a" width = 400>

<br>

첫 번째 예시를 그대로 적용하면 아래와 같이 해시 함수가 적용될 것입니다.  

<img src = "https://github.com/user-attachments/assets/9be3037c-138f-4153-b584-721090f76f0f" width = 400>

<br>

#### hash code function
해시 함수를 구현하는 방법은 위의 예시처럼 구현하기 나름입니다. 하지만 모종의 이유로 대표적인 함수들이 있습니다.

**해시 코드 함수는 `Key` 값을 정수형 값으로 변환하는 함수**입니다. 결국 **해시값은 데이터를 나타내는 위치**이기 때문에 숫자로 나타내줘야 하기 때문입니다.  

만약 `Key` 값이 문자열이면 어떨까요? 객체가 될 수도 있겠네요. 정수형 범위가 넘어갈 수도 있습니다. 즉, **`Key` 값이 어떠한 형식의 데이터이든 정수형 범위의 값으로 변환하는 과정이 필요**하고, 이것을 해시 코드 함수가 해주는 것입니다.  

<br>

#### compression function
**압축 함수는 해시 코드 함수로 얻은 해시값을 해시 테이블의 크기에 맞게 압축**해주는 함수입니다. 만약 해시 해시 코드 함수로 얻은 해시값이 해시 테이블의 크기 N을 넘어가면, [0, N-1] 범위에 맞게 압축해야 할 것입니다.

<img src = "https://github.com/user-attachments/assets/6ded1aa6-e8bd-4df7-af99-5836203afdf9" width = 500>

<br>

#### Collision
해시 테이블에 모든 `<Key, Value>` 데이터들을 ***유일한*** 위치에 저장하는 것은 사실 불가능합니다. 해시 테이블의 크기가 N이고 저장하고자 하는 데이터들의 개수가 N을 초과한다면, 언젠가 같은 위치에 데이터를 저장하게 될 것이기 때문입니다. 이를 **해시 충돌**이라고 합니다.  

해시 충돌이 발생하지 않도록 해시 함수를 효율적으로 설계하는 것이 가장 중요하겠지만, 만약 해시 충돌이 발생했을 때의 대안도 필요할 것입니다. 대표적으로 Separate Chaining과 Linear Probing 방법이 있습니다.
