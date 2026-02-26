# Python 고급 개념 완전 정복
### Senior Engineer를 위한 박사급 심층 해설 + 해외 대기업 면접 Q&A 35선

---

# 1부. Python 고급 개념 심층 해설

---

## Chapter 1. Python 객체 모델과 메모리 아키텍처

### 1.1 모든 것은 객체다 — PyObject 구조

CPython에서 모든 값은 `PyObject` 또는 `PyVarObject` 구조체를 기반으로 합니다. `PyObject`는 참조 카운트(`ob_refcnt`)와 타입 포인터(`ob_type`)만 가지며, 이 단순한 구조로부터 Python의 동적 타이핑과 가비지 컬렉션이 시작됩니다.

```python
import sys
import ctypes

x = 42
# 참조 카운트 확인
print(sys.getrefcount(x))  # 최소 2 이상 (함수 인자 전달 시 +1)

# 실제 메모리 레이아웃 확인 (CPython 내부)
# id()는 객체의 메모리 주소를 반환
print(id(x))

# 작은 정수 캐싱: -5 ~ 256은 싱글턴
a, b = 256, 256
print(a is b)   # True  — 같은 객체 재사용

c, d = 257, 257
print(c is d)   # False (인터프리터 환경에 따라 다를 수 있음)
```

CPython은 `-5`부터 `256`까지의 정수를 미리 할당하여 캐싱합니다. 이는 빈번한 정수 연산의 메모리 할당/해제 오버헤드를 제거하기 위한 최적화입니다. 이 사실은 면접에서 `is` vs `==` 문제로 자주 등장합니다.

### 1.2 참조 카운팅과 순환 참조 문제

Python의 기본 GC 메커니즘은 참조 카운팅(Reference Counting)입니다. 객체가 더 이상 참조되지 않으면 즉시 메모리를 해제합니다. 그러나 순환 참조(Cyclic Reference)는 참조 카운트가 절대 0이 되지 않는 문제를 일으킵니다.

```python
import gc

class Node:
    def __init__(self, val):
        self.val = val
        self.next = None
    
    def __del__(self):
        print(f"Node {self.val} 소멸됨")

# 순환 참조 생성
a = Node(1)
b = Node(2)
a.next = b
b.next = a  # 순환!

del a
del b
# __del__이 즉시 호출되지 않음 — 참조 카운트가 여전히 1

gc.collect()  # 세대별 GC가 순환 참조를 탐지하고 수거
```

CPython의 세대별 GC(Generational GC)는 0세대(신생 객체), 1세대, 2세대로 나뉘며, 각 세대의 임계값(`gc.get_threshold()` → 기본 `700, 10, 10`)을 초과하면 수거가 실행됩니다. 장기 실행 서버나 LLM 추론 서비스에서 메모리 누수의 주범이 바로 이 순환 참조입니다.

### 1.3 `__slots__`으로 메모리 최적화

일반 클래스는 인스턴스마다 `__dict__`(딕셔너리)를 가지므로 수십 바이트의 오버헤드가 발생합니다. `__slots__`을 선언하면 딕셔너리 대신 고정 크기 배열을 사용하여 30~50%의 메모리를 절약할 수 있습니다.

```python
import sys

class WithDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

obj1 = WithDict(1, 2)
obj2 = WithSlots(1, 2)

print(sys.getsizeof(obj1))           # ~48 bytes
print(sys.getsizeof(obj2))           # ~56 bytes (헤더 크기)
print(sys.getsizeof(obj1.__dict__))  # ~232 bytes (딕셔너리 오버헤드!)
# __slots__ 객체는 __dict__ 자체가 없음
```

MLOps 파이프라인에서 수백만 건의 데이터 레코드를 Python 객체로 다룰 때, `__slots__` 적용만으로도 수 GB의 메모리 절감 효과를 볼 수 있습니다.

---

## Chapter 2. 디스크립터 프로토콜과 메타클래스

### 2.1 디스크립터 프로토콜 — 속성 접근의 핵심 엔진

디스크립터(Descriptor)는 `__get__`, `__set__`, `__delete__` 중 하나 이상을 구현한 객체입니다. Python의 `property`, `classmethod`, `staticmethod`, `function` 모두 디스크립터로 구현되어 있습니다.

```python
class TypedAttribute:
    """타입 검증 디스크립터 — Django ORM Field의 단순화 버전"""
    
    def __set_name__(self, owner, name):
        # Python 3.6+: 클래스 정의 시 자동 호출
        self.name = name
        self.private_name = f'_{name}'
    
    def __init__(self, expected_type):
        self.expected_type = expected_type
    
    def __get__(self, obj, objtype=None):
        if obj is None:
            # 클래스 레벨 접근 시 디스크립터 자체 반환
            return self
        return getattr(obj, self.private_name, None)
    
    def __set__(self, obj, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"{self.name}: {self.expected_type.__name__} 타입이어야 합니다. "
                f"실제 값: {type(value).__name__}"
            )
        setattr(obj, self.private_name, value)

class Model:
    name = TypedAttribute(str)
    learning_rate = TypedAttribute(float)
    
    def __init__(self, name, lr):
        self.name = name
        self.learning_rate = lr

m = Model("GPT-5", 0.001)
# m.name = 42  # TypeError 발생

# 클래스 레벨 접근 → 디스크립터 객체 반환
print(type(Model.name))  # <class 'TypedAttribute'>
```

속성 조회 우선순위(MRO 기반)는 다음과 같습니다: **데이터 디스크립터** → **인스턴스 `__dict__`** → **비데이터 디스크립터 / 클래스 변수**. 이 순서를 이해하지 못하면 예측 불가능한 버그가 발생합니다.

### 2.2 메타클래스 — 클래스를 만드는 클래스

메타클래스는 클래스의 타입입니다. `type`이 기본 메타클래스이며, `type(클래스명, 부모클래스_튜플, 네임스페이스_딕셔너리)`로 동적으로 클래스를 생성할 수 있습니다.

```python
from typing import get_type_hints

class ModelRegistryMeta(type):
    """자동 모델 레지스트리 메타클래스"""
    _registry: dict = {}
    
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        # 추상 기반 클래스는 등록 제외
        if bases:
            mcs._registry[name] = cls
            print(f"[Registry] 모델 등록: {name}")
        return cls
    
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
    
    @classmethod
    def get_model(mcs, name: str):
        return mcs._registry.get(name)

class BaseModel(metaclass=ModelRegistryMeta):
    def forward(self, x):
        raise NotImplementedError

class TransformerModel(BaseModel):    # → Registry에 자동 등록
    def forward(self, x):
        return x @ x.T

class CNNModel(BaseModel):            # → Registry에 자동 등록
    def forward(self, x):
        return x

# 팩토리 패턴: 이름으로 모델 인스턴스화
ModelClass = ModelRegistryMeta.get_model("TransformerModel")
```

이 패턴은 PyTorch의 `nn.Module`, Django의 ORM, FastAPI의 라우터 자동 등록에서 실제로 사용됩니다. MLOps에서 모델 팩토리를 구현할 때 매우 유용합니다.

---

## Chapter 3. 제너레이터, 코루틴, 비동기 아키텍처

### 3.1 제너레이터의 내부 동작 원리

제너레이터는 `PyFrameObject`를 일시 중단(suspend)하고 재개(resume)하는 메커니즘입니다. `yield` 표현식이 있는 함수를 호출하면 즉시 실행되지 않고 `generator` 객체를 반환합니다.

```python
import dis

def fibonacci_gen():
    a, b = 0, 1
    while True:
        yield a          # 실행 프레임을 저장하고 값을 반환
        a, b = b, a + b  # next() 호출 시 여기서 재개

# 바이트코드 확인
dis.dis(fibonacci_gen)
# YIELD_VALUE 명령어가 프레임 상태를 보존함을 확인 가능

# send()를 이용한 양방향 통신
def accumulator():
    total = 0
    while True:
        value = yield total   # yield는 표현식: send()로 전달된 값을 받음
        if value is None:
            break
        total += value

gen = accumulator()
next(gen)        # 제너레이터 초기화 (첫 yield까지 실행)
gen.send(10)     # → 10
gen.send(20)     # → 30
gen.send(5)      # → 35
```

### 3.2 `yield from`과 서브제너레이터 위임

`yield from`은 단순한 문법 설탕이 아닙니다. 서브제너레이터와 호출자 사이의 채널을 직접 연결하여 `send()`와 `throw()`까지 투명하게 전달합니다.

```python
def producer():
    """데이터를 생성하는 서브제너레이터"""
    for i in range(5):
        received = yield i
        print(f"producer received: {received}")

def delegator():
    """yield from으로 producer에게 제어권 위임"""
    result = yield from producer()
    # producer가 StopIteration을 raise하면 result에 return값이 담김
    print(f"delegator: producer 완료, result={result}")

gen = delegator()
print(next(gen))      # 0
print(gen.send("A"))  # producer received: A → 1
print(gen.send("B"))  # producer received: B → 2
```

이 메커니즘이 바로 Python `asyncio`의 기반입니다. `async/await`는 사실 코루틴 기반 이벤트 루프를 위한 제너레이터의 특화 형태입니다.

### 3.3 asyncio 이벤트 루프 내부 구조

```python
import asyncio
import aiohttp
import time
from typing import List

# 실제 LLM API 병렬 호출 패턴
async def fetch_embedding(session: aiohttp.ClientSession, 
                           text: str, 
                           semaphore: asyncio.Semaphore) -> List[float]:
    """세마포어로 동시 연결 수 제한"""
    async with semaphore:
        async with session.post(
            "https://api.openai.com/v1/embeddings",
            json={"input": text, "model": "text-embedding-3-small"},
            headers={"Authorization": "Bearer $TOKEN"}
        ) as resp:
            data = await resp.json()
            return data["data"][0]["embedding"]

async def batch_embed(texts: List[str], max_concurrent: int = 50):
    semaphore = asyncio.Semaphore(max_concurrent)
    async with aiohttp.ClientSession() as session:
        tasks = [
            fetch_embedding(session, text, semaphore) 
            for text in texts
        ]
        # gather는 모든 태스크를 이벤트 루프에 등록하고 결과를 수집
        results = await asyncio.gather(*tasks, return_exceptions=True)
    
    # 실패한 태스크 처리
    embeddings, errors = [], []
    for r in results:
        if isinstance(r, Exception):
            errors.append(r)
        else:
            embeddings.append(r)
    return embeddings, errors
```

이벤트 루프는 `selectors` 모듈(epoll/kqueue/IOCP)을 기반으로 I/O 이벤트를 비동기로 감시합니다. `asyncio.gather`는 내부적으로 각 코루틴을 `Task`로 감싸 이벤트 루프에 등록하며, I/O 대기 중에는 다른 Task로 컨텍스트를 전환합니다.

---

## Chapter 4. 타입 시스템과 정적 분석

### 4.1 Python 타입 시스템의 심층 이해

Python의 타입 힌트는 런타임에 강제되지 않지만, mypy, pyright, pytype 같은 정적 분석 도구의 입력이 됩니다. 고급 타입 표현을 이해하면 대규모 코드베이스의 안전성을 크게 높일 수 있습니다.

```python
from typing import (
    TypeVar, Generic, Protocol, 
    Callable, ParamSpec, Concatenate,
    overload, Literal, TypeGuard
)
from typing import runtime_checkable

# TypeVar와 Generic으로 타입 안전한 컨테이너
T = TypeVar('T')
S = TypeVar('S')

class Pipeline(Generic[T]):
    def __init__(self, value: T) -> None:
        self._value = value
    
    def map(self, func: Callable[[T], S]) -> 'Pipeline[S]':
        return Pipeline(func(self._value))
    
    def get(self) -> T:
        return self._value

# 사용 예: mypy가 전체 체인의 타입을 추론
result: Pipeline[int] = (
    Pipeline("hello world")         # Pipeline[str]
    .map(str.split)                  # Pipeline[list[str]]
    .map(len)                        # Pipeline[int]
)

# ParamSpec: 데코레이터의 시그니처를 보존
P = ParamSpec('P')
R = TypeVar('R')

def retry(times: int) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise
            raise RuntimeError("unreachable")
        return wrapper
    return decorator

@retry(times=3)
def call_api(url: str, timeout: float = 5.0) -> dict:
    ...
# call_api의 시그니처가 완전히 보존됨 — mypy가 검증 가능

# Protocol: 구조적 서브타이핑 (덕 타이핑의 정적 버전)
@runtime_checkable
class Vectorizable(Protocol):
    def to_vector(self) -> list[float]: ...
    def dim(self) -> int: ...

def cosine_similarity(a: Vectorizable, b: Vectorizable) -> float:
    import math
    va, vb = a.to_vector(), b.to_vector()
    dot = sum(x * y for x, y in zip(va, vb))
    norm_a = math.sqrt(sum(x**2 for x in va))
    norm_b = math.sqrt(sum(x**2 for x in vb))
    return dot / (norm_a * norm_b + 1e-9)
```

