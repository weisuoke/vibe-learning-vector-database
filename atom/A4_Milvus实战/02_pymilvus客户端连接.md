# PyMilvus 客户端连接

## 1. 【30字核心】

**PyMilvus 是 Milvus 的 Python SDK，提供连接管理和 API 封装，是 Python 应用操作向量数据库的桥梁。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### PyMilvus 连接的第一性原理 🎯

#### 1. 最基础的定义

**PyMilvus = gRPC 客户端 + API 封装 + 连接池管理**

仅此而已！没有更基础的了。

- **gRPC 客户端**：与 Milvus 服务器通信的协议
- **API 封装**：把复杂的 gRPC 调用封装成简单的 Python 函数
- **连接池管理**：复用连接，提高性能

#### 2. 为什么需要 PyMilvus？

**核心问题：应用程序如何与 Milvus 服务器交互？**

Milvus 是一个独立的服务进程，运行在某个端口（默认 19530）。你的 Python 程序需要：
1. 建立网络连接
2. 序列化请求数据
3. 发送请求，接收响应
4. 反序列化响应数据
5. 处理错误和重试

这些底层细节很繁琐！PyMilvus 把它们全部封装好了。

#### 3. PyMilvus 的三层价值

##### 价值1：简化开发
把复杂的 gRPC 调用封装成简单的 Python 函数。

```python
# 没有 PyMilvus：需要手写 gRPC 调用
# stub.Search(search_pb2.SearchRequest(...))

# 有 PyMilvus：一行代码
collection.search(vectors, "embedding", params, limit=10)
```

##### 价值2：连接管理
自动管理连接池、重连、超时等。

##### 价值3：类型安全
提供 Python 对象（Collection、Partition 等），IDE 可以自动补全。

#### 4. 从第一性原理推导连接流程

**推理链：**
```
1. Milvus 服务运行在 19530 端口
   ↓
2. Python 程序需要与之通信
   ↓
3. 通信协议是 gRPC
   ↓
4. PyMilvus 封装了 gRPC 细节
   ↓
5. 调用 connections.connect() 建立连接
   ↓
6. 连接成功后可以操作 Collection
```

#### 5. 一句话总结第一性原理

**PyMilvus 是 Milvus 的 Python 门户，屏蔽网络通信细节，让开发者专注于业务逻辑。**

---

## 3. 【3个核心概念】

### 核心概念1：connections 连接管理器 🔌

**connections 是 PyMilvus 的全局连接管理器，负责创建、维护、复用与 Milvus 服务器的连接。**

```python
from pymilvus import connections

# 建立连接（给连接起个别名 "default"）
connections.connect(
    alias="default",      # 连接别名，后续操作使用
    host="localhost",     # Milvus 服务器地址
    port="19530"          # Milvus 服务器端口
)

# 检查连接状态
print(connections.list_connections())  # [('default', {...})]

# 断开连接
connections.disconnect("default")
```

**为什么用别名（alias）？**

```python
# 可以同时连接多个 Milvus 实例
connections.connect(alias="dev", host="dev-milvus", port="19530")
connections.connect(alias="prod", host="prod-milvus", port="19530")

# 操作时指定使用哪个连接
Collection("my_collection", using="dev")   # 使用开发环境
Collection("my_collection", using="prod")  # 使用生产环境
```

**在向量数据库中的应用：**
实际项目中可能有多个 Milvus 实例（开发、测试、生产），别名机制让切换变得简单。

---

### 核心概念2：MilvusClient 简化客户端 🎯

**MilvusClient 是 PyMilvus 2.x 新增的简化接口，一个对象完成所有操作，适合快速开发。**

```python
from pymilvus import MilvusClient

# 一行代码完成连接
client = MilvusClient(uri="http://localhost:19530")

# 直接操作，无需关心 Collection 对象
client.create_collection(
    collection_name="my_collection",
    dimension=128
)

# 插入数据
client.insert(
    collection_name="my_collection",
    data=[{"id": 1, "vector": [0.1] * 128}]
)

# 搜索
results = client.search(
    collection_name="my_collection",
    data=[[0.1] * 128],
    limit=10
)

# 关闭连接
client.close()
```

**MilvusClient vs connections + Collection：**

| 特性 | MilvusClient | connections + Collection |
|------|-------------|-------------------------|
| 代码量 | 少 | 多 |
| 灵活性 | 一般 | 高 |
| 适用场景 | 快速原型、简单应用 | 复杂应用、精细控制 |
| 多实例 | 创建多个 client | 使用 alias |

