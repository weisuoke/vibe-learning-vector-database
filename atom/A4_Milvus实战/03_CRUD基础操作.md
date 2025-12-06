# CRUD 基础操作

## 1. 【30字核心】

**CRUD 是 Create/Read/Update/Delete 的缩写，对应 Milvus 的 create_collection、search/query、upsert、delete 操作。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### CRUD 的第一性原理 🎯

#### 1. 最基础的定义

**CRUD = 数据的四种基本操作 = 增、查、改、删**

仅此而已！没有更基础的了。

- **Create**：创建数据容器（Collection）和插入数据（Insert）
- **Read**：查询数据（Search 向量搜索 / Query 标量查询）
- **Update**：更新数据（Upsert = Update + Insert）
- **Delete**：删除数据

#### 2. 为什么需要 CRUD？

**核心问题：如何管理向量数据的生命周期？**

向量数据库存储的是 embedding 向量，这些向量需要：
1. **被创建**：把文本/图片转成向量，存入数据库
2. **被检索**：根据查询向量找到相似的向量
3. **被更新**：内容变化时更新对应的向量
4. **被删除**：过期或无效的数据需要清理

CRUD 就是管理这些向量数据的基本操作。

#### 3. CRUD 的三层价值

##### 价值1：数据完整性
提供标准化的数据操作接口，确保数据的一致性和完整性。

##### 价值2：灵活性
支持多种查询方式（向量搜索、标量查询、混合查询）。

##### 价值3：性能优化
Milvus 针对每种操作都做了专门优化（批量插入、ANN 搜索等）。

#### 4. 从第一性原理推导 Milvus CRUD

**推理链：**
```
1. 向量数据需要存储在某个容器中
   ↓
2. 创建 Collection 定义数据结构
   ↓
3. 插入向量数据到 Collection
   ↓
4. 建立索引加速搜索
   ↓
5. 使用 Search/Query 检索数据
   ↓
6. 使用 Upsert/Delete 维护数据
```

#### 5. 一句话总结第一性原理

**CRUD 是向量数据管理的基础，Milvus 通过 Collection 组织数据，通过 Search 实现相似性检索。**

---

## 3. 【3个核心概念】

### 核心概念1：Collection 集合 📦

**Collection 是 Milvus 的数据容器，类似关系型数据库的表（Table），定义了数据的结构。**

```python
from pymilvus import MilvusClient

client = MilvusClient(uri="http://localhost:19530")

# 创建 Collection（最简方式）
client.create_collection(
    collection_name="my_docs",
    dimension=128  # 向量维度
)

# 创建 Collection（完整配置）
from pymilvus import CollectionSchema, FieldSchema, DataType

# 定义字段
fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="title", dtype=DataType.VARCHAR, max_length=256),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=128),
]

# 创建 Schema
schema = CollectionSchema(fields=fields, description="文档集合")

# 创建 Collection
from pymilvus import Collection, connections
connections.connect(host="localhost", port="19530")
collection = Collection(name="my_docs", schema=schema)
```

**Collection 的关键属性：**
- **name**：集合名称（唯一标识）
- **schema**：数据结构定义
- **primary_key**：主键字段
- **vector_field**：向量字段（支持多个）

**在向量数据库中的应用：**
每个业务场景通常对应一个 Collection，如 `product_embeddings`、`document_embeddings`。

---

### 核心概念2：Insert 插入数据 ➕

**Insert 是将向量数据写入 Collection 的操作，支持单条和批量插入。**

```python
from pymilvus import MilvusClient
import numpy as np

client = MilvusClient(uri="http://localhost:19530")

# 准备数据
data = [
    {"id": 1, "title": "Python 入门", "embedding": np.random.rand(128).tolist()},
    {"id": 2, "title": "机器学习基础", "embedding": np.random.rand(128).tolist()},
    {"id": 3, "title": "深度学习实战", "embedding": np.random.rand(128).tolist()},
]

# 插入数据
result = client.insert(
    collection_name="my_docs",
    data=data
)

print(f"插入成功，ID: {result['ids']}")
```

