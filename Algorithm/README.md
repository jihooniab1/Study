# 알고리즘
파이썬을 이용한 코딩 테스트 준비

# Index
- [1. 파이썬 기초](#파이썬-기초-문법)

# 파이썬 기초 문법
## 자료형
정수형: 양의 정수, 음의 정수, 0이 포함됨
```
a = 1000
print(a)

a = -7
print(a)
```

실수형: 소수점 아래의 데이터를 포함하는 수 자료형 (0 생략 가능)
```
a = 157.93
print(a)

a = 5.
print(a)

a = -.7
print(a)
```

컴퓨터 시스템 특성상 실수 정보를 정확하게 표현하는데 한계가 있음 => **round()** 함수를 써보자(소수 n째자리까지 반올림림)
```
a = 0.3 + 0.6
print(round(a,4)) #0.9가 나옴
```

리스트: 여러 데이터를 연속적으로 담아 처리 (STL vector와 유사), 배열과 연결 리스트의 기능 지원. 비어있는 리스트를 선언하려면
`list()`나 `[]` 형태로 선언 가능. 원소에 접근할 때는 **Index** 값을 괄호에 넣어 사용하고, 0부터 시작함 
```
a = [1,2,3,4,5]
print(a[3])

n = 10
a = [0] * 10 #크기 10의 모든 값이 0인 1차원 리스트 초기화
```

**인덱싱(Indexing)**: 인덱스 값을 통해 특정 원소에 접근. 음의 정수를 넣으면 원소를 거꾸로 탐색함
```
a = [1,2,3,4,5]
print(a[1]) # 1
print(a[-1]) # 5
```

**슬라이싱(Slicing)**: 리스트에서 연속적인 위치를 갖는 원소들을 가져옴, 대괄호 안에 콜론`:`을 넣어서 시작, 끝 인덱스 설정. 끝 인덱스는 시작 인덱스보다 1 더 크게 설정
```
a = [1,2,3,4,5]
print(a[1:4]) # [2,3,4]
```

**list comprehension**: 리스트를 초기화하는 방법 중 하나. **대괄호 안에 조건문과 반목문** 을 적용하여 리스트를 초기화 할 수 있음
```
array = [i for i in range(5)]
print(array) # [0, 1, 2, 3, 4]

array = [i for i in range(10) if i % 2 == 1]
print(array) # [1, 3, 5, 7, 9]

array =[[0] * m for _ in range(n)] # 이 방식으로 N X M 크기의 2차원 리스트 쉽게 초기화 가능능
```

| 함수명 | 사용법 | 설명 | 시간 복잡도 |
| :--- | :--- | :--- | :--- |
| `append()` | `변수명.append()` | 리스트에 원소를 하나 삽입할 때 사용한다. | $O(1)$ |
| `sort()` | `변수명.sort()`<br>`변수명.sort(reverse = True)` | 기본 정렬 기능으로 오름차순으로 정렬한다.<br>내림차순으로 정렬한다. | $O(N \log N)$ |
| `reverse()` | `변수명.reverse()` | 리스트의 원소의 순서를 모두 뒤집어 놓는다. | $O(N)$ |
| `insert()` | `insert(삽입할 위치 인덱스, 삽입할 값)` | 특정한 인덱스 위치에 원소를 삽입할 때 사용한다. | $O(N)$ |
| `count()` | `변수명.count(특정 값)` | 리스트에서 특정한 값을 가지는 데이터의 개수를 셀 때 사용한다. | $O(N)$ |
| `remove()` | `변수명.remove(특정 값)` | 특정한 값을 갖는 원소를 제거하는데, 값을 가진 원소가 여러 개면 하나만 제거한다. | $O(N)$ |

문자열: 초기화할 때 큰 따옴표나 작은 따옴표를 이용. 
```
data = "don't you know \"Python\"?
print(data)
```

문자열 연산: 덧셈(+)을 이용하면 **연결(Concatenate)**, 인덱싱과 슬라이싱이 가능하지만 값은 못 바꿈

튜플 자료형: 리스트와 비슷하지만 차이가 존재. 한 번 선언된 값을 바꾸지 못하고, 소괄호를 이용함. **서로 다른 성질의 데이터** 를 묶어서 관리할 때 유용함.  데이터의 나열의 **해싱의 키 값** 으로 사용해랴 할 때도 사용 
```
a = (1,2,3,4,5)
print(a[1:4]) # (2, 3, 4)
```

사전 자료형: **키와 값의 쌍** 을 데이터로 가지는 자료형. 순차적으로 저장하는게 아님. **임의의 변경 불가능한 자료형** 을 키로 사용할 수 있음.
해시 테이블을 사용하기에 데이터의 조회와 수정이 ** O(1)** 시간내에 처리 가능
```
data = dict()
data['사과'] = 'Apple'
data['바나나'] = 'Banana'
data['코코넛'] = 'Coconut'

if '사과' in data:
    print("사과 데이터 존재")
```

키 데이터만 뽑아서 리스트로 이용: **keys()** 함수. 값은 **values()** 함수

집합 자료형: 중복 허용 X, 순서 X. 리스트나 문자열을 이용해서 초기화할 수 있고, 이때 **set() 함수** 를 이용함. 혹은 중괄호와 콤마로 원소 표현
```
data = set([1,1,2,3,4,4])
print(data) # [1,2,3,4]
```

집합 연산: 합집합, 교집합, 차집합 
```
a = set([1,2,3,4,5])
b = set([3,4,5,6,7])

print(a | b) # 합집합
print(a & b) # 교집합
print(a - b) # 차집합합
```

set에 대한 추가 연산은 다음과 같다. 사전의 키나 집합의 원소를 이용해 **O(1)** 의 시간 복잡도로 조회함 
```
data = set([1,2,3])

data.add(4)
data.update([5,6]) # 원소 여러개 추가가
data.remove(3)
```

## 기본 입출력
- input(): **한 줄의 문자열** 을 입력 받는 함수
- map(): 리스트의 모든 원소에 각각 특정한 함수를 적용할 때 사용 

아래와 같이 활용할 수 있음
```
n = int(input()) # 데이터 개수
data = list(map(int, input().split())) # 각 데이터를 공백을 기준으로 구분하여 입력

data.sort(reverse=True)
print(data)
```

입력을 최대한 빨리받아야 하는 경우 => `sys.stdin.readline()` 메서드를 쓸 수 있음. 다만 엔터가 줄 바꿈 기호로 입력되기에 **rstip()** 메서드도 같이 쓴다. 
```
import sys

data = sys.stdin.readline().rstrip()
print(data)
```

기본 출력: **print() 함수**. 기본적으로 출력 후 줄바꿈을 하기에 `end` 속성으로 이를 바꿀 수 있음
```
a = 1
b = 2
print(a, b)
print(7, end=" ")

print("test " + str(a) + "test")
```

f-string: 문자열 앞에 **접두사 f** 를 붙여 중괄호 안에 변수명을 기입하여 간단히 문자열과 정수를 함께 넣을 수 있음
```
answer = 7
print(f"정답은 {answer}.")
```

## 조건문과 반복문
조건문 예제. 파이썬에서는 **코드의 블록을 들여쓰기로 지정** 함.
```
x = 15
if x >= 10:
    print("1")
if x >= 0:
    print("2")
```

if ~ elif ~ else는 다음과 같이 사용
```
a = 5

if a >= 0:
    print("1")
elif a >= 10:
    print("2")
else
    print("3")
```

논리 연산자는 **논리 값(True/False)** 사이의 연산을 수행할 때 사용
- X and Y
- X or Y
- not X

다수의 데이터를 담는 자료형을 위해 **in, not in** 연산자가 제공됨
- x in 리스트 : 리스트 안에 x 들어가있으면 True
- x not in 문자열 : 문자열 안에 x 포함되어 있지 않으면 True

반복문 => for 문이 더 간결하다. 특정한 변수를 이용하여  `in` 뒤에 오는 데이터(리스트, 튜플 등)에 포함된 원소를
첫 번째 인덱스부터 차례대로 하나씩 방문. 연속적인 값을 차례대로 순회할 때는 `range()`를 주로 사용. (시작, 끝 + 1). 인자를 하나만 넣으면 자동으로 시작 값은 0.
```
array = [9,8,7,6,5]
for x in array:
    print(x)

result = 0
for i in range(1, 10):
    result += i
```

## 함수
- 내장 함수: 기본적으로 제공
- 사용자 정의: 개발자가 직접 정의

매개변수: 함수 내부에서 사용할 변수. 반환값: 함수에서 처리 된 결과 반환, 여러 개의 반환 값도 가질 수 있음음
```
def add(a, b):
    return a + b
```

**global** 키워드로 변수를 지정하면 함수 바깥에 선언된 변수를 바로 참조하게 됨
```
a = 0

def func():
    global a
    a += 1
```

람다 표현식: 함수를 간단하게 작성 가능. 이름 없는 함수라고 봐도 됨 
```
array = [('홍길동', 50),('이순신',32),('아무개',74)]

def my_key(x):
    return x[1]

print(sorted(array, key=my_key))
print(sorted(array, key=lambda x: x[1]))
```

```
list1 = [1, 2, 3, 4, 5]
list2 = [6, 7, 8, 9, 10]

result = map(lambda a,b: a + b, list1, list2) # 두 리스트의 각 원소를 함수에 적용
```

## 유용한 표준 라이브러리
- itertools: 반복되는 형태의 데이터를 처리하기 위한 유용한 기능 제공(순열, 조합...)
- heapq: 힙 자료구조를 제공
- bisect: 이진 탐색 기능을 제공
- collections: 덱(deque), 카운터(Counter) 등의 유용한 자료구조 포함
- math: 팩토리얼, 제곱근, 최대공약수 등 다양한 수학 기능 포함 

```
result = sum([1,2,3,4,5])

min_result = min(7, 3, 5, 2)

result = eval("(3+5)*7") # 수식을 실제로 계산
```

**sorted**: 반복가능한 객체가 들어왔을 때, 정렬한 결과를 반환 

- 순열: 서로 다른 n개에서 r개를 선택해서 일렬로 나열 (순서 고려)
- 조합: 서로 다른 n개에서 순서 상관없이 r개를 선택 

```
from itertools import permutations

data = ['A', 'B', 'C']

result = list(permutations(data, 3)) # 3개를 골라서 나열
```

```
from itertools import combinations

data = ['A', 'B', 'C']

result = list(combinations(data, 2)) # 2개를 뽑는 모든 조합
```

중복 순열(product 라이브러리), 중복 조합(combinations_with_replacement 라이브러리)도 가능

**카운터**: 반복 가능한 객체가 주어질 때 내부의 원소가 몇번 씩 등장했는지 알려줌
```
from collections import Counter

counter = Counter(['red','blue','green','red'])
print(counter['green']) # 1
```