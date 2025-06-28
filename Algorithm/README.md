# 알고리즘
파이썬을 이용한 코딩 테스트 준비

# Index
- [1. 파이썬 기초](#파이썬-기초-문법)
- [2. 배열](#배열)
- [3. 연결 리스트](#연결-리스트)
- [4. 스택](#스택)

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

# 배열
## 3273 두 수의 합

문제
n개의 서로 다른 양의 정수 a1, a2, ..., an으로 이루어진 수열이 있다. ai의 값은 1보다 크거나 같고, 1000000보다 작거나 같은 자연수이다. 자연수 x가 주어졌을 때, ai + aj = x (1 ≤ i < j ≤ n)을 만족하는 (ai, aj)쌍의 수를 구하는 프로그램을 작성하시오.

입력
첫째 줄에 수열의 크기 n이 주어진다. 다음 줄에는 수열에 포함되는 수가 주어진다. 셋째 줄에는 x가 주어진다. (1 ≤ n ≤ 100000, 1 ≤ x ≤ 2000000)

출력
문제의 조건을 만족하는 쌍의 개수를 출력한다.

https://www.acmicpc.net/problem/3273

제한시간이 1초라 단순하게 반복문을 탐색하는 방법으로는 힙들다. 이때는 **투포인터** 기법을 활용할 수 있다. 배열이 정렬되어 있을 때, 왼쪽 끝과 오른쪽 끝에 포인터를 배치해두고 조건에 따라 각 포인터를 이동시킨다.

어차피 두 수의 합은 정해져 있으고, 배열의 각 원소는 서로 다른 양의 정수니 
- arr[left] + arr[right] == x: left, right를 각각 1씩 증감
- arr[left] + arr[right] < x: left만 1 증가
- arr[left] + arr[right] > x: right만 1 감소

**left < right** 조건이 유지되는 동안 반복문을 돌면서 위 연산을 반복하고, 카운트를 측정하면 된다.

```
import bisect

n = int(input())
arr = list(map(int,input().split()))
x = int(input())

arr.sort()

cnt = 0
left = 0
right = n - 1

while left < right:
    cur = arr[left] + arr[right]

    if cur == x:
        cnt = cnt + 1
        left = left + 1
        right = right - 1
    elif cur < x:
        left = left + 1
    else:
        right = right - 1
print(cnt)
```

# 연결 리스트
## 1406 에디터
문제
한 줄로 된 간단한 에디터를 구현하려고 한다. 이 편집기는 영어 소문자만을 기록할 수 있는 편집기로, 최대 600,000글자까지 입력할 수 있다.

이 편집기에는 '커서'라는 것이 있는데, 커서는 문장의 맨 앞(첫 번째 문자의 왼쪽), 문장의 맨 뒤(마지막 문자의 오른쪽), 또는 문장 중간 임의의 곳(모든 연속된 두 문자 사이)에 위치할 수 있다. 즉 길이가 L인 문자열이 현재 편집기에 입력되어 있으면, 커서가 위치할 수 있는 곳은 L+1가지 경우가 있다.

이 편집기가 지원하는 명령어는 다음과 같다.
- L: 커서를 왼쪽으로 한 칸 옮김 (커서가 문장의 맨 앞이면 무시됨)
- D: 커서를 오른쪽으로 한 칸 옮김 (커서가 문장의 맨 뒤이면 무시됨)
- B: 커서 왼쪽에 있는 문자를 삭제함 (커서가 문장의 맨 앞이면 무시됨)
삭제로 인해 커서는 한 칸 왼쪽으로 이동한 것처럼 나타나지만, 실제로 커서의 오른쪽에 있던 문자는 그대로임
- P $: $라는 문자를 커서 왼쪽에 추가함

초기에 편집기에 입력되어 있는 문자열이 주어지고, 그 이후 입력한 명령어가 차례로 주어졌을 때, 모든 명령어를 수행하고 난 후 편집기에 입력되어 있는 문자열을 구하는 프로그램을 작성하시오. 단, 명령어가 수행되기 전에 커서는 문장의 맨 뒤에 위치하고 있다고 한다.

입력
첫째 줄에는 초기에 편집기에 입력되어 있는 문자열이 주어진다. 이 문자열은 길이가 N이고, 영어 소문자로만 이루어져 있으며, 길이는 100,000을 넘지 않는다. 둘째 줄에는 입력할 명령어의 개수를 나타내는 정수 M(1 ≤ M ≤ 500,000)이 주어진다. 셋째 줄부터 M개의 줄에 걸쳐 입력할 명령어가 순서대로 주어진다. 명령어는 위의 네 가지 중 하나의 형태로만 주어진다.

출력
첫째 줄에 모든 명령어를 수행하고 난 후 편집기에 입력되어 있는 문자열을 출력한다.

https://www.acmicpc.net/problem/1406

전형적인 **에디터 시뮬레이터** 문제이다. 명령어가 최대 50만개까지 주어지는데, 직접 풀어보면 **insert** 메서드를 썼을 때 시간 초과가 나는 것을 알 수 있다. 파이썬의 리스트 구조 상 맨 끝과 앞에 빼고 넣는 `append`, `pop()`은 연산 속도가 빠르지만, 중간에 빼고 넣는 작업은 시간이 굉장히 오래 걸린다. 

접근은 다음과 같다. 커서를 기준으로 왼쪽, 오른쪽 배열을 나누는 것이다. 커서가 왼쪽으로 움직이는 것은 왼쪽 배열의 끝 원소가 오른쪽 원소로 간다는 뜻이고, 뭘 쓰고 지울 때마다 왼쪽 배열에서 더하고 빼면 된다. 

```
from sys import stdin

left = list(input())
right = []

for _ in range(int(input())):
    command = list(stdin.readline().split())
    if command[0] == 'L' and left:
        right.append(left.pop())
    elif command[0] == 'D' and right:
        left.append(right.pop())
    elif command[0] == 'B' and left:
        left.pop()
    elif command[0] == 'P':
        left.append(command[1])

answer = left + right[::-1]
print(''.join(answer))
```

# 스택
## 2493 탑
문제
KOI 통신연구소는 레이저를 이용한 새로운 비밀 통신 시스템 개발을 위한 실험을 하고 있다. 실험을 위하여 일직선 위에 N개의 높이가 서로 다른 탑을 수평 직선의 왼쪽부터 오른쪽 방향으로 차례로 세우고, 각 탑의 꼭대기에 레이저 송신기를 설치하였다. 모든 탑의 레이저 송신기는 레이저 신호를 지표면과 평행하게 수평 직선의 왼쪽 방향으로 발사하고, 탑의 기둥 모두에는 레이저 신호를 수신하는 장치가 설치되어 있다. 하나의 탑에서 발사된 레이저 신호는 가장 먼저 만나는 단 하나의 탑에서만 수신이 가능하다.

예를 들어 높이가 6, 9, 5, 7, 4인 다섯 개의 탑이 수평 직선에 일렬로 서 있고, 모든 탑에서는 주어진 탑 순서의 반대 방향(왼쪽 방향)으로 동시에 레이저 신호를 발사한다고 하자. 그러면, 높이가 4인 다섯 번째 탑에서 발사한 레이저 신호는 높이가 7인 네 번째 탑이 수신을 하고, 높이가 7인 네 번째 탑의 신호는 높이가 9인 두 번째 탑이, 높이가 5인 세 번째 탑의 신호도 높이가 9인 두 번째 탑이 수신을 한다. 높이가 9인 두 번째 탑과 높이가 6인 첫 번째 탑이 보낸 레이저 신호는 어떤 탑에서도 수신을 하지 못한다.

탑들의 개수 N과 탑들의 높이가 주어질 때, 각각의 탑에서 발사한 레이저 신호를 어느 탑에서 수신하는지를 알아내는 프로그램을 작성하라.

입력
첫째 줄에 탑의 수를 나타내는 정수 N이 주어진다. N은 1 이상 500,000 이하이다. 둘째 줄에는 N개의 탑들의 높이가 직선상에 놓인 순서대로 하나의 빈칸을 사이에 두고 주어진다. 탑들의 높이는 1 이상 100,000,000 이하의 정수이다.

출력
첫째 줄에 주어진 탑들의 순서대로 각각의 탑들에서 발사한 레이저 신호를 수신한 탑들의 번호를 하나의 빈칸을 사이에 두고 출력한다. 만약 레이저 신호를 수신하는 탑이 존재하지 않으면 0을 출력한다.

https://www.acmicpc.net/problem/2493

탑의 수가 많아 반복문 같은 구조로 일일이 세고 있으면 무조건 시간 초과가 발생한다. **단조 감소 스택** 을 사용해서 풀어야 한다. 각 탑들은 신호를 왼쪽으로 보내기 때문에, 오른쪽에 있는 탑보다 작은 탑들은 영원히 신호를 받을 수 없다. 즉 **스택에서 바로 제거** 해도 상관이 없다. 이 원리를 바탕으로 탑이 입력되자마자 처리하고 출력을 해줄 수 있다.

```
n = int(input())
height = list(map(int, input().split()))
tower = list()

for i in range(n):
    h = height[i]
    while True:
        if len(tower) == 0:
            tower.append((h, i))
            break
        top = tower[len(tower) - 1]
        if top[0] < h:
            tower.pop()
        else:
            tower.append((h, i))
            break
    if len(tower) == 1:
        print(0, end=' ')
    else:
        top = tower[len(tower) - 2]
        print(f"{top[1] + 1}",end=' ')
```

## 6198 옥상 정원 꾸미기
도시에는 N개의 빌딩이 있다.

빌딩 관리인들은 매우 성실 하기 때문에, 다른 빌딩의 옥상 정원을 벤치마킹 하고 싶어한다.

i번째 빌딩의 키가 hi이고, 모든 빌딩은 일렬로 서 있고 오른쪽으로만 볼 수 있다.

i번째 빌딩 관리인이 볼 수 있는 다른 빌딩의 옥상 정원은 i+1, i+2, .... , N이다.

그런데 자신이 위치한 빌딩보다 높거나 같은 빌딩이 있으면 그 다음에 있는 모든 빌딩의 옥상은 보지 못한다.

예) N=6, H = {10, 3, 7, 4, 12, 2}인 경우

             = 
 =           = 
 =     -     = 
 =     =     =        -> 관리인이 보는 방향
 =  -  =  =  =   
 =  =  =  =  =  = 
10  3  7  4  12 2     -> 빌딩의 높이
[1][2][3][4][5][6]    -> 빌딩의 번호
1번 관리인은 2, 3, 4번 빌딩의 옥상을 확인할 수 있다.
2번 관리인은 다른 빌딩의 옥상을 확인할 수 없다.
3번 관리인은 4번 빌딩의 옥상을 확인할 수 있다.
4번 관리인은 다른 빌딩의 옥상을 확인할 수 없다.
5번 관리인은 6번 빌딩의 옥상을 확인할 수 있다.
6번 관리인은 마지막이므로 다른 빌딩의 옥상을 확인할 수 없다.
따라서, 관리인들이 옥상정원을 확인할 수 있는 총 수는 3 + 0 + 1 + 0 + 1 + 0 = 5이다.

입력
첫 번째 줄에 빌딩의 개수 N이 입력된다.(1 ≤ N ≤ 80,000)
두 번째 줄 부터 N+1번째 줄까지 각 빌딩의 높이가 hi 입력된다. (1 ≤ hi ≤ 1,000,000,000)
출력
각 관리인들이 벤치마킹이 가능한 빌딩의 수의 합을 출력한다.

https://www.acmicpc.net/problem/6198

이 문제도 ![탑](#2493-탑) 문제와 비슷하다. 결국 방향만 다를 뿐 작은 원소가 더 큰 원소에 의해 가려진다는 공통점이 있기 때문이다. 
본인은 더 큰 빌딩이 작은 빌딩을 제거할 때 카운팅을 해주었다. 

```
import sys

n = int(input())

building = list()
stack = list()

for i in range(n):
    building.append(int(sys.stdin.readline().rstrip()))

count = 0

for i in range(len(building)-1,-1,-1):
    stack.append([building[i],0])
    while True:
        if len(stack) == 1: break
        front = stack[len(stack) - 2]
        if building[i] > front[0]:
            stack[len(stack) - 1][1] = stack[len(stack) - 1][1] + 1 + front[1]
            count = count + front[1]
            stack.pop(len(stack) - 2)
        else:
            break

for s in stack:
    count = count + s[1]
print(count)
```

## 3015 오아시스 재결합
문제
오아시스의 재결합 공연에 N명이 한 줄로 서서 기다리고 있다.

이 역사적인 순간을 맞이하기 위해 줄에서 기다리고 있던 백준이는 갑자기 자기가 볼 수 있는 사람의 수가 궁금해졌다.

두 사람 A와 B가 서로 볼 수 있으려면, 두 사람 사이에 A 또는 B보다 키가 큰 사람이 없어야 한다.

줄에 서 있는 사람의 키가 주어졌을 때, 서로 볼 수 있는 쌍의 수를 구하는 프로그램을 작성하시오.

입력
첫째 줄에 줄에서 기다리고 있는 사람의 수 N이 주어진다. (1 ≤ N ≤ 500,000)

둘째 줄부터 N개의 줄에는 각 사람의 키가 나노미터 단위로 주어진다. 모든 사람의 키는 231 나노미터 보다 작다.

사람들이 서 있는 순서대로 입력이 주어진다.

출력
서로 볼 수 있는 쌍의 수를 출력한다.

https://www.acmicpc.net/problem/3015

이 문제 역시 단조 스택을 활용 하지만, **키가 같은 사람** 을 처리하는 것이 까다로운 문제이다. 키가 같은 사람들이 나란히 입력되는 경우 그룹을 구성해서 
- 키가 같은 사람들끼리의 pair
- 기존의 키가 큰 사람과의 pair

을 고려해서 계산해주어야 한다.

```
import sys

n = int(input())
count = 0
stack = []

for _ in range(n):
    h = int(sys.stdin.readline().rstrip())
    
    while stack and stack[-1][0] < h:
        count += stack.pop()[1]
    
    if not stack:
        stack.append((h, 1))
        continue
    
    if stack[-1][0] == h:
        group_size = stack.pop()[1]
        count += group_size
        
        if stack:
            count += 1
        
        stack.append((h, group_size + 1))
    
    else:
        count += 1
        stack.append((h, 1))
```

# 큐
## 10845 큐
문제
정수를 저장하는 큐를 구현한 다음, 입력으로 주어지는 명령을 처리하는 프로그램을 작성하시오.

명령은 총 여섯 가지이다.

- push X: 정수 X를 큐에 넣는 연산이다.
- pop: 큐에서 가장 앞에 있는 정수를 빼고, 그 수를 출력한다. 만약 큐에 들어있는 정수가 없는 경우에는 -1을 출력한다.
- size: 큐에 들어있는 정수의 개수를 출력한다.
- empty: 큐가 비어있으면 1, 아니면 0을 출력한다.
- front: 큐의 가장 앞에 있는 정수를 출력한다. 만약 큐에 들어있는 정수가 없는 경우에는 -1을 출력한다.
- back: 큐의 가장 뒤에 있는 정수를 출력한다. 만약 큐에 들어있는 정수가 없는 경우에는 -1을 출력한다.

입력
첫째 줄에 주어지는 명령의 수 N (1 ≤ N ≤ 10,000)이 주어진다. 둘째 줄부터 N개의 줄에는 명령이 하나씩 주어진다. 주어지는 정수는 1보다 크거나 같고, 100,000보다 작거나 같다. 문제에 나와있지 않은 명령이 주어지는 경우는 없다.

출력
출력해야하는 명령이 주어질 때마다, 한 줄에 하나씩 출력한다.

https://www.acmicpc.net/problem/10845

전형적인 선입선출의 큐 자료구조 문제이다. 문제는 파이썬에서 이를 리스트로 구현하면 시간 초과의 가능성이 생긴다. **pop(), append()** 와 같이 끝에 뭘 더하고 빼는건 빠르지만, 맨앞에서 제거하는 연산은 `O(n)`이 걸리기 때문이다. 

이럴 때 쓸 수 있는게 파이썬의 **deque, double-ended queue** 자료구조이다. 양쪽 끝에서 삽입/삭제가 빈번하게 발생할 때 쓰기 좋다. `popleft, appendleft`가 전부 O(1)이다. 

```
from collections import deque
import sys

n = int(input())

q = deque()

for _ in range(n):
    command = sys.stdin.readline().split()
    
    if command[0] == 'push':
        num = int(command[1])
        q.append(num)
    elif command[0] == 'pop':
        if len(q) == 0: print(-1)
        else:
            left = q.popleft()
            print(left)
    elif command[0] == 'size':
        print(len(q))
    elif command[0] == 'empty':
        if len(q) == 0: print(1)
        else: print(0)
    elif command[0] == 'front':
        if len(q) == 0: print(-1)
        else:
            print(q[0])
    else:
        if len(q) == 0: print(-1)
        else:
            print(q[-1])
```

# 덱
## 5430 AC
문제
선영이는 주말에 할 일이 없어서 새로운 언어 AC를 만들었다. AC는 정수 배열에 연산을 하기 위해 만든 언어이다. 이 언어에는 두 가지 함수 R(뒤집기)과 D(버리기)가 있다.

함수 R은 배열에 있는 수의 순서를 뒤집는 함수이고, D는 첫 번째 수를 버리는 함수이다. 배열이 비어있는데 D를 사용한 경우에는 에러가 발생한다.

함수는 조합해서 한 번에 사용할 수 있다. 예를 들어, "AB"는 A를 수행한 다음에 바로 이어서 B를 수행하는 함수이다. 예를 들어, "RDD"는 배열을 뒤집은 다음 처음 두 수를 버리는 함수이다.

배열의 초기값과 수행할 함수가 주어졌을 때, 최종 결과를 구하는 프로그램을 작성하시오.

입력
첫째 줄에 테스트 케이스의 개수 T가 주어진다. T는 최대 100이다.

각 테스트 케이스의 첫째 줄에는 수행할 함수 p가 주어진다. p의 길이는 1보다 크거나 같고, 100,000보다 작거나 같다.

다음 줄에는 배열에 들어있는 수의 개수 n이 주어진다. (0 ≤ n ≤ 100,000)

다음 줄에는 [x1,...,xn]과 같은 형태로 배열에 들어있는 정수가 주어진다. (1 ≤ xi ≤ 100)

전체 테스트 케이스에 주어지는 p의 길이의 합과 n의 합은 70만을 넘지 않는다.

출력
각 테스트 케이스에 대해서, 입력으로 주어진 정수 배열에 함수를 수행한 결과를 출력한다. 만약, 에러가 발생한 경우에는 error를 출력한다.

https://www.acmicpc.net/problem/5430

문제 자체는 어렵지 않으나 **파싱** 방법을 기억할만 한 거 같았다. 전형적인 deque을 사용하는 문제이다. R로 방향을 기록하고, D를 쓰기 전에 비어있는지만 확인하면 되는데, **파싱과 빈 덱 처리** 를 못해서 틀렸었다. 

문제 입력 자체가 `[1,2,3,4]` 상태로 들어오는데 처음에는 `read(1)` 를 이용해서 일일이 입력 받았으나, 더 좋은 방법이 있었다. 
```
arr_str = arr[1:-1]  # 이렇게 하면 1,2,3,4 나 빈 배열이 남는다.
...
numbers = arr_str.split(',')
```
이런 식으로 하면 숫자 배열을 깔끔하게 얻을 수 있고, 빈 배열 처리도 쉽게 가능하다. 출력할 때도 빈 배열 처리에 신경을 써주어야 한다. 
```
import sys
from collections import deque

t = int(input())

for _ in range(t):
    dir = 0
    error = 0
    arr = deque()
    p = list(sys.stdin.readline().rstrip())
    l = int(input())

    arr_input = input().strip()
    arr_str = arr_input[1:-1]
    if arr_str:
        numbers = arr_str.split(',')
        for num in numbers:
            arr.append(num.strip())
    for c in p:
        if c == 'R':
            dir = not dir
        else:
            if len(arr) == 0:
                    error = 1
                    break
            if dir == 0:
                arr.popleft()
            else:
                arr.pop()
    if error == 1:
        print('error')
        continue
    print('[', end='')
    if len(arr) == 0:
        print(']')
    elif dir == 0:
        for i in range(len(arr)):
            print(arr[i], end='')
            if i == len(arr) - 1: 
                print(']')
            else: 
                print(',', end='')
    else:
        for i in range(len(arr)-1, -1, -1):
            print(arr[i], end='')
            if i == 0: 
                print(']')
            else: 
                print(',', end='')
```

## 11003 최솟값 찾기
제
N개의 수 A1, A2, ..., AN과 L이 주어진다.

Di = Ai-L+1 ~ Ai 중의 최솟값이라고 할 때, D에 저장된 수를 출력하는 프로그램을 작성하시오. 이때, i ≤ 0 인 Ai는 무시하고 D를 구해야 한다.

입력
첫째 줄에 N과 L이 주어진다. (1 ≤ L ≤ N ≤ 5,000,000)

둘째 줄에는 N개의 수 Ai가 주어진다. (-109 ≤ Ai ≤ 109)

출력
첫째 줄에 Di를 공백으로 구분하여 순서대로 출력한다.

https://www.acmicpc.net/problem/11003

**슬라이딩 윈도우의 최솟값** 을 찾는 문제이다. 이 문제의 경우 deque을 사용하여 최솟값을 구할 수 있고, **단조 덱** 을 이용하여 덱의 원소들을 특정 우선순위에 맞게 유지시킬 수 있다. 이 문제도 스택 문제처럼 최솟값이 될 수 없는 값들을 없애버리는 방법을 사용할 수 있다.

1. 덱에는 입력 배열의 인덱스를 저장하고, 값은 항상 오름차순을 유지해야 한다.
2. 윈도우의 크기가 l을 넘으면 덱의 맨 앞을 제거한다.
3. 그리고 새로 들어온 값이 덱의 마지막보다 크다면 그대로 넣고, 작으면 오름차순 유지를 위해 계속 pop을 한다. 

이런 알고리즘을 반복하면 결국 덱의 맨 앞은 항상 그 윈도우의 최솟값 인덱스를 가리키게 된다. 

```
from collections import deque
import sys

n, l = map(int, input().split())
arr = list(map(int, sys.stdin.readline().split()))
q = deque()

q.append(0)
print(arr[0],end=' ')

for i in range(1, len(arr)):
    cur = arr[i]
    if q[0] < i - l + 1:
        q.popleft()
    while True:
        if not q: break
        if arr[q[-1]] > cur:
            q.pop()
        else:
            break
    q.append(i)
    print(arr[q[0]],end=' ')
```

