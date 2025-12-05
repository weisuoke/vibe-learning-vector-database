# 生成器 - yield 基础

## 1. 【30字核心】

**生成器是一种特殊函数，用 yield 逐个产出值而非一次返回，实现惰性计算和内存高效的数据流处理。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### 生成器的第一性原理 🎯

#### 1. 最基础的定义

**生成器 = 可以暂停和恢复的函数**

仅此而已！没有更基础的了。

普通函数：从头执行到尾，一次性返回结果
生成器函数：执行到 `yield` 暂停，下次从暂停处继续

#### 2. 为什么需要生成器？

**核心问题：当数据量很大时，一次性加载到内存会爆炸。**

想象一个场景：
- 你要处理 1亿条向量数据
- 每条数据占用 1KB
- 全部加载需要 100GB 内存
- 问题：你的电脑只有 16GB 内存

```python
# 没有生成器的困境
def get_all_vectors():
    result = []
    for i in range(100_000_000):
        result.append(compute_vector(i))  # 内存爆炸！
    return result

# 有了生成器
def get_all_vectors():
    for i in range(100_000_000):
        yield compute_vector(i)  # 一次只在内存中保留一个！
```

#### 3. 生成器的三层价值

##### 价值1：内存效率
只在需要时计算，不提前占用内存。

```python
# 列表：立即计算，占用内存
squares_list = [x**2 for x in range(10_000_000)]  # ~80MB

# 生成器：惰性计算，几乎不占内存
squares_gen = (x**2 for x in range(10_000_000))   # ~120 bytes
```

##### 价值2：处理无限流
可以表示无限序列，因为值是按需产生的。

```python
def infinite_ids():
    id = 0
    while True:
        yield id
        id += 1

id_gen = infinite_ids()
print(next(id_gen))  # 0
print(next(id_gen))  # 1
# 可以无限调用下去...
```

##### 价值3：流水线处理
多个生成器可以串联，形成高效的数据处理流水线。

```python
def read_vectors(file):
    for line in open(file):
        yield parse_vector(line)

def normalize(vectors):
    for v in vectors:
        yield v / np.linalg.norm(v)

def filter_valid(vectors):
    for v in vectors:
        if not np.isnan(v).any():
            yield v

# 流水线：内存中同时只有一个向量
pipeline = filter_valid(normalize(read_vectors('huge.txt')))
```

#### 4. 从第一性原理推导向量数据库应用

**推理链：**
```
1. 向量数据库存储海量向量（可能数十亿）
   ↓
2. 不可能一次性加载所有向量到内存
   ↓
3. 需要一种"按需取用"的机制
   ↓
4. 生成器提供"惰性求值"能力
   ↓
5. 应用：批量导入、流式查询、数据预处理
```

```python
def batch_vectors(vectors, batch_size=1000):
    """将向量流分批，用于批量插入数据库"""
    batch = []
    for vector in vectors:
        batch.append(vector)
        if len(batch) >= batch_size:
            yield batch
            batch = []
    if batch:
        yield batch

# 使用：内存友好的批量插入
for batch in batch_vectors(huge_vector_stream, batch_size=1000):
    db.insert_batch(batch)
```

#### 5. 一句话总结第一性原理

**生成器是"按需生产"的函数，用暂停/恢复机制实现惰性计算，是处理大数据流的内存友好方案。**

---

## 3. 【3个核心概念】

### 核心概念1：yield 关键字 🎯

**yield 是生成器的心脏，它让函数可以"暂停"并产出一个值。**

```python
def simple_generator():
    print("开始")
    yield 1        # 暂停，产出 1
    print("继续")
    yield 2        # 暂停，产出 2
    print("结束")
    yield 3        # 暂停，产出 3

gen = simple_generator()
print(next(gen))  # 输出"开始"，返回 1
print(next(gen))  # 输出"继续"，返回 2
print(next(gen))  # 输出"结束"，返回 3
```

**yield vs return：**

| 特性 | yield | return |
|-----|-------|--------|
| 函数状态 | 暂停，保留状态 | 终止，清除状态 |
| 可执行次数 | 多次 | 一次 |
| 返回值类型 | 生成器对象 | 具体值 |
| 内存使用 | 惰性，按需 | 立即，全部 |

**在向量数据库中的应用：**
逐条读取和处理向量，避免内存溢出。

---

### 核心概念2：生成器对象 📦