**在向量数据库中的应用：**
学习和原型开发推荐用 MilvusClient，生产环境可根据需要选择。

---

### 核心概念3：连接参数配置 ⚙️

**连接参数控制超时、认证、安全等行为，生产环境必须正确配置。**

```python
from pymilvus import connections

# 完整连接参数
connections.connect(
    alias="default",
    host="localhost",
    port="19530",
    
    # 认证（如果启用了用户认证）
    user="username",
    password="password",
    
    # 安全连接（TLS/SSL）
    secure=True,
    server_pem_path="/path/to/server.pem",
    
    # 超时设置（秒）
    timeout=30,
)

# 使用 URI 格式（更简洁）
connections.connect(
    alias="default",
    uri="http://localhost:19530"
)

# 带认证的 URI
connections.connect(
    alias="default",
    uri="http://username:password@localhost:19530"
)
```

**关键参数说明：**

| 参数 | 说明 | 默认值 |
|------|------|--------|
| host | 服务器地址 | localhost |
| port | 服务器端口 | 19530 |
| user/password | 认证凭据 | 无 |
| secure | 是否启用 TLS | False |
| timeout | 连接超时（秒） | 无限制 |

**在向量数据库中的应用：**
生产环境必须配置认证和超时，避免安全风险和连接泄漏。

---

## 4. 【最小可用】

掌握以下内容，就能开始连接 Milvus：

### 4.1 安装 PyMilvus

```bash
# 安装最新版
pip install pymilvus

# 安装指定版本（与 Milvus 服务器版本匹配）
pip install pymilvus==2.3.3

# 验证安装
python -c "import pymilvus; print(pymilvus.__version__)"
```

### 4.2 基础连接（推荐 MilvusClient）

```python
from pymilvus import MilvusClient

# 连接本地 Milvus
client = MilvusClient(uri="http://localhost:19530")

# 验证连接
print(client.list_collections())

# 使用完毕关闭
client.close()
```

### 4.3 传统连接方式

```python
from pymilvus import connections, utility

# 建立连接
connections.connect(host="localhost", port="19530")

# 验证连接
print(f"Milvus 版本: {utility.get_server_version()}")
print(f"已有集合: {utility.list_collections()}")

# 断开连接
connections.disconnect("default")
```

### 4.4 连接上下文管理器

```python
from pymilvus import MilvusClient

# 使用 with 语句自动管理连接
with MilvusClient(uri="http://localhost:19530") as client:
    collections = client.list_collections()
    print(f"集合列表: {collections}")
# 退出 with 块自动关闭连接
```

**这些知识足以：**
- 连接到本地或远程 Milvus 实例
- 验证连接是否成功
- 正确关闭连接释放资源
- 为后续 CRUD 操作做准备

---

## 5. 【1个类比】

### 类比1：PyMilvus = Axios/Fetch API 🌐

**前端用 Axios 调用后端 API，Python 用 PyMilvus 调用 Milvus API。**

```javascript
// 前端：Axios 连接后端
import axios from 'axios';

// 创建 axios 实例（类似 MilvusClient）
const api = axios.create({
  baseURL: 'http://localhost:3000',
  timeout: 5000,
  headers: { 'Authorization': 'Bearer token' }
});

// 发送请求
const response = await api.get('/users');
```

```python
# Python：PyMilvus 连接 Milvus
from pymilvus import MilvusClient

# 创建 client 实例
client = MilvusClient(
    uri="http://localhost:19530",
    timeout=5.0,
    token="username:password"
)

# 发送请求
collections = client.list_collections()
```

---

### 类比2：connections.connect = React Context Provider 🎭

**connections 管理全局连接，就像 React Context 管理全局状态。**

```jsx
// React：Context 提供全局状态
const DatabaseContext = createContext();

function App() {
  return (
    <DatabaseContext.Provider value={dbConnection}>
      <MyComponent />
    </DatabaseContext.Provider>
  );
}

// 子组件使用全局状态
function MyComponent() {
  const db = useContext(DatabaseContext);
  // 使用 db 进行操作
}
```

```python
# PyMilvus：connections 提供全局连接
from pymilvus import connections, Collection

# 建立全局连接
connections.connect(alias="default", host="localhost", port="19530")

# 任何地方都可以使用这个连接
def my_function():
    # 自动使用 "default" 连接
    collection = Collection("my_collection")
    # 使用 collection 进行操作
```

---

