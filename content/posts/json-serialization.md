---
title: "JSON이란? 직렬화와 역직렬화 개념 정리"
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
  "age": 25,
  "skills": ["Python", "Java", "React"]
}
```

---

## 직렬화(Serialization)란?

프로그램 내부에서 사용하는 **객체(Object)** 를 외부로 전송하거나 저장할 수 있도록
**문자열(String) 형태로 변환하는 과정**이다.

JavaScript에서 객체를 JSON 문자열로 직렬화하는 예시:
```javascript
const user = {
  name: "이경준",
  age: 25
};

const jsonString = JSON.stringify(user);
console.log(jsonString);
// 출력: {"name":"이경준","age":25}
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

json_string = '{"name": "이경준", "age": 25}'

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