**调用生成器函数不会执行函数体，而是返回一个生成器对象。**

```python
def my_gen():
    print("执行了！")
    yield 1

# 调用函数，不会打印任何东西！
gen = my_gen()
print(type(gen))  # <class 'generator'>

# 调用 next() 才开始执行
value = next(gen)  # 现在打印"执行了！"
print(value)       # 1
```

**生成器对象的关键方法：**

```python
gen = (x for x in range(3))

# __next__() / next() - 获取下一个值
print(next(gen))  # 0

# __iter__() - 返回自身，所以可以用 for 循环
for x in gen:
    print(x)  # 1, 2
```

**生成器的状态：**
```python
import inspect

def gen_func():
    yield 1
    yield 2

gen = gen_func()
print(inspect.getgeneratorstate(gen))  # GEN_CREATED
next(gen)
print(inspect.getgeneratorstate(gen))  # GEN_SUSPENDED
next(gen)
print(inspect.getgeneratorstate(gen))  # GEN_SUSPENDED
try:
    next(gen)
except StopIteration:
    print(inspect.getgeneratorstate(gen))  # GEN_CLOSED
```

---

### 核心概念3：StopIteration 异常 🛑

**当生成器耗尽时，会抛出 StopIteration 异常。**

```python
def short_gen():
    yield 1
    yield 2

gen = short_gen()
print(next(gen))  # 1
print(next(gen))  # 2
print(next(gen))  # StopIteration 异常！
```

**for 循环自动处理：**
```python
# for 循环会自动捕获 StopIteration
for x in short_gen():
    print(x)  # 1, 2 - 不会报错
```

**手动处理：**
```python
gen = short_gen()
while True:
    try:
        value = next(gen)
        print(value)
    except StopIteration:
        print("生成器耗尽")
        break
```

**return 值存储在异常中：**
```python
def gen_with_return():
    yield 1
    yield 2
    return "完成"

gen = gen_with_return()
next(gen)  # 1
next(gen)  # 2
try:
    next(gen)
except StopIteration as e:
    print(e.value)  # "完成"
```

---

## 4. 【最小可用】

掌握以下内容，就能在80%的场景中使用生成器：

### 4.1 定义生成器函数

```python
def count_up(n):
    """从 0 数到 n-1"""
    for i in range(n):
        yield i

# 使用
for num in count_up(5):
    print(num)  # 0 1 2 3 4
```

### 4.2 生成器表达式（一行搞定）

```python
# 列表推导式 -> 立即计算
squares_list = [x**2 for x in range(10)]

# 生成器表达式 -> 惰性计算（把 [] 换成 ()）
squares_gen = (x**2 for x in range(10))

# 遍历
for s in squares_gen:
    print(s)
```

### 4.3 用 next() 手动获取值

```python
gen = (x for x in range(3))

print(next(gen))  # 0
print(next(gen))  # 1
print(next(gen))  # 2
# print(next(gen))  # StopIteration!

# 安全获取：提供默认值
gen2 = (x for x in range(2))
print(next(gen2, '没了'))  # 0
print(next(gen2, '没了'))  # 1
print(next(gen2, '没了'))  # '没了'（不会报错）
```

### 4.4 读取大文件

```python
def read_large_file(filepath):
    """逐行读取大文件"""
    with open(filepath, 'r') as f:
        for line in f:
            yield line.strip()

# 处理 10GB 日志文件也不会爆内存
for line in read_large_file('huge.log'):
    if 'ERROR' in line:
        print(line)
```

### 4.5 批量处理（向量数据库常用）

```python
def batch_iter(iterable, batch_size):
    """将任意可迭代对象分批"""
    batch = []
    for item in iterable:
        batch.append(item)
        if len(batch) >= batch_size:
            yield batch
            batch = []
    if batch:
        yield batch

# 批量插入向量
vectors = (generate_vector(i) for i in range(10000))
for batch in batch_iter(vectors, batch_size=100):
    db.insert_many(batch)  # 每次插入100条
```

**这些知识足以：**
- 处理大文件和大数据流
- 实现内存友好的数据处理
- 为向量数据库批量操作打下基础
- 理解异步编程的基础（async/await 基于生成器）

---

## 5. 【1个类比】

### 类比1：生成器 = JavaScript Generator 🎨

Python 生成器和 JavaScript Generator 几乎一模一样！