**插入的最佳实践：**
```python
# ✅ 批量插入（推荐）
batch_size = 1000
for i in range(0, len(all_data), batch_size):
    batch = all_data[i:i+batch_size]
    client.insert(collection_name="my_docs", data=batch)

# ❌ 单条插入（性能差）
for item in all_data:
    client.insert(collection_name="my_docs", data=[item])
```

**在向量数据库中的应用：**
RAG 系统中，将文档分块后生成 embedding，批量插入到 Collection。

---

### 核心概念3：Search 向量搜索 🔍

**Search 是 Milvus 的核心能力，通过 ANN（近似最近邻）算法快速找到相似向量。**

```python
from pymilvus import MilvusClient
import numpy as np

client = MilvusClient(uri="http://localhost:19530")

# 查询向量（模拟用户问题的 embedding）
query_vector = np.random.rand(128).tolist()

# 执行搜索
results = client.search(
    collection_name="my_docs",
    data=[query_vector],          # 支持多个查询向量
    anns_field="embedding",       # 向量字段名
    limit=10,                     # 返回数量
    output_fields=["id", "title"] # 返回哪些字段
)

# 解析结果
for hits in results:
    for hit in hits:
        print(f"ID: {hit['id']}, 距离: {hit['distance']}, 标题: {hit['entity']['title']}")
```

**Search 的关键参数：**
| 参数 | 说明 | 示例 |
|------|------|------|
| data | 查询向量列表 | [[0.1, 0.2, ...]] |
| anns_field | 向量字段名 | "embedding" |
| limit | 返回数量 | 10 |
| output_fields | 返回字段 | ["id", "title"] |
| search_params | 搜索参数 | {"metric_type": "L2"} |

**在向量数据库中的应用：**
这是向量数据库的核心功能！RAG 系统用它找到与用户问题最相关的文档。

---

## 4. 【最小可用】

掌握以下内容，就能完成 Milvus 的基本数据操作：

### 4.1 创建 Collection

```python
from pymilvus import MilvusClient

client = MilvusClient(uri="http://localhost:19530")

# 最简单的创建方式
client.create_collection(
    collection_name="quick_start",
    dimension=128
)
```

### 4.2 插入数据

```python
import numpy as np

# 准备数据
data = [
    {"id": i, "vector": np.random.rand(128).tolist()}
    for i in range(100)
]

# 批量插入
client.insert(collection_name="quick_start", data=data)
```

### 4.3 搜索数据

```python
# 查询向量
query = np.random.rand(128).tolist()

# 搜索
results = client.search(
    collection_name="quick_start",
    data=[query],
    limit=5
)

# 打印结果
for hit in results[0]:
    print(f"ID: {hit['id']}, 距离: {hit['distance']}")
```

### 4.4 删除数据

```python
# 按 ID 删除
client.delete(
    collection_name="quick_start",
    ids=[1, 2, 3]
)

# 按条件删除
client.delete(
    collection_name="quick_start",
    filter="id > 50"
)
```

### 4.5 删除 Collection

```python
# 删除整个 Collection
client.drop_collection(collection_name="quick_start")
```

**这些知识足以：**
- 创建向量数据集合
- 存储和检索向量数据
- 维护数据（删除过期数据）
- 构建简单的 RAG 系统

---

## 5. 【1个类比】

### 类比1：Collection = React Component State 📦

**Collection 定义数据结构，就像 TypeScript 定义 React State 的类型。**

```typescript
// React：定义 State 类型
interface DocumentState {
  id: number;
  title: string;
  embedding: number[];  // 向量
}

const [documents, setDocuments] = useState<DocumentState[]>([]);
```

