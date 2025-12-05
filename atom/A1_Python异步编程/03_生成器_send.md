# 生成器 - send() 方法

## 1. 【30字核心】

**send() 方法让生成器变成双向通道，不仅能产出值，还能接收外部输入，是协程和异步编程的基础。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### send() 的第一性原理 🎯

#### 1. 最基础的定义

**send() = 向生成器的 yield 表达式发送一个值**

仅此而已！没有更基础的了。

- `next(gen)` 等价于 `gen.send(None)`
- `send(value)` 让 `yield` 表达式的值变成 `value`

#### 2. 为什么需要 send()？

**核心问题：普通生成器是单向的（只能产出），但有时我们需要与生成器"对话"。**

想象一个场景：
- 生成器正在处理数据流
- 你想根据处理结果，动态调整它的行为
- 问题：怎么把信息"传进去"？

```python
# 没有 send() 的困境
def processor():
    while True:
        data = yield  # 我怎么拿到外部的数据？
        process(data)

# 有了 send()
def processor():
    while True:
        data = yield  # data 会接收 send() 的值
        print(f"处理: {data}")

gen = processor()
next(gen)       # 启动生成器，走到第一个 yield
gen.send(100)   # 把 100 发送进去，data = 100
gen.send(200)   # 把 200 发送进去，data = 200
```

#### 3. send() 的三层价值

##### 价值1：双向通信
生成器不再是单向的数据源，而是可以接收外部控制的"智能"处理器。

```python
def smart_accumulator():
    """智能累加器：可以动态重置"""
    total = 0
    while True:
        value = yield total
        if value is None:
            total = 0  # 收到 None 就重置
        else:
            total += value

acc = smart_accumulator()
next(acc)        # 启动，返回 0
print(acc.send(10))  # 10
print(acc.send(20))  # 30
print(acc.send(None))  # 0（重置了）
print(acc.send(5))   # 5
```

##### 价值2：协程基础
`send()` 是 Python 协程的历史基础，理解它有助于理解 `async/await`。

```python
# 早期的协程写法（Python 2.5+）
def old_coroutine():
    while True:
        x = yield
        print(f"收到: {x}")

# 现代写法（Python 3.5+）
async def new_coroutine():
    while True:
        x = await some_async_operation()
        print(f"收到: {x}")
```

##### 价值3：状态机实现
`send()` 让生成器可以实现复杂的状态机。

```python
def connection_state_machine():
    """连接状态机"""
    state = 'disconnected'
    while True:
        command = yield state
        if state == 'disconnected' and command == 'connect':
            state = 'connected'
        elif state == 'connected' and command == 'disconnect':
            state = 'disconnected'
        elif state == 'connected' and command == 'send':
            state = 'sending'
        elif state == 'sending':
            state = 'connected'

sm = connection_state_machine()
print(next(sm))           # disconnected
print(sm.send('connect')) # connected
print(sm.send('send'))    # sending
print(sm.send(None))      # connected
```

#### 4. 从第一性原理推导向量数据库应用

**推理链：**
```
1. 向量数据库操作需要流式处理
   ↓
2. 处理过程中可能需要动态调整参数（如阈值、批大小）
   ↓
3. send() 允许在处理过程中注入控制信息
   ↓
4. 应用：可暂停/可调参的向量处理管道
```

```python
def adaptive_vector_processor(initial_threshold=0.8):
    """自适应向量处理器：可以动态调整阈值"""
    threshold = initial_threshold
    results = []
    
    while True:
        # 接收向量或控制命令
        input_data = yield results
        
        if isinstance(input_data, dict):
            # 控制命令
            if 'threshold' in input_data:
                threshold = input_data['threshold']
                print(f"阈值调整为: {threshold}")
        elif input_data is not None:
            # 处理向量
            vector, score = input_data
            if score >= threshold:
                results.append((vector, score))

# 使用
processor = adaptive_vector_processor()
next(processor)  # 启动

# 发送向量
processor.send(([0.1, 0.2], 0.85))
processor.send(([0.3, 0.4], 0.75))  # 低于阈值，被过滤

# 动态调整阈值
processor.send({'threshold': 0.7})

# 继续处理
processor.send(([0.5, 0.6], 0.72))  # 现在会被接受
```

#### 5. 一句话总结第一性原理

**send() 把生成器从单向数据流变成双向通信管道，是实现协程和复杂数据流控制的基础机制。**