```javascript
// JavaScript Generator
function* countUp(n) {
    for (let i = 0; i < n; i++) {
        yield i;
    }
}

const gen = countUp(3);
console.log(gen.next().value);  // 0
console.log(gen.next().value);  // 1
console.log(gen.next().value);  // 2
console.log(gen.next().done);   // true
```

```python
# Python Generator
def count_up(n):
    for i in range(n):
        yield i

gen = count_up(3)
print(next(gen))  # 0
print(next(gen))  # 1
print(next(gen))  # 2
# next(gen) -> StopIteration
```

**主要区别：**
- JS 用 `function*` 标记，Python 有 `yield` 就是生成器
- JS 的 `next()` 返回 `{value, done}` 对象
- Python 直接返回值，耗尽抛 `StopIteration`

---

### 类比2：生成器 = 懒加载图片 🖼️

前端经常用懒加载优化性能，生成器就是数据的"懒加载"！

```javascript
// 前端：图片懒加载
// 不是一次加载所有图片，而是滚动到可视区域才加载
<img loading="lazy" src="image.jpg">
```

```python
# Python：数据懒加载
# 不是一次计算所有数据，而是需要时才计算
def lazy_vectors():
    for i in range(1000000):
        yield expensive_computation(i)  # 用到才算

# 可能只用前 10 个
gen = lazy_vectors()
first_10 = [next(gen) for _ in range(10)]
# 只计算了 10 次，不是 100 万次！
```

---

### 类比3：生成器 = 流式 API 响应 📡

前端调用流式 API（如 ChatGPT）时，数据是一点一点到达的。

```javascript
// 前端：流式接收 API 响应
const response = await fetch('/api/stream');
const reader = response.body.getReader();

while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    console.log(new TextDecoder().decode(value));
}
```

```python
# Python：流式产出数据
def stream_response(query):
    """模拟流式 LLM 响应"""
    for token in llm.generate_stream(query):
        yield token  # 一个词一个词产出

# 使用
for token in stream_response("什么是向量数据库？"):
    print(token, end='', flush=True)
```

---

### 类比4：生成器 = React 的 Suspense/Lazy 🔄

```javascript
// React: 懒加载组件
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

// 需要时才加载
<Suspense fallback={<Loading />}>
    <HeavyComponent />
</Suspense>
```

```python
# Python: 懒计算数据
def lazy_embeddings(texts):
    """需要时才计算 embedding"""
    for text in texts:
        yield model.encode(text)  # 用到才算

# 可能不需要全部
embeddings = lazy_embeddings(million_texts)
first_100 = list(itertools.islice(embeddings, 100))
```

---

### 类比5：生成器 = 分页 API 📄

```javascript
// 前端：分页获取数据
async function* fetchPages(url) {
    let page = 1;
    while (true) {
        const response = await fetch(`${url}?page=${page}`);
        const data = await response.json();
        if (data.length === 0) break;
        yield data;
        page++;
    }
}

// 使用
for await (const page of fetchPages('/api/users')) {
    renderUsers(page);
}
```

```python
# Python：分页查询向量数据库
def query_pages(collection, query_vector, page_size=100):
    """分页获取相似向量"""
    offset = 0
    while True:
        results = collection.search(
            query_vector,
            limit=page_size,
            offset=offset
        )
        if not results:
            break
        yield results
        offset += page_size

# 使用
for page in query_pages(my_collection, query_vec):
    process_results(page)
```

---

### 类比总结表

| Python 生成器概念 | 前端对应概念 |
|-----------------|-------------|
| yield | JavaScript yield |
| 生成器函数 | function* |
| 惰性计算 | 懒加载 (lazy loading) |
| next() | iterator.next() |
| StopIteration | { done: true } |
| 生成器表达式 | 无直接对应（类似懒数组） |
| 流式处理 | ReadableStream |

---

## 6. 【反直觉点】

### 误区1：调用生成器函数会执行函数体 ❌

**为什么错？**
- 调用生成器函数只是创建生成器对象
- 函数体的代码一行都不会执行
- 只有调用 `next()` 或遍历时才执行

**为什么人们容易这样错？**
因为普通函数调用就会执行，我们习惯性地认为所有函数都这样。

**正确理解：**
```python
def my_gen():
    print("A")
    yield 1
    print("B")
    yield 2

# 调用函数 - 什么都不打印！
gen = my_gen()
print("创建了生成器")

# 调用 next() 才开始执行
print(next(gen))  # 打印"A"，返回 1
print(next(gen))  # 打印"B"，返回 2

# 输出：
# 创建了生成器
# A
# 1
# B
# 2
```