```python
# Milvus：定义 Collection Schema
from pymilvus import FieldSchema, DataType

fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True),
    FieldSchema(name="title", dtype=DataType.VARCHAR, max_length=256),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=128),
]
```

---

### 类比2：Insert = setState/dispatch 📝

**插入数据就像 React 的 setState 或 Redux 的 dispatch。**

```javascript
// React：添加数据
setDocuments(prev => [...prev, newDocument]);

// Redux：dispatch action
dispatch({ type: 'ADD_DOCUMENT', payload: newDocument });
```

```python
# Milvus：插入数据
client.insert(
    collection_name="documents",
    data=[{"id": 1, "title": "新文档", "embedding": [...]}]
)
```

---

### 类比3：Search = Array.filter + sort 🔍

**向量搜索就像数组的过滤和排序，只是用"相似度"而不是精确匹配。**

```javascript
// JavaScript：精确匹配 + 排序
const results = documents
  .filter(doc => doc.category === 'tech')
  .sort((a, b) => b.relevance - a.relevance)
  .slice(0, 10);
```

```python
# Milvus：相似度匹配 + 排序（自动完成）
results = client.search(
    collection_name="documents",
    data=[query_vector],
    limit=10  # 自动按相似度排序，返回前10个
)
```

**关键区别：**
- JS filter 是**精确匹配**：`category === 'tech'`
- Milvus search 是**相似度匹配**：找向量空间中最近的点

---

### 类比4：Delete = Array.filter 删除 🗑️

**删除操作就像用 filter 过滤掉不要的元素。**

```javascript
// JavaScript：删除指定 ID
setDocuments(prev => prev.filter(doc => ![1, 2, 3].includes(doc.id)));
```

```python
# Milvus：删除指定 ID
client.delete(collection_name="documents", ids=[1, 2, 3])
```

---

### 类比5：Schema = PropTypes/TypeScript 接口 📋

**Collection Schema 就像定义组件的 Props 类型。**

```typescript
// TypeScript：定义 Props 接口
interface DocumentProps {
  id: number;           // 必需，数字
  title: string;        // 必需，字符串，最大256
  embedding: number[];  // 必需，数组，长度128
}
```

```python
# Milvus：定义 Schema
fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True),
    FieldSchema(name="title", dtype=DataType.VARCHAR, max_length=256),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=128),
]
```

---

### 类比总结表

| Milvus CRUD | 前端对应概念 | 作用 |
|-------------|------------|------|
| Collection | State 类型定义 | 定义数据结构 |
| Schema | TypeScript Interface | 类型约束 |
| Insert | setState/dispatch | 添加数据 |
| Search | filter + sort | 查询数据 |
| Query | filter（精确匹配） | 条件查询 |
| Delete | filter 删除 | 移除数据 |
| Drop | 清空 State | 删除容器 |

---

## 6. 【反直觉点】

### 误区1：Insert 后立即可以 Search ❌

**为什么错？**
- Insert 只是把数据写入内存/日志
- 数据需要**flush**到磁盘才能被搜索
- 但 MilvusClient 默认会自动 flush

**为什么人们容易这样错？**
习惯了关系型数据库的即时一致性（INSERT 后立即 SELECT 可见）。

**正确理解：**
```python
from pymilvus import Collection, connections

connections.connect(host="localhost", port="19530")
collection = Collection("my_docs")

# 插入数据
collection.insert(data)

# 方式1：手动 flush（确保数据持久化）
collection.flush()

# 方式2：等待一段时间让自动 flush 生效

# 然后才能搜索到
collection.load()  # 加载到内存
results = collection.search(...)

# 注意：MilvusClient 默认自动处理 flush 和 load
# 所以用 MilvusClient 时通常不需要手动操作
```

---

### 误区2：Search 返回的是精确的最近邻 ❌

**为什么错？**
- Milvus 使用 **ANN（近似最近邻）** 算法
- 返回的是"足够近"的结果，不保证是"最近"
- 这是性能和精度的权衡

