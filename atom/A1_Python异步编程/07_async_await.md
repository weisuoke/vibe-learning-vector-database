# async/await

## 1. 【30字核心】

**async 定义协程函数，await 暂停等待异步操作完成，两者配合让异步代码像同步一样清晰易读。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### async/await 的第一性原理 🎯

#### 1. 最基础的定义

**async = 声明"这是一个可暂停的函数"**
**await = 声明"在这里暂停，等待结果"**

仅此而已！没有更基础的了。

```python
async def fetch_data():      # async: 这个函数可以暂停
    result = await api_call() # await: 在这里暂停等待
    return result
```

#### 2. 为什么需要 async/await？

**核心问题：异步代码用回调写会变成"回调地狱"，难以阅读和维护。**

```python
# 没有 async/await 的世界（回调地狱）
def fetch_user(callback):
    def on_user(user):
        def on_posts(posts):
            def on_comments(comments):
                callback(user, posts, comments)
            fetch_comments(posts, on_comments)
        fetch_posts(user, on_posts)
    api_get_user(on_user)

# 有了 async/await（清晰明了）
async def fetch_all():
    user = await api_get_user()
    posts = await fetch_posts(user)
    comments = await fetch_comments(posts)
    return user, posts, comments
```

#### 3. async/await 的三层价值

##### 价值1：代码可读性
异步代码看起来像同步代码，符合人类思维习惯。

```python
# 同步代码
def sync_version():
    data = fetch_data()
    result = process(data)
    save(result)
    return result

# 异步代码（结构完全一样！）
async def async_version():
    data = await fetch_data()
    result = await process(data)
    await save(result)
    return result
```

##### 价值2：错误处理
可以用标准的 try/except，不用在回调里处理错误。

```python
async def safe_fetch():
    try:
        data = await risky_api_call()
        return data
    except NetworkError as e:
        logger.error(f"网络错误: {e}")
        return None
    finally:
        await cleanup()
```

##### 价值3：组合性
异步函数可以像普通函数一样组合、嵌套、复用。

```python
async def get_user_with_posts(user_id):
    user = await get_user(user_id)
    posts = await get_posts(user_id)
    return {**user, 'posts': posts}

async def get_all_users_with_posts(user_ids):
    tasks = [get_user_with_posts(uid) for uid in user_ids]
    return await asyncio.gather(*tasks)
```

#### 4. 从第一性原理推导向量数据库应用

**推理链：**
```
1. 向量数据库操作是 I/O 密集型
   ↓
2. 需要异步来避免等待浪费
   ↓
3. async/await 让异步代码清晰可维护
   ↓
4. 应用：异步查询、批量操作、流式处理
```

```python
async def vector_db_operations():
    """向量数据库异步操作示例"""
    
    # 1. 异步连接
    client = await AsyncVectorDB.connect('localhost:8000')
    
    # 2. 异步插入
    await client.insert(vectors)
    
    # 3. 异步查询
    results = await client.search(query_vector, top_k=10)
    
    # 4. 并行批量查询
    batch_results = await asyncio.gather(*[
        client.search(q) for q in query_vectors
    ])
    
    return batch_results
```

#### 5. 一句话总结第一性原理

**async/await 是异步编程的语法糖，让"暂停-恢复"的异步逻辑可以用顺序代码表达，兼顾性能和可读性。**

---

## 3. 【3个核心概念】

### 核心概念1：协程函数 vs 协程对象 🔄

**async def 定义的是协程函数，调用它返回协程对象。**

```python
# 定义协程函数
async def my_coroutine():
    await asyncio.sleep(1)
    return "完成"

# 调用协程函数，得到协程对象（不会执行！）
coro = my_coroutine()
print(type(coro))  # <class 'coroutine'>

# 协程对象必须被 await 或交给事件循环才会执行
result = await coro  # 现在才执行
# 或
result = asyncio.run(my_coroutine())
```