---

### 误区2：生成器可以重复使用 ❌

**为什么错？**
- 生成器是一次性的，耗尽后不能重置
- 想再次遍历，必须重新创建生成器

**为什么人们容易这样错？**
因为列表可以反复遍历，以为生成器也一样。

**正确理解：**
```python
def gen():
    yield 1
    yield 2

g = gen()

# 第一次遍历
print(list(g))  # [1, 2]

# 第二次遍历
print(list(g))  # [] 空！生成器已耗尽

# 正确做法：重新创建
g = gen()
print(list(g))  # [1, 2]

# 或者：保存为列表（如果内存允许）
data = list(gen())
print(data)  # [1, 2]
print(data)  # [1, 2] - 可以重复使用
```

---

### 误区3：生成器比列表慢 ❌

**为什么错？**
- 生成器在大多数场景下更快
- 避免了内存分配和垃圾回收的开销
- 只有在需要随机访问时，列表才更合适

**为什么人们容易这样错？**
因为"每次调用 next() 都要恢复状态"听起来像是额外开销。

**正确理解：**
```python
import time

# 测试：计算前 100 个满足条件的数
def find_first_n(n, condition):
    """方式1：列表推导式"""
    all_results = [x for x in range(10_000_000) if condition(x)]
    return all_results[:n]

def find_first_n_gen(n, condition):
    """方式2：生成器"""
    count = 0
    for x in range(10_000_000):
        if condition(x):
            yield x
            count += 1
            if count >= n:
                break

condition = lambda x: x % 1000 == 0

# 列表方式：需要遍历完所有 1000 万个数
start = time.time()
result1 = find_first_n(100, condition)
print(f"列表方式: {time.time() - start:.3f}s")

# 生成器方式：找到 100 个就停止
start = time.time()
result2 = list(find_first_n_gen(100, condition))
print(f"生成器方式: {time.time() - start:.6f}s")

# 输出示例：
# 列表方式: 0.850s
# 生成器方式: 0.000100s（快 8500 倍！）
```

---

## 7. 【实战代码】