### 类比3：alias = 环境变量 NODE_ENV 🏷️

**连接别名区分不同环境，就像 NODE_ENV 区分开发和生产。**

```javascript
// 前端：根据环境选择 API 地址
const API_URL = process.env.NODE_ENV === 'production'
  ? 'https://api.prod.com'
  : 'http://localhost:3000';

const api = axios.create({ baseURL: API_URL });
```

```python
# PyMilvus：用别名区分环境
import os
from pymilvus import connections, Collection

# 连接多个环境
connections.connect(alias="dev", host="dev-milvus", port="19530")
connections.connect(alias="prod", host="prod-milvus", port="19530")

# 根据环境选择连接
env = os.getenv("MILVUS_ENV", "dev")
collection = Collection("my_collection", using=env)
```

---

### 类比4：timeout = Promise.race 超时控制 ⏱️

**连接超时机制，就像前端用 Promise.race 实现请求超时。**

```javascript
// 前端：超时控制
const fetchWithTimeout = (url, timeout = 5000) => {
  return Promise.race([
    fetch(url),
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Timeout')), timeout)
    )
  ]);
};
```

```python
# PyMilvus：超时控制
from pymilvus import connections

connections.connect(
    host="localhost",
    port="19530",
    timeout=5.0  # 5秒超时
)
```

---

### 类比总结表

| PyMilvus 概念 | 前端对应概念 | 作用 |
|--------------|------------|------|
| MilvusClient | axios.create() | 创建 API 客户端 |
| connections.connect | Context Provider | 建立全局连接 |
| alias | NODE_ENV | 区分不同环境 |
| timeout | Promise.race | 超时控制 |
| client.close() | cleanup function | 资源释放 |
| uri 格式 | baseURL | 服务器地址 |

---

## 6. 【反直觉点】

### 误区1：每次操作都需要 connect/disconnect ❌

**为什么错？**
- connections 维护的是**连接池**
- 一次 connect 可以复用很多次
- 频繁 connect/disconnect 会严重影响性能

**为什么人们容易这样错？**
习惯了"用完就关"的思维，类似文件操作的 open/close。但数据库连接更像"打开水龙头"，开一次用很久。

**正确理解：**
```python
# ❌ 错误：每次都连接/断开
def search_vector(vector):
    connections.connect(host="localhost", port="19530")
    collection = Collection("my_collection")
    results = collection.search([vector], "embedding", {}, limit=10)
    connections.disconnect("default")
    return results

# ✅ 正确：应用启动时连接一次
# app.py
connections.connect(host="localhost", port="19530")

# 任何地方都可以直接使用
def search_vector(vector):
    collection = Collection("my_collection")
    return collection.search([vector], "embedding", {}, limit=10)

# 应用关闭时断开
# atexit.register(lambda: connections.disconnect("default"))
```

---

### 误区2：MilvusClient 和 connections 可以混用 ❌

**为什么错？**
- MilvusClient 内部维护自己的连接
- connections 维护的是全局连接池
- 混用会导致连接混乱、资源泄漏

**为什么人们容易这样错？**
看文档时两种方式都学了，就想着"都用上"。

**正确理解：**
```python
# ❌ 错误：混用两种方式
from pymilvus import MilvusClient, connections, Collection

client = MilvusClient(uri="http://localhost:19530")
connections.connect(host="localhost", port="19530")

collection = Collection("test")  # 用的是哪个连接？

# ✅ 正确：选择一种方式，坚持使用

# 方式1：MilvusClient（推荐简单场景）
client = MilvusClient(uri="http://localhost:19530")
client.create_collection(...)
client.insert(...)

# 方式2：connections + Collection（复杂场景）
connections.connect(host="localhost", port="19530")
collection = Collection("test")
collection.insert(...)
```

---

### 误区3：连接失败就是服务器挂了 ❌

**为什么错？**
连接失败的原因很多：
- 网络不通（防火墙、VPN）
- 端口错误（19530 vs 19531）
- 服务未启动
- 认证失败
- 超时设置太短

**为什么人们容易这样错？**
看到"Connection refused"就以为服务器出问题了。

**正确理解：**
```python
from pymilvus import connections, MilvusException

def connect_with_retry():
    """带重试的连接函数"""
    import time
    
    for attempt in range(3):
        try:
            connections.connect(
                host="localhost",
                port="19530",
                timeout=10  # 给足够的超时时间
            )
            print("✅ 连接成功")
            return True
        except MilvusException as e:
            print(f"⚠️ 连接失败 (尝试 {attempt + 1}/3): {e}")
            time.sleep(2)
    
    return False

# 排查步骤
# 1. 检查服务是否运行: docker-compose ps
# 2. 检查端口是否开放: curl localhost:19530
# 3. 检查网络连通性: ping milvus-host
# 4. 检查认证配置: 用户名密码是否正确
```