---

## 3. 【3个核心概念】

### 核心概念1：yield 表达式的值 🎯

**yield 不仅能产出值，它本身也是一个表达式，可以接收值。**

```python
def demo():
    # yield 表达式的值 = send() 发送的值
    received = yield "产出值"
    print(f"收到: {received}")
    yield "结束"

gen = demo()
print(next(gen))       # "产出值"（走到 yield，暂停）
print(gen.send(100))   # 打印"收到: 100"，然后返回"结束"
```

**关键理解：**

```
gen = demo()
next(gen)           # 执行到 yield "产出值"，返回 "产出值"，暂停
                    # 此时 received 还没有被赋值！

gen.send(100)       # 1. 把 100 赋给 yield 表达式
                    # 2. received = 100
                    # 3. 继续执行 print(...)
                    # 4. 执行到 yield "结束"，返回 "结束"
```

**可视化流程：**
```
代码                          执行流程
---                           ---
def demo():
    received = yield "产出"    <-- next(gen): 执行到这，返回"产出"，暂停
                               <-- send(100): 100赋给yield，received=100
    print(f"收到: {received}")  <-- 继续执行
    yield "结束"               <-- 执行到这，返回"结束"，暂停
```

---

### 核心概念2：启动生成器 🚀

**首次调用必须用 next() 或 send(None)，不能发送非 None 值。**

```python
def gen():
    x = yield 1
    yield x

g = gen()

# 正确的启动方式
next(g)        # OK
# 或
g.send(None)   # OK

# 错误的启动方式
g = gen()
g.send(100)    # TypeError: can't send non-None value to a just-started generator
```

**为什么？**
- 生成器刚创建时，还没有执行到任何 `yield`
- 没有 `yield` 表达式来接收 `send()` 的值
- 所以只能发送 `None`（等价于 `next()`）

**最佳实践：用装饰器自动启动**
```python
def coroutine(func):
    """装饰器：自动启动生成器"""
    def wrapper(*args, **kwargs):
        gen = func(*args, **kwargs)
        next(gen)  # 自动启动
        return gen
    return wrapper

@coroutine
def receiver():
    while True:
        data = yield
        print(f"收到: {data}")

r = receiver()  # 已经启动了，可以直接 send
r.send("hello")  # 收到: hello
```

---

### 核心概念3：throw() 和 close() 🛑

**除了 send()，还可以向生成器发送异常或关闭它。**

```python
def robust_generator():
    try:
        while True:
            try:
                value = yield
                print(f"处理: {value}")
            except ValueError as e:
                print(f"捕获 ValueError: {e}")
    except GeneratorExit:
        print("生成器被关闭")
        # 不要在这里 yield！

gen = robust_generator()
next(gen)

gen.send("数据1")           # 处理: 数据1
gen.throw(ValueError, "坏数据")  # 捕获 ValueError: 坏数据
gen.send("数据2")           # 处理: 数据2
gen.close()                 # 生成器被关闭
```

**三个方法对比：**

| 方法 | 作用 | 生成器内部表现 |
|-----|------|--------------|
| `send(value)` | 发送值 | `yield` 表达式 = value |
| `throw(exc)` | 发送异常 | `yield` 处抛出异常 |
| `close()` | 关闭生成器 | `yield` 处抛出 `GeneratorExit` |

---

## 4. 【最小可用】

掌握以下内容，就能在80%的场景中使用 send()：

### 4.1 基本的双向通信

```python
def echo():
    """回声：返回收到的值"""
    received = None
    while True:
        received = yield received

gen = echo()
next(gen)           # 启动，返回 None
print(gen.send(1))  # 1
print(gen.send(2))  # 2
```

### 4.2 累加器模式

```python
def accumulator():
    """累加器：send 值会被累加"""
    total = 0
    while True:
        value = yield total
        if value is not None:
            total += value

acc = accumulator()
next(acc)            # 启动，返回 0
print(acc.send(10))  # 10
print(acc.send(20))  # 30
print(acc.send(5))   # 35
```

### 4.3 带控制的处理器

```python
def processor():
    """处理器：可以控制行为"""
    running = True
    while running:
        command = yield
        if command == 'stop':
            running = False
            yield "已停止"
        elif command == 'status':
            yield "运行中"
        else:
            yield f"处理: {command}"

p = processor()
next(p)
print(p.send('hello'))   # 处理: hello
print(p.send('status'))  # 运行中
print(p.send('stop'))    # 已停止
```