```python
"""
生成器实战示例：向量数据处理流水线
展示生成器在向量数据库场景中的应用
"""

import time
import random

# ===== 1. 基础：创建生成器 =====
print("=== 1. 基础：创建生成器 ===")

def count_vectors(n):
    """生成 n 个模拟向量"""
    for i in range(n):
        yield [random.random() for _ in range(4)]  # 4维向量

# 只创建生成器，不执行
gen = count_vectors(1000000)
print(f"生成器类型: {type(gen)}")
print(f"获取前3个: {[next(gen) for _ in range(3)]}")

# ===== 2. 生成器表达式 =====
print("\n=== 2. 生成器表达式 ===")

import sys

# 内存对比
list_comp = [x**2 for x in range(1000000)]
gen_expr = (x**2 for x in range(1000000))

print(f"列表内存: {sys.getsizeof(list_comp):,} bytes")
print(f"生成器内存: {sys.getsizeof(gen_expr)} bytes")

# ===== 3. 读取大文件（模拟）=====
print("\n=== 3. 模拟读取大量向量数据 ===")

def read_vectors_from_source(source, limit=None):
    """从数据源逐条读取向量"""
    count = 0
    while limit is None or count < limit:
        # 模拟从文件/数据库读取
        vector = {
            'id': count,
            'embedding': [random.random() for _ in range(128)],
            'metadata': {'source': source}
        }
        yield vector
        count += 1
        if limit and count >= limit:
            break

# 只读取需要的数量
reader = read_vectors_from_source('documents', limit=5)
for vec in reader:
    print(f"ID: {vec['id']}, embedding 维度: {len(vec['embedding'])}")

# ===== 4. 流水线处理 =====
print("\n=== 4. 向量处理流水线 ===")

def normalize_vectors(vectors):
    """归一化向量"""
    for vec in vectors:
        embedding = vec['embedding']
        norm = sum(x**2 for x in embedding) ** 0.5
        vec['embedding'] = [x/norm for x in embedding]
        yield vec

def filter_by_norm(vectors, min_norm=0.5):
    """过滤范数过小的向量"""
    for vec in vectors:
        embedding = vec['embedding']
        norm = sum(x**2 for x in embedding) ** 0.5
        if norm >= min_norm:
            yield vec

def add_timestamp(vectors):
    """添加时间戳"""
    for vec in vectors:
        vec['timestamp'] = time.time()
        yield vec

# 组合流水线：读取 -> 归一化 -> 过滤 -> 添加时间戳
pipeline = add_timestamp(
    filter_by_norm(
        normalize_vectors(
            read_vectors_from_source('docs', limit=10)
        )
    )
)

print("流水线处理结果:")
for i, vec in enumerate(pipeline):
    if i < 3:  # 只打印前3个
        print(f"  ID: {vec['id']}, 有时间戳: {'timestamp' in vec}")

# ===== 5. 批量处理 =====
print("\n=== 5. 批量处理（向量数据库插入）===")

def batch_generator(iterable, batch_size):
    """将数据流分批"""
    batch = []
    for item in iterable:
        batch.append(item)
        if len(batch) >= batch_size:
            yield batch
            batch = []
    if batch:
        yield batch

def mock_db_insert(batch):
    """模拟数据库批量插入"""
    time.sleep(0.01)  # 模拟 IO 延迟
    return len(batch)

# 批量插入 100 条数据，每批 20 条
vectors = read_vectors_from_source('images', limit=100)
total_inserted = 0

for batch_num, batch in enumerate(batch_generator(vectors, batch_size=20)):
    inserted = mock_db_insert(batch)
    total_inserted += inserted
    print(f"  批次 {batch_num + 1}: 插入 {inserted} 条")

print(f"总共插入: {total_inserted} 条")

# ===== 6. 无限生成器 =====
print("\n=== 6. 无限ID生成器 ===")

def infinite_id_generator(prefix='vec'):
    """生成无限唯一ID"""
    counter = 0
    while True:
        yield f"{prefix}_{counter:08d}"
        counter += 1

id_gen = infinite_id_generator()
print("生成的ID:", [next(id_gen) for _ in range(5)])

# ===== 7. 生成器组合：yield from =====
print("\n=== 7. 使用 yield from 组合生成器 ===")

def read_from_multiple_sources(sources):
    """从多个数据源读取"""
    for source in sources:
        yield from read_vectors_from_source(source, limit=2)

sources = ['docs', 'images', 'audio']
combined = read_from_multiple_sources(sources)

print("多源数据:")
for vec in combined:
    print(f"  来源: {vec['metadata']['source']}, ID: {vec['id']}")

# ===== 8. 带状态的生成器 =====
print("\n=== 8. 带统计的生成器 ===")

def counting_generator(iterable):
    """包装生成器，统计处理数量"""
    count = 0
    for item in iterable:
        count += 1
        yield item
    print(f"[统计] 共处理 {count} 个元素")

data = counting_generator(range(10))
result = sum(data)  # 触发遍历
print(f"求和结果: {result}")

# ===== 9. 向量相似度搜索结果流 =====
print("\n=== 9. 模拟流式搜索结果 ===")

def similarity_search_stream(query_vector, threshold=0.5, max_results=100):
    """流式返回相似度搜索结果"""
    for i in range(max_results):
        # 模拟计算相似度
        similarity = random.random()
        if similarity >= threshold:
            yield {
                'id': f'doc_{i}',
                'similarity': round(similarity, 3),
                'content': f'Document {i} content...'
            }

query = [0.1, 0.2, 0.3, 0.4]
results = similarity_search_stream(query, threshold=0.7, max_results=20)

print("搜索结果（相似度 >= 0.7）:")
for r in results:
    print(f"  {r['id']}: 相似度 {r['similarity']}")

# ===== 10. 性能对比 =====
print("\n=== 10. 性能对比 ===")

def measure_time(func, *args):
    start = time.time()
    result = func(*args)
    return time.time() - start, result

# 列表方式
def list_approach(n):
    data = [x**2 for x in range(n)]
    return sum(x for x in data if x % 2 == 0)

# 生成器方式
def gen_approach(n):
    data = (x**2 for x in range(n))
    return sum(x for x in data if x % 2 == 0)

n = 1000000
time_list, result_list = measure_time(list_approach, n)
time_gen, result_gen = measure_time(gen_approach, n)

print(f"列表方式: {time_list:.3f}s")
print(f"生成器方式: {time_gen:.3f}s")
print(f"结果一致: {result_list == result_gen}")
```