**为什么人们容易这样错？**
函数名叫"search"，让人以为会返回最佳结果。

**正确理解：**
```python
# Milvus Search 是近似搜索
# 可能错过真正最近的点，但速度极快

# 搜索参数可以调整精度
results = client.search(
    collection_name="my_docs",
    data=[query],
    limit=10,
    search_params={
        "metric_type": "L2",
        "params": {"nprobe": 16}  # nprobe 越大越精确，但越慢
    }
)

# 如果需要精确搜索，可以用暴力搜索（不推荐大数据量）
# 或者调高 nprobe 值
```

---

### 误区3：向量维度可以随意设置 ❌

**为什么错？**
- 向量维度由 **Embedding 模型决定**
- OpenAI text-embedding-ada-002 是 1536 维
- 不同模型维度不同，必须匹配

**为什么人们容易这样错？**
觉得维度是"配置参数"，可以自由选择。

**正确理解：**
```python
# ❌ 错误：随意设置维度
client.create_collection(collection_name="docs", dimension=128)
# 然后用 OpenAI embedding（1536维）插入 → 报错！

# ✅ 正确：根据 Embedding 模型设置维度
# OpenAI text-embedding-ada-002
client.create_collection(collection_name="docs", dimension=1536)

# 或者查询模型文档确认维度
# - OpenAI ada-002: 1536
# - BERT base: 768
# - Sentence-BERT: 384
# - BGE-large: 1024
```

---

## 7. 【实战代码】

