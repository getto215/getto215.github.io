---
title: "Python - Decorator"
tags:
  - python
---

데코레이터에 대해 이해하려면 퍼스트클래스 함수와 클로저에 대해 이해가 필요함.
따라서 클로저와 퍼스트클래스 함수에 대해 간단히 정리하고 설명함.

* 참고
  * 강좌 : http://schoolofweb.net/blog/posts/%ED%8C%8C%EC%9D%B4%EC%8D%AC-%EB%8D%B0%EC%B
  D%94%EB%A0%88%EC%9D%B4%ED%84%B0-decorator/
  * 도서 : 파이썬 코딩의 기술. Better way 42. functools.wraps로 함수 데코레이터를 정의하자


## 퍼스트클래스 함수(First class function)
언어가 함수(function)을 first-class citizen으로 취급하는 것
* 함수 자체를 인자(arg)로써 다른 함수에 전달
* 다른 함수의 결과값으로 리턴
* 함수를 변수에 할당하거나 데이터 구조안에 저장할 수 있음

~~~python
def logger(msg):
  def log_message():
    print 'Log: ', msg

  return log_message

log_hi = logger('Hi')
print log_hi  # <function log_message at 0x10b7511b8>
log_hi()      # Log:  Hi

del logger    # 오브젝트 삭제

try:
  print logger
except NameError:
  print 'NameError: logger는 존재하지 않습니다.' # <-- 출력

log_hi()      # Log:  Hi
~~~

## 클로저(Closure)
퍼스트클래스 함수를 지원하는 언어의 네임 바인딩 기술.
클로저는 어떤 함수를 함수 자신이 가지고 있는 환경과 함께 저장한 레코드다.
또한 함수가 가진 프리변수(free variable)를 클로저가 만들어지는 당시의 값과 레퍼런스에 맵핑하여 주는 역할을 한다. 클로저는 일반 함수와는 다르게, 자신의 영역 밖에서 호출된 함수의 변수값과 레퍼런스를 복사하고 저장한 뒤, 이 캡처한 값들에 엑세스할 수 있게 도와준다.

> 프리변수(free variable)
파이썬에서 프리변수는 코드 블럭 안에서 사용은 되었지만, 그 코드 블럭 안에서 정의되지 않은 변수를 뜻함.

~~~python
def outer_func(num):
  def inner_func(add_num):
    print "%d + %d = %d" % (num, add_num, num + add_num)
  return inner_func # inner_func() --> X

func = outer_func(10)
print func  # <function inner_func at 0x10e56e0c8>
func(20)    # 10 + 20 = 30
~~~


## 데코레이터(decorator)
기존의 코드에 여러가지 기능을 추가. 함수를 호출하기 전이나 후에 추가로 코드를 실행하는 기능을 갖춤. 이 기능으로 입력 인수와 반환 값을 접근하거나 수정할 수 있음. 이 기능은 시맨틱 강조, 디버깅, 함수 등록 등으로 활용할 수 있음

~~~python
from functools import wraps

def trace(func):
  @wraps(func)  # 내부 함수에 있는 중요한 메타 데이터가 모두 외부 함수로 복사
  def wrapper(*args, **kwargs):
    result = func(*args, **kwargs)
    print('%s(%r, %r) -> %r' % (func.__name__, args, kwargs, result))
    return result
  return wrapper

@trace # decorator
def fibonacci(n):
  if n in (0, 1):
    return n
  return (fibonacci(n-2) + fibonacci(n-1))

print(fibonacci)  # <function wrapper at 0x10e5771b8> --> <function fibonacci at 0x10df48f50>

fibonacci(2)

# --> 결과
# fibonacci((0,), {}) -> 0
# fibonacci((1,), {}) -> 1
# fibonacci((2,), {}) -> 1
~~~