**关键区分：**
```python
async def fetch():
    return "data"

# 这只是创建了协程对象，没有执行！
coro = fetch()  # 警告：coroutine was never awaited

# 正确做法
result = await fetch()  # 或 asyncio.run(fetch())
```

**在向量数据库中的应用：**
```python
async def search(query):
    return await db.search(query)

# 错误：创建了协程但没执行
results = [search(q) for q in queries]  # 列表里是协程对象！

# 正确：用 gather 执行所有协程
results = await asyncio.gather(*[search(q) for q in queries])
```

---

### 核心概念2：await 的执行流程 ⏸️

**await 做三件事：暂停当前协程、等待结果、恢复执行。**

```python
async def demo():
    print("1. 开始")
    
    # await 在这里：
    # 1. 暂停 demo 协程
    # 2. 让事件循环去做别的事
    # 3. sleep 完成后恢复 demo
    await asyncio.sleep(1)
    
    print("2. 1秒后继续")
    
    # 又一个 await
    result = await fetch_data()
    
    print(f"3. 拿到数据: {result}")
    return result
```

**可视化执行流程：**
```
时间线：
0ms   demo 开始执行
      print("1. 开始")
      遇到 await sleep(1)，暂停 demo
      
0-1000ms  事件循环可以执行其他协程
      
1000ms  sleep 完成，恢复 demo
        print("2. 1秒后继续")
        遇到 await fetch_data()，暂停 demo
        
1000-1050ms  等待网络请求
        
1050ms  fetch 完成，恢复 demo
        print("3. 拿到数据")
        return
```

**关键理解：await 不是"卡住程序"，而是"让出 CPU 给别人"。**

---

### 核心概念3：可等待对象 (Awaitable) ⏳

**await 后面可以跟三种对象：协程、Task、Future。**

```python
import asyncio

# 1. 协程 (Coroutine)
async def coro_func():
    return "协程结果"

result = await coro_func()  # 直接 await 协程

# 2. Task
async def task_demo():
    task = asyncio.create_task(coro_func())  # 创建 Task
    # Task 会立即开始执行
    result = await task  # await Task
    return result

# 3. Future
async def future_demo():
    loop = asyncio.get_event_loop()
    future = loop.create_future()
    
    # 某处设置结果
    future.set_result("Future 结果")
    
    result = await future  # await Future
    return result
```

**三者关系：**
```
Awaitable (可等待)
├── Coroutine (协程)
│   └── async def 创建
├── Task (任务)
│   └── asyncio.create_task() 创建
│   └── 是 Future 的子类
│   └── 包装协程，加入事件循环调度
└── Future (未来对象)
    └── loop.create_future() 创建
    └── 底层的"结果占位符"
```

**最常用的模式：**
```python
async def main():
    # 直接 await 协程
    result1 = await some_async_func()
    
    # 创建 Task 实现并发
    task1 = asyncio.create_task(async_func1())
    task2 = asyncio.create_task(async_func2())
    
    # await 多个 Task
    result1 = await task1
    result2 = await task2
    
    # 或者用 gather
    results = await asyncio.gather(task1, task2)
```

---

## 4. 【最小可用】

掌握以下内容，就能在80%的场景中使用 async/await：

### 4.1 定义和调用异步函数

```python
import asyncio

# 定义异步函数
async def greet(name):
    await asyncio.sleep(1)  # 模拟异步操作
    return f"Hello, {name}!"

# 调用方式1：在另一个 async 函数中 await
async def main():
    result = await greet("World")
    print(result)

# 调用方式2：用 asyncio.run
asyncio.run(main())
```

### 4.2 并发执行多个协程

```python
import asyncio

async def fetch(id):
    await asyncio.sleep(0.5)
    return f"Result {id}"

async def main():
    # 方式1：gather（推荐）
    results = await asyncio.gather(
        fetch(1),
        fetch(2),
        fetch(3)
    )
    print(results)  # ['Result 1', 'Result 2', 'Result 3']
    
    # 方式2：create_task
    task1 = asyncio.create_task(fetch(1))
    task2 = asyncio.create_task(fetch(2))
    
    result1 = await task1
    result2 = await task2
    print(result1, result2)

asyncio.run(main())
```