```python
"""
Milvus CRUD 完整实战示例
演示 Collection 的创建、数据插入、搜索、删除
"""

from pymilvus import MilvusClient
import numpy as np

# ===== 0. 连接 Milvus =====
print("=== 连接 Milvus ===")
client = MilvusClient(uri="http://localhost:19530")
print("✅ 连接成功")

# ===== 1. 创建 Collection =====
print("\n=== 1. 创建 Collection ===")

collection_name = "book_embeddings"

# 如果已存在，先删除
if client.has_collection(collection_name):
    client.drop_collection(collection_name)
    print(f"已删除旧的 Collection: {collection_name}")

# 创建 Collection（简化方式）
client.create_collection(
    collection_name=collection_name,
    dimension=128,  # 向量维度
)
print(f"✅ 创建 Collection: {collection_name}")

# 查看所有 Collection
collections = client.list_collections()
print(f"当前所有 Collection: {collections}")

# ===== 2. 插入数据 =====
print("\n=== 2. 插入数据 ===")

# 模拟书籍数据
books = [
    {"id": 1, "title": "Python 编程入门", "category": "编程", "price": 59.0},
    {"id": 2, "title": "机器学习实战", "category": "AI", "price": 89.0},
    {"id": 3, "title": "深度学习基础", "category": "AI", "price": 99.0},
    {"id": 4, "title": "JavaScript 高级程序设计", "category": "编程", "price": 79.0},
    {"id": 5, "title": "React 从入门到精通", "category": "前端", "price": 69.0},
    {"id": 6, "title": "Vue.js 实战", "category": "前端", "price": 65.0},
    {"id": 7, "title": "向量数据库原理", "category": "数据库", "price": 88.0},
    {"id": 8, "title": "Milvus 实战指南", "category": "数据库", "price": 78.0},
    {"id": 9, "title": "自然语言处理入门", "category": "AI", "price": 85.0},
    {"id": 10, "title": "推荐系统实践", "category": "AI", "price": 95.0},
]

# 为每本书生成模拟 embedding（实际应用中使用 Embedding 模型）
np.random.seed(42)  # 固定随机种子，保证结果可复现
data = []
for book in books:
    data.append({
        "id": book["id"],
        "vector": np.random.rand(128).tolist(),  # 模拟 embedding
        # 注意：MilvusClient 简化模式下，额外字段会被忽略
        # 需要用完整 Schema 模式才能存储 title, category 等
    })

# 批量插入
result = client.insert(
    collection_name=collection_name,
    data=data
)
print(f"✅ 插入 {len(data)} 条数据")
print(f"插入的 ID: {result['ids']}")

# ===== 3. 搜索数据 =====
print("\n=== 3. 搜索数据 ===")

# 模拟查询向量（比如用户搜索"机器学习入门"的 embedding）
np.random.seed(100)
query_vector = np.random.rand(128).tolist()

# 执行搜索
results = client.search(
    collection_name=collection_name,
    data=[query_vector],
    limit=5,  # 返回前5个最相似的
    output_fields=["id"]  # 返回 id 字段
)

print("搜索结果（前5个最相似）：")
for i, hit in enumerate(results[0]):
    # 找到对应的书籍信息
    book = books[hit["id"] - 1]
    print(f"  {i+1}. ID={hit['id']}, 距离={hit['distance']:.4f}, 书名={book['title']}")

# ===== 4. 批量搜索 =====
print("\n=== 4. 批量搜索 ===")

# 同时搜索多个向量
query_vectors = [
    np.random.rand(128).tolist(),
    np.random.rand(128).tolist(),
]

batch_results = client.search(
    collection_name=collection_name,
    data=query_vectors,
    limit=3
)

for i, hits in enumerate(batch_results):
    print(f"查询 {i+1} 的结果：")
    for hit in hits:
        print(f"  ID={hit['id']}, 距离={hit['distance']:.4f}")

# ===== 5. 查询数据（Query - 标量查询）=====
print("\n=== 5. Query 查询 ===")

# 按 ID 查询
query_result = client.query(
    collection_name=collection_name,
    filter="id in [1, 2, 3]",  # 查询 ID 为 1, 2, 3 的数据
    output_fields=["id"]
)

print("按 ID 查询结果：")
for item in query_result:
    print(f"  ID={item['id']}")

# ===== 6. 获取数据统计 =====
print("\n=== 6. 数据统计 ===")

# 获取 Collection 信息
stats = client.get_collection_stats(collection_name)
print(f"Collection 统计: {stats}")

# ===== 7. 删除数据 =====
print("\n=== 7. 删除数据 ===")

# 按 ID 删除
client.delete(
    collection_name=collection_name,
    ids=[9, 10]  # 删除 ID 为 9, 10 的数据
)
print("✅ 已删除 ID=9, 10 的数据")

# 按条件删除
client.delete(
    collection_name=collection_name,
    filter="id > 7"  # 删除 ID > 7 的数据
)
print("✅ 已删除 ID > 7 的数据")

# 验证删除
remaining = client.query(
    collection_name=collection_name,
    filter="id >= 1",
    output_fields=["id"]
)
print(f"剩余数据数量: {len(remaining)}")

# ===== 8. 更新数据（Upsert）=====
print("\n=== 8. 更新数据（Upsert）===")

# Upsert = Update + Insert
# 如果 ID 存在则更新，不存在则插入
upsert_data = [
    {"id": 1, "vector": np.random.rand(128).tolist()},  # 更新 ID=1
    {"id": 100, "vector": np.random.rand(128).tolist()}, # 新增 ID=100
]

client.upsert(
    collection_name=collection_name,
    data=upsert_data
)
print("✅ Upsert 完成（更新 ID=1，新增 ID=100）")

# ===== 9. 清理 =====
print("\n=== 9. 清理 ===")

# 删除 Collection
client.drop_collection(collection_name)
print(f"✅ 已删除 Collection: {collection_name}")

# 验证
if not client.has_collection(collection_name):
    print(f"✅ 确认 Collection 已删除")

# 关闭连接
client.close()
print("✅ 连接已关闭")

print("\n=== CRUD 示例完成 ===")
```