### 4.2 TypeGuard와 런타임 타입 좁히기

```python
from typing import TypeGuard, Union

def is_list_of_str(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

def process(data: Union[list[str], list[int]]) -> str:
    if is_list_of_str(data):
        # 이 블록 안에서 mypy는 data를 list[str]로 인식
        return ", ".join(data)
    else:
        return str(sum(data))  # data는 list[int]로 좁혀짐
```

---

## Chapter 5. 컨텍스트 관리자와 리소스 생명주기

### 5.1 `contextlib`을 이용한 고급 컨텍스트 관리자

```python
import contextlib
import time
import logging
from typing import Iterator, Optional

@contextlib.contextmanager
def timer(label: str, logger: Optional[logging.Logger] = None) -> Iterator[None]:
    """코드 블록 실행 시간 측정 컨텍스트 매니저"""
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        msg = f"[{label}] {elapsed:.4f}s"
        if logger:
            logger.info(msg)
        else:
            print(msg)

# ExitStack: 동적으로 컨텍스트 매니저를 쌓기
def process_multiple_files(paths: list[str]):
    with contextlib.ExitStack() as stack:
        # 파일 개수가 런타임에 결정되어도 안전하게 관리
        files = [stack.enter_context(open(p)) for p in paths]
        # 어떤 파일 열기가 실패해도 이미 열린 파일들은 모두 닫힘
        for f in files:
            process(f.read())

# 비동기 컨텍스트 매니저
class AsyncModelSession:
    """ML 모델 세션의 비동기 생명주기 관리"""
    
    async def __aenter__(self):
        self._session = await self._load_model()
        return self._session
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self._session.close()
        # True를 반환하면 예외 억제, False/None이면 예외 전파
        return False
    
    async def _load_model(self):
        await asyncio.sleep(0.1)  # 모의 I/O
        return {"status": "loaded"}
```

---

## Chapter 6. 함수형 프로그래밍 패턴

### 6.1 함수를 일급 시민으로 다루기 — 클로저와 커링

```python
from functools import reduce, partial, lru_cache
from typing import Callable, TypeVar, Iterable

A = TypeVar('A')
B = TypeVar('B')
C = TypeVar('C')

# 커링 구현
def curry(func: Callable) -> Callable:
    """임의 인자 수를 가진 함수를 자동으로 커링"""
    import inspect
    arity = len(inspect.signature(func).parameters)
    
    def curried(*args):
        if len(args) >= arity:
            return func(*args[:arity])
        return lambda *more: curried(*(args + more))
    
    return curried

@curry
def add(a: int, b: int, c: int) -> int:
    return a + b + c

add5 = add(5)          # 부분 적용 → 새 함수 반환
add5_3 = add5(3)       # 부분 적용 → 또 다른 함수
print(add5_3(2))       # 10

# 모나드 패턴: Maybe/Option
class Maybe(Generic[T]):
    """None-safe 연산 체인"""
    
    def __init__(self, value: Optional[T]):
        self._value = value
    
    @classmethod
    def of(cls, value: Optional[T]) -> 'Maybe[T]':
        return cls(value)
    
    def map(self, func: Callable[[T], S]) -> 'Maybe[S]':
        if self._value is None:
            return Maybe(None)
        try:
            return Maybe(func(self._value))
        except Exception:
            return Maybe(None)
    
    def flat_map(self, func: Callable[[T], 'Maybe[S]']) -> 'Maybe[S]':
        if self._value is None:
            return Maybe(None)
        return func(self._value)
    
    def get_or_else(self, default: T) -> T:
        return self._value if self._value is not None else default

# 사용 예: None 체크 없는 안전한 체인
result = (
    Maybe.of(user_data)
    .map(lambda d: d.get("profile"))
    .map(lambda p: p.get("embedding"))
    .map(lambda e: e[:512])  # 차원 축소
    .get_or_else([0.0] * 512)
)
```

### 6.2 `lru_cache`와 메모이제이션의 심층 이해

```python
from functools import lru_cache, cache
import sys

# lru_cache의 내부: 해시맵 + 이중 연결 리스트 (LRU 교체 정책)
@lru_cache(maxsize=128)
def expensive_tokenize(text: str, vocab_size: int) -> tuple[int, ...]:
    """토크나이저 결과 캐싱 — 같은 텍스트는 재계산 안 함"""
    # 실제 토크나이저 호출을 시뮬레이션
    return tuple(hash(w) % vocab_size for w in text.split())

# 캐시 통계 확인
expensive_tokenize("hello world", 50000)
expensive_tokenize("hello world", 50000)  # 캐시 히트
print(expensive_tokenize.cache_info())
# CacheInfo(hits=1, misses=1, maxsize=128, currsize=1)

# 주의: lru_cache는 인자가 hashable해야 함
# list, dict 같은 mutable 타입은 사용 불가
# 해결책: tuple로 변환하거나 frozenset 사용
@lru_cache(maxsize=None)
def compute_with_list(items: tuple[int, ...]) -> int:
    return sum(items)

# 클래스 메서드에서 lru_cache 사용 시 주의점
class ModelCache:
    def __init__(self):
        # 인스턴스 메서드에 직접 lru_cache 불가 (self가 hashable하지 않을 수 있음)
        # methodtools.lru_cache 또는 아래 패턴 사용
        self._cache = {}
    
    def predict(self, input_hash: int) -> float:
        if input_hash not in self._cache:
            self._cache[input_hash] = self._compute(input_hash)
        return self._cache[input_hash]
```

---

## Chapter 7. 성능 최적화 — CPython, NumPy, Cython, Numba

### 7.1 CPython 바이트코드 최적화

```python
import dis
import timeit

# 리스트 컴프리헨션 vs for 루프: 왜 컴프리헨션이 빠른가?
def loop_version(n):
    result = []
    for i in range(n):
        result.append(i * i)
    return result

def comp_version(n):
    return [i * i for i in range(n)]

# 컴프리헨션은 LOAD_FAST(result.append) 오버헤드가 없고
# LIST_APPEND 전용 opcode를 사용하여 메서드 조회 비용이 제거됨
dis.dis(comp_version)

# 성능 비교
print(timeit.timeit(lambda: loop_version(10000), number=1000))
print(timeit.timeit(lambda: comp_version(10000), number=1000))

# Global vs Local 변수 접근 속도
import math

def slow_sqrt(n):
    # LOAD_GLOBAL math, LOAD_ATTR sqrt → 두 번의 딕셔너리 조회
    return [math.sqrt(i) for i in range(n)]

def fast_sqrt(n):
    _sqrt = math.sqrt  # 로컬 변수에 바인딩
    # LOAD_FAST _sqrt → 하나의 배열 인덱스 접근 (2~3배 빠름)
    return [_sqrt(i) for i in range(n)]
```

### 7.2 NumPy 벡터화와 브로드캐스팅

```python
import numpy as np

# 브로드캐스팅 규칙: 차원이 오른쪽부터 정렬, 1은 자동 확장
A = np.random.randn(1000, 512)   # (1000, 512)
B = np.random.randn(512)          # (512,) → 브로드캐스팅으로 (1000, 512)
C = A + B                          # 메모리 효율적, SIMD 최적화

# einsum: 복잡한 텐서 연산을 명시적으로
# 배치 행렬 곱 (Transformer attention에서 핵심)
Q = np.random.randn(32, 8, 128)   # batch, heads, dim
K = np.random.randn(32, 8, 128)
scores = np.einsum('bhi,bhj->bhij', Q, K) / np.sqrt(128)
# 이는 torch.matmul(Q, K.transpose(-2, -1))과 동일하지만 의도가 명확

# stride tricks: 복사 없이 슬라이딩 윈도우
def sliding_window_view(arr: np.ndarray, window: int) -> np.ndarray:
    """메모리 복사 없이 슬라이딩 윈도우 뷰 생성"""
    shape = (arr.shape[0] - window + 1, window)
    strides = (arr.strides[0], arr.strides[0])
    return np.lib.stride_tricks.as_strided(arr, shape=shape, strides=strides)

signal = np.arange(10, dtype=float)
windows = sliding_window_view(signal, 3)
# shape: (8, 3) — 데이터 복사 없음!
```

### 7.3 Numba JIT 컴파일

```python
from numba import jit, prange, cuda
import numpy as np

@jit(nopython=True, parallel=True, cache=True)
def pairwise_distance_numba(X: np.ndarray) -> np.ndarray:
    """
    N×N 거리 행렬 계산
    nopython=True: CPython 인터프리터 완전히 우회, LLVM으로 컴파일
    parallel=True: prange로 병렬 루프 자동 생성
    """
    n = X.shape[0]
    dist = np.zeros((n, n))
    for i in prange(n):  # 자동 병렬화
        for j in range(i + 1, n):
            d = 0.0
            for k in range(X.shape[1]):
                diff = X[i, k] - X[j, k]
                d += diff * diff
            dist[i, j] = dist[j, i] = np.sqrt(d)
    return dist

# 첫 호출 시 JIT 컴파일 (수 초 소요)
X = np.random.randn(1000, 128)
result = pairwise_distance_numba(X)
# 이후 호출은 C/FORTRAN 수준 속도
```

---

## Chapter 8. 동시성 모델 심층 비교

### 8.1 GIL과 멀티스레딩의 진실

CPython의 GIL(Global Interpreter Lock)은 한 번에 하나의 스레드만 Python 바이트코드를 실행하도록 보장합니다. 이는 참조 카운팅의 thread-safety를 위한 설계입니다. 그러나 I/O 바운드 작업이나 C 확장(NumPy, Pandas 등)은 GIL을 릴리즈하므로 멀티스레딩이 효과적입니다.

```python
import threading
import multiprocessing
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import time
import numpy as np

def cpu_bound(n: int) -> int:
    """순수 Python CPU 바운드 작업 — GIL로 인해 스레드 병렬화 불가"""
    return sum(i * i for i in range(n))

def numpy_cpu_bound(n: int) -> float:
    """NumPy 연산 — GIL 릴리즈하므로 스레드 병렬화 가능"""
    arr = np.random.randn(n)
    return float(np.sum(arr ** 2))

# CPU-bound: 멀티프로세싱이 효과적
with ProcessPoolExecutor(max_workers=4) as exe:
    results = list(exe.map(cpu_bound, [10_000_000] * 4))

# I/O-bound 또는 NumPy: 스레드풀이 효과적 (GIL 해제됨)
with ThreadPoolExecutor(max_workers=4) as exe:
    results = list(exe.map(numpy_cpu_bound, [1_000_000] * 4))

# Python 3.12+: Per-Interpreter GIL (PEP 684)
# 각 서브인터프리터가 독립적인 GIL을 가짐
# → 진정한 멀티코어 Python 병렬 처리 가능해짐
```

### 8.2 프로세스 간 공유 메모리