### 4.3 异常处理

```python
async def risky_operation():
    await asyncio.sleep(0.1)
    raise ValueError("Something went wrong")

async def main():
    # 单个协程的异常处理
    try:
        result = await risky_operation()
    except ValueError as e:
        print(f"捕获异常: {e}")
    
    # gather 的异常处理
    results = await asyncio.gather(
        fetch(1),
        risky_operation(),
        fetch(3),
        return_exceptions=True  # 异常作为结果返回
    )
    for r in results:
        if isinstance(r, Exception):
            print(f"异常: {r}")
        else:
            print(f"结果: {r}")

asyncio.run(main())
```

### 4.4 超时控制

```python
async def slow_operation():
    await asyncio.sleep(10)
    return "Done"

async def main():
    try:
        # 设置 2 秒超时
        result = await asyncio.wait_for(
            slow_operation(),
            timeout=2.0
        )
    except asyncio.TimeoutError:
        print("操作超时！")

asyncio.run(main())
```

### 4.5 向量数据库异步查询

```python
import asyncio

async def async_vector_search(client, query, name=""):
    """模拟异步向量搜索"""
    print(f"[{name}] 开始搜索")
    await asyncio.sleep(0.1)  # 模拟网络延迟
    print(f"[{name}] 完成")
    return {"query": name, "results": [1, 2, 3]}

async def batch_search(client, queries):
    """批量异步搜索"""
    tasks = [
        async_vector_search(client, q, f"Q{i}")
        for i, q in enumerate(queries)
    ]
    return await asyncio.gather(*tasks)

async def main():
    client = None  # 模拟客户端
    queries = [[0.1, 0.2], [0.3, 0.4], [0.5, 0.6]]
    
    results = await batch_search(client, queries)
    print(f"获得 {len(results)} 个结果")

asyncio.run(main())
```

**这些知识足以：**
- 编写异步函数
- 实现并发操作
- 处理异常和超时
- 构建向量数据库的异步客户端

---

## 5. 【1个类比】

### 类比1：async/await = JavaScript async/await 🎨

Python 和 JavaScript 的 async/await 语法几乎一模一样！

```javascript
// JavaScript
async function fetchData() {
    const response = await fetch('/api/data');
    const data = await response.json();
    return data;
}

// 调用
fetchData().then(data => console.log(data));

// 并发
const results = await Promise.all([
    fetch('/api/1'),
    fetch('/api/2'),
    fetch('/api/3')
]);
```

```python
# Python
async def fetch_data():
    response = await aiohttp_session.get('/api/data')
    data = await response.json()
    return data

# 调用
asyncio.run(fetch_data())

# 并发
results = await asyncio.gather(
    fetch('/api/1'),
    fetch('/api/2'),
    fetch('/api/3')
)
```

**主要区别：**
| JavaScript | Python |
|-----------|--------|
| `Promise` | `coroutine/Task/Future` |
| `Promise.all()` | `asyncio.gather()` |
| `.then()` 链式调用 | 直接 await |
| 自动运行 | 需要 `asyncio.run()` |

---

### 类比2：await = JavaScript 的 await 暂停 ⏸️

```javascript
// JavaScript
async function demo() {
    console.log('1');
    await sleep(1000);  // 暂停，让出控制权
    console.log('2');
}
```

```python
# Python（完全一样的语义）
async def demo():
    print('1')
    await asyncio.sleep(1)  # 暂停，让出控制权
    print('2')
```

---

### 类比3：asyncio.gather = Promise.all 📦

```javascript
// JavaScript：等待所有 Promise
const [user, posts, comments] = await Promise.all([
    fetchUser(),
    fetchPosts(),
    fetchComments()
]);
```

```python
# Python：等待所有协程
user, posts, comments = await asyncio.gather(
    fetch_user(),
    fetch_posts(),
    fetch_comments()
)
```