**运行输出示例：**
```
=== 1. 基础：创建生成器 ===
生成器类型: <class 'generator'>
获取前3个: [[0.23, 0.45, 0.67, 0.89], ...]

=== 2. 生成器表达式 ===
列表内存: 8,448,728 bytes
生成器内存: 112 bytes

=== 3. 模拟读取大量向量数据 ===
ID: 0, embedding 维度: 128
ID: 1, embedding 维度: 128
...

=== 4. 向量处理流水线 ===
流水线处理结果:
  ID: 0, 有时间戳: True
  ID: 1, 有时间戳: True
  ID: 2, 有时间戳: True

=== 5. 批量处理（向量数据库插入）===
  批次 1: 插入 20 条
  批次 2: 插入 20 条
  批次 3: 插入 20 条
  批次 4: 插入 20 条
  批次 5: 插入 20 条
总共插入: 100 条

=== 6. 无限ID生成器 ===
生成的ID: ['vec_00000000', 'vec_00000001', ...]

=== 7. 使用 yield from 组合生成器 ===
多源数据:
  来源: docs, ID: 0
  来源: docs, ID: 1
  来源: images, ID: 0
  来源: images, ID: 1
  来源: audio, ID: 0
  来源: audio, ID: 1

=== 8. 带统计的生成器 ===
[统计] 共处理 10 个元素
求和结果: 45

=== 9. 模拟流式搜索结果 ===
搜索结果（相似度 >= 0.7）:
  doc_3: 相似度 0.823
  doc_7: 相似度 0.756
  ...

=== 10. 性能对比 ===
列表方式: 0.156s
生成器方式: 0.089s
结果一致: True
```

---

## 8. 【面试必问】

### 问题1："什么是生成器？和普通函数有什么区别？"

**普通回答（❌ 不出彩）：**
"生成器是用 yield 的函数，可以产出多个值。"

**出彩回答（✅ 推荐）：**

> **生成器有三个关键区别：**
>
> 1. **执行方式不同**：普通函数调用立即执行完毕；生成器函数调用只创建生成器对象，需要 `next()` 才开始执行，遇到 `yield` 暂停。
>
> 2. **返回值不同**：普通函数 `return` 一个值就结束；生成器可以 `yield` 多次，每次暂停并产出一个值。
>
> 3. **内存使用不同**：普通函数需要一次性计算所有结果；生成器是惰性计算，按需产出，内存占用恒定。
>
> **实际应用场景：**
> 在向量数据库开发中，处理百万级向量时，我会用生成器逐批读取和处理：
>
> ```python
> def batch_vectors(file, batch_size=1000):
>     batch = []
>     for line in open(file):
>         batch.append(parse(line))
>         if len(batch) >= batch_size:
>             yield batch
>             batch = []
>     if batch:
>         yield batch
> ```
>
> 这样即使处理 10GB 的向量文件，内存也只占用一个批次的大小。

**为什么这个回答出彩？**
1. ✅ 清晰的三点对比，结构化思维
2. ✅ 提到"惰性计算"这个核心概念
3. ✅ 结合实际场景（向量数据库批处理）
4. ✅ 代码示例简洁实用

---

### 问题2："生成器表达式和列表推导式怎么选？"

**普通回答（❌ 不出彩）：**
"数据量大用生成器，小用列表。"

**出彩回答（✅ 推荐）：**

> **选择依据有三个维度：**
>
> | 维度 | 列表推导式 | 生成器表达式 |
> |-----|-----------|-------------|
> | 内存 | O(n) | O(1) |
> | 遍历次数 | 可多次 | 只能一次 |
> | 随机访问 | 支持 | 不支持 |
>
> **具体选择策略：**
>
> 1. **需要多次遍历或随机访问** → 列表
>    ```python
>    data = [x**2 for x in range(100)]
>    print(data[50])  # 需要索引访问
>    ```
>
> 2. **数据量大、只遍历一次** → 生成器
>    ```python
>    vectors = (compute_embedding(doc) for doc in million_docs)
>    for vec in vectors:
>        db.insert(vec)
>    ```
>
> 3. **需要提前知道长度** → 列表（生成器不能 `len()`）
>
> 4. **流式处理、管道操作** → 生成器
>    ```python
>    pipeline = (process(x) for x in (filter(x) for x in raw_data))
>    ```

---

## 9. 【化骨绵掌】

### 卡片1：什么是生成器？ 🎯

**一句话：** 生成器是可以暂停和恢复的函数。