---

## 7. 【实战代码】

```python
"""
PyMilvus 客户端连接实战示例
演示多种连接方式和最佳实践
"""

from pymilvus import (
    connections,
    utility,
    MilvusClient,
    MilvusException
)
import time

# ===== 1. 基础连接（MilvusClient 方式）=====
print("=== 1. MilvusClient 基础连接 ===")

try:
    # 最简单的连接方式
    client = MilvusClient(uri="http://localhost:19530")
    
    # 验证连接
    collections = client.list_collections()
    print(f"✅ 连接成功！现有集合: {collections}")
    
    # 关闭连接
    client.close()
    print("✅ 连接已关闭")
    
except MilvusException as e:
    print(f"❌ 连接失败: {e}")

# ===== 2. 传统连接方式（connections 模块）=====
print("\n=== 2. connections 模块连接 ===")

try:
    # 建立连接
    connections.connect(
        alias="default",
        host="localhost",
        port="19530"
    )
    
    # 验证连接
    version = utility.get_server_version()
    collections = utility.list_collections()
    
    print(f"✅ 连接成功！")
    print(f"   Milvus 版本: {version}")
    print(f"   现有集合: {collections}")
    
    # 查看连接列表
    print(f"   活跃连接: {connections.list_connections()}")
    
    # 断开连接
    connections.disconnect("default")
    print("✅ 连接已断开")
    
except MilvusException as e:
    print(f"❌ 连接失败: {e}")

# ===== 3. 多环境连接（使用别名）=====
print("\n=== 3. 多环境连接示例 ===")

def connect_multi_env():
    """演示连接多个 Milvus 实例（这里用同一个实例模拟）"""
    
    # 连接"开发环境"
    connections.connect(
        alias="dev",
        host="localhost",
        port="19530"
    )
    
    # 连接"测试环境"（实际项目中是不同的 host）
    connections.connect(
        alias="test",
        host="localhost",
        port="19530"
    )
    
    print(f"✅ 已连接环境: {connections.list_connections()}")
    
    # 使用特定环境
    dev_collections = utility.list_collections(using="dev")
    test_collections = utility.list_collections(using="test")
    
    print(f"   开发环境集合: {dev_collections}")
    print(f"   测试环境集合: {test_collections}")
    
    # 断开所有连接
    connections.disconnect("dev")
    connections.disconnect("test")
    print("✅ 所有连接已断开")

try:
    connect_multi_env()
except MilvusException as e:
    print(f"❌ 多环境连接失败: {e}")

# ===== 4. 带重试的连接 =====
print("\n=== 4. 带重试的连接 ===")

def connect_with_retry(host="localhost", port="19530", max_retries=3, retry_delay=2):
    """带重试机制的连接函数"""
    
    for attempt in range(max_retries):
        try:
            connections.connect(
                alias="default",
                host=host,
                port=port,
                timeout=10
            )
            print(f"✅ 连接成功（尝试 {attempt + 1}/{max_retries}）")
            return True
            
        except MilvusException as e:
            print(f"⚠️ 连接失败（尝试 {attempt + 1}/{max_retries}）: {e}")
            
            if attempt < max_retries - 1:
                print(f"   {retry_delay} 秒后重试...")
                time.sleep(retry_delay)
    
    print("❌ 达到最大重试次数，连接失败")
    return False

# 测试重试连接
if connect_with_retry():
    connections.disconnect("default")

# ===== 5. 上下文管理器方式 =====
print("\n=== 5. 上下文管理器（自动关闭）===")

class MilvusConnection:
    """自定义上下文管理器，确保连接正确关闭"""
    
    def __init__(self, alias="default", **kwargs):
        self.alias = alias
        self.kwargs = kwargs
    
    def __enter__(self):
        connections.connect(alias=self.alias, **self.kwargs)
        print(f"✅ 连接已建立 (alias={self.alias})")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        connections.disconnect(self.alias)
        print(f"✅ 连接已关闭 (alias={self.alias})")
        return False

# 使用上下文管理器
try:
    with MilvusConnection(host="localhost", port="19530") as conn:
        version = utility.get_server_version()
        print(f"   Milvus 版本: {version}")
except MilvusException as e:
    print(f"❌ 连接失败: {e}")

# ===== 6. 连接健康检查 =====
print("\n=== 6. 连接健康检查 ===")

def health_check(alias="default"):
    """检查连接是否健康"""
    try:
        # 尝试执行一个简单操作
        utility.list_collections(using=alias)
        return True
    except:
        return False

# 建立连接
connections.connect(host="localhost", port="19530")

# 健康检查
if health_check():
    print("✅ 连接健康")
else:
    print("❌ 连接异常")

# 清理
connections.disconnect("default")

# ===== 7. 完整的生产级连接示例 =====
print("\n=== 7. 生产级连接配置 ===")

def create_production_connection():
    """生产环境推荐的连接配置"""
    
    import os
    
    # 从环境变量读取配置
    config = {
        "host": os.getenv("MILVUS_HOST", "localhost"),
        "port": os.getenv("MILVUS_PORT", "19530"),
        "user": os.getenv("MILVUS_USER", ""),
        "password": os.getenv("MILVUS_PASSWORD", ""),
    }
    
    # 构建连接参数
    connect_params = {
        "alias": "default",
        "host": config["host"],
        "port": config["port"],
        "timeout": 30,  # 生产环境给足够的超时时间
    }
    
    # 如果配置了认证
    if config["user"] and config["password"]:
        connect_params["user"] = config["user"]
        connect_params["password"] = config["password"]
    
    print(f"连接配置: {config['host']}:{config['port']}")
    
    # 建立连接
    connections.connect(**connect_params)
    
    # 验证连接
    version = utility.get_server_version()
    print(f"✅ 生产连接成功，Milvus 版本: {version}")
    
    return True

try:
    create_production_connection()
    connections.disconnect("default")
except MilvusException as e:
    print(f"❌ 生产连接失败: {e}")

print("\n=== 示例完成 ===")
```