### 4.4 自动启动装饰器

```python
from functools import wraps

def auto_start(func):
    """自动启动生成器的装饰器"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        gen = func(*args, **kwargs)
        next(gen)
        return gen
    return wrapper

@auto_start
def counter(start=0):
    count = start
    while True:
        increment = yield count
        count += (increment or 1)

c = counter(10)  # 已启动
print(c.send(5))   # 15
print(c.send(None))  # 16（默认加1）
```

**这些知识足以：**
- 实现双向数据流
- 创建可控制的生成器
- 理解协程的基础原理
- 为学习 async/await 打下基础

---

## 5. 【1个类比】

### 类比1：send() = JavaScript Generator.next(value) 🎨

JavaScript 和 Python 的生成器 send 机制几乎一样！

```javascript
// JavaScript
function* echo() {
    while (true) {
        const received = yield;
        console.log(`收到: ${received}`);
    }
}

const gen = echo();
gen.next();        // 启动（等同于 Python 的 next()）
gen.next(100);     // 发送 100（等同于 Python 的 send(100)）
gen.next(200);     // 发送 200
```

```python
# Python
def echo():
    while True:
        received = yield
        print(f"收到: {received}")

gen = echo()
next(gen)        # 启动
gen.send(100)    # 发送 100
gen.send(200)    # 发送 200
```

**主要区别：**
- JS 用 `gen.next(value)` 发送
- Python 用 `gen.send(value)` 发送
- JS 首次 `next(value)` 的 value 会被忽略
- Python 首次 `send(非None)` 会报错

---

### 类比2：send() = React 的 useReducer dispatch 📬

```javascript
// React useReducer
const [state, dispatch] = useReducer(reducer, initialState);

// dispatch 发送 action，reducer 接收并处理
dispatch({ type: 'increment', payload: 5 });
```

```python
# Python 生成器实现类似效果
def reducer_gen(initial_state):
    state = initial_state
    while True:
        action = yield state
        if action['type'] == 'increment':
            state = state + action['payload']
        elif action['type'] == 'reset':
            state = initial_state

# 使用
store = reducer_gen(0)
next(store)  # 启动
print(store.send({'type': 'increment', 'payload': 5}))  # 5
print(store.send({'type': 'increment', 'payload': 3}))  # 8
print(store.send({'type': 'reset'}))  # 0
```

---

### 类比3：send() = WebSocket 双向通信 🔄

```javascript
// WebSocket 是双向的
const ws = new WebSocket('ws://server');

// 可以发送
ws.send('Hello');

// 也可以接收
ws.onmessage = (event) => {
    console.log('收到:', event.data);
};
```

```python
# 生成器也是双向的
def websocket_like():
    while True:
        # yield 既能产出也能接收
        message = yield f"Echo: {message}" if 'message' in dir() else "Ready"
        print(f"收到: {message}")

gen = websocket_like()
print(next(gen))        # Ready（产出）
print(gen.send("Hi"))   # 打印"收到: Hi"，返回"Echo: Hi"（收发）
```

---

### 类比4：send() = 事件驱动的回调 📡

```javascript
// 前端：事件回调
element.addEventListener('click', (event) => {
    // event 是外部传入的数据
    console.log(event.target);
});
```

```python
# Python 生成器：类似的"等待输入"模式
def event_handler():
    while True:
        event = yield  # 等待外部发送事件
        print(f"处理事件: {event['type']}")
        if event['type'] == 'click':
            print(f"点击位置: {event['position']}")

handler = event_handler()
next(handler)
handler.send({'type': 'click', 'position': (100, 200)})
handler.send({'type': 'hover', 'position': (150, 250)})
```

---

### 类比5：send() = Promise 的 resolve 🤝

```javascript
// Promise：外部 resolve 传入值
new Promise((resolve, reject) => {
    // 某个时候调用 resolve(value)
    setTimeout(() => resolve(42), 1000);
}).then(value => {
    console.log(value);  // 42
});
```

```python
# 生成器：send() 类似 resolve
def promise_like():
    print("等待结果...")
    result = yield  # 等待外部 send
    print(f"得到结果: {result}")
    return result

gen = promise_like()
next(gen)        # 打印"等待结果..."
gen.send(42)     # 打印"得到结果: 42"
```