```python
from multiprocessing import shared_memory, Process
import numpy as np

def worker(shm_name: str, shape: tuple, dtype: str, result_idx: int):
    """공유 메모리를 통해 대용량 배열 교환 — 복사 없음"""
    shm = shared_memory.SharedMemory(name=shm_name)
    arr = np.ndarray(shape, dtype=dtype, buffer=shm.buf)
    # 배열 수정 (같은 물리 메모리에 반영)
    arr[result_idx] = arr[result_idx] ** 2
    shm.close()

shm = shared_memory.SharedMemory(create=True, size=8 * 1000)
data = np.ndarray((1000,), dtype=np.float64, buffer=shm.buf)
data[:] = np.arange(1000, dtype=np.float64)

procs = [
    Process(target=worker, args=(shm.name, (1000,), 'float64', i))
    for i in range(0, 100, 10)
]
for p in procs: p.start()
for p in procs: p.join()

print(data[0], data[10])  # 0.0, 100.0 — 제곱 적용됨
shm.close()
shm.unlink()  # 반드시 명시적으로 해제
```

---

## Chapter 9. 디자인 패턴과 SOLID 원칙의 Python적 구현

### 9.1 전략 패턴 + 프로토콜

```python
from typing import Protocol
import numpy as np

class SamplingStrategy(Protocol):
    def sample(self, logits: np.ndarray, temperature: float) -> int: ...

class GreedySampling:
    def sample(self, logits: np.ndarray, temperature: float) -> int:
        return int(np.argmax(logits))

class TemperatureSampling:
    def sample(self, logits: np.ndarray, temperature: float) -> int:
        scaled = logits / max(temperature, 1e-9)
        probs = np.exp(scaled - np.max(scaled))
        probs /= probs.sum()
        return int(np.random.choice(len(probs), p=probs))

class TopKSampling:
    def __init__(self, k: int = 50):
        self.k = k
    
    def sample(self, logits: np.ndarray, temperature: float) -> int:
        top_k_idx = np.argpartition(logits, -self.k)[-self.k:]
        top_k_logits = logits[top_k_idx]
        scaled = top_k_logits / max(temperature, 1e-9)
        probs = np.exp(scaled - np.max(scaled))
        probs /= probs.sum()
        local_idx = np.random.choice(self.k, p=probs)
        return int(top_k_idx[local_idx])

class LLMDecoder:
    def __init__(self, strategy: SamplingStrategy):
        self._strategy = strategy
    
    def set_strategy(self, strategy: SamplingStrategy) -> None:
        self._strategy = strategy
    
    def decode_step(self, logits: np.ndarray, temperature: float = 1.0) -> int:
        return self._strategy.sample(logits, temperature)
```

### 9.2 옵저버 패턴 — 이벤트 드리븐 MLOps 파이프라인

```python
from typing import Callable, Any
from collections import defaultdict
import asyncio

class EventBus:
    """비동기 이벤트 버스 — MLOps 파이프라인 오케스트레이션"""
    
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = defaultdict(list)
    
    def subscribe(self, event: str, handler: Callable) -> None:
        self._subscribers[event].append(handler)
    
    def unsubscribe(self, event: str, handler: Callable) -> None:
        self._subscribers[event].remove(handler)
    
    async def publish(self, event: str, payload: Any) -> None:
        handlers = self._subscribers.get(event, [])
        await asyncio.gather(*[
            h(payload) if asyncio.iscoroutinefunction(h) 
            else asyncio.to_thread(h, payload)
            for h in handlers
        ])

# 사용 예
bus = EventBus()

async def log_metrics(payload):
    print(f"[Metrics] {payload}")

async def trigger_retraining(payload):
    if payload.get("accuracy") < 0.9:
        print(f"[Alert] 정확도 하락 → 재학습 트리거")

bus.subscribe("model.evaluation", log_metrics)
bus.subscribe("model.evaluation", trigger_retraining)

asyncio.run(bus.publish("model.evaluation", {"accuracy": 0.87, "loss": 0.43}))
```

---

## Chapter 10. Python 데이터 직렬화와 스키마 설계

### 10.1 Pydantic v2 — Rust 기반 초고속 검증

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from pydantic import ConfigDict
from typing import Annotated
import numpy as np

PositiveFloat = Annotated[float, Field(gt=0)]

class TrainingConfig(BaseModel):
    model_config = ConfigDict(
        frozen=True,          # 불변 객체 (hashable)
        populate_by_name=True,
        json_schema_extra={"examples": [{"learning_rate": 0.001}]}
    )
    
    learning_rate: PositiveFloat = Field(
        default=1e-4,
        alias="lr",           # JSON에서 "lr"로도 받을 수 있음
        description="Adam optimizer learning rate"
    )
    batch_size: int = Field(default=32, ge=1, le=8192)
    model_name: str = Field(default="gpt2", pattern=r'^[a-z0-9\-]+$')
    mixed_precision: bool = False
    gradient_clip: float | None = None
    
    @field_validator('batch_size')
    @classmethod
    def batch_size_must_be_power_of_2(cls, v: int) -> int:
        if v & (v - 1) != 0:
            raise ValueError(f"batch_size는 2의 거듭제곱이어야 합니다. 입력: {v}")
        return v
    
    @model_validator(mode='after')
    def check_gradient_clip_with_mixed_precision(self) -> 'TrainingConfig':
        if self.mixed_precision and self.gradient_clip is None:
            # mixed precision에서는 gradient clipping 권장
            object.__setattr__(self, 'gradient_clip', 1.0)
        return self

config = TrainingConfig(lr=0.001, batch_size=64, mixed_precision=True)
print(config.model_dump_json(by_alias=True))
```

---

# 2부. 해외 대기업 면접 Q&A 35선

---

## 🟠 Amazon (SDE / Applied Scientist)

---

**Q1. Python의 GIL이란 무엇이며, CPU-bound 작업에서 병렬성을 달성하는 방법을 설명하세요.**

**A.** GIL(Global Interpreter Lock)은 CPython 인터프리터가 한 번에 하나의 OS 스레드만 Python 바이트코드를 실행하도록 보장하는 뮤텍스입니다. 이것은 CPython의 참조 카운팅 기반 메모리 관리가 thread-safe하게 동작하도록 하기 위해 도입되었습니다.

CPU-bound 작업에서 GIL은 멀티스레딩이 실질적인 병렬성을 제공하지 못하게 막습니다. 해결책은 세 가지입니다. 첫째, `multiprocessing` 모듈을 사용하여 별도 Python 인터프리터 프로세스를 생성하면 각 프로세스는 독립적인 GIL을 가집니다. 둘째, NumPy, SciPy, PyTorch 같은 C 확장 라이브러리는 연산 중 GIL을 해제하므로 멀티스레딩이 효과적입니다. 셋째, Cython, Numba, C 확장을 직접 작성하여 `Py_BEGIN_ALLOW_THREADS` / `Py_END_ALLOW_THREADS` 매크로로 GIL을 명시적으로 해제할 수 있습니다. Python 3.12에서 도입된 Per-Interpreter GIL(PEP 684)은 향후 이 제약을 근본적으로 완화할 것입니다.

**면접 포인트:** 단순히 "multiprocessing 쓰면 됩니다"로 끝내지 말고, GIL의 근본 원인, I/O-bound vs CPU-bound 차이, Python 3.12의 변화 방향까지 언급하면 고점을 받습니다.

---

**Q2. Python `__new__`와 `__init__`의 차이를 설명하고, 싱글턴 패턴을 `__new__`로 구현하세요.**

**A.** `__new__`는 객체의 메모리를 할당하고 인스턴스를 생성하는 클래스 메서드입니다. `__init__`은 이미 생성된 인스턴스를 초기화합니다. `__new__`가 같은 클래스의 인스턴스를 반환하지 않으면 `__init__`은 호출되지 않습니다.

```python
class Singleton:
    _instance = None
    _initialized = False
    
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self, value: int):
        # __new__가 기존 인스턴스를 반환해도 __init__은 매번 호출됨
        if not self.__class__._initialized:
            self.value = value
            self.__class__._initialized = True

s1 = Singleton(42)
s2 = Singleton(99)
print(s1 is s2)      # True
print(s1.value)      # 42 (재초기화 방지)
```

실무에서는 이보다 thread-safe한 구현(double-checked locking + threading.Lock)이나 모듈 레벨 싱글턴 패턴이 더 권장됩니다.

---

**Q3. Python의 `yield from`과 `asyncio`의 관계를 설명하세요. `await`는 내부적으로 어떻게 동작합니까?**

**A.** `await`는 사실 `yield from`의 문법적 특화입니다. `async def` 함수는 내부적으로 코루틴 제너레이터를 생성하며, `await expr`은 `yield from expr.__await__()`로 변환됩니다.

`asyncio`의 이벤트 루프는 코루틴을 `Task`로 감싸 실행하다가 `await`를 만나면 현재 코루틴의 프레임을 저장하고 이벤트 루프로 제어권을 반환합니다. 이벤트 루프는 `selectors.select()`(내부적으로 epoll/kqueue)로 I/O 이벤트를 감시하다가 해당 파일 디스크립터가 준비되면 대기 중인 코루틴을 깨웁니다.

핵심은 `await`가 OS 스레드를 블록하지 않는다는 점입니다. 단지 Python 실행 프레임을 일시 정지시킬 뿐입니다. 이 덕분에 수천 개의 동시 연결을 단일 스레드에서 처리할 수 있습니다.

---

**Q4. 대용량 로그 파일(100GB)을 메모리 효율적으로 처리하는 Python 코드를 작성하세요.**

**A.** 제너레이터 파이프라인을 사용하면 전체 파일을 메모리에 올리지 않고 스트리밍 처리할 수 있습니다.

```python
from typing import Generator, Iterator
import re
from collections import Counter

def read_lines(path: str, encoding: str = 'utf-8') -> Generator[str, None, None]:
    """파일을 한 줄씩 스트리밍 — 메모리 O(1)"""
    with open(path, encoding=encoding, buffering=1 << 20) as f:  # 1MB 버퍼
        yield from f

def parse_log_line(line: str) -> dict | None:
    pattern = r'(\d{4}-\d{2}-\d{2}) (\w+) (.+)'
    m = re.match(pattern, line.rstrip())
    if m:
        return {"date": m[1], "level": m[2], "message": m[3]}
    return None

def filter_errors(records: Iterator[dict | None]) -> Generator[dict, None, None]:
    for r in records:
        if r and r["level"] == "ERROR":
            yield r

def count_by_date(records: Iterator[dict]) -> Counter:
    counter = Counter()
    for r in records:
        counter[r["date"]] += 1
    return counter

# 파이프라인 조합 — 메모리 사용량 O(unique_dates) 수준
pipeline = filter_errors(map(parse_log_line, read_lines("app.log")))
result = count_by_date(pipeline)
```

추가적으로, 멀티코어를 활용하려면 파일을 청크로 나누어 `ProcessPoolExecutor`로 병렬 처리한 뒤 결과를 병합할 수 있습니다.

---

**Q5. Python의 `dataclass`와 `NamedTuple`, `TypedDict`의 차이와 적합한 사용 상황을 설명하세요.**

**A.** 세 가지는 목적과 동작 방식이 다릅니다.

`dataclass`는 일반 클래스를 자동 생성 메서드(`__init__`, `__repr__`, `__eq__`)로 강화한 것으로, 변경 가능한 상태를 가진 객체가 필요할 때 최적입니다. `frozen=True`로 불변으로 만들 수 있고 상속도 지원합니다. ML 모델의 configuration 객체나 학습 결과 저장에 적합합니다.

`NamedTuple`은 `tuple`의 서브클래스로 불변이며, 인덱스와 이름 모두로 접근 가능합니다. 튜플이므로 `hash`가 가능하고 딕셔너리 키로 사용할 수 있습니다. `struct`처럼 메모리 레이아웃이 중요하거나 반환값이 여러 필드를 가질 때 적합합니다.

`TypedDict`는 순수하게 타입 검사 목적으로, 런타임에는 일반 `dict`와 동일합니다. 기존 딕셔너리 기반 API와의 호환성을 유지하면서 타입 안전성을 추가할 때 사용합니다. JSON API 응답 타입 정의에 매우 유용합니다.

---

**Q6. Python에서 메모리 누수를 진단하고 수정하는 방법을 설명하세요.**

**A.** Python 메모리 누수의 주요 원인은 세 가지입니다. 첫째, 순환 참조(Cyclic Reference)로 인해 GC가 수거하지 못하는 경우입니다. `gc.collect()`를 호출하거나 `weakref`를 사용하여 해결합니다. 둘째, 전역 변수나 클래스 변수에 대용량 데이터가 누적되는 경우입니다. 캐시 크기 제한(`lru_cache(maxsize=N)`) 또는 명시적 삭제로 해결합니다. 셋째, C 확장의 메모리 해제 누락입니다.

진단 도구로는 `tracemalloc`(표준 라이브러리), `memory_profiler`, `objgraph`를 사용합니다.

```python
import tracemalloc
import gc