**运行输出示例：**
```
=== 1. MilvusClient 基础连接 ===
✅ 连接成功！现有集合: []
✅ 连接已关闭

=== 2. connections 模块连接 ===
✅ 连接成功！
   Milvus 版本: v2.3.3
   现有集合: []
   活跃连接: [('default', {'address': 'localhost:19530'})]
✅ 连接已断开

=== 3. 多环境连接示例 ===
✅ 已连接环境: [('dev', {...}), ('test', {...})]
   开发环境集合: []
   测试环境集合: []
✅ 所有连接已断开

=== 4. 带重试的连接 ===
✅ 连接成功（尝试 1/3）

=== 5. 上下文管理器（自动关闭）===
✅ 连接已建立 (alias=default)
   Milvus 版本: v2.3.3
✅ 连接已关闭 (alias=default)

=== 6. 连接健康检查 ===
✅ 连接健康

=== 7. 生产级连接配置 ===
连接配置: localhost:19530
✅ 生产连接成功，Milvus 版本: v2.3.3

=== 示例完成 ===
```

---

## 8. 【面试必问】

### 问题："如何用 Python 连接 Milvus？有哪些注意事项？"

**普通回答（❌ 不出彩）：**
"用 pymilvus 库，调用 connections.connect() 就行了，传入 host 和 port。"

**出彩回答（✅ 推荐）：**

> **PyMilvus 提供两种连接方式：**
>
> 1. **MilvusClient（推荐快速开发）**
>    - 一个对象完成所有操作
>    - `client = MilvusClient(uri="http://localhost:19530")`
>    - 适合原型开发、简单应用
>
> 2. **connections 模块（精细控制）**
>    - 支持别名管理多个连接
>    - 支持全局连接池复用
>    - 适合复杂应用、多环境切换
>
> **生产环境的注意事项：**
>
> 1. **连接复用**：应用启动时连接一次，不要每次操作都 connect/disconnect
> 2. **超时设置**：必须设置合理的 timeout，避免无限等待
> 3. **认证配置**：生产环境必须启用用户认证
> 4. **健康检查**：定期检查连接状态，实现自动重连
> 5. **资源释放**：应用关闭时正确断开连接
>
> **在我们的项目中**，使用 connections 模块配合环境变量，开发环境用 localhost，生产环境从 K8s ConfigMap 读取配置。

**为什么这个回答出彩？**
1. ✅ 对比了两种连接方式
2. ✅ 指出各自适用场景
3. ✅ 强调生产环境注意事项
4. ✅ 有实际项目经验

---