---

### 类比4：create_task = 启动后台任务 🚀

```javascript
// JavaScript：启动不等待
fetchData();  // 立即返回 Promise，后台执行
doOtherThings();

// 需要结果时再等
const result = await fetchData();
```

```python
# Python：create_task 类似效果
task = asyncio.create_task(fetch_data())  # 立即开始执行
do_other_things()  # 继续做别的

# 需要结果时再等
result = await task
```

---

### 类比5：async 函数 = React 的 async 组件 ⚛️

```javascript
// React Server Component（概念类似）
async function UserProfile({ userId }) {
    const user = await fetchUser(userId);
    const posts = await fetchPosts(userId);
    
    return (
        <div>
            <h1>{user.name}</h1>
            <PostList posts={posts} />
        </div>
    );
}
```

```python
# Python async 函数
async def get_user_profile(user_id):
    user = await fetch_user(user_id)
    posts = await fetch_posts(user_id)
    
    return {
        "name": user["name"],
        "posts": posts
    }
```

---

### 类比总结表

| Python | JavaScript | 说明 |
|--------|-----------|------|
| `async def` | `async function` | 定义异步函数 |
| `await` | `await` | 等待异步操作 |
| `asyncio.gather()` | `Promise.all()` | 并发等待多个 |
| `asyncio.create_task()` | 不显式调用 | 启动后台任务 |
| `asyncio.wait_for()` | `Promise.race()` | 超时/竞争 |
| `asyncio.run()` | 自动执行 | 运行入口 |

---

## 6. 【反直觉点】

### 误区1：await 会阻塞整个程序 ❌

**为什么错？**
- await 只是暂停当前协程
- 事件循环会去执行其他就绪的协程
- 其他协程可以继续运行

**为什么人们容易这样错？**
因为 await 看起来像"等待"，让人以为整个程序都在等。

**正确理解：**
```python
async def task_a():
    print("A: 开始")
    await asyncio.sleep(2)  # 暂停 A，不影响 B
    print("A: 结束")

async def task_b():
    print("B: 开始")
    await asyncio.sleep(1)  # 暂停 B，不影响 A
    print("B: 结束")

async def main():
    await asyncio.gather(task_a(), task_b())

asyncio.run(main())
# 输出：
# A: 开始
# B: 开始
# B: 结束（1秒后）
# A: 结束（2秒后）
# 总共只需要 2 秒，不是 3 秒！
```

---

### 误区2：调用 async 函数就会执行 ❌

**为什么错？**
- 调用 async 函数只是创建协程对象
- 协程对象必须被 await 或交给事件循环才会执行

**为什么人们容易这样错？**
因为普通函数调用会立即执行，习惯性以为 async 函数也一样。

**正确理解：**
```python
async def fetch():
    print("正在获取数据...")
    return "data"

# 错误：以为调用就执行了
def wrong():
    result = fetch()  # 只创建了协程对象！
    print(result)     # <coroutine object fetch at ...>
    # 警告：RuntimeWarning: coroutine 'fetch' was never awaited

# 正确：必须 await
async def right():
    result = await fetch()  # 现在才执行
    print(result)           # "data"

asyncio.run(right())
```

---

### 误区3：async 函数内可以随意用 time.sleep ❌

**为什么错？**
- `time.sleep()` 是阻塞的，会卡住整个线程
- 应该用 `asyncio.sleep()`，只暂停当前协程

**为什么人们容易这样错？**
因为两个都叫 sleep，以为功能一样。

**正确理解：**
```python
import time
import asyncio

async def bad_sleep():
    print("开始")
    time.sleep(2)  # 阻塞整个线程！其他协程也被卡住
    print("结束")

async def good_sleep():
    print("开始")
    await asyncio.sleep(2)  # 只暂停这个协程
    print("结束")

# 对比效果
async def main():
    # bad: 两个任务串行，总共 4 秒
    # await asyncio.gather(bad_sleep(), bad_sleep())
    
    # good: 两个任务并发，总共 2 秒
    await asyncio.gather(good_sleep(), good_sleep())

asyncio.run(main())
```