tracemalloc.start()

# 의심되는 코드 실행
snapshot_before = tracemalloc.take_snapshot()
run_workload()
snapshot_after = tracemalloc.take_snapshot()

top_stats = snapshot_after.compare_to(snapshot_before, 'lineno')
for stat in top_stats[:10]:
    print(stat)  # 메모리 증가가 큰 라인 순으로 출력

# 순환 참조 객체 탐지
gc.set_debug(gc.DEBUG_LEAK)
gc.collect()
print(f"추적 불가 객체: {len(gc.garbage)}")
```

---

## 🍎 Apple (Software Engineer)

---

**Q7. Python 디스크립터 프로토콜을 사용하여 thread-safe한 레이지(lazy) 프로퍼티를 구현하세요.**

**A.** 레이지 프로퍼티는 처음 접근할 때만 값을 계산하고 이후에는 캐싱된 값을 반환합니다. 스레드 안전성을 위해 `threading.Lock`을 사용합니다.

```python
import threading
from typing import TypeVar, Callable, Generic, Optional

T = TypeVar('T')

class lazy_property(Generic[T]):
    """스레드 안전한 레이지 디스크립터"""
    
    def __init__(self, func: Callable[..., T]):
        self._func = func
        self._attr_name = f'_lazy_{func.__name__}'
        self._lock_name = f'_lazy_lock_{func.__name__}'
        self.__doc__ = func.__doc__
    
    def __set_name__(self, owner, name):
        self._attr_name = f'_lazy_{name}'
        self._lock_name = f'_lazy_lock_{name}'
    
    def __get__(self, obj, objtype=None) -> T:
        if obj is None:
            return self  # 클래스 레벨 접근
        
        # 잠금 없이 먼저 확인 (fast path)
        value = obj.__dict__.get(self._attr_name, _MISSING := object())
        if value is not _MISSING:
            return value
        
        # Double-checked locking
        lock = obj.__dict__.setdefault(self._lock_name, threading.Lock())
        with lock:
            value = obj.__dict__.get(self._attr_name, _MISSING)
            if value is _MISSING:
                value = self._func(obj)
                obj.__dict__[self._attr_name] = value
        return value

class MLModel:
    def __init__(self, path: str):
        self.path = path
    
    @lazy_property
    def weights(self) -> dict:
        """모델 가중치 — 처음 접근 시에만 디스크에서 로드"""
        print(f"[Loading] {self.path}에서 가중치 로드 중...")
        import time; time.sleep(0.1)  # 시뮬레이션
        return {"layer1": [0.1, 0.2], "layer2": [0.3, 0.4]}
```

---

**Q8. `asyncio.Queue`를 이용한 생산자-소비자 패턴을 구현하고, 백프레셔(backpressure) 처리 방법을 설명하세요.**

**A.** 백프레셔는 소비자가 처리할 수 없는 속도로 생산자가 데이터를 쏟아낼 때 시스템이 과부하되지 않도록 압력을 역방향으로 전달하는 메커니즘입니다.

```python
import asyncio
from dataclasses import dataclass
from typing import AsyncIterator

@dataclass
class InferenceRequest:
    request_id: str
    prompt: str

async def producer(
    queue: asyncio.Queue[InferenceRequest | None],
    requests: list[InferenceRequest]
) -> None:
    for req in requests:
        # maxsize가 찼을 때 put은 자동으로 대기 — 이것이 백프레셔
        await queue.put(req)
        print(f"[Producer] 큐에 추가: {req.request_id}, 크기: {queue.qsize()}")
    
    # 소독 시그널 (sentinel) 전송
    await queue.put(None)

async def consumer(
    queue: asyncio.Queue[InferenceRequest | None],
    worker_id: int
) -> None:
    while True:
        item = await queue.get()
        if item is None:
            await queue.put(None)  # 다른 워커에게 종료 신호 전파
            break
        
        # 실제 추론 처리
        await asyncio.sleep(0.05)  # 모의 추론
        print(f"[Worker-{worker_id}] 처리 완료: {item.request_id}")
        queue.task_done()

async def inference_pipeline(n_workers: int = 4):
    queue: asyncio.Queue[InferenceRequest | None] = asyncio.Queue(maxsize=10)
    
    requests = [InferenceRequest(f"req-{i}", f"prompt {i}") for i in range(20)]
    
    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(queue, requests))
        for i in range(n_workers):
            tg.create_task(consumer(queue, i))
    
    print("모든 추론 완료")

asyncio.run(inference_pipeline())
```

---

**Q9. Python에서 `__getattr__`과 `__getattribute__`의 차이를 설명하고, 동적 프록시 패턴을 구현하세요.**

**A.** `__getattribute__`는 모든 속성 접근 시 무조건 호출됩니다. `__getattr__`는 일반적인 방법으로 속성을 찾지 못했을 때만 호출됩니다 (폴백). `__getattribute__`를 오버라이드할 때는 무한 재귀를 피하기 위해 `super().__getattribute__(name)` 또는 `object.__getattribute__(self, name)`을 사용해야 합니다.

```python
class RemoteModelProxy:
    """원격 모델 메서드를 로컬에서 투명하게 호출하는 프록시"""
    
    def __init__(self, endpoint: str):
        # __setattr__보다 먼저 처리하기 위해 직접 __dict__ 사용
        object.__setattr__(self, '_endpoint', endpoint)
        object.__setattr__(self, '_call_log', [])
    
    def __getattr__(self, name: str):
        # 일반적인 방법으로 못 찾았을 때만 호출됨
        def remote_call(*args, **kwargs):
            self._call_log.append({"method": name, "args": args})
            # 실제로는 HTTP/gRPC 호출
            return f"[Remote] {name}({args}) at {self._endpoint}"
        return remote_call
    
    def __getattribute__(self, name: str):
        # 모든 속성 접근 시 호출 — 접근 로깅
        if not name.startswith('_'):
            # 내부 속성 제외하고 외부 접근만 로깅
            pass
        return object.__getattribute__(self, name)

proxy = RemoteModelProxy("https://model-api.example.com")
proxy.predict([1, 2, 3])   # → __getattr__ 호출 → remote_call 실행
proxy.embed("hello world") # → 마찬가지
print(proxy._call_log)
```

---

**Q10. Python의 ABC(Abstract Base Class)와 Protocol의 차이를 실제 사용 사례와 함께 설명하세요.**

**A.** ABC는 명목적 서브타이핑(Nominal Subtyping)을 강제합니다. 자식 클래스는 반드시 `ABC`를 명시적으로 상속해야 하며, 추상 메서드를 구현하지 않으면 인스턴스화 시 `TypeError`가 발생합니다. 상속 관계가 필요하고 공통 구현(non-abstract 메서드)을 제공해야 할 때 적합합니다.

Protocol은 구조적 서브타이핑(Structural Subtyping)을 지원합니다. 명시적 상속 없이도, 필요한 메서드/속성을 가지면 해당 Protocol을 만족한다고 간주됩니다. 외부 라이브러리의 클래스처럼 수정 불가능한 타입과의 호환성을 정의할 때 강력합니다.

```python
from abc import ABC, abstractmethod
from typing import Protocol, runtime_checkable

# ABC: 공통 구현이 있는 계층 구조
class BaseTransform(ABC):
    @abstractmethod
    def transform(self, x: np.ndarray) -> np.ndarray: ...
    
    def fit_transform(self, x: np.ndarray) -> np.ndarray:
        self.fit(x)  # ABC는 공통 로직을 기본 구현으로 제공 가능
        return self.transform(x)
    
    @abstractmethod
    def fit(self, x: np.ndarray) -> None: ...

# Protocol: 덕타이핑의 정적 버전
@runtime_checkable
class Serializable(Protocol):
    def to_json(self) -> str: ...
    def from_json(cls, data: str) -> 'Serializable': ...

# NumPy 배열도 특정 Protocol을 만족시킬 수 있음 — 상속 없이
def save_to_cache(obj: Serializable, key: str) -> None:
    cache[key] = obj.to_json()
```

---

## 🟢 Nvidia (Deep Learning Infrastructure)

---

**Q11. Python에서 CUDA 커널을 호출하는 방법을 설명하고, CuPy와 Numba CUDA의 차이를 논하세요.**

**A.** Python에서 CUDA를 활용하는 계층은 여러 단계로 나뉩니다. 가장 하위 레벨은 `ctypes`나 `cffi`로 CUDA 공유 라이브러리를 직접 호출하는 것입니다. 그 위에 `PyCUDA`(CUDA C 커널을 Python에서 JIT 컴파일), `CuPy`(NumPy 호환 GPU 배열), `Numba CUDA`(Python 함수를 GPU 커널로 JIT 컴파일)가 있습니다.

CuPy는 NumPy API를 GPU 위에서 실행하며, 기존 NumPy 코드를 `import numpy as np`를 `import cupy as cp`로 바꾸기만 해도 동작합니다. 주로 배열 연산과 행렬 연산에 최적화되어 있습니다.

```python
import cupy as cp
import numpy as np

# CuPy: NumPy를 GPU로
x_gpu = cp.random.randn(10000, 512)  # GPU 메모리
y_gpu = cp.dot(x_gpu, x_gpu.T)       # cuBLAS 호출
y_cpu = cp.asnumpy(y_gpu)            # GPU → CPU 전송

# Numba CUDA: Python 함수를 GPU 커널로
from numba import cuda

@cuda.jit
def vector_add_kernel(a, b, out):
    idx = cuda.grid(1)  # 스레드 인덱스 계산
    if idx < a.shape[0]:
        out[idx] = a[idx] + b[idx]

n = 1_000_000
a_gpu = cuda.to_device(np.ones(n))
b_gpu = cuda.to_device(np.ones(n))
out_gpu = cuda.device_array(n)

threads_per_block = 256
blocks = (n + threads_per_block - 1) // threads_per_block
vector_add_kernel[blocks, threads_per_block](a_gpu, b_gpu, out_gpu)
result = out_gpu.copy_to_host()
```

Numba는 커스텀 연산이 필요하거나 특정 메모리 접근 패턴을 최적화할 때 강력합니다. CuPy는 표준 수치 연산에서 더 편리하고 성숙된 생태계를 갖습니다.

---

**Q12. Python 멀티프로세싱에서 GPU 텐서를 프로세스 간에 공유하는 올바른 방법은 무엇입니까?**

**A.** CUDA 텐서(GPU 메모리)를 프로세스 간에 공유하는 것은 복잡합니다. 세 가지 접근 방식이 있습니다.

첫 번째는 PyTorch의 `tensor.share_memory_()`와 `multiprocessing.Queue`를 사용하는 방법으로, CPU 공유 메모리를 거쳐 전달합니다. 두 번째는 CUDA IPC(Inter-Process Communication) 핸들을 사용하는 방법으로, `torch.multiprocessing`이 자동으로 처리합니다. 세 번째는 프로세스마다 독립적인 GPU 컨텍스트를 갖고 필요한 데이터만 CPU를 거쳐 동기화하는 방법입니다.

```python
import torch
import torch.multiprocessing as mp

def worker(rank: int, shared_tensor: torch.Tensor, result_queue: mp.Queue):
    """각 워커가 공유 텐서의 일부를 처리"""
    chunk_size = shared_tensor.shape[0] // 4
    start = rank * chunk_size
    local = shared_tensor[start:start + chunk_size].clone().cuda(rank)
    result = local.pow(2).mean()
    result_queue.put((rank, result.item()))