---

### 类比总结表

| Python send() 概念 | 前端对应概念 |
|-------------------|-------------|
| `gen.send(value)` | `gen.next(value)` (JS) |
| yield 表达式接收值 | `const x = yield` |
| 双向通信 | WebSocket |
| 发送控制命令 | dispatch action |
| 等待外部输入 | Promise resolve |
| throw() | reject / throw error |
| close() | WebSocket close |

---

## 6. 【反直觉点】

### 误区1：send() 会立即执行到下一个 yield ❌

**为什么错？**
- `send(value)` 的完整流程是：
  1. 把 value 赋给当前暂停的 yield 表达式
  2. 继续执行直到下一个 yield
  3. 返回下一个 yield 产出的值

**为什么人们容易这样错？**
因为 `next()` 是"获取下一个"，所以以为 `send()` 也是直接跳到下一个。

**正确理解：**
```python
def demo():
    print("A")
    x = yield 1
    print(f"B: x={x}")
    y = yield 2
    print(f"C: y={y}")
    yield 3

gen = demo()
print(next(gen))      # 打印 A，返回 1
print(gen.send(100))  # 打印 B: x=100，返回 2
print(gen.send(200))  # 打印 C: y=200，返回 3

# 输出：
# A
# 1
# B: x=100
# 2
# C: y=200
# 3
```

---

### 误区2：可以在任何时候调用 send() ❌

**为什么错？**
- 刚创建的生成器必须先用 `next()` 或 `send(None)` 启动
- 发送非 None 值给刚创建的生成器会报错

**为什么人们容易这样错？**
因为其他语言（如 JavaScript）的 `next(value)` 可以直接用（虽然首次的 value 会被忽略）。

**正确理解：**
```python
def gen():
    x = yield 1
    yield x

g = gen()

# 错误：不能直接 send 非 None 值
# g.send(100)  # TypeError!

# 正确：先启动
next(g)      # 或 g.send(None)
g.send(100)  # 现在可以了

# 最佳实践：用装饰器自动启动
def auto_start(f):
    def wrapper(*a, **kw):
        g = f(*a, **kw)
        next(g)
        return g
    return wrapper

@auto_start
def safe_gen():
    while True:
        x = yield
        print(x)

sg = safe_gen()  # 已启动
sg.send(100)     # 直接用
```

---

### 误区3：send() 和 next() 完全不同 ❌

**为什么错？**
- `next(gen)` 完全等价于 `gen.send(None)`
- 它们做的事情一模一样

**为什么人们容易这样错？**
因为它们的名字和语法不同，看起来像是两个独立的操作。

**正确理解：**
```python
def gen():
    while True:
        x = yield
        print(f"收到: {x}")

g = gen()

# 这两个完全等价：
next(g)       # 等于 g.send(None)，x = None
g.send(None)  # 等于 next(g)，x = None

# 只有发送非 None 值时才有区别
g.send(100)   # x = 100
```

**证明：**
```python
def gen():
    x = yield 1
    print(f"x = {x}")
    yield 2

g1 = gen()
print(next(g1))      # 1
print(next(g1))      # 打印 "x = None"，返回 2

g2 = gen()
print(g2.send(None)) # 1（等同于 next）
print(g2.send(None)) # 打印 "x = None"，返回 2（等同于 next）
```

---

## 7. 【实战代码】

