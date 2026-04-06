---
title: "[CS 공부]JSON이란? 직렬화와 역직렬화 개념 정리"
date: 2026-04-06
tags: ["CS", "JSON"]
categories: ["CS"]
draft: false
---

## JSON이란?

JSON(JavaScript Object Notation)은 데이터를 표현하기 위한 텍스트 기반의 경량 데이터 형식이다.
이름에 JavaScript가 들어가지만 특정 언어에 종속되지 않으며, 대부분의 프로그래밍 언어에서 지원한다.
```json
{
  "name": "이경준",
  "age": 27,
  "skills": ["Python", "Java", "React"]
}
```

---

## JSON의 중첩 구조

JSON은 값으로 또 다른 객체나 배열을 가질 수 있어 계층적으로 데이터를 표현할 수 있다.
즉, 객체 안에 객체, 배열 안에 객체처럼 단계별로 중첩하여 복잡한 데이터를 구조화할 수 있다.

```json
{
  "name": "이경준",
  "age": 27,
  "skills": {
      "frontend": ["React", "Javascript"],
      "backend": ["Python", "Java"]
    }
}
```

위 예시에서 `skills`는 단순한 값이 아니라 `frontend`와 `backend`라는 키를 가진 또 다른 객체다.
이처럼 JSON은 재귀적으로 중첩이 가능하여 현실의 복잡한 데이터 구조를 표현하는 데 적합하다.

---

## JavaScript에서 특정 데이터 접근하기

JavaScript에서는 `.`(점 표기법) 또는 `[]`(괄호 표기법)으로 JSON 데이터의 특정 값을 꺼낼 수 있다.

```javascript
const user = {
  "name": "이경준",
  "age": 27,
  "skills": {
    "frontend": ["React", "Javascript"],
    "backend": ["Python", "Java"]
  }
};

console.log(user.name);
// 출력: 이경준

console.log(user.skills.frontend);
// 출력: ["React", "Javascript"]

console.log(user.skills.frontend[0]);
// 출력: React
```

중첩된 구조에서는 `.`을 연속으로 사용하여 단계별로 접근하면 된다.
배열의 경우 `[인덱스]`로 특정 요소를 꺼낼 수 있다.

---

## 직렬화(Serialization)란?

프로그램 내부에서 사용하는 **객체(Object)** 를 외부로 전송하거나 저장할 수 있도록
**문자열(String) 형태로 변환하는 과정**이다.

JavaScript에서 객체를 JSON 문자열로 직렬화하는 예시:
```javascript
const user = {
  name: "이경준",
  age: 27
};

const jsonString = JSON.stringify(user);
console.log(jsonString);
// 출력: {"name":"이경준","age":27}
console.log(typeof jsonString);
// 출력: string
```

`JSON.stringify()`를 통해 JavaScript 객체가 문자열로 변환되었다.
이 문자열은 네트워크를 통해 전송하거나 파일에 저장할 수 있다.

---

## 역직렬화(Deserialization)란?

직렬화된 **문자열 데이터를 다시 프로그램에서 사용할 수 있는 객체로 변환하는 과정**이다.

JavaScript에서 직렬화된 JSON 문자열을 Python에서 역직렬화하는 예시:
```python
import json

json_string = '{"name": "이경준", "age": 27}'

user = json.loads(json_string)
print(user["name"])
# 출력: 이경준
print(type(user))
# 출력: <class 'dict'>
```

`json.loads()`를 통해 문자열이 Python의 딕셔너리 객체로 변환되었다.

---

## 왜 직렬화/역직렬화가 필요한가?

프로그램 내부의 객체는 메모리에 저장되는 방식이 언어마다 다르다.
따라서 JavaScript에서 만든 객체를 Python에 그대로 전달할 수 없다.

이를 해결하기 위해 **모든 언어가 공통으로 이해할 수 있는 문자열 형태(JSON)로 변환**한다.