if __name__ == '__main__':
    mp.set_start_method('spawn')  # CUDA를 사용할 때는 반드시 'spawn' 또는 'forkserver'
    
    data = torch.randn(10000, 512)
    data.share_memory_()  # 공유 메모리에 배치
    
    queue = mp.Queue()
    procs = [mp.Process(target=worker, args=(i, data, queue)) for i in range(4)]
    for p in procs: p.start()
    for p in procs: p.join()
    
    results = [queue.get() for _ in range(4)]
```

**주의:** `fork` 방식으로 CUDA 컨텍스트를 가진 프로세스를 복제하면 CUDA 상태가 손상될 수 있습니다. 반드시 `spawn`을 사용해야 합니다.

---

**Q13. Python에서 컴파일된 확장 모듈(C Extension)을 만들 때 메모리 관리에서 가장 주의해야 할 점은 무엇입니까?**

**A.** C 확장에서 Python 객체를 다룰 때 가장 중요한 것은 참조 카운트를 정확히 관리하는 것입니다. 모든 `PyObject*`는 "소유된 참조(owned reference)"와 "빌린 참조(borrowed reference)"의 개념으로 구분됩니다.

소유된 참조는 `Py_DECREF`를 호출할 책임이 있습니다. `PyLong_FromLong`, `PyList_New` 등 `New`나 `From`으로 시작하는 함수들은 소유된 참조를 반환합니다. 반면 `PyList_GetItem`은 빌린 참조를 반환하므로 `Py_INCREF`를 명시적으로 호출해야 합니다.

```c
// 잘못된 예 — 메모리 누수
static PyObject* bad_function(PyObject* self, PyObject* args) {
    PyObject* list = PyList_New(3);
    // ... list를 채우는 작업 ...
    // 에러 발생 시 list를 Py_DECREF하지 않으면 누수
    return list;
}

// 올바른 예 — 에러 처리 포함
static PyObject* good_function(PyObject* self, PyObject* args) {
    PyObject* list = PyList_New(3);
    if (!list) return NULL;  // 할당 실패
    
    PyObject* item = PyLong_FromLong(42);
    if (!item) {
        Py_DECREF(list);  // 에러 시 이미 생성된 객체 해제
        return NULL;
    }
    PyList_SET_ITEM(list, 0, item);  // list가 item의 소유권을 가져감
    // item을 별도로 Py_DECREF하면 안 됨 (double-free)
    
    return list;
}
```

pybind11이나 Cython을 사용하면 이런 저수준 참조 관리를 대부분 자동화할 수 있습니다.

---

## 🔵 Google (Software Engineer / Research Engineer)

---

**Q14. Python의 MRO(Method Resolution Order)를 설명하고 C3 선형화 알고리즘의 동작 원리를 서술하세요.**

**A.** MRO는 다중 상속 시 메서드를 찾는 순서입니다. Python 2.3부터 C3 선형화 알고리즘을 사용합니다. C3의 핵심 규칙은 자식 클래스가 부모보다 먼저 오고, 부모 클래스들이 선언된 순서가 유지되어야 한다는 것입니다.

```python
class A:
    def method(self): return "A"

class B(A):
    def method(self): return "B"

class C(A):
    def method(self): return "C"

class D(B, C):  # 다이아몬드 상속
    pass

# D의 MRO: D → B → C → A → object
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)

# C3 계산 과정:
# L[D] = D + merge(L[B], L[C], [B, C])
# L[B] = [B, A, object]
# L[C] = [C, A, object]
# merge([B,A,object], [C,A,object], [B,C]):
#   1. B는 어느 리스트의 tail에도 없음 → 선택, 제거
#   2. C는 어느 리스트의 tail에도 없음 → 선택, 제거
#   3. A → 선택, object → 선택
# 결과: [D, B, C, A, object]

print(D().method())  # "B" — MRO 첫 번째 발견인 B.method 호출
```

`super()`를 이해하려면 MRO가 필수입니다. `super()`는 "나 자신을 제외한 MRO 순서에서 다음 클래스"를 의미합니다.

---

**Q15. Python의 `contextlib.asynccontextmanager`를 이용하여 분산 락(Distributed Lock)을 구현하세요.**

**A.** Redis를 이용한 분산 락은 MLOps 파이프라인에서 모델 업데이트나 공유 리소스 접근 시 필수입니다.

```python
import asyncio
import contextlib
import uuid
from typing import AsyncIterator

# 실제 구현에서는 aioredis 사용
class MockRedis:
    def __init__(self):
        self._store = {}
        self._lock = asyncio.Lock()
    
    async def set(self, key, value, nx=False, ex=None):
        async with self._lock:
            if nx and key in self._store:
                return False
            self._store[key] = value
            return True
    
    async def delete(self, key, expected_value=None):
        async with self._lock:
            if expected_value and self._store.get(key) != expected_value:
                return False
            self._store.pop(key, None)
            return True

@contextlib.asynccontextmanager
async def distributed_lock(
    redis: MockRedis,
    lock_name: str,
    timeout: float = 10.0,
    retry_interval: float = 0.1
) -> AsyncIterator[str]:
    """
    Redis 기반 분산 락 컨텍스트 매니저
    - 고유 token으로 락 소유권 보장 (다른 프로세스의 실수로 인한 해제 방지)
    - timeout으로 데드락 방지
    """
    token = str(uuid.uuid4())
    acquired = False
    
    try:
        # 락 획득 시도 (재시도 포함)
        deadline = asyncio.get_event_loop().time() + timeout
        while asyncio.get_event_loop().time() < deadline:
            acquired = await redis.set(
                lock_name, token, nx=True, ex=int(timeout)
            )
            if acquired:
                break
            await asyncio.sleep(retry_interval)
        
        if not acquired:
            raise TimeoutError(f"락 획득 시간 초과: {lock_name}")
        
        yield token  # 컨텍스트 블록 실행
    
    finally:
        if acquired:
            # 내 토큰인 경우에만 해제 (원자적 연산 필요 → 실제로는 Lua 스크립트)
            await redis.delete(lock_name, expected_value=token)

# 사용 예
async def update_model_weights(redis: MockRedis, model_id: str):
    async with distributed_lock(redis, f"model_lock:{model_id}", timeout=30.0):
        print(f"모델 {model_id} 업데이트 중...")
        await asyncio.sleep(1.0)  # 실제 업데이트 작업
        print(f"모델 {model_id} 업데이트 완료")
```

---

**Q16. Python에서 사용자 정의 `__eq__`와 `__hash__`를 구현할 때 지켜야 할 불변식(invariant)을 설명하세요.**

**A.** Python의 해싱 계약(contract)은 두 가지 불변식을 요구합니다. 첫째, `a == b`이면 반드시 `hash(a) == hash(b)`여야 합니다. 둘째, `hash(a) != hash(b)`이면 `a != b`가 보장됩니다(역은 성립하지 않아도 됨 — 해시 충돌 허용).

`__eq__`를 정의하면 `__hash__`가 자동으로 `None`으로 설정됩니다(unhashable). 딕셔너리 키나 집합 원소로 사용하려면 반드시 `__hash__`도 함께 구현해야 합니다.

```python
class EmbeddingVector:
    def __init__(self, data: tuple[float, ...], model_id: str):
        self._data = data
        self.model_id = model_id
    
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, EmbeddingVector):
            return NotImplemented  # ← NotImplemented 반환이 올바름 (False가 아님)
        return self._data == other._data and self.model_id == other.model_id
    
    def __hash__(self) -> int:
        # mutable 필드(model_id)도 hash에 포함해야 일관성 유지
        # 단, hash 계산 후 _data나 model_id를 변경하면 안 됨!
        return hash((self._data, self.model_id))
    
    def __repr__(self) -> str:
        return f"EmbeddingVector(dim={len(self._data)}, model={self.model_id})"

v1 = EmbeddingVector((0.1, 0.2, 0.3), "gpt-4")
v2 = EmbeddingVector((0.1, 0.2, 0.3), "gpt-4")

print(v1 == v2)         # True
print(hash(v1) == hash(v2))  # True (불변식 만족)
seen = {v1, v2}
print(len(seen))        # 1 (같은 값이므로 중복 제거)
```

중요한 점은 `mutable` 객체에서 `__hash__`를 구현하면 딕셔너리나 집합에 넣은 후 상태가 변경될 경우 해당 객체를 더 이상 찾을 수 없게 됩니다. 이것이 `list`와 `dict`가 기본적으로 unhashable한 이유입니다.

---

**Q17. Python에서 함수 호출 오버헤드를 최소화하는 기법들을 CPython 내부 구조 관점에서 설명하세요.**

**A.** CPython에서 함수 호출 시 발생하는 비용은 상당합니다. 새 `PyFrameObject` 생성, 지역 변수 공간 할당, 인수 개수 및 타입 검사, 키워드 인자 처리 등이 포함됩니다.

최적화 기법으로는 다음이 있습니다. 첫째, 자주 호출되는 전역 함수를 로컬 변수에 바인딩합니다(`_append = list.append`). 이는 `LOAD_GLOBAL`(딕셔너리 조회)을 `LOAD_FAST`(배열 인덱스 접근)로 대체합니다. 둘째, `map`, `filter`보다 내장 C 함수를 직접 사용합니다. 셋째, 인라이닝이 가능한 단순 함수는 Python 레벨보다 C 확장으로 구현합니다. 넷째, Python 3.11+의 specializing adaptive interpreter(`CALL_PY_EXACT_ARGS` 같은 특화 opcode)를 활용하면 자동으로 최적화됩니다.

```python
import timeit

