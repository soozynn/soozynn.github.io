---
title: "[자바스크립트] Map에 관하여 | LeetCode 594번 문제를 풀다가..."
date: 2026-02-07
categories: [Dev Blog, JavaScript]
tags: [Map, Object, JavaScript, Algorithms]
---

LeetCode 594번 문제를 풀다가,
무의식적으로 사용한 `Map` 자료구조에서 예상과 다른 동작을 마주했다.

습관처럼 Object를 다루듯이 `map[key] = value` 형태로 값을 저장했는데,
에러는 나지 않았지만 `map.size`는 늘지 않았고,
`get`, `has`, `for...of` 순회에서도 값이 잡히지 않았다.

> **왜 에러는 안 나는데, Map처럼 동작하지 않을까?**

이 글은 그 질문에서 출발해
**Object와 Map의 근본적인 차이**,
그리고 **Map의 엔트리(entry) 개념**을 정리한 기록이다.

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

겉보기엔 값이 들어간 것 같지만,
Map이 의도한 방식으로는 아무것도 저장되지 않았다.

---

### 이유: Map도 결국 객체(Object)이기 때문이다
JavaScript에서 `.`(점 표기)와 `[]`(대괄호 표기)는
“자료구조에 값을 넣는다”는 의미가 아니라,
항상 *객체의 프로퍼티(property)를 읽고/쓰는 문법*이다.

```js
map.foo = 1;
map["bar"] = 2;
```
이 문법의 의미는 항상 동일하다.

> “map 객체에 foo / bar라는 프로퍼티를 추가한다”

JavaScript는 **동적 언어**이기 때문에
객체에 새 프로퍼티를 추가하는 행위를 에러로 취급하지 않는다.

---

### 그럼 Map은 왜 특별 취급을 하지 않을까?
Map의 실제 데이터는
객체 프로퍼티가 아니라 **내부 슬롯(internal slot)** 에 저장된다.

개념적으로 보면 Map은 이렇게 생겼다
```lua
Map 객체
 ├─ [[MapData]]  ← 엔트리 저장소 (internal slot)
 │    ├─ Entry 1: (key, value)
 │    ├─ Entry 2: (key, value)
 │    └─ Entry 3: (key, value)
 └─ 기타 메서드 (get, set, has, size 등)
```
- [[MapData]]는 **JS 코드에서 직접 접근 불가**
- 오직 set / get / has / delete로만 조작 가능

따라서,
```js
map.set(k, v); // ⭕ Map의 엔트리 저장소에 저장
map[k] = v;    // ❌ Map 객체의 겉에 프로퍼티 추가
```
map[key] = value는
엔트리를 만든 게 아니라, 그냥 객체 프로퍼티를 하나 붙인 것이다.

---

#### Map의 핵심 개념: 엔트리(entry)란 무엇인가?
Map에서 말하는 엔트리(entry) 는 다음을 의미한다.

(key, value) 한 쌍 = Map의 엔트리
```js
map.set(1, "A");
map.set("1", "B");
```

이 Map에는 엔트리 2개가 존재한다.

- Entry 1: key 1, value "A"
- Entry 2: key "1", value "B"

중요한 점은:
- map.size는 엔트리 개수
- 엔트리는 Map 내부 저장 단위
- 프로퍼티와는 완전히 다른 개념

---

#### Map을 Object처럼 쓰면 생기는 문제들
**1️⃣ 기능적으로 Map의 장점을 전부 잃는다**
- map.size 증가 ❌
- map.get() / map.has() / map.delete() 사용 ❌
- for (const [k, v] of map) 순회 ❌
- map.keys() / map.values() ❌

즉,
> Map을 쓰는 의미 자체가 사라진다

#### Object에도 Object.entries()가 있는데, 그럼 Object도 엔트리를 가지는가?
결론부터 말하면 아니다.

- Object의 기본 단위는 **property**
- Object 내부에는 **엔트리 저장소가 존재하지 않는다**

`Object.entries()`는
Object의 프로퍼티를 *엔트리처럼 보이게 변환한 결과*일 뿐이다.

```js
const obj = { a: 1, b: 2 };

Object.entries(obj);
// [["a", 1], ["b", 2]]
```

이 배열은 **새로 만들어진 데이터**이며,
Object 내부 구조와는 무관하다.

---

#### 그렇담, Object key 순서 규칙은 왜 이렇게 복잡해졌을까? 왜 정렬을 하지?
**원래 Object는 순서 개념이 없었다.**
단순한 property bag에 가까웠다.

하지만 사용 패턴이 늘어나면서 다음과 같은 규칙이 생겼다.

1. 정수처럼 생긴 key → 오름차순
2. 일반 문자열 key → 삽입 순서
3. Symbol key → 삽입 순서 (마지막)

이로 인해 Object는
**순서를 예측하기 어려운 구조가 되었다.**

#### Object vs Map vs Set 계열 비교
| 구조      | 내부 기반      | 핵심 목적  |
| ------- | ---------- | ------ |
| Object  | 상황별 구조     | 데이터 모델 |
| Map     | 해시 + 순서    | 컬렉션    |
| Set     | 해시         | 중복 제거  |
| WeakMap | 약한 해시      | 메타데이터  |
| WeakSet | WeakMap 기반 | 객체 추적  |

### Object와 Map의 근본적인 차이
#### Object

데이터 모델, 상태(state) 표현
정적인 구조에 적합

```js
const obj = { a: 1, b: 2 };
```

#### Map

key-value 컬렉션에 특화
집계, 카운팅, 알고리즘 문제에 적합

```js
const map = new Map();
map.set("a", 1);
map.set("b", 2);
```

| 구분      | Object          | Map   |
| ------- | --------------- | ----- |
| 설계 목적   | 데이터 모델          | 컬렉션   |
| 기본 단위   | Property        | Entry |
| key 타입  | string / symbol | 모든 타입 |
| 순서 보장   | 규칙 복잡           | 명확    |
| size    | ❌               | ⭕     |
| 알고리즘 용도 | 대체 가능           | 정석    |

#### 정리
Object와 Map은 모두 key-value 구조를 가지지만,
Map은 Object의 확장이나 하위 개념이 아니다.

>Object는 상태를 표현하는 데이터 모델이고,
Map은 엔트리를 관리하는 컬렉션 자료구조다.

알고리즘 문제에서
“카운팅”, “집계”, “동적 key 관리”가 필요하다면
Map을 Object처럼 쓰지 말고,
Map답게 set / get / has를 사용해야 한다.

이 작은 차이가
코드의 의도와 정확성을 크게 바꾼다.

---
다음 글에선,
객체의 프로토타입에 대해 정리해보려고 한다.