**规则：在 async 函数里，所有 I/O 操作都要用异步版本！**

---

## 7. 【实战代码】

```python
"""
async/await 实战示例：向量数据库异步客户端
展示 async/await 在实际开发中的应用
"""

import asyncio
import random
import time

# ===== 1. 基础：定义和调用异步函数 =====
print("=== 1. 基础异步函数 ===")

async def simple_async():
    """最简单的异步函数"""
    print("开始异步操作")
    await asyncio.sleep(0.5)
    print("异步操作完成")
    return "结果"

# 运行
result = asyncio.run(simple_async())
print(f"返回值: {result}")

# ===== 2. 并发执行 =====
print("\n=== 2. 并发执行 ===")

async def task(name, delay):
    print(f"[{name}] 开始 (delay={delay}s)")
    await asyncio.sleep(delay)
    print(f"[{name}] 完成")
    return f"{name}_result"

async def concurrent_demo():
    start = time.time()
    
    # gather 并发执行
    results = await asyncio.gather(
        task("A", 1),
        task("B", 0.5),
        task("C", 0.8)
    )
    
    print(f"\n耗时: {time.time() - start:.2f}s (并发，不是 2.3s)")
    print(f"结果: {results}")

asyncio.run(concurrent_demo())

# ===== 3. create_task 详解 =====
print("\n=== 3. create_task ===")

async def create_task_demo():
    start = time.time()
    
    # create_task 立即开始执行（不等待）
    task1 = asyncio.create_task(task("Task1", 0.5))
    task2 = asyncio.create_task(task("Task2", 0.5))
    
    # 可以先做其他事
    print("任务已启动，做其他事...")
    await asyncio.sleep(0.1)
    print("其他事做完了")
    
    # 需要结果时再等
    result1 = await task1
    result2 = await task2
    
    print(f"耗时: {time.time() - start:.2f}s")

asyncio.run(create_task_demo())

# ===== 4. 异常处理 =====
print("\n=== 4. 异常处理 ===")

async def might_fail(should_fail=False):
    await asyncio.sleep(0.1)
    if should_fail:
        raise ValueError("模拟错误")
    return "成功"

async def exception_demo():
    # 单个协程异常处理
    try:
        result = await might_fail(True)
    except ValueError as e:
        print(f"捕获异常: {e}")
    
    # gather 中的异常处理
    print("\ngather 异常处理:")
    results = await asyncio.gather(
        might_fail(False),
        might_fail(True),
        might_fail(False),
        return_exceptions=True
    )
    
    for i, r in enumerate(results):
        if isinstance(r, Exception):
            print(f"  任务{i}: 失败 - {r}")
        else:
            print(f"  任务{i}: {r}")

asyncio.run(exception_demo())

# ===== 5. 超时控制 =====
print("\n=== 5. 超时控制 ===")

async def slow_operation(duration):
    await asyncio.sleep(duration)
    return "完成"

async def timeout_demo():
    # wait_for 超时
    try:
        result = await asyncio.wait_for(
            slow_operation(5),
            timeout=1.0
        )
        print(f"结果: {result}")
    except asyncio.TimeoutError:
        print("操作超时！")
    
    # 带超时的批量操作
    print("\n批量操作超时处理:")
    tasks = [
        asyncio.wait_for(slow_operation(0.5), timeout=1),
        asyncio.wait_for(slow_operation(2), timeout=1),  # 会超时
        asyncio.wait_for(slow_operation(0.3), timeout=1),
    ]
    
    results = await asyncio.gather(*tasks, return_exceptions=True)
    for i, r in enumerate(results):
        if isinstance(r, asyncio.TimeoutError):
            print(f"  任务{i}: 超时")
        else:
            print(f"  任务{i}: {r}")

asyncio.run(timeout_demo())

# ===== 6. 并发限制（信号量）=====
print("\n=== 6. 并发限制 ===")

async def limited_task(sem, task_id):
    async with sem:
        print(f"[{task_id}] 获取信号量，开始执行")
        await asyncio.sleep(0.3)
        print(f"[{task_id}] 完成，释放信号量")
        return task_id

async def semaphore_demo():
    # 限制最多 3 个并发
    sem = asyncio.Semaphore(3)
    
    start = time.time()
    tasks = [limited_task(sem, i) for i in range(9)]
    results = await asyncio.gather(*tasks)
    
    print(f"\n总耗时: {time.time() - start:.2f}s")
    print(f"预期: 9任务/3并发 * 0.3s = 0.9s")

asyncio.run(semaphore_demo())

# ===== 7. 向量数据库异步客户端 =====
print("\n=== 7. 向量数据库异步客户端 ===")

class AsyncVectorDB:
    """模拟异步向量数据库客户端"""
    
    def __init__(self):
        self.data = {}
        self.connected = False
    
    async def connect(self):
        """异步连接"""
        await asyncio.sleep(0.1)  # 模拟连接延迟
        self.connected = True
        print("数据库已连接")
    
    async def disconnect(self):
        """异步断开"""
        await asyncio.sleep(0.05)
        self.connected = False
        print("数据库已断开")
    
    async def insert(self, id, vector):
        """异步插入单条"""
        await asyncio.sleep(0.05)
        self.data[id] = vector
        return True
    
    async def batch_insert(self, vectors_dict):
        """异步批量插入"""
        await asyncio.sleep(0.05)  # 一次网络往返
        self.data.update(vectors_dict)
        return len(vectors_dict)
    
    async def search(self, query_vector, top_k=5):
        """异步搜索"""
        await asyncio.sleep(0.05)
        # 模拟返回结果
        return [
            {"id": f"doc_{i}", "score": random.random()}
            for i in range(top_k)
        ]
    
    async def batch_search(self, query_vectors):
        """异步批量搜索"""
        tasks = [self.search(q) for q in query_vectors]
        return await asyncio.gather(*tasks)

async def vector_db_demo():
    db = AsyncVectorDB()
    
    # 连接
    await db.connect()
    
    # 批量插入
    print("\n批量插入 100 个向量:")
    vectors = {f"vec_{i}": [random.random() for _ in range(4)] 
               for i in range(100)}
    
    start = time.time()
    count = await db.batch_insert(vectors)
    print(f"  插入 {count} 条，耗时: {time.time() - start:.3f}s")
    
    # 批量搜索
    print("\n批量搜索 10 个查询:")
    queries = [[random.random() for _ in range(4)] for _ in range(10)]
    
    start = time.time()
    results = await db.batch_search(queries)
    print(f"  完成 {len(results)} 个查询，耗时: {time.time() - start:.3f}s")
    
    # 断开
    await db.disconnect()

asyncio.run(vector_db_demo())

# ===== 8. 异步上下文管理器 =====
print("\n=== 8. 异步上下文管理器 ===")

class AsyncDBConnection:
    """异步数据库连接上下文管理器"""
    
    def __init__(self, name):
        self.name = name
    
    async def __aenter__(self):
        print(f"[{self.name}] 建立连接...")
        await asyncio.sleep(0.1)
        print(f"[{self.name}] 连接成功")
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print(f"[{self.name}] 关闭连接...")
        await asyncio.sleep(0.05)
        print(f"[{self.name}] 连接已关闭")
    
    async def query(self, sql):
        await asyncio.sleep(0.05)
        return f"结果: {sql}"

async def context_manager_demo():
    async with AsyncDBConnection("主库") as conn:
        result = await conn.query("SELECT * FROM vectors")
        print(f"查询结果: {result}")
    # 退出时自动关闭连接

asyncio.run(context_manager_demo())

# ===== 9. 异步迭代器 =====
print("\n=== 9. 异步迭代器 ===")

class AsyncVectorStream:
    """异步向量流"""
    
    def __init__(self, count):
        self.count = count
        self.index = 0
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.index >= self.count:
            raise StopAsyncIteration
        
        await asyncio.sleep(0.1)  # 模拟异步获取
        vector = [random.random() for _ in range(4)]
        self.index += 1
        return {"id": self.index, "vector": vector}

async def async_iterator_demo():
    print("异步迭代向量流:")
    async for item in AsyncVectorStream(3):
        print(f"  收到: {item['id']}")

asyncio.run(async_iterator_demo())

# ===== 10. 实际应用模式 =====
print("\n=== 10. 实际应用模式 ===")

async def retry_async(coro_func, max_retries=3, delay=0.5):
    """带重试的异步调用"""
    for attempt in range(max_retries):
        try:
            return await coro_func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            print(f"  尝试 {attempt + 1} 失败: {e}，{delay}s 后重试")
            await asyncio.sleep(delay)

async def flaky_operation():
    """可能失败的操作"""
    if random.random() < 0.7:
        raise ConnectionError("连接失败")
    return "成功"

async def retry_demo():
    print("重试模式:")
    try:
        result = await retry_async(flaky_operation, max_retries=5, delay=0.2)
        print(f"  最终结果: {result}")
    except ConnectionError:
        print("  所有重试都失败了")

asyncio.run(retry_demo())
```