# 전역 조회 vs 로컬 조회 비교
def slow_sum(n):
    total = 0
    for i in range(n):
        total += abs(i - n // 2)  # abs는 전역 조회
    return total

def fast_sum(n):
    _abs = abs  # 로컬에 바인딩
    total = 0
    for i in range(n):
        total += _abs(i - n // 2)
    return total

# Python 3.11+ CALL_INTRINSIC 최적화로 차이가 줄었지만 여전히 유효
print(timeit.timeit(lambda: slow_sum(100000), number=100))
print(timeit.timeit(lambda: fast_sum(100000), number=100))
```

---

**Q18. Python `dataclass(frozen=True)` vs `NamedTuple`의 해싱 동작 차이와 성능 비교를 하세요.**

**A.** `frozen=True` dataclass는 `__setattr__`과 `__delattr__`을 재정의하여 수정 시 `FrozenInstanceError`를 발생시키며, `__hash__`를 자동으로 생성합니다. 반면 `NamedTuple`은 실제로 `tuple`의 서브클래스이므로 인덱스 접근, 언패킹, 슬라이싱이 가능합니다.

```python
from dataclasses import dataclass
from typing import NamedTuple
import sys, timeit

@dataclass(frozen=True)
class FrozenPoint:
    x: float
    y: float

class NamedPoint(NamedTuple):
    x: float
    y: float

fp = FrozenPoint(1.0, 2.0)
np_ = NamedPoint(1.0, 2.0)

# 메모리 크기 비교
print(sys.getsizeof(fp))   # ~56 bytes (dataclass 인스턴스)
print(sys.getsizeof(np_))  # ~72 bytes (tuple + NamedTuple 오버헤드)

# 하지만 NamedTuple은 튜플이므로 인덱스 접근 가능
print(np_[0])     # 1.0
print(np_.x)      # 1.0
# fp[0]은 TypeError!

# 해싱 성능
print(timeit.timeit(lambda: hash(fp), number=1_000_000))
print(timeit.timeit(lambda: hash(np_), number=1_000_000))
# NamedTuple이 약간 빠름 (tuple의 네이티브 해시 알고리즘 사용)
```

상속이 필요하거나 메서드를 추가해야 한다면 `frozen dataclass`, 단순 값 컨테이너로 튜플 호환성이 필요하다면 `NamedTuple`을 선택하세요.

---

## 🔷 Microsoft (Principal Software Engineer / Azure ML)

---

**Q19. Python `__init_subclass__`를 이용하여 플러그인 아키텍처를 구현하세요.**

**A.** `__init_subclass__`는 Python 3.6에서 추가된 클래스 훅으로, 서브클래스가 정의될 때마다 슈퍼클래스에서 자동으로 호출됩니다. 메타클래스보다 간단하게 서브클래스 등록, 검증, 변환 로직을 구현할 수 있습니다.

```python
from typing import ClassVar, Type

class MLPlugin:
    """자동 플러그인 등록 시스템"""
    _registry: ClassVar[dict[str, Type['MLPlugin']]] = {}
    _required_methods: ClassVar[list[str]] = ['fit', 'predict']
    
    def __init_subclass__(
        cls,
        plugin_name: str | None = None,
        version: str = "1.0",
        **kwargs
    ):
        super().__init_subclass__(**kwargs)
        
        # 추상 메서드 구현 여부 확인
        for method in cls._required_methods:
            if not callable(getattr(cls, method, None)):
                raise TypeError(
                    f"{cls.__name__}은 '{method}' 메서드를 구현해야 합니다."
                )
        
        # 레지스트리에 등록
        name = plugin_name or cls.__name__
        if name in cls._registry:
            raise ValueError(f"플러그인 이름 중복: '{name}'")
        
        cls._registry[name] = cls
        cls._version = version
        print(f"[Plugin] '{name}' v{version} 등록됨")
    
    @classmethod
    def create(cls, name: str, **kwargs) -> 'MLPlugin':
        if name not in cls._registry:
            raise KeyError(f"알 수 없는 플러그인: '{name}'")
        return cls._registry[name](**kwargs)
    
    def fit(self, X, y): raise NotImplementedError
    def predict(self, X): raise NotImplementedError

class RandomForestPlugin(MLPlugin, plugin_name="random_forest", version="2.1"):
    def fit(self, X, y):
        self._model = {"trained": True}
    
    def predict(self, X):
        return [0] * len(X)

class XGBoostPlugin(MLPlugin, plugin_name="xgboost", version="1.7"):
    def fit(self, X, y):
        self._booster = {"rounds": 100}
    
    def predict(self, X):
        return [1] * len(X)

# 팩토리 패턴
model = MLPlugin.create("random_forest")
model.fit([[1, 2], [3, 4]], [0, 1])
print(model.predict([[5, 6]]))
```

---

**Q20. Python의 `struct` 모듈과 `ctypes`를 이용한 바이너리 프로토콜 파싱을 설명하세요.**

**A.** `struct` 모듈은 Python 값을 C 구조체의 바이너리 표현으로 직렬화/역직렬화합니다. 네트워크 프로토콜, 바이너리 파일 파싱, 임베디드 시스템과의 통신에 필수입니다.

```python
import struct
import ctypes
from dataclasses import dataclass

# 커스텀 ML 모델 체크포인트 파일 파싱
# 헤더 형식: magic(4B) | version(2B) | n_layers(4B) | timestamp(8B)
HEADER_FORMAT = '!4sHIq'  # ! = big-endian, 4s = 4-char string
HEADER_SIZE = struct.calcsize(HEADER_FORMAT)

@dataclass
class CheckpointHeader:
    magic: bytes
    version: int
    n_layers: int
    timestamp: int

def parse_checkpoint_header(data: bytes) -> CheckpointHeader:
    if len(data) < HEADER_SIZE:
        raise ValueError(f"헤더 크기 부족: {len(data)} < {HEADER_SIZE}")
    
    magic, version, n_layers, timestamp = struct.unpack_from(
        HEADER_FORMAT, data, offset=0
    )
    
    if magic != b'MDLC':
        raise ValueError(f"잘못된 매직 바이트: {magic}")
    
    return CheckpointHeader(magic, version, n_layers, timestamp)

# 바이너리 직렬화
def write_checkpoint_header(header: CheckpointHeader) -> bytes:
    return struct.pack(
        HEADER_FORMAT,
        header.magic, header.version, header.n_layers, header.timestamp
    )

# ctypes: C 구조체를 Python에서 직접 정의
class LayerDescriptor(ctypes.Structure):
    _fields_ = [
        ("layer_id", ctypes.c_uint32),
        ("n_params", ctypes.c_uint64),
        ("dtype", ctypes.c_uint8),
        ("_padding", ctypes.c_uint8 * 7),  # 8바이트 정렬
    ]
    # _pack_ = 1  # 패딩 없이 빡빡하게 패킹할 경우

desc = LayerDescriptor(layer_id=0, n_params=768*768, dtype=2)
print(ctypes.sizeof(LayerDescriptor))  # 16 bytes (정렬 포함)
```

---

**Q21. Python에서 타입 안전한 이벤트 시스템(Event System)을 구현하세요. 이때 메모리 누수를 방지하는 방법도 포함하세요.**

**A.** 이벤트 시스템에서 메모리 누수의 주요 원인은 이벤트 핸들러가 구독 취소 없이 삭제될 때 강한 참조로 인해 구독자 객체가 GC되지 않는 것입니다. `weakref`를 사용하면 이를 방지할 수 있습니다.

```python
import weakref
from typing import TypeVar, Generic, Callable, Any
from collections import defaultdict
import inspect

E = TypeVar('E')

class WeakEventSystem:
    """약한 참조를 이용한 메모리 누수 방지 이벤트 시스템"""
    
    def __init__(self):
        # 메서드는 WeakMethod, 함수는 WeakRef, 람다는 강한 참조(수명 관리 필요)
        self._handlers: dict[str, list] = defaultdict(list)
    
    def on(self, event: str, handler: Callable) -> Callable:
        """구독 등록 — 약한 참조 사용"""
        if inspect.ismethod(handler):
            ref = weakref.WeakMethod(handler, self._make_cleanup(event))
        else:
            ref = weakref.ref(handler, self._make_cleanup(event))
        self._handlers[event].append(ref)
        return handler
    
    def _make_cleanup(self, event: str):
        """핸들러가 GC될 때 자동으로 구독 목록에서 제거"""
        def cleanup(dead_ref):
            self._handlers[event] = [
                r for r in self._handlers[event] if r is not dead_ref
            ]
        return cleanup
    
    def emit(self, event: str, *args, **kwargs) -> None:
        dead_refs = []
        for ref in self._handlers.get(event, []):
            handler = ref()
            if handler is None:
                dead_refs.append(ref)
            else:
                handler(*args, **kwargs)
        # 죽은 참조 정리
        for ref in dead_refs:
            self._handlers[event].remove(ref)

events = WeakEventSystem()

class ModelMonitor:
    def on_accuracy_drop(self, accuracy: float):
        print(f"[Alert] 정확도 하락: {accuracy:.3f}")

monitor = ModelMonitor()
events.on("accuracy.drop", monitor.on_accuracy_drop)
events.emit("accuracy.drop", 0.72)

del monitor  # monitor 삭제
import gc; gc.collect()
# 이후 emit은 아무것도 하지 않음 — 메모리 누수 없음
events.emit("accuracy.drop", 0.65)
```

---

**Q22. Python에서 `ast` 모듈을 이용한 코드 분석 및 변환 예제를 보여주세요.**

**A.** AST(Abstract Syntax Tree) 조작은 코드 린팅, 자동 리팩토링, 커스텀 최적화 패스, 도메인 특화 언어(DSL) 구현에 활용됩니다.

```python
import ast
import textwrap

class PrintToLogTransformer(ast.NodeTransformer):
    """print() 호출을 logging.info()로 자동 변환하는 AST 트랜스포머"""
    
    def visit_Call(self, node: ast.Call) -> ast.AST:
        self.generic_visit(node)  # 자식 노드 먼저 방문
        
        if (isinstance(node.func, ast.Name) and 
            node.func.id == 'print'):
            # print(args) → logging.info(args)
            new_node = ast.Call(
                func=ast.Attribute(
                    value=ast.Name(id='logging', ctx=ast.Load()),
                    attr='info',
                    ctx=ast.Load()
                ),
                args=node.args,
                keywords=[]
            )
            return ast.copy_location(new_node, node)
        return node

source = textwrap.dedent("""
    def train_epoch(loss):
        print(f"Loss: {loss:.4f}")
        print("Epoch 완료")
""")

tree = ast.parse(source)
transformer = PrintToLogTransformer()
new_tree = transformer.visit(tree)
ast.fix_missing_locations(new_tree)

# 변환된 코드 출력
import astunparse  # pip install astunparse
# print(astunparse.unparse(new_tree))

# 변환된 코드 실행
code = compile(new_tree, "<string>", "exec")
namespace = {"logging": __import__("logging")}
exec(code, namespace)
namespace['train_epoch'](0.1234)
```

---

## 🔴 Tesla (Autopilot / AI Infrastructure)

---

**Q23. Python에서 실시간 센서 데이터 스트림을 처리하는 비동기 파이프라인을 구현하세요. 지연 시간(latency)과 처리량(throughput)을 모두 최적화하는 방법을 포함하세요.**

**A.** 실시간 처리에서 지연 시간과 처리량은 trade-off 관계입니다. 배치 크기를 키우면 처리량이 올라가지만 지연 시간이 증가합니다. adaptive batching으로 이 균형을 동적으로 조절할 수 있습니다.

```python
import asyncio
import time
from collections import deque
from dataclasses import dataclass, field
from typing import Callable, AsyncIterator

@dataclass
class SensorFrame:
    timestamp: float
    camera_id: int
    data: bytes  # 실제로는 numpy 배열
    
@dataclass 
class BatchMetrics:
    batch_size: int
    wait_time_ms: float
    process_time_ms: float

class AdaptiveBatcher:
    """
    지연 시간과 처리량을 동적으로 균형 잡는 어댑티브 배처
    - max_batch_size: 최대 배치 크기
    - max_wait_ms: 최대 대기 시간 (이 시간이 지나면 작은 배치도 처리)
    """
    
    def __init__(
        self,
        processor: Callable,
        max_batch_size: int = 32,
        max_wait_ms: float = 10.0
    ):
        self._queue: asyncio.Queue[SensorFrame] = asyncio.Queue(maxsize=1000)
        self._processor = processor
        self._max_batch = max_batch_size
        self._max_wait = max_wait_ms / 1000.0
        self._metrics: deque[BatchMetrics] = deque(maxlen=100)
    
    async def ingest(self, frame: SensorFrame) -> None:
        try:
            self._queue.put_nowait(frame)
        except asyncio.QueueFull:
            # 백프레셔: 오래된 프레임 드롭 (실시간 시스템에서는 최신 데이터 우선)
            self._queue.get_nowait()  # 가장 오래된 것 버림
            await self._queue.put(frame)
    
    async def process_loop(self) -> None:
        while True:
            batch = []
            deadline = asyncio.get_event_loop().time() + self._max_wait
            
            # 첫 번째 아이템 대기
            frame = await self._queue.get()
            batch.append(frame)
            wait_start = time.perf_counter()
            
            # 배치 채우기 (최대 대기 시간까지)
            while len(batch) < self._max_batch:
                remaining = deadline - asyncio.get_event_loop().time()
                if remaining <= 0:
                    break
                try:
                    frame = await asyncio.wait_for(
                        self._queue.get(), timeout=remaining
                    )
                    batch.append(frame)
                except asyncio.TimeoutError:
                    break
            
            wait_ms = (time.perf_counter() - wait_start) * 1000
            
            # 배치 처리
            proc_start = time.perf_counter()
            await self._processor(batch)
            proc_ms = (time.perf_counter() - proc_start) * 1000
            
            self._metrics.append(BatchMetrics(len(batch), wait_ms, proc_ms))
    
    def get_avg_latency_ms(self) -> float:
        if not self._metrics:
            return 0.0
        return sum(m.wait_time_ms + m.process_time_ms for m in self._metrics) / len(self._metrics)
```

---

**Q24. Python에서 Cython 없이 C 수준 성능을 달성하는 방법을 설명하세요 (ctypes, cffi, pybind11 비교).**

**A.** 네이티브 성능을 달성하는 방법은 사용 목적에 따라 다릅니다.

`ctypes`는 표준 라이브러리에 포함되어 있고 컴파일이 필요 없으나, 타입 표현이 번거롭고 에러가 나기 쉽습니다. 기존 공유 라이브러리(`.so`, `.dll`)를 빠르게 호출할 때 적합합니다.

`cffi`는 실제 C 헤더 문법을 그대로 사용하므로 `ctypes`보다 자연스럽고, ABI 모드와 API 모드를 모두 지원합니다.

`pybind11`은 헤더 전용 C++17 라이브러리로, C++ 코드를 Python 확장으로 노출하는 가장 현대적인 방법입니다. STL 컨테이너, 예외, numpy 배열을 자동으로 변환합니다.

```python
# cffi 예제: 커스텀 벡터 연산
import cffi
ffi = cffi.FFI()

ffi.cdef("""
    void dot_product_batch(
        float* A, float* B, float* results,
        int n_vectors, int dim
    );
""")

# 인라인 C 코드 컴파일
lib = ffi.verify("""
    #include <string.h>
    
    void dot_product_batch(
        float* A, float* B, float* results,
        int n_vectors, int dim
    ) {
        for (int i = 0; i < n_vectors; i++) {
            float sum = 0.0f;
            for (int j = 0; j < dim; j++) {
                sum += A[i*dim + j] * B[i*dim + j];
            }
            results[i] = sum;
        }
    }
""", extra_compile_args=['-O3', '-march=native', '-ffast-math'])

import numpy as np
n, d = 10000, 512
A = np.random.randn(n, d).astype(np.float32)
B = np.random.randn(n, d).astype(np.float32)
results = np.zeros(n, dtype=np.float32)

lib.dot_product_batch(
    ffi.cast("float*", A.ctypes.data),
    ffi.cast("float*", B.ctypes.data),
    ffi.cast("float*", results.ctypes.data),
    n, d
)
```

---

**Q25. Python에서 무한 데이터 스트림을 처리하기 위한 이동 평균, 분위수, 이상치 탐지를 메모리 O(1)로 구현하세요.**

**A.** 스트리밍 통계는 전체 데이터를 저장하지 않고 요약 통계를 점진적으로 업데이트합니다.

```python
import math
from collections import deque

class StreamingStats:
    """Welford 알고리즘 기반 스트리밍 평균/분산 — O(1) 메모리"""
    
    def __init__(self):
        self._n = 0
        self._mean = 0.0
        self._M2 = 0.0   # 분산 계산을 위한 누적 제곱합
    
    def update(self, x: float) -> None:
        self._n += 1
        delta = x - self._mean
        self._mean += delta / self._n
        delta2 = x - self._mean
        self._M2 += delta * delta2
    
    @property
    def mean(self) -> float:
        return self._mean
    
    @property
    def variance(self) -> float:
        return self._M2 / (self._n - 1) if self._n > 1 else 0.0
    
    @property
    def std(self) -> float:
        return math.sqrt(self.variance)

class EWMA:
    """지수 가중 이동 평균 — 최근 데이터에 더 높은 가중치"""
    
    def __init__(self, alpha: float = 0.1):
        self._alpha = alpha
        self._value: float | None = None
    
    def update(self, x: float) -> float:
        if self._value is None:
            self._value = x
        else:
            self._value = self._alpha * x + (1 - self._alpha) * self._value
        return self._value

class ZScoreAnomalyDetector:
    """Z-Score 기반 실시간 이상치 탐지"""
    
    def __init__(self, threshold: float = 3.0, warmup: int = 30):
        self._stats = StreamingStats()
        self._threshold = threshold
        self._warmup = warmup
    
    def update(self, x: float) -> tuple[float, bool]:
        self._stats.update(x)
        
        if self._stats._n < self._warmup:
            return 0.0, False
        
        z_score = abs(x - self._stats.mean) / (self._stats.std + 1e-9)
        is_anomaly = z_score > self._threshold
        return z_score, is_anomaly

# 실시간 사용 예
detector = ZScoreAnomalyDetector(threshold=3.0)
import random

for i in range(1000):
    value = random.gauss(0, 1)
    if i == 500:
        value = 10.0  # 이상치 주입
    
    z, anomaly = detector.update(value)
    if anomaly:
        print(f"[t={i}] 이상치 탐지! value={value:.3f}, z={z:.2f}")
```

---

## 🔧 공통 심화 질문 (전 회사 해당)

---

**Q26. Python의 `__class_getitem__`과 제네릭 클래스의 런타임 동작을 설명하세요.**

**A.** `__class_getitem__`은 `MyClass[int]`처럼 클래스에 대괄호 구독 연산자를 사용할 때 호출됩니다. `list[int]`, `dict[str, float]` 같은 제네릭 타입 힌트가 이것으로 구현됩니다.

```python
from typing import ClassVar, Any

class TypedList:
    """런타임 타입 검사가 있는 제네릭 리스트"""
    _item_type: type | None = None
    
    def __class_getitem__(cls, item_type: type) -> type:
        # 새 클래스를 동적으로 생성하여 타입 정보를 포함
        new_cls = type(
            f'TypedList[{item_type.__name__}]',
            (cls,),
            {'_item_type': item_type}
        )
        return new_cls
    
    def __init__(self):
        self._data: list = []
    
    def append(self, item: Any) -> None:
        if self._item_type and not isinstance(item, self._item_type):
            raise TypeError(
                f"{type(self).__name__}에는 {self._item_type.__name__} 타입만 가능합니다."
            )
        self._data.append(item)

IntList = TypedList[int]
lst = IntList()
lst.append(42)       # OK
# lst.append("hello") # TypeError 발생

# typing 모듈의 Generic은 런타임에 타입 소거(type erasure)됨
from typing import Generic, TypeVar
T = TypeVar('T')

class Container(Generic[T]):
    def __init__(self, value: T) -> None:
        self.value = value

c = Container[int](42)
# 런타임에 Container[int]와 Container[str]는 동일한 클래스
print(type(c))  # <class '__main__.Container'>
print(c.__orig_class__ if hasattr(c, '__orig_class__') else "타입 소거됨")
```

---

**Q27. Python의 `importlib`과 동적 임포트를 활용하여 플러그인 로더를 구현하세요.**

**A.** 동적 임포트는 런타임에 모듈을 로드하는 기법으로, 플러그인 아키텍처, 지연 로딩, 조건부 의존성에 활용됩니다.

```python
import importlib
import importlib.util
import sys
from pathlib import Path
from typing import Any

class PluginLoader:
    """파일 경로로부터 직접 Python 모듈을 로드하는 플러그인 시스템"""
    
    def __init__(self, plugin_dir: str):
        self._plugin_dir = Path(plugin_dir)
        self._loaded: dict[str, Any] = {}
    
    def load(self, plugin_name: str) -> Any:
        """플러그인을 동적으로 로드"""
        if plugin_name in self._loaded:
            return self._loaded[plugin_name]
        
        plugin_path = self._plugin_dir / f"{plugin_name}.py"
        if not plugin_path.exists():
            raise FileNotFoundError(f"플러그인 파일 없음: {plugin_path}")
        
        # spec → module → exec 순서로 모듈 로드
        spec = importlib.util.spec_from_file_location(
            name=f"plugins.{plugin_name}",
            location=plugin_path
        )
        if spec is None or spec.loader is None:
            raise ImportError(f"플러그인 로드 실패: {plugin_name}")
        
        module = importlib.util.module_from_spec(spec)
        sys.modules[spec.name] = module  # sys.modules에 등록해야 pickle 등 작동
        spec.loader.exec_module(module)
        
        self._loaded[plugin_name] = module
        return module
    
    def reload(self, plugin_name: str) -> Any:
        """개발 중 변경 사항 반영을 위한 핫 리로드"""
        if plugin_name in self._loaded:
            old_module = self._loaded.pop(plugin_name)
            # sys.modules에서도 제거하여 완전 재로드 보장
            sys.modules.pop(f"plugins.{plugin_name}", None)
        return self.load(plugin_name)

# 사용 예
loader = PluginLoader("./plugins")
transform_plugin = loader.load("image_transform")
transform_fn = getattr(transform_plugin, 'transform', None)
if transform_fn:
    result = transform_fn(data)
```

---

**Q28. Python에서 `dataclasses.field`의 `default_factory`와 인스턴스 변수 공유 문제를 설명하세요.**

**A.** Python 클래스에서 mutable 객체를 기본값으로 사용하면 모든 인스턴스가 같은 객체를 공유하는 치명적인 버그가 발생합니다. 이것은 Python 초보자에서 고급 개발자까지 실수하는 대표적인 함정입니다.

```python
from dataclasses import dataclass, field
from typing import List

# 잘못된 방법 — 전형적인 Python 함정
class BadModel:
    def __init__(self):
        self.history = []  # 이건 OK (매 호출마다 새 리스트)

# 클래스 변수로 mutable 기본값 — 버그!
class ReallyBadModel:
    shared_history = []  # 모든 인스턴스가 공유!
    
    def log(self, event):
        self.shared_history.append(event)  # 모든 인스턴스에 영향

m1 = ReallyBadModel()
m2 = ReallyBadModel()
m1.log("epoch 1")
print(m2.shared_history)  # ['epoch 1'] — 버그!

# dataclass에서도 같은 함정
# @dataclass
# class BadConfig:
#     history: list = []  # ValueError: mutable default is not allowed

# 올바른 방법 — default_factory
@dataclass
class TrainingState:
    epoch: int = 0
    history: List[float] = field(default_factory=list)  # 인스턴스마다 새 리스트
    hyperparams: dict = field(default_factory=lambda: {
        "lr": 1e-4,
        "warmup_steps": 1000
    })
    
    # init=False: __init__에서 인자로 받지 않음
    _internal_cache: dict = field(default_factory=dict, init=False, repr=False)

s1 = TrainingState()
s2 = TrainingState()
s1.history.append(0.5)
print(s2.history)  # [] — 독립적인 리스트
```

---

**Q29. Python의 `__slots__`을 사용할 때 상속에서 발생하는 문제와 해결책을 설명하세요.**

**A.** `__slots__`은 상속 시 주의가 필요합니다. 부모 클래스가 `__slots__`을 정의하더라도, 자식 클래스가 `__slots__`을 정의하지 않으면 자식 클래스는 자동으로 `__dict__`를 갖게 되어 부모의 `__slots__` 최적화가 무효화됩니다.

```python
import sys

class SlottedBase:
    __slots__ = ('x', 'y')
    
    def __init__(self, x, y):
        self.x = x
        self.y = y

class UnslottedChild(SlottedBase):
    # __slots__ 미정의 → __dict__ 자동 생성 → 최적화 손실
    def __init__(self, x, y, z):
        super().__init__(x, y)
        self.z = z  # __dict__에 저장됨

class SlottedChild(SlottedBase):
    # 부모의 슬롯을 다시 선언하면 안 됨 (중복 슬롯 → 메모리 낭비)
    __slots__ = ('z',)  # 새로운 슬롯만 선언
    
    def __init__(self, x, y, z):
        super().__init__(x, y)
        self.z = z

uc = UnslottedChild(1, 2, 3)
sc = SlottedChild(1, 2, 3)

print(hasattr(uc, '__dict__'))  # True — 최적화 손실
print(hasattr(sc, '__dict__'))  # False — 완전 최적화
print(sys.getsizeof(uc))        # ~56 + __dict__ 오버헤드
print(sys.getsizeof(sc))        # ~56 (슬롯만)

# 믹스인 클래스에서 __slots__ = () 빈 튜플 선언
class LoggingMixin:
    __slots__ = ()  # 새 슬롯 없음, __dict__ 생성도 방지
    
    def log(self, msg):
        print(f"[{self.__class__.__name__}] {msg}")
```

---

**Q30. Python의 `contextvar`(contextvars 모듈)을 이용한 비동기 컨텍스트 전파를 설명하세요.**

**A.** `contextvars.ContextVar`는 asyncio 태스크, 스레드 등 서로 다른 실행 컨텍스트에서 값을 독립적으로 유지합니다. 이는 분산 추적의 `trace_id`, per-request 로깅, 트랜잭션 ID 같은 컨텍스트 정보를 전역 변수 없이 전파할 때 매우 유용합니다.

```python
import asyncio
import contextvars
import uuid
import logging

# 요청 ID — 각 asyncio 태스크에서 독립적으로 유지됨
request_id_var: contextvars.ContextVar[str] = contextvars.ContextVar(
    'request_id', 
    default='unknown'
)

class RequestContextLogger:
    """요청 컨텍스트가 자동으로 포함되는 로거"""
    
    def info(self, msg: str) -> None:
        req_id = request_id_var.get()
        logging.info(f"[req:{req_id}] {msg}")

logger = RequestContextLogger()

async def process_request(request_data: dict) -> dict:
    # 이 태스크에만 적용되는 request_id 설정
    token = request_id_var.set(str(uuid.uuid4())[:8])
    try:
        logger.info(f"요청 처리 시작: {request_data}")
        await asyncio.sleep(0.01)  # I/O 시뮬레이션
        await sub_operation()
        logger.info("요청 처리 완료")
        return {"status": "ok"}
    finally:
        request_id_var.reset(token)  # 이전 값으로 복원

async def sub_operation():
    # 부모 태스크의 request_id를 자동으로 상속
    logger.info("서브 작업 실행 중")
    await asyncio.sleep(0.005)

async def main():
    # 여러 요청을 동시에 처리 — 각자 독립된 request_id를 가짐
    tasks = [
        asyncio.create_task(process_request({"user": f"user_{i}"}))
        for i in range(5)
    ]
    results = await asyncio.gather(*tasks)
    return results

asyncio.run(main())
```

---

**Q31. Python에서 `__prepare__` 메타클래스 훅을 이용하여 클래스 본문의 정의 순서를 추적하는 방법을 설명하세요.**

**A.** 일반적으로 클래스 본문은 딕셔너리로 수집됩니다. `__prepare__`는 이 딕셔너리 대신 사용할 객체를 반환하는 메타클래스 훅입니다. 순서 보존 딕셔너리나 커스텀 매핑을 반환하여 정의 순서 추적, 중복 검사 등을 구현할 수 있습니다.

```python
from collections import OrderedDict
from typing import Any

class OrderTrackingDict(OrderedDict):
    """클래스 속성 정의 순서를 추적하는 특수 딕셔너리"""
    
    def __init__(self):
        super().__init__()
        self._definition_order: list[str] = []
    
    def __setitem__(self, key: str, value: Any) -> None:
        if not key.startswith('_') and key not in self:
            self._definition_order.append(key)
        super().__setitem__(key, value)

class OrderedMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases, **kwargs):
        # 클래스 본문 실행 전에 네임스페이스 딕셔너리를 제공
        return OrderTrackingDict()
    
    def __new__(mcs, name, bases, namespace, **kwargs):
        cls = super().__new__(mcs, name, bases, dict(namespace))
        cls._field_order = namespace._definition_order
        return cls

class DataRecord(metaclass=OrderedMeta):
    # 정의 순서가 보장됨 (Python 3.7+ 기본 dict도 순서 보장하지만
    # 이 방법으로 명시적 추적 및 커스텀 로직 적용 가능)
    timestamp: float
    sensor_id: int
    value: float
    quality: str
    
    def __init__(self, **kwargs):
        for field in self._field_order:
            setattr(self, field, kwargs.get(field))

print(DataRecord._field_order)
# ['timestamp', 'sensor_id', 'value', 'quality'] — 정의 순서 보장
```

이 기법은 SQLAlchemy의 Column 순서 보존, Django Form 필드 순서 제어 등에 실제로 사용됩니다.

---

**Q32. Python에서 함수 합성(Function Composition)과 파이프라인 연산자를 구현하는 방법을 설명하세요.**

**A.** Python에는 Haskell의 `.` 연산자 같은 내장 함수 합성이 없지만, `functools`와 연산자 오버로딩으로 우아하게 구현할 수 있습니다.

```python
from typing import Callable, TypeVar, Generic
from functools import reduce
import operator

A = TypeVar('A')
B = TypeVar('B')
C = TypeVar('C')

class Fn(Generic[A, B]):
    """함수를 일급 시민으로 감싸는 래퍼 — 합성 연산자 제공"""
    
    def __init__(self, func: Callable[[A], B]):
        self._func = func
        self.__doc__ = func.__doc__
        self.__name__ = getattr(func, '__name__', repr(func))
    
    def __call__(self, x: A) -> B:
        return self._func(x)
    
    def __rshift__(self, other: 'Fn[B, C]') -> 'Fn[A, C]':
        """f >> g : f를 먼저 적용, 그 결과에 g 적용"""
        return Fn(lambda x: other(self(x)))
    
    def __lshift__(self, other: 'Fn[C, A]') -> 'Fn[C, B]':
        """f << g : g를 먼저 적용, 그 결과에 f 적용 (수학적 합성 순서)"""
        return Fn(lambda x: self(other(x)))
    
    def __repr__(self) -> str:
        return f"Fn({self.__name__})"

# ML 전처리 파이프라인을 함수 합성으로 표현
import numpy as np

normalize = Fn(lambda x: (x - x.mean()) / (x.std() + 1e-8))
clip = Fn(lambda x: np.clip(x, -3.0, 3.0))
to_float32 = Fn(lambda x: x.astype(np.float32))
add_batch_dim = Fn(lambda x: x[np.newaxis, :])

# 파이프라인: normalize → clip → to_float32 → add_batch_dim
pipeline = normalize >> clip >> to_float32 >> add_batch_dim

data = np.array([1.0, 2.0, 100.0, -50.0, 3.5])
result = pipeline(data)
print(result.shape, result.dtype)  # (1, 5) float32
```

---

**Q33. Python의 `__subclasshook__`을 이용하여 가상 서브클래스 등록 없이 ABC를 만족시키는 방법을 설명하세요.**

**A.** `__subclasshook__`은 `isinstance(obj, ABC)`와 `issubclass(Cls, ABC)` 호출 시 ABC가 직접 판단하는 로직을 정의합니다. 이를 통해 명시적 상속이나 `register()` 없이도 특정 조건을 만족하는 클래스를 자동으로 해당 ABC의 서브클래스로 인식시킬 수 있습니다.

```python
from abc import ABC, abstractmethod

class Iterable(ABC):
    @abstractmethod
    def __iter__(self): ...
    
    @classmethod
    def __subclasshook__(cls, C):
        if cls is Iterable:
            # C 클래스의 MRO 어딘가에 __iter__가 있으면 Iterable로 인정
            if any("__iter__" in B.__dict__ for B in C.__mro__):
                return True
        return NotImplemented  # 기본 판단 로직에 위임

class MyStream:
    """Iterable을 상속하지 않지만 __iter__를 구현함"""
    def __init__(self, data):
        self._data = data
    
    def __iter__(self):
        return iter(self._data)

print(isinstance(MyStream([1, 2, 3]), Iterable))  # True
print(issubclass(MyStream, Iterable))             # True
print(issubclass(list, Iterable))                 # True
print(issubclass(int, Iterable))                  # False

# 이것이 바로 collections.abc.Iterable이 실제로 동작하는 방식
from collections.abc import Iterable as AbcIterable
print(isinstance([1, 2, 3], AbcIterable))  # True (list는 __iter__ 보유)
```

---

**Q34. Python의 `__set_name__` 디스크립터 훅을 이용하여 자동 검증 시스템을 만드세요.**

**A.** `__set_name__`은 Python 3.6에서 추가된 훅으로, 디스크립터가 클래스에 할당될 때 클래스와 속성 이름을 자동으로 받습니다. 이를 이용하면 `property`처럼 이름을 수동으로 지정할 필요 없이 재사용 가능한 검증 디스크립터를 만들 수 있습니다.

```python
from typing import Any, Callable, TypeVar, Type

T = TypeVar('T')

class Validated:
    """재사용 가능한 검증 디스크립터 기반 클래스"""
    
    def __set_name__(self, owner: type, name: str) -> None:
        self.public_name = name
        self.private_name = f'_{type(self).__name__}_{name}'
    
    def __get__(self, obj: Any, objtype: type | None = None) -> Any:
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)
    
    def __set__(self, obj: Any, value: Any) -> None:
        self.validate(value)
        setattr(obj, self.private_name, value)
    
    def validate(self, value: Any) -> None:
        raise NotImplementedError