```python
"""
生成器 send() 实战示例：向量数据处理管道
展示 send() 在实际开发中的应用
"""

import time
import random
from functools import wraps

# ===== 工具：自动启动装饰器 =====
def coroutine(func):
    """自动启动生成器的装饰器"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        gen = func(*args, **kwargs)
        next(gen)
        return gen
    return wrapper

# ===== 1. 基础：双向通信 =====
print("=== 1. 基础：双向通信 ===")

def simple_receiver():
    """简单的接收器"""
    print("接收器已启动")
    while True:
        data = yield
        print(f"收到: {data}")

recv = simple_receiver()
next(recv)  # 启动
recv.send("Hello")
recv.send("World")
recv.send(123)

# ===== 2. 累加器：产出累计值 =====
print("\n=== 2. 累加器 ===")

@coroutine
def accumulator(initial=0):
    """累加器：每次 send 后返回累计值"""
    total = initial
    while True:
        value = yield total
        if value is not None:
            total += value

acc = accumulator(100)
print(f"初始: {acc.send(0)}")    # 100
print(f"加 10: {acc.send(10)}")  # 110
print(f"加 20: {acc.send(20)}")  # 130
print(f"加 -30: {acc.send(-30)}") # 100

# ===== 3. 可配置的过滤器 =====
print("\n=== 3. 可配置过滤器 ===")

@coroutine
def configurable_filter(initial_threshold=0.5):
    """可动态调整阈值的过滤器"""
    threshold = initial_threshold
    results = []
    
    while True:
        data = yield results
        
        if isinstance(data, dict) and 'set_threshold' in data:
            threshold = data['set_threshold']
            print(f"[配置] 阈值更新为: {threshold}")
        elif isinstance(data, tuple) and len(data) == 2:
            value, score = data
            if score >= threshold:
                results.append((value, score))
                print(f"[通过] {value}: {score:.2f} >= {threshold}")
            else:
                print(f"[过滤] {value}: {score:.2f} < {threshold}")

flt = configurable_filter(0.7)

# 发送数据
flt.send(("文档A", 0.85))
flt.send(("文档B", 0.65))  # 被过滤

# 调整阈值
flt.send({'set_threshold': 0.5})

# 继续发送
flt.send(("文档C", 0.55))  # 现在通过了
print(f"结果: {flt.send(None)}")

# ===== 4. 状态机：数据库连接模拟 =====
print("\n=== 4. 状态机：数据库连接 ===")

@coroutine
def db_connection_state_machine():
    """数据库连接状态机"""
    state = 'DISCONNECTED'
    
    while True:
        command = yield state
        
        if state == 'DISCONNECTED':
            if command == 'connect':
                print("[状态机] 正在连接...")
                state = 'CONNECTING'
        
        elif state == 'CONNECTING':
            if command == 'connected':
                print("[状态机] 连接成功")
                state = 'CONNECTED'
            elif command == 'fail':
                print("[状态机] 连接失败")
                state = 'DISCONNECTED'
        
        elif state == 'CONNECTED':
            if command == 'disconnect':
                print("[状态机] 断开连接")
                state = 'DISCONNECTED'
            elif command == 'query':
                print("[状态机] 执行查询...")
                state = 'QUERYING'
        
        elif state == 'QUERYING':
            if command == 'done':
                print("[状态机] 查询完成")
                state = 'CONNECTED'

sm = db_connection_state_machine()
print(f"初始状态: {sm.send(None)}")
print(f"-> {sm.send('connect')}")
print(f"-> {sm.send('connected')}")
print(f"-> {sm.send('query')}")
print(f"-> {sm.send('done')}")
print(f"-> {sm.send('disconnect')}")

# ===== 5. 批处理器：可控制的批量操作 =====
print("\n=== 5. 可控制的批处理器 ===")

@coroutine
def batch_processor(batch_size=3):
    """可控制的批处理器"""
    batch = []
    processed_count = 0
    
    while True:
        command = yield {'buffer': len(batch), 'processed': processed_count}
        
        if command == 'flush':
            # 强制处理当前批次
            if batch:
                print(f"[批处理] 强制处理 {len(batch)} 条")
                processed_count += len(batch)
                batch = []
        elif command == 'status':
            # 只返回状态，不做操作
            pass
        elif command == 'reset':
            batch = []
            processed_count = 0
            print("[批处理] 已重置")
        elif command is not None:
            # 添加数据
            batch.append(command)
            if len(batch) >= batch_size:
                print(f"[批处理] 批次已满，处理 {len(batch)} 条")
                processed_count += len(batch)
                batch = []

bp = batch_processor(batch_size=3)

# 添加数据
for i in range(5):
    status = bp.send(f"item_{i}")
    print(f"  添加 item_{i}, 缓冲: {status['buffer']}")

# 强制刷新
bp.send('flush')
print(f"最终状态: {bp.send('status')}")

# ===== 6. 向量处理管道 =====
print("\n=== 6. 向量处理管道 ===")

@coroutine
def vector_processor():
    """向量处理管道：支持多种操作"""
    vectors = []
    
    while True:
        command = yield
        
        if isinstance(command, dict):
            action = command.get('action')
            
            if action == 'add':
                vec = command['vector']
                vectors.append(vec)
                print(f"[管道] 添加向量，总数: {len(vectors)}")
            
            elif action == 'normalize':
                # 归一化所有向量
                for i, v in enumerate(vectors):
                    norm = sum(x**2 for x in v) ** 0.5
                    if norm > 0:
                        vectors[i] = [x/norm for x in v]
                print(f"[管道] 已归一化 {len(vectors)} 个向量")
            
            elif action == 'filter':
                threshold = command.get('threshold', 0.5)
                before = len(vectors)
                vectors = [v for v in vectors if sum(v) > threshold]
                print(f"[管道] 过滤: {before} -> {len(vectors)}")
            
            elif action == 'get':
                print(f"[管道] 当前向量: {vectors}")
            
            elif action == 'clear':
                vectors = []
                print("[管道] 已清空")

pipeline = vector_processor()

pipeline.send({'action': 'add', 'vector': [0.1, 0.2, 0.3]})
pipeline.send({'action': 'add', 'vector': [0.4, 0.5, 0.6]})
pipeline.send({'action': 'add', 'vector': [0.01, 0.02, 0.03]})
pipeline.send({'action': 'get'})
pipeline.send({'action': 'filter', 'threshold': 0.1})
pipeline.send({'action': 'get'})
pipeline.send({'action': 'normalize'})
pipeline.send({'action': 'get'})

# ===== 7. throw() 和 close() =====
print("\n=== 7. 异常处理和关闭 ===")

@coroutine
def robust_processor():
    """健壮的处理器：处理异常和关闭"""
    try:
        while True:
            try:
                data = yield
                if data == 'error':
                    raise ValueError("手动触发的错误")
                print(f"[处理] {data}")
            except ValueError as e:
                print(f"[捕获] ValueError: {e}")
    except GeneratorExit:
        print("[清理] 生成器被关闭，执行清理...")
    finally:
        print("[完成] finally 块执行")

rp = robust_processor()
rp.send("数据1")
rp.send("数据2")
rp.throw(ValueError, "外部注入的错误")  # 注入异常
rp.send("数据3")
rp.close()  # 关闭生成器

# ===== 8. 实际应用：流式相似度计算 =====
print("\n=== 8. 流式相似度计算 ===")

def cosine_similarity(v1, v2):
    """计算余弦相似度"""
    dot = sum(a*b for a, b in zip(v1, v2))
    norm1 = sum(a**2 for a in v1) ** 0.5
    norm2 = sum(b**2 for b in v2) ** 0.5
    return dot / (norm1 * norm2) if norm1 * norm2 > 0 else 0

@coroutine
def similarity_matcher(query_vector, top_k=3):
    """流式匹配器：持续接收向量，维护 top-k"""
    top_matches = []  # [(similarity, vector_id, vector)]
    
    while True:
        data = yield [
            {'id': m[1], 'score': round(m[0], 3)} 
            for m in top_matches
        ]
        
        if isinstance(data, dict) and 'vector' in data:
            vec_id = data.get('id', 'unknown')
            vector = data['vector']
            sim = cosine_similarity(query_vector, vector)
            
            # 维护 top-k
            top_matches.append((sim, vec_id, vector))
            top_matches.sort(reverse=True, key=lambda x: x[0])
            top_matches = top_matches[:top_k]
        
        elif data == 'reset':
            top_matches = []

# 使用
query = [0.5, 0.5, 0.5]
matcher = similarity_matcher(query, top_k=3)

# 流式发送向量
vectors = [
    {'id': 'doc1', 'vector': [0.4, 0.5, 0.6]},
    {'id': 'doc2', 'vector': [0.1, 0.2, 0.3]},
    {'id': 'doc3', 'vector': [0.5, 0.5, 0.5]},
    {'id': 'doc4', 'vector': [0.6, 0.4, 0.5]},
    {'id': 'doc5', 'vector': [0.2, 0.8, 0.1]},
]

for vec in vectors:
    result = matcher.send(vec)
    print(f"发送 {vec['id']}, Top-3: {result}")
```

