---
title: "[자바스크립트] Map에 관하여 | LeetCode 594번 문제를 풀다가..."
date: 2026-02-07
categories: [Dev Blog, JavaScript]
tags: [Map, Object, JavaScript, Algorithms]
---

LeetCode 594번 문제를 풀던 중,
Map 자료구조에서 예상과 다른 동작을 마주했다.

무의식적으로 Map을 Object처럼 다루며
map[key] = value 형태로 값을 저장했지만,
에러는 발생하지 않았다.

`map.size`, `get`, `has`,`for...of` 순회에서도
해당 값이 전혀 잡히지 않았다.
그럼에도 런타임 환경에서는 아무런 에러가 발생하지 않았다.

> **왜 잘못된 코드인데도 에러가 발생하지 않았을까?**

이 글은 그 질문에서 출발해
Map과 Object의 기본 사용법을 알고 있다는 전제하에,
Map을 Object처럼 사용했을 때 발생하는
**JavaScript의 ‘silent failure’**와
그에 관련된 개념들을 정리한 기록이다.

---

## 문제 상황: 왜 `map[key] = value`가 에러 없이 동작할까?

```js
const map = new Map();
map[1] = "A";
```

이 코드는 에러 없이 실행된다.
하지만 결과는 다음과 같다.

```js
map.size      // 0
map.get(1)    // undefined
map[1]        // "A"
```

겉보기엔 값이 들어간 것처럼 보이지만,
Map이 의도한 방식으로는 아무것도 저장되지 않았다.

---


<br/>

당연하지만,

### 이유 1: Map도 결국 객체(Object)이기 때문이다

JavaScript에서 `.`(점 표기)와 `[]`(대괄호 표기)는
“자료구조에 값을 넣는다”는 의미가 아니라,<br/>
항상 *객체의 프로퍼티(property)를 읽고/쓰는 문법*이다.

```js
map.foo = 1;
map["bar"] = 2;
```

이 문법의 의미는 항상 동일하다.

> “map 객체에 foo / bar라는 프로퍼티를 추가한다”

❗️ JavaScript는 **동적 언어**이기 때문에 객체에 새로운 프로퍼티를 추가하는 행위를
에러로 취급하지 않는다.<br/>

Map 역시 객체이므로,
map[key] = value와 같은 문법을
특별히 차단하지 않는다.

---

### 이유 2: Map의 실제 데이터는 내부 슬롯에 저장된다

Map의 실제 데이터는
객체 프로퍼티가 아니라 **⛔️ 내부 슬롯(internal slot)** 에 저장된다.

개념적으로 Map은 다음과 같은 구조를 가진다.

```text
Map 객체
 ├─ [[MapData]]  ← 엔트리 저장소 (internal slot)
 │    ├─ Entry 1: (key, value)
 │    ├─ Entry 2: (key, value)
 │    └─ Entry 3: (key, value)
 └─ 기타 인스턴스 메서드 (get, set, has, size 등 / prototype)
```

- [[MapData]]는 **JS 코드에서 직접 접근 할 수 없다**
- 오직 set / get / has / delete를 통해서만 조작된다

따라서 다음 두 코드는 완전히 다른 의미를 가진다.

```js
map.set(k, v); // ⭕ Map의 엔트리 저장소에 저장
map[k] = v;    // ❌ Map 객체의 겉에 프로퍼티 추가
```

`map[key] = value`는
엔트리를 만든 게 아니라, Map 객체의 겉에 프로퍼티를 하나 붙인 것에 불과하다.

---

#### 그래서 왜 에러가 아니라 ‘조용히 실패’할까?

```js
const obj = {};
obj.foo = 1; // 정상 동작
```

JavaScript는 객체에 존재하지 않는 프로퍼티를
추가하는 행위를 에러로 취급하지 않는다.

Map 역시 객체이기 때문에,
잘못된 접근 방식이라 하더라도
런타임 에러는 발생하지 않는다.

그 결과,
코드는 실패했지만
JavaScript는 이를 실패로 간주하지 않는다.

이것이 Map을 Object처럼 사용할 때
가장 위험한 지점이다.

---

#### Map의 핵심 개념: 엔트리(entry)

Map에서 말하는 엔트리(entry) 는 다음을 의미한다.

(key, value) 한 쌍 = Map의 엔트리

```js
map.set(1, "A");
map.set("1", "B");
```

이 Map에는 엔트리 2개가 존재한다.

- Entry 1: key 1, value "A"
- Entry 2: key "1", value "B"

중요한 점은 다음과 같다.

- map.size는 엔트리 개수
- 엔트리는 Map 내부 저장 단위
- **프로퍼티와는 완전히 다른 개념**

---

#### Map을 Object처럼 쓰면 생기는 문제

Map을 Object처럼 사용하면,
Map이 제공하는 모든 장점을 잃게 된다.

- map.size 증가 ❌
- map.get() / map.has() / map.delete() 사용 ❌
- for (const [k, v] of map) 순회 ❌
- map.keys() / map.values() ❌

즉,
> Map을 사용하는 의미 자체가 사라진다

#### Object와 Map의 차이

Object와 Map은 모두 key-value 구조를 가지지만,
설계 목적은 완전히 다르다.

| 구분     | Object          | Map   |
| ------ | --------------- | ----- |
| 설계 목적  | 데이터 모델          | 컬렉션   |
| 기본 단위  | Property        | Entry |
| key 타입 | string / symbol | 모든 타입 |
| 순서 보장  | 규칙 복잡           | 명확    |
| size   | ❌               | ⭕     |

---
JavaScript에서는 많은 경우,</br>
잘못된 코드가 에러 없이 동작한다.

그래서 더더욱,</br>
자료구조의 ‘사용법’이 아니라
‘의도된 모델’을 이해한 상태에서
코드를 작성해야 한다.


---

다음 글에서는,
객체의 프로토타입과 프로퍼티 탐색 과정에 대해
정리해보려고 한다.