class InRange(Validated):
    def __init__(self, min_val: float, max_val: float):
        self.min_val = min_val
        self.max_val = max_val
    
    def validate(self, value: float) -> None:
        if not (self.min_val <= value <= self.max_val):
            raise ValueError(
                f"{self.public_name}: [{self.min_val}, {self.max_val}] 범위여야 합니다. "
                f"입력값: {value}"
            )

class NonEmpty(Validated):
    def validate(self, value: str) -> None:
        if not value or not value.strip():
            raise ValueError(f"{self.public_name}은 빈 문자열일 수 없습니다.")

class ModelConfig:
    learning_rate = InRange(1e-7, 1.0)
    dropout_rate = InRange(0.0, 0.9)
    model_name = NonEmpty()
    
    def __init__(self, name: str, lr: float, dropout: float):
        self.model_name = name
        self.learning_rate = lr
        self.dropout_rate = dropout

cfg = ModelConfig("bert-base", 0.001, 0.1)
# ModelConfig("bert-base", 5.0, 0.1)  # ValueError: learning_rate 범위 초과
```

---

**Q35. Python에서 `__future__` 어노테이션과 PEP 563/649의 차이, 그리고 실제 프로덕션 코드에서의 영향을 설명하세요.**

**A.** `from __future__ import annotations`(PEP 563)는 모든 어노테이션 평가를 지연시켜(lazy evaluation) 문자열로 저장합니다. 이를 통해 전방 참조(forward reference) 문제를 해결하고 임포트 시간을 단축합니다. 그러나 런타임에 실제 타입 객체가 필요한 `pydantic`, `attrs`, `dataclasses` 같은 라이브러리에서 `get_type_hints()`를 호출해야 하는 추가 비용이 발생하고, 일부 경우 정보가 손실됩니다.

```python
# PEP 563: 모든 어노테이션이 지연 평가됨
from __future__ import annotations

