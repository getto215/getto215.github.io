---
title: "Python - contextlib, with 사용"
tags:
  - python
---
* 참고 : 파이썬 코딩의 기술. Better way 43. 재사용 가능한 try/finally 동작을 만들려면 contextlib와 with 문을 고려하자

#### with 문을 이용하면 try/finally 블록의 로직을 재사용할 수 있고, 코드를 깔끔하게 만들 수 있다.
~~~python
lock.acquire()
try:
  print('Lock is held')
finally:
  lock.release()

# 위와 동일한 기능
lock = Lock()
with lock:
  print('Lock is held')

# 자주 사용하는 with문. file close를 해주지 않아도 된다.
with open('/tmp/tmp.txt', 'w') as handle:
  handle.write('This is some data!')
~~~

#### 내장 모듈 contextlib의 contextmanager 데코레이터를 이용하면 직접 작성한 함수를 with문에서 쉽게 사용할 수 있다.
~~~python
import logging
from contextlib import contextmanager


logging.error('Default logging level: error!')
logging.debug('This will not print..')
logging.error('Error!')

@contextmanager
def log_level(level, name):
  logger = logging.getLogger(name)
  old_level = logger.getEffectiveLevel()
  logger.setLevel(level)
  try:
    yield logger
  finally:
    logger.setLevel(old_level)

with log_level(logging.DEBUG, 'my-log') as logger:
  logger.debug('--- with logger')
  logger.debug('This is my message!')
  logging.debug('This will not print')

logger = logging.getLogger('my-log')
logger.error('--- not with logger')
logger.debug('Debug will not print')
logger.error('Error will print')

# --> 결과
# ERROR:root:Default logging level: error!
# ERROR:root:Error!

# DEBUG:my-log:--- with logger
# DEBUG:my-log:This is my message!

# ERROR:my-log:--- not with logger
# ERROR:my-log:Error will print
~~~
* contextmanager에서 넘겨준 값은 with문의 as 부분에 할당된다.