**运行输出示例：**
```
=== 1. 基础异步函数 ===
开始异步操作
异步操作完成
返回值: 结果

=== 2. 并发执行 ===
[A] 开始 (delay=1s)
[B] 开始 (delay=0.5s)
[C] 开始 (delay=0.8s)
[B] 完成
[C] 完成
[A] 完成

耗时: 1.00s (并发，不是 2.3s)
结果: ['A_result', 'B_result', 'C_result']

=== 7. 向量数据库异步客户端 ===
数据库已连接

批量插入 100 个向量:
  插入 100 条，耗时: 0.050s

批量搜索 10 个查询:
  完成 10 个查询，耗时: 0.050s

数据库已断开
...
```

---

## 8. 【面试必问】

### 问题1："async/await 的原理是什么？"

**普通回答（❌ 不出彩）：**
"async 定义异步函数，await 等待异步操作完成。"

**出彩回答（✅ 推荐）：**

> **async/await 的原理有三层：**
>
> 1. **语法层面**：
>    - `async def` 把函数变成协程函数
>    - `await` 只能在 async 函数内使用
>    - 调用协程函数返回协程对象，不立即执行
>
> 2. **执行层面**：
>    - await 让当前协程暂停，把控制权交给事件循环
>    - 事件循环调度其他就绪的协程执行
>    - 等待的操作完成后，协程被唤醒继续执行
>
> 3. **底层原理**：
>    - 协程基于生成器实现（`yield from` 的语法糖）
>    - 事件循环维护就绪队列和等待队列
>    - 通过 `select/epoll` 监听 I/O 事件
>
> **关键理解**：await 不是"阻塞"，而是"让出"。它只暂停当前协程，其他协程可以继续运行，这就是异步高效的原因。
>
> **实际应用**：向量数据库批量查询时，10 个 `await search()` 用 `gather` 并发执行，只需要一个查询的时间。