## 9. 【化骨绵掌】

### 卡片1：什么是 PyMilvus？ 🐍

**一句话：** PyMilvus 是 Milvus 的 Python SDK，封装了与 Milvus 服务器通信的所有细节。

**举例：**
```python
pip install pymilvus
from pymilvus import MilvusClient
```

**应用：** 所有 Python 应用连接 Milvus 的必备库。

---

### 卡片2：MilvusClient 简化连接 🎯

**一句话：** MilvusClient 是最简单的连接方式，一个对象完成所有操作。

**举例：**
```python
client = MilvusClient(uri="http://localhost:19530")
client.list_collections()
client.close()
```

**应用：** 学习、原型开发首选方式。

---

### 卡片3：connections 模块 🔌

**一句话：** connections 是全局连接管理器，支持多连接和别名。

**举例：**
```python
from pymilvus import connections
connections.connect(alias="default", host="localhost", port="19530")
connections.disconnect("default")
```

**应用：** 需要连接多个 Milvus 实例时使用。

---

### 卡片4：连接别名 alias 🏷️

**一句话：** 别名让你可以同时管理多个连接，用名字区分。

**举例：**
```python
connections.connect(alias="dev", host="dev-server", port="19530")
connections.connect(alias="prod", host="prod-server", port="19530")
Collection("test", using="dev")  # 使用开发环境
```

**应用：** 开发、测试、生产环境切换。

---

### 卡片5：连接参数 ⚙️

**一句话：** 关键参数有 host（地址）、port（端口）、timeout（超时）、user/password（认证）。

**举例：**
```python
connections.connect(
    host="localhost",
    port="19530",
    timeout=30,
    user="admin",
    password="secret"
)
```

**应用：** 生产环境必须配置认证和超时。

---

### 卡片6：连接复用 ♻️

**一句话：** 建立一次连接，多次复用，不要频繁 connect/disconnect。

**举例：**
```python
# ✅ 正确：启动时连接一次
connections.connect(...)

# 多次操作都复用这个连接
collection.insert(...)
collection.search(...)
collection.query(...)
```

**应用：** 提高性能，减少网络开销。

---

### 卡片7：健康检查 🏥

**一句话：** 定期检查连接状态，发现问题及时重连。

**举例：**
```python
def is_healthy():
    try:
        utility.list_collections()
        return True
    except:
        return False
```

**应用：** 生产环境保持服务稳定性。

---

### 卡片8：URI 格式 🔗

**一句话：** URI 格式更简洁，把地址信息写成一个字符串。

**举例：**
```python
# URI 格式
client = MilvusClient(uri="http://localhost:19530")

# 带认证的 URI
client = MilvusClient(uri="http://user:pass@localhost:19530")
```

**应用：** 配置简化，便于环境变量管理。

---

### 卡片9：错误处理 ⚠️

**一句话：** 使用 MilvusException 捕获连接错误，实现优雅降级。

**举例：**
```python
from pymilvus import MilvusException

try:
    connections.connect(...)
except MilvusException as e:
    print(f"连接失败: {e}")
    # 降级处理
```

**应用：** 提高应用健壮性。

---

### 卡片10：资源释放 🧹

**一句话：** 应用结束时必须关闭连接，释放资源。

**举例：**
```python
import atexit

# 注册退出时的清理函数
atexit.register(lambda: connections.disconnect("default"))

# 或使用 with 语句
with MilvusClient(...) as client:
    # 自动关闭
```

**应用：** 避免资源泄漏，保持系统稳定。

---

## 10. 【一句话总结】

**PyMilvus 是 Milvus 的 Python SDK，提供 MilvusClient（简化）和 connections（精细）两种连接方式，是 Python 应用与向量数据库交互的桥梁。**

---

## 📚 学习检查清单

- [ ] 能用 MilvusClient 连接 Milvus
- [ ] 能用 connections 模块连接 Milvus
- [ ] 理解连接别名的作用
- [ ] 知道生产环境需要配置哪些参数
- [ ] 理解连接复用的重要性
- [ ] 能正确处理连接错误

## 🔗 下一步学习

学完连接，下一步学习 **CRUD 基础操作**，掌握 Collection 的增删改查！

## 📖 参考资源

- [PyMilvus 官方文档](https://milvus.io/docs/install-pymilvus.md)
- [PyMilvus GitHub](https://github.com/milvus-io/pymilvus)
- [API Reference](https://milvus.io/api-reference/pymilvus/v2.3.x/About.md)