**运行输出示例：**
```
=== 1. 基础：双向通信 ===
接收器已启动
收到: Hello
收到: World
收到: 123

=== 2. 累加器 ===
初始: 100
加 10: 110
加 20: 130
加 -30: 100

=== 3. 可配置过滤器 ===
[通过] 文档A: 0.85 >= 0.7
[过滤] 文档B: 0.65 < 0.7
[配置] 阈值更新为: 0.5
[通过] 文档C: 0.55 >= 0.5
结果: [('文档A', 0.85), ('文档C', 0.55)]

=== 4. 状态机：数据库连接 ===
初始状态: DISCONNECTED
[状态机] 正在连接...
-> CONNECTING
[状态机] 连接成功
-> CONNECTED
[状态机] 执行查询...
-> QUERYING
[状态机] 查询完成
-> CONNECTED
[状态机] 断开连接
-> DISCONNECTED

=== 5. 可控制的批处理器 ===
  添加 item_0, 缓冲: 1
  添加 item_1, 缓冲: 2
[批处理] 批次已满，处理 3 条
  添加 item_2, 缓冲: 0
  添加 item_3, 缓冲: 1
  添加 item_4, 缓冲: 2
[批处理] 强制处理 2 条
最终状态: {'buffer': 0, 'processed': 5}

=== 6. 向量处理管道 ===
[管道] 添加向量，总数: 1
[管道] 添加向量，总数: 2
[管道] 添加向量，总数: 3
[管道] 当前向量: [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6], [0.01, 0.02, 0.03]]
[管道] 过滤: 3 -> 2
[管道] 当前向量: [[0.1, 0.2, 0.3], [0.4, 0.5, 0.6]]
[管道] 已归一化 2 个向量
[管道] 当前向量: [[0.267..., 0.534..., 0.801...], ...]

=== 7. 异常处理和关闭 ===
[处理] 数据1
[处理] 数据2
[捕获] ValueError: 外部注入的错误
[处理] 数据3
[清理] 生成器被关闭，执行清理...
[完成] finally 块执行

=== 8. 流式相似度计算 ===
发送 doc1, Top-3: [{'id': 'doc1', 'score': 0.99}]
发送 doc2, Top-3: [{'id': 'doc1', 'score': 0.99}, {'id': 'doc2', 'score': 0.927}]
发送 doc3, Top-3: [{'id': 'doc3', 'score': 1.0}, {'id': 'doc1', 'score': 0.99}, ...]
...
```