**运行输出示例：**
```
=== 连接 Milvus ===
✅ 连接成功

=== 1. 创建 Collection ===
✅ 创建 Collection: book_embeddings
当前所有 Collection: ['book_embeddings']

=== 2. 插入数据 ===
✅ 插入 10 条数据
插入的 ID: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

=== 3. 搜索数据 ===
搜索结果（前5个最相似）：
  1. ID=7, 距离=5.2341, 书名=向量数据库原理
  2. ID=3, 距离=5.4521, 书名=深度学习基础
  3. ID=10, 距离=5.6234, 书名=推荐系统实践
  4. ID=1, 距离=5.8123, 书名=Python 编程入门
  5. ID=8, 距离=5.9456, 书名=Milvus 实战指南

=== 4. 批量搜索 ===
查询 1 的结果：
  ID=4, 距离=5.1234
  ID=6, 距离=5.3456
  ID=2, 距离=5.5678
查询 2 的结果：
  ID=5, 距离=4.9876
  ID=9, 距离=5.2345
  ID=1, 距离=5.4567

=== 5. Query 查询 ===
按 ID 查询结果：
  ID=1
  ID=2
  ID=3

=== 6. 数据统计 ===
Collection 统计: {'row_count': 10}

=== 7. 删除数据 ===
✅ 已删除 ID=9, 10 的数据
✅ 已删除 ID > 7 的数据
剩余数据数量: 7

=== 8. 更新数据（Upsert）===
✅ Upsert 完成（更新 ID=1，新增 ID=100）

=== 9. 清理 ===
✅ 已删除 Collection: book_embeddings
✅ 确认 Collection 已删除
✅ 连接已关闭

=== CRUD 示例完成 ===
```

---

## 8. 【面试必问】

### 问题："Milvus 的 Search 和 Query 有什么区别？"

**普通回答（❌ 不出彩）：**
"Search 是向量搜索，Query 是普通查询。"

**出彩回答（✅ 推荐）：**

> **Search 和 Query 是两种不同的检索方式：**
>
> 1. **Search（向量搜索）**
>    - 基于 **ANN（近似最近邻）算法**
>    - 输入：查询向量
>    - 输出：相似度最高的 Top-K 结果
>    - 特点：**模糊匹配**，按相似度排序
>    - 场景：语义搜索、推荐系统
>
> 2. **Query（标量查询）**
>    - 基于 **表达式过滤**
>    - 输入：过滤条件（如 `id > 10`）
>    - 输出：满足条件的所有结果
>    - 特点：**精确匹配**，类似 SQL WHERE
>    - 场景：按 ID 查询、条件筛选
>
> **实际应用中的组合：**
> ```python
> # 混合查询：先向量搜索，再标量过滤
> results = collection.search(
>     data=[query_vector],
>     filter="category == 'AI' and price < 100",  # 标量过滤
>     limit=10
> )
> ```
>
> **在 RAG 系统中**，通常先用 Search 找到语义相关的文档，再用 Query 按元数据（时间、来源）进一步筛选。

**为什么这个回答出彩？**
1. ✅ 清晰对比两种查询方式
2. ✅ 解释了底层原理（ANN vs 过滤）
3. ✅ 给出了实际应用场景
4. ✅ 展示了高级用法（混合查询）

---

## 9. 【化骨绵掌】

### 卡片1：什么是 CRUD？ 📝

**一句话：** CRUD 是 Create/Read/Update/Delete 的缩写，代表数据的四种基本操作。

**举例：**
```
Create → 创建 Collection、插入数据
Read   → Search（向量搜索）、Query（标量查询）
Update → Upsert（更新或插入）
Delete → 删除数据、删除 Collection
```

**应用：** 所有数据库操作都可以归类为 CRUD。

---

### 卡片2：Collection 是什么？ 📦

**一句话：** Collection 是 Milvus 的数据容器，类似关系型数据库的表。

**举例：**
```python
client.create_collection(
    collection_name="documents",
    dimension=128
)
```

**应用：** 每个业务场景通常对应一个 Collection。