---

### 问题2："什么情况下用 create_task，什么情况下直接 await？"

**普通回答（❌ 不出彩）：**
"需要并发时用 create_task。"

**出彩回答（✅ 推荐）：**

> **选择取决于你是否需要"启动后先做别的"：**
>
> **直接 await**：顺序执行，等待完成再继续
> ```python
> result = await fetch()  # 必须等 fetch 完成
> process(result)         # 才能处理结果
> ```
>
> **create_task**：立即启动，稍后等待结果
> ```python
> task = asyncio.create_task(fetch())  # 立即开始
> do_other_things()                     # 可以先做别的
> result = await task                   # 需要时再等
> ```
>
> **典型场景**：
> 1. **顺序依赖** → 直接 await
>    ```python
>    user = await get_user(id)
>    posts = await get_posts(user.id)  # 依赖 user
>    ```
>
> 2. **并发独立** → create_task 或 gather
>    ```python
>    task1 = asyncio.create_task(get_user(id))
>    task2 = asyncio.create_task(get_posts(id))
>    user, posts = await task1, await task2
>    # 或
>    user, posts = await asyncio.gather(get_user(id), get_posts(id))
>    ```
>
> **向量数据库例子**：
> - 单个查询：`result = await db.search(query)`
> - 批量查询：`results = await asyncio.gather(*[db.search(q) for q in queries])`