---

## 8. 【面试必问】

### 问题1："send() 和 next() 有什么区别？"

**普通回答（❌ 不出彩）：**
"next() 获取下一个值，send() 可以发送值进去。"

**出彩回答（✅ 推荐）：**

> **核心区别在于 yield 表达式的返回值：**
>
> 1. **本质关系**：`next(gen)` 完全等价于 `gen.send(None)`
>
> 2. **区别点**：
>    - `next(gen)` 让 yield 表达式的值为 `None`
>    - `gen.send(value)` 让 yield 表达式的值为 `value`
>
> 3. **使用限制**：
>    - 刚创建的生成器必须先用 `next()` 或 `send(None)` 启动
>    - 不能对刚创建的生成器 `send(非None值)`
>
> **代码说明：**
> ```python
> def demo():
>     x = yield 1
>     print(f"x = {x}")
>
> g = demo()
> next(g)        # 走到 yield，返回 1
> g.send(100)    # x 被赋值为 100
>
> # next(g) 等价于 g.send(None)，此时 x = None
> ```
>
> **实际应用**：send() 让生成器从单向数据源变成双向通信管道，是实现协程和状态机的基础。

---

### 问题2："生成器和协程是什么关系？"

**普通回答（❌ 不出彩）：**
"协程是用生成器实现的。"

**出彩回答（✅ 推荐）：**

> **生成器是 Python 协程的历史基础：**
>
> 1. **Python 2.5**：引入 `send()`，让生成器可以双向通信
>
> 2. **Python 3.3**：引入 `yield from`，简化生成器委托
>
> 3. **Python 3.4**：引入 `@asyncio.coroutine` + `yield from` 实现协程
>
> 4. **Python 3.5**：引入 `async/await` 语法糖，但底层仍基于生成器机制
>
> **代码演进：**
> ```python
> # 阶段1：原始生成器
> def gen():
>     result = yield request()
>     return result
>
> # 阶段2：asyncio 协程（Python 3.4）
> @asyncio.coroutine
> def coro():
>     result = yield from async_request()
>     return result
>
> # 阶段3：现代协程（Python 3.5+）
> async def coro():
>     result = await async_request()
>     return result
> ```
>
> **理解 send() 的价值**：它是理解 Python 异步编程历史和原理的关键环节。

---

## 9. 【化骨绵掌】

### 卡片1：send() 是什么？ 🎯