class Node:
    def __init__(self, value: int, next: Node | None = None):  
        # 'Node'가 아직 정의되지 않았어도 OK — 문자열로 저장
        self.value = value
        self.next = next

import typing
print(typing.get_type_hints(Node.__init__))
# {'next': Node | None, 'return': <class 'NoneType'>}
# get_type_hints()가 문자열을 실제 타입으로 평가

# PEP 649 (Python 3.14 예정): Deferred Evaluation of Annotations
# 어노테이션을 람다로 감싸 진정한 지연 평가
# → get_type_hints() 없이도 __annotations__이 올바른 타입 반환
# → PEP 563의 문제점 해결

# 현재 프로덕션 코드에서의 권장사항
# Pydantic v2 사용 시: model_rebuild()로 순환 참조 해결
from pydantic import BaseModel

class Category(BaseModel):
    name: str
    subcategories: list['Category'] = []  # 전방 참조

Category.model_rebuild()  # 실제 타입 해석

# dataclass 사용 시
from dataclasses import dataclass
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from some_module import HeavyClass  # 타입 체크 시에만 임포트

@dataclass
class Config:
    heavy: 'HeavyClass | None' = None  # 문자열 어노테이션으로 런타임 임포트 회피
```

PEP 563은 Python 3.10 이후 논란으로 기본값 적용이 취소되었으며, PEP 649가 더 올바른 해결책으로 채택되어 Python 3.14에 포함될 예정입니다. 이 변화를 추적하는 것이 미래 코드 호환성에 중요합니다.

---

# 📌 마무리: 면접 고득점을 위한 핵심 전략

해외 대기업 Python 면접에서 차별화되는 포인트는 단순히 문법을 아는 것이 아니라, **왜 그렇게 동작하는지(CPython 내부)**, **어떤 trade-off가 있는지(성능 vs 가독성 vs 안전성)**, **실제 대규모 시스템에서 어떻게 적용하는지(MLOps, 분산 시스템, 실시간 처리)**를 함께 설명하는 것입니다.

면접 답변의 구조는 항상 **정의 → 동작 원리 → 실제 사용 사례 → 한계점 및 대안** 순서로 전개하세요. 이것이 시니어 엔지니어다운 사고방식을 보여주는 가장 효과적인 방법입니다.