---

## 9. 【化骨绵掌】

### 卡片1：async 是什么？ 🎯

**一句话：** async 声明"这是一个可以暂停的函数"。

```python
async def my_func():
    return "Hello"

# 调用返回协程对象，不是结果
coro = my_func()  # <coroutine object>
```

---

### 卡片2：await 是什么？ ⏸️

**一句话：** await 在这里暂停，等待结果，让出 CPU。

```python
async def demo():
    result = await async_operation()
    # await 期间，其他协程可以运行
    return result
```

---

### 卡片3：协程不会自动执行 ⚠️

**一句话：** 调用 async 函数只创建协程，必须 await 才执行。

```python
# 错误：没有执行
coro = fetch_data()  # 只是协程对象

# 正确：await 执行
result = await fetch_data()
```

---

### 卡片4：asyncio.run() 入口 🚪

**一句话：** asyncio.run() 是运行异步代码的入口。

```python
async def main():
    result = await async_func()
    return result

# 在同步代码中运行异步
result = asyncio.run(main())
```

---

### 卡片5：gather 并发 ⚡

**一句话：** asyncio.gather() 并发执行多个协程。

```python
# 串行：3秒
a = await task(1)
b = await task(1)
c = await task(1)

# 并发：1秒
a, b, c = await asyncio.gather(
    task(1), task(1), task(1)
)
```

---

### 卡片6：create_task 后台启动 🚀

**一句话：** create_task() 立即启动协程，不等待。

```python
task = asyncio.create_task(fetch())
# fetch 已开始执行
do_other_things()
# 需要结果时再等
result = await task
```

---

### 卡片7：异常处理 🛡️

**一句话：** async 函数用标准 try/except。

```python
async def safe():
    try:
        return await risky()
    except ValueError as e:
        return None
```

---

### 卡片8：超时控制 ⏱️

**一句话：** wait_for() 设置超时。

```python
try:
    result = await asyncio.wait_for(
        slow_func(), timeout=5.0
    )
except asyncio.TimeoutError:
    print("超时")
```

---

### 卡片9：信号量限流 🚦

**一句话：** Semaphore 限制并发数量。

```python
sem = asyncio.Semaphore(10)

async def limited():
    async with sem:
        await operation()
```

---

### 卡片10：向量数据库应用 🗄️

**一句话：** 批量查询用 gather 并发。

```python
async def batch_search(queries):
    return await asyncio.gather(*[
        db.search(q) for q in queries
    ])

# 10 个查询并发，只需 1 个的时间
```

---

## 10. 【一句话总结】

**async/await 是 Python 异步编程的核心语法，async 定义可暂停的协程函数，await 暂停等待并让出控制权，配合 asyncio.gather 实现并发，让向量数据库的批量操作从串行变并行，代码清晰且高效。**

---

## 📚 学习检查清单

- [ ] 理解 async def 和普通 def 的区别
- [ ] 知道调用 async 函数返回协程对象
- [ ] 会用 await 等待异步操作
- [ ] 会用 asyncio.gather 并发执行
- [ ] 理解 create_task 的作用
- [ ] 能处理异步代码的异常
- [ ] 会设置超时和并发限制

## 🔗 下一步学习

学完 async/await 后，建议学习：
1. **asyncio.run** - 详细的运行机制
2. **aiohttp** - 异步 HTTP 客户端
3. **异步数据库驱动** - asyncpg, motor 等