**一句话：** send() 向生成器的 yield 表达式发送一个值。

**举例：**
```python
def gen():
    x = yield 1
    print(f"x = {x}")

g = gen()
next(g)        # 走到 yield，返回 1
g.send(100)    # x = 100
```

**应用：** 让生成器变成双向通信管道。

---

### 卡片2：send() vs next() ⚖️

**一句话：** `next(gen)` 等价于 `gen.send(None)`。

**对比：**
```python
def gen():
    x = yield
    print(x)

g = gen()
next(g)       # x 将是 None
g.send(100)   # x 将是 100
```

---

### 卡片3：启动生成器 🚀

**一句话：** 刚创建的生成器必须先启动，不能直接 send 非 None 值。

**规则：**
```python
g = gen()
# g.send(100)  # 错！TypeError

next(g)        # 正确：启动
g.send(100)    # 现在可以了

# 或者
g.send(None)   # 也是启动
```

---

### 卡片4：自动启动装饰器 🎀

**一句话：** 用装饰器自动执行首次 next()。

**代码：**
```python
def coroutine(func):
    def wrapper(*args, **kwargs):
        gen = func(*args, **kwargs)
        next(gen)  # 自动启动
        return gen
    return wrapper

@coroutine
def receiver():
    while True:
        data = yield
        print(data)

r = receiver()  # 已启动
r.send("hello")  # 直接用
```

---

### 卡片5：yield 表达式的值 📥

**一句话：** yield 既能产出值（右边），也能接收值（整个表达式的值）。

**理解：**
```python
def gen():
    # yield 1 产出 1
    # x 接收 send() 的值
    x = yield 1
```

**执行流程：**
1. `next(gen)` → 执行到 yield，返回 1，暂停
2. `gen.send(100)` → x = 100，继续执行

---

### 卡片6：双向累加器 ➕

**一句话：** send 数值累加，yield 返回总和。

**代码：**
```python
@coroutine
def accumulator():
    total = 0
    while True:
        value = yield total
        total += value or 0

acc = accumulator()
acc.send(10)  # 10
acc.send(20)  # 30
```

---

### 卡片7：throw() 方法 💥

**一句话：** 向生成器内部抛出异常。

**代码：**
```python
def gen():
    try:
        yield
    except ValueError as e:
        print(f"捕获: {e}")
        yield "已恢复"

g = gen()
next(g)
g.throw(ValueError, "错误")  # 捕获: 错误
```

---

### 卡片8：close() 方法 🛑

**一句话：** 关闭生成器，触发 GeneratorExit。

**代码：**
```python
def gen():
    try:
        while True:
            yield
    except GeneratorExit:
        print("清理资源")

g = gen()
next(g)
g.close()  # 打印"清理资源"
```

---

### 卡片9：状态机模式 🔄

**一句话：** 用 send() 发送命令，生成器根据状态响应。

**代码：**
```python
@coroutine
def state_machine():
    state = 'idle'
    while True:
        cmd = yield state
        if state == 'idle' and cmd == 'start':
            state = 'running'
        elif state == 'running' and cmd == 'stop':
            state = 'idle'

sm = state_machine()
sm.send('start')  # running
sm.send('stop')   # idle
```

---

### 卡片10：协程基础 🔮

**一句话：** send() 是 Python 协程的历史基础，理解它有助于理解 async/await。

**演进：**
```python
# 生成器协程（旧）
def old_coro():
    result = yield async_call()
    return result

# 现代协程（新）
async def new_coro():
    result = await async_call()
    return result
```

---

## 10. 【一句话总结】

**send() 方法让生成器从单向数据流变成双向通信管道，通过向 yield 表达式发送值实现外部控制，是理解 Python 协程和异步编程的关键基础，在向量数据库开发中用于构建可控制的数据处理管道。**

---

## 📚 学习检查清单

- [ ] 理解 send() 和 next() 的关系
- [ ] 知道为什么要先启动生成器
- [ ] 能用 send() 实现双向通信
- [ ] 会写自动启动的装饰器
- [ ] 理解 throw() 和 close() 的作用
- [ ] 能用 send() 实现简单状态机
- [ ] 理解 send() 与协程的关系

## 🔗 下一步学习

学完 send() 后，建议学习：
1. **迭代器协议** - 理解生成器的底层原理
2. **async/await** - 现代协程语法
3. **asyncio** - Python 异步编程框架