---

### 卡片3：Schema 数据结构 📋

**一句话：** Schema 定义 Collection 的字段类型，包括主键、向量、标量字段。

**举例：**
```python
fields = [
    FieldSchema("id", DataType.INT64, is_primary=True),
    FieldSchema("embedding", DataType.FLOAT_VECTOR, dim=128),
    FieldSchema("title", DataType.VARCHAR, max_length=256),
]
```

**应用：** 生产环境建议用完整 Schema，支持存储更多元数据。

---

### 卡片4：Insert 插入数据 ➕

**一句话：** Insert 将向量数据写入 Collection，推荐批量插入提高性能。

**举例：**
```python
data = [{"id": i, "vector": [...]} for i in range(1000)]
client.insert(collection_name="docs", data=data)
```

**应用：** RAG 系统将文档 embedding 批量插入到 Milvus。

---

### 卡片5：Search 向量搜索 🔍

**一句话：** Search 通过 ANN 算法找到与查询向量最相似的 Top-K 结果。

**举例：**
```python
results = client.search(
    collection_name="docs",
    data=[query_vector],
    limit=10
)
```

**应用：** 语义搜索、相似推荐的核心操作。

---

### 卡片6：Query 标量查询 📊

**一句话：** Query 根据过滤条件精确查询数据，类似 SQL 的 WHERE。

**举例：**
```python
results = client.query(
    collection_name="docs",
    filter="id in [1, 2, 3]"
)
```

**应用：** 按 ID 获取数据、按条件筛选。

---

### 卡片7：Delete 删除数据 🗑️

**一句话：** Delete 支持按 ID 删除或按条件删除。

**举例：**
```python
# 按 ID 删除
client.delete(collection_name="docs", ids=[1, 2])

# 按条件删除
client.delete(collection_name="docs", filter="id > 100")
```

**应用：** 清理过期数据、删除无效记录。

---

### 卡片8：Upsert 更新数据 🔄

**一句话：** Upsert = Update + Insert，ID 存在则更新，不存在则插入。

**举例：**
```python
client.upsert(
    collection_name="docs",
    data=[{"id": 1, "vector": [...]}]  # 更新 ID=1
)
```

**应用：** 内容更新后重新生成 embedding。

---

### 卡片9：Drop 删除 Collection ⚠️

**一句话：** Drop 删除整个 Collection，包括所有数据，谨慎使用！

**举例：**
```python
client.drop_collection(collection_name="docs")
```

**应用：** 清理测试数据、重建索引时使用。

---

### 卡片10：CRUD 最佳实践 ✨

**一句话：** 批量操作、合理索引、定期清理是提高性能的关键。

**举例：**
```
✅ 批量 Insert（1000条/批）
✅ 合适的索引类型（IVF_FLAT/HNSW）
✅ 定期删除过期数据
❌ 单条插入
❌ 不建索引直接搜索
```

**应用：** 生产环境必须遵循的性能优化原则。

---

## 10. 【一句话总结】

**CRUD 是向量数据库的基本操作，Milvus 通过 Collection 组织数据，Insert 批量写入，Search 实现 ANN 相似搜索，是构建 RAG/推荐系统的核心能力。**

---

## 📚 学习检查清单

- [ ] 能创建 Collection 并定义 Schema
- [ ] 能批量插入向量数据
- [ ] 能使用 Search 进行向量搜索
- [ ] 能使用 Query 进行标量查询
- [ ] 理解 Search 和 Query 的区别
- [ ] 能正确删除数据和 Collection

## 🔗 下一步学习

学完 CRUD，下一步学习 **混合查询**，掌握向量搜索与标量过滤的组合使用！

## 📖 参考资源

- [Milvus CRUD 官方文档](https://milvus.io/docs/insert_data.md)
- [Collection 管理](https://milvus.io/docs/manage-collections.md)
- [Search 参数详解](https://milvus.io/docs/search.md)