**举例：**
```python
def gen():
    yield 1  # 暂停，产出 1
    yield 2  # 暂停，产出 2

g = gen()
print(next(g))  # 1
print(next(g))  # 2
```

**应用：** 向量数据库中用于流式读取海量数据。

---

### 卡片2：yield vs return 🔄

**一句话：** yield 暂停函数并产出值，return 终止函数并返回值。

**对比：**
```python
# return：执行一次就结束
def normal():
    return 1
    return 2  # 永远不会执行

# yield：可以多次产出
def generator():
    yield 1
    yield 2  # 会执行
```

---

### 卡片3：生成器表达式 📝

**一句话：** 把列表推导式的 `[]` 换成 `()` 就是生成器表达式。

**举例：**
```python
# 列表：立即计算，占内存
list_comp = [x**2 for x in range(1000000)]

# 生成器：惰性计算，几乎不占内存
gen_expr = (x**2 for x in range(1000000))
```

---

### 卡片4：next() 函数 ➡️

**一句话：** next() 让生成器执行到下一个 yield 并返回其值。

**举例：**
```python
gen = (x for x in [1, 2, 3])
print(next(gen))  # 1
print(next(gen))  # 2
print(next(gen))  # 3
print(next(gen, '没了'))  # '没了'（默认值，避免报错）
```

---

### 卡片5：StopIteration 🛑

**一句话：** 生成器耗尽时抛出 StopIteration 异常。

**举例：**
```python
gen = (x for x in [1])
next(gen)  # 1
next(gen)  # StopIteration!

# for 循环自动处理
for x in (x for x in [1]):
    print(x)  # 正常结束，不报错
```

---

### 卡片6：生成器是一次性的 ⚠️

**一句话：** 生成器耗尽后不能重置，需要重新创建。

**举例：**
```python
gen = (x for x in [1, 2, 3])
print(list(gen))  # [1, 2, 3]
print(list(gen))  # [] 空了！

# 解决：重新创建
gen = (x for x in [1, 2, 3])
print(list(gen))  # [1, 2, 3]
```

---

### 卡片7：yield from 🔗

**一句话：** yield from 委托给另一个生成器，简化嵌套迭代。

**举例：**
```python
def combined():
    yield from [1, 2]
    yield from [3, 4]

list(combined())  # [1, 2, 3, 4]

# 等价于
def combined():
    for x in [1, 2]:
        yield x
    for x in [3, 4]:
        yield x
```

---

### 卡片8：惰性计算 💤

**一句话：** 生成器只在需要时才计算，不提前占用资源。

**举例：**
```python
def expensive():
    for i in range(1000000):
        yield i ** 2  # 不是一次算完

gen = expensive()
# 只算前 3 个
result = [next(gen) for _ in range(3)]  # [0, 1, 4]
```

---

### 卡片9：批处理模式 📦

**一句话：** 用生成器将数据流分批，是向量数据库插入的常用模式。

**举例：**
```python
def batch(iterable, size):
    batch = []
    for item in iterable:
        batch.append(item)
        if len(batch) >= size:
            yield batch
            batch = []
    if batch:
        yield batch

for b in batch(range(7), 3):
    print(b)  # [0,1,2] [3,4,5] [6]
```

---

### 卡片10：生成器与异步 🔮

**一句话：** Python 的 async/await 底层基于生成器实现。

**关联：**
```python
# 生成器：yield 暂停
def gen():
    result = yield request()
    return result

# 协程：await 暂停
async def coro():
    result = await request()
    return result
```

**学完生成器，异步编程就有了基础！**

---

## 10. 【一句话总结】

**生成器是通过 yield 实现"暂停-恢复"能力的函数，按需产出值而非一次性计算，在向量数据库开发中用于内存友好地处理海量向量数据的读取、转换和批量操作。**

---

## 📚 学习检查清单

- [ ] 理解 yield 和 return 的区别
- [ ] 会写生成器函数和生成器表达式
- [ ] 知道 next() 的用法和默认值
- [ ] 理解 StopIteration 异常
- [ ] 知道生成器是一次性的
- [ ] 能用生成器处理大文件
- [ ] 能实现批处理生成器
- [ ] 理解惰性计算的优势

## 🔗 下一步学习

学完 yield 基础后，建议学习：
1. **生成器 - send()** - 向生成器发送数据
2. **迭代器协议** - 生成器的底层原理
3. **async/await** - 生成器的异步扩展
