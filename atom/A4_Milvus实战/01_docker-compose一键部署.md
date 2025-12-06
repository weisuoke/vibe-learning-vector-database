# Docker-Compose 一键部署 Milvus

## 1. 【30字核心】

**Docker-Compose 是容器编排工具，通过一个 YAML 文件定义多容器应用，实现 Milvus 向量数据库的一键部署。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### Docker-Compose 部署的第一性原理 🎯

#### 1. 最基础的定义

**Docker-Compose = 多容器应用的声明式配置 + 一键启动**

仅此而已！没有更基础的了。

- **声明式配置**：用 YAML 文件描述"我想要什么"
- **一键启动**：一条命令 `docker-compose up -d` 搞定所有

#### 2. 为什么需要 Docker-Compose？

**核心问题：Milvus 不是单一服务，而是多组件系统**

Milvus 的架构包含：
- **etcd**：元数据存储（知道数据在哪）
- **MinIO**：对象存储（存实际的向量数据）
- **Milvus**：核心服务（处理向量检索）

如果手动启动：
```bash
# 需要分别启动3个容器，配置网络，设置依赖...
docker run etcd ...
docker run minio ...
docker run milvus ...
```

这太麻烦了！Docker-Compose 解决了这个问题。

#### 3. Docker-Compose 的三层价值

##### 价值1：简化部署
一个文件定义所有服务，一条命令全部启动。

##### 价值2：环境一致性
开发、测试、生产环境配置完全相同，避免"在我电脑上能跑"的问题。

##### 价值3：易于维护
升级版本只需改一行配置，重启即可。

#### 4. 从第一性原理推导部署流程

**推理链：**
```
1. Milvus 需要多个组件协同工作
   ↓
2. 手动管理多个容器太复杂
   ↓
3. 需要一种方式统一管理多容器
   ↓
4. Docker-Compose 提供声明式配置
   ↓
5. 用户只需一个 YAML 文件 + 一条命令
   ↓
6. 实现 Milvus 的一键部署
```

#### 5. 一句话总结第一性原理

**Docker-Compose 是"基础设施即代码"的实践，把复杂的多容器部署简化为一个配置文件。**

---

## 3. 【3个核心概念】

### 核心概念1：docker-compose.yml 配置文件 📄

**docker-compose.yml 是 Docker-Compose 的核心，用 YAML 格式定义所有服务、网络、卷。**

```yaml
# Milvus Standalone 最小配置
version: '3.5'

services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.5
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
    volumes:
      - etcd_data:/etcd

  minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    volumes:
      - minio_data:/minio_data
    command: minio server /minio_data

  standalone:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.3.3
    depends_on:
      - etcd
      - minio
    ports:
      - "19530:19530"  # Milvus 服务端口
      - "9091:9091"    # 健康检查端口

volumes:
  etcd_data:
  minio_data:
```

**关键字段解释：**
- `services`：定义要启动的容器
- `image`：使用的镜像
- `ports`：端口映射（主机:容器）
- `volumes`：数据持久化
- `depends_on`：启动依赖顺序

**在向量数据库中的应用：**
Milvus 官方提供了多种 docker-compose.yml 模板，适用于不同场景（单机版、集群版）。

---

### 核心概念2：服务依赖与启动顺序 🔗

**`depends_on` 定义服务启动顺序，确保 Milvus 在 etcd 和 MinIO 之后启动。**

```yaml
services:
  standalone:
    depends_on:
      - etcd   # 先启动 etcd
      - minio  # 再启动 minio
    # 最后启动 milvus standalone
```

**为什么顺序重要？**

```
启动顺序：etcd → minio → milvus

milvus 启动时会：
1. 连接 etcd 读取/写入元数据
2. 连接 minio 存储/读取向量数据

如果 etcd/minio 没启动，milvus 会报错！
```

**注意：** `depends_on` 只保证启动顺序，不保证服务"就绪"。生产环境需要额外的健康检查。

**在向量数据库中的应用：**
理解依赖关系有助于排查启动失败的问题——通常是某个依赖服务没准备好。

---

### 核心概念3：数据持久化（Volumes） 💾

**Volumes 是 Docker 的数据持久化机制，确保容器删除后数据不丢失。**

```yaml
volumes:
  etcd_data:      # 存储元数据（collection 定义、索引信息等）
  minio_data:     # 存储实际的向量数据和索引文件

services:
  etcd:
    volumes:
      - etcd_data:/etcd  # 挂载到容器内的 /etcd 目录
  
  minio:
    volumes:
      - minio_data:/minio_data
```

**为什么需要持久化？**

```
没有 volumes：
┌─────────────┐
│  Container  │  → 删除容器 → 数据全没了！
│  (数据在内部) │
└─────────────┘

有 volumes：
┌─────────────┐
│  Container  │  → 删除容器 → 数据还在！
└──────┬──────┘
       │ 挂载
┌──────┴──────┐
│   Volume    │  ← 数据持久保存
│ (宿主机磁盘) │
└─────────────┘
```

**在向量数据库中的应用：**
向量数据库存储的 embedding 是宝贵资产（生成成本高），必须持久化保存。

---

## 4. 【最小可用】

掌握以下内容，就能一键部署 Milvus：

### 4.1 安装 Docker 和 Docker-Compose

```bash
# macOS：安装 Docker Desktop（自带 Docker-Compose）
# https://www.docker.com/products/docker-desktop

# Linux：
sudo apt-get update
sudo apt-get install docker.io docker-compose

# 验证安装
docker --version
docker-compose --version
```

### 4.2 下载官方配置文件

```bash
# 下载 Milvus Standalone 配置
wget https://github.com/milvus-io/milvus/releases/download/v2.3.3/milvus-standalone-docker-compose.yml -O docker-compose.yml

# 或者直接创建（见上面的配置示例）
```

### 4.3 一键启动

```bash
# 启动所有服务（后台运行）
docker-compose up -d

# 查看运行状态
docker-compose ps

# 查看日志
docker-compose logs -f milvus-standalone
```

### 4.4 停止和清理

```bash
# 停止服务（保留数据）
docker-compose down

# 停止服务并删除数据（谨慎！）
docker-compose down -v
```

**这些知识足以：**
- 在本地快速搭建 Milvus 开发环境
- 理解 Milvus 的基本架构组件
- 排查常见的部署问题
- 为后续学习 pymilvus 连接做准备

---

## 5. 【1个类比】

### 类比1：Docker-Compose = package.json + npm install 📦

**前端项目用 `package.json` 定义依赖，Docker-Compose 用 `docker-compose.yml` 定义服务。**

```javascript
// package.json - 定义前端项目依赖
{
  "name": "my-app",
  "dependencies": {
    "react": "^18.0.0",
    "axios": "^1.0.0",
    "lodash": "^4.17.0"
  }
}

// 一条命令安装所有依赖
// npm install
```

```yaml
# docker-compose.yml - 定义服务依赖
version: '3.5'
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.5
  minio:
    image: minio/minio:latest
  milvus:
    image: milvusdb/milvus:v2.3.3

# 一条命令启动所有服务
# docker-compose up -d
```

---

### 类比2：services = React 组件树 🌳

**Docker-Compose 的服务依赖就像 React 组件的父子关系。**

```jsx
// React 组件树：父组件先渲染，子组件后渲染
<App>                    {/* 最先渲染 */}
  <DatabaseProvider>     {/* 依赖 App */}
    <StorageProvider>    {/* 依赖 DatabaseProvider */}
      <MainContent />    {/* 依赖所有上层 */}
    </StorageProvider>
  </DatabaseProvider>
</App>
```

```yaml
# Docker-Compose 服务依赖
services:
  etcd:           # 最先启动（元数据存储）
    ...
  
  minio:          # 第二启动（对象存储）
    ...
  
  standalone:     # 最后启动（主服务）
    depends_on:
      - etcd      # 依赖 etcd
      - minio     # 依赖 minio
```

---

### 类比3：Volumes = localStorage 持久化 💾

**Docker Volumes 持久化数据，就像浏览器 localStorage 保存用户数据。**

```javascript
// 前端：localStorage 持久化
// 刷新页面后数据还在
localStorage.setItem('user_preferences', JSON.stringify({
  theme: 'dark',
  language: 'zh-CN'
}));

// 页面刷新后
const prefs = JSON.parse(localStorage.getItem('user_preferences'));
// 数据还在！
```

```yaml
# Docker：volumes 持久化
# 删除容器后数据还在
volumes:
  milvus_data:

services:
  milvus:
    volumes:
      - milvus_data:/var/lib/milvus  # 数据持久化

# docker-compose down && docker-compose up -d
# 数据还在！
```

---

### 类比4：端口映射 = 开发服务器端口 🚪

**Docker 的端口映射就像前端开发服务器的端口配置。**

```javascript
// Vite 配置开发服务器端口
// vite.config.js
export default {
  server: {
    port: 3000,  // 本地访问：http://localhost:3000
    host: true   // 允许外部访问
  }
}
```

```yaml
# Docker-Compose 端口映射
services:
  standalone:
    ports:
      - "19530:19530"  # 本地访问：localhost:19530
      # 格式：宿主机端口:容器端口
```

---

### 类比总结表

| Docker-Compose 概念 | 前端对应概念 | 作用 |
|-------------------|------------|------|
| docker-compose.yml | package.json | 声明式配置文件 |
| docker-compose up | npm install + npm start | 一键安装启动 |
| services | 组件树 | 定义依赖关系 |
| volumes | localStorage | 数据持久化 |
| ports | dev server port | 端口暴露 |
| depends_on | 组件挂载顺序 | 启动顺序控制 |

---

## 6. 【反直觉点】

### 误区1：Docker-Compose 适合生产环境 ❌

**为什么错？**
- Docker-Compose 适合**开发和测试环境**
- 生产环境需要 **Kubernetes** 或 **Milvus Operator**
- 原因：缺乏高可用、自动扩展、滚动更新等能力

**为什么人们容易这样错？**
因为 Docker-Compose 太方便了，一条命令就能跑，让人产生"这就够了"的错觉。

**正确理解：**
```
开发/测试环境：Docker-Compose ✅
├── 快速启动
├── 易于调试
└── 配置简单

生产环境：Kubernetes + Helm ✅
├── 高可用（多副本）
├── 自动扩展
├── 滚动更新
└── 资源隔离
```

---

### 误区2：depends_on 保证服务就绪 ❌

**为什么错？**
- `depends_on` 只保证**启动顺序**
- 不保证依赖服务**完全就绪**
- etcd 容器启动 ≠ etcd 服务可用

**为什么人们容易这样错？**
"依赖"这个词让人以为是"等依赖准备好了再启动"，但实际上只是"先启动依赖容器"。

**正确理解：**
```yaml
# 基础版：只保证启动顺序
services:
  standalone:
    depends_on:
      - etcd

# 生产版：加入健康检查
services:
  standalone:
    depends_on:
      etcd:
        condition: service_healthy
    
  etcd:
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 20s
      retries: 3
```

---

### 误区3：docker-compose down 会删除数据 ❌

**为什么错？**
- `docker-compose down`：只删除容器和网络，**保留 volumes**
- `docker-compose down -v`：删除容器、网络和 **volumes（数据丢失！）**

**为什么人们容易这样错？**
习惯性加 `-v` 参数"清理干净"，却忘了 volumes 里有重要数据。

**正确理解：**
```bash
# 停止服务，保留数据（推荐）
docker-compose down

# 完全清理，包括数据（谨慎！）
docker-compose down -v

# 查看 volumes
docker volume ls

# 单独删除某个 volume
docker volume rm milvus_etcd_data
```

---

## 7. 【实战代码】

```bash
#!/bin/bash
# ===== Milvus Docker-Compose 一键部署实战 =====

# ===== 1. 环境检查 =====
echo "=== 检查 Docker 环境 ==="

# 检查 Docker 是否安装
if ! command -v docker &> /dev/null; then
    echo "❌ Docker 未安装，请先安装 Docker"
    exit 1
fi

# 检查 Docker-Compose 是否安装
if ! command -v docker-compose &> /dev/null; then
    echo "❌ Docker-Compose 未安装，请先安装"
    exit 1
fi

echo "✅ Docker 版本: $(docker --version)"
echo "✅ Docker-Compose 版本: $(docker-compose --version)"

# ===== 2. 创建配置文件 =====
echo ""
echo "=== 创建 docker-compose.yml ==="

cat > docker-compose.yml << 'EOF'
version: '3.5'

services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.5
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ETCD_SNAPSHOT_COUNT=50000
    volumes:
      - etcd_data:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    ports:
      - "9001:9001"   # MinIO Console
      - "9000:9000"   # MinIO API
    volumes:
      - minio_data:/minio_data
    command: minio server /minio_data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  standalone:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.3.3
    command: ["milvus", "run", "standalone"]
    security_opt:
      - seccomp:unconfined
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
    volumes:
      - milvus_data:/var/lib/milvus
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
      interval: 30s
      start_period: 90s
      timeout: 20s
      retries: 3
    ports:
      - "19530:19530"  # Milvus gRPC
      - "9091:9091"    # Milvus Health
    depends_on:
      - etcd
      - minio

volumes:
  etcd_data:
  minio_data:
  milvus_data:
EOF

echo "✅ docker-compose.yml 已创建"

# ===== 3. 启动服务 =====
echo ""
echo "=== 启动 Milvus 服务 ==="
docker-compose up -d

# ===== 4. 等待服务就绪 =====
echo ""
echo "=== 等待服务就绪（约60秒）==="
sleep 10

# 检查服务状态
for i in {1..6}; do
    echo "检查中... ($i/6)"
    
    # 检查 Milvus 健康状态
    if curl -s http://localhost:9091/healthz > /dev/null 2>&1; then
        echo ""
        echo "✅ Milvus 服务已就绪！"
        break
    fi
    
    if [ $i -eq 6 ]; then
        echo "⚠️ 服务启动较慢，请稍后检查"
    fi
    
    sleep 10
done

# ===== 5. 显示服务状态 =====
echo ""
echo "=== 服务状态 ==="
docker-compose ps

# ===== 6. 显示连接信息 =====
echo ""
echo "=== 连接信息 ==="
echo "Milvus gRPC:    localhost:19530"
echo "Milvus Health:  http://localhost:9091/healthz"
echo "MinIO Console:  http://localhost:9001 (minioadmin/minioadmin)"

# ===== 7. 验证连接（使用 Python） =====
echo ""
echo "=== 验证连接（Python 示例）==="
cat << 'PYEOF'
# 安装 pymilvus
# pip install pymilvus

from pymilvus import connections, utility

# 连接 Milvus
connections.connect(host="localhost", port="19530")

# 检查连接
print(f"Milvus 版本: {utility.get_server_version()}")
print("✅ 连接成功！")
PYEOF

echo ""
echo "=== 常用命令 ==="
echo "查看日志:     docker-compose logs -f"
echo "停止服务:     docker-compose down"
echo "重启服务:     docker-compose restart"
echo "删除数据:     docker-compose down -v  (谨慎!)"
```

**运行输出示例：**
```
=== 检查 Docker 环境 ===
✅ Docker 版本: Docker version 24.0.6
✅ Docker-Compose 版本: Docker Compose version v2.22.0

=== 创建 docker-compose.yml ===
✅ docker-compose.yml 已创建

=== 启动 Milvus 服务 ===
[+] Running 4/4
 ✔ Network milvus_default    Created
 ✔ Container milvus-etcd     Started
 ✔ Container milvus-minio    Started
 ✔ Container milvus-standalone Started

=== 等待服务就绪（约60秒）===
检查中... (1/6)
检查中... (2/6)

✅ Milvus 服务已就绪！

=== 服务状态 ===
NAME                 STATUS
milvus-etcd          running (healthy)
milvus-minio         running (healthy)
milvus-standalone    running (healthy)

=== 连接信息 ===
Milvus gRPC:    localhost:19530
Milvus Health:  http://localhost:9091/healthz
MinIO Console:  http://localhost:9001 (minioadmin/minioadmin)
```

---

## 8. 【面试必问】

### 问题："如何快速部署一个 Milvus 向量数据库？"

**普通回答（❌ 不出彩）：**
"用 Docker-Compose，下载官方配置文件，然后 `docker-compose up -d` 就行了。"

**出彩回答（✅ 推荐）：**

> **Milvus 部署有三种方式，根据场景选择：**
>
> 1. **开发环境：Docker-Compose**
>    - 一个 YAML 文件定义 etcd、MinIO、Milvus 三个组件
>    - 一条命令 `docker-compose up -d` 启动
>    - 适合本地开发和功能验证
>
> 2. **测试环境：Milvus Lite**
>    - 纯 Python 包，`pip install milvus` 即可
>    - 无需任何外部依赖
>    - 适合 CI/CD 和单元测试
>
> 3. **生产环境：Kubernetes + Helm**
>    - 支持高可用、自动扩展
>    - 使用 Milvus Operator 管理
>    - 适合大规模线上服务
>
> **关于 Docker-Compose 部署的关键点：**
> - Milvus 依赖 etcd（元数据）和 MinIO（对象存储）
> - `depends_on` 只保证启动顺序，生产环境需要健康检查
> - 必须配置 volumes 做数据持久化
>
> **在实际项目中**，我们用 Docker-Compose 做本地开发，用 K8s 部署生产环境，通过环境变量区分配置。

**为什么这个回答出彩？**
1. ✅ 展示了全面的技术视野（三种部署方式）
2. ✅ 理解 Milvus 架构（三个组件的作用）
3. ✅ 知道 Docker-Compose 的局限性（不适合生产）
4. ✅ 有实际项目经验（开发 vs 生产环境）

---

## 9. 【化骨绵掌】

### 卡片1：什么是 Docker-Compose？ 🐳

**一句话：** Docker-Compose 是管理多个 Docker 容器的工具，用一个 YAML 文件定义所有服务。

**举例：**
```yaml
# docker-compose.yml
services:
  web:
    image: nginx
  db:
    image: mysql
```

**应用：** Milvus 由多个组件组成，Docker-Compose 可以一键启动所有组件。

---

### 卡片2：Milvus 的三个组件 🧩

**一句话：** Milvus Standalone 需要 etcd（元数据）、MinIO（存储）、Milvus（核心服务）三个组件。

**举例：**
```
etcd   → 存储 Collection 定义、索引信息
MinIO  → 存储实际的向量数据文件
Milvus → 处理查询请求，执行向量检索
```

**应用：** 理解组件作用有助于排查问题——查询慢可能是 Milvus，数据丢失可能是 MinIO。

---

### 卡片3：docker-compose.yml 核心结构 📄

**一句话：** YAML 文件有三个核心部分：version（版本）、services（服务）、volumes（存储）。

**举例：**
```yaml
version: '3.5'        # Compose 文件版本
services:             # 定义容器服务
  milvus:
    image: milvusdb/milvus:v2.3.3
volumes:              # 定义持久化存储
  milvus_data:
```

**应用：** 官方提供的配置文件可以直接使用，也可以根据需要修改。

---

### 卡片4：端口映射 🚪

**一句话：** 端口映射让宿主机可以访问容器内的服务，格式是 `宿主机端口:容器端口`。

**举例：**
```yaml
ports:
  - "19530:19530"  # Milvus gRPC 端口
  - "9091:9091"    # 健康检查端口
```

**应用：** Python 客户端通过 `localhost:19530` 连接 Milvus。

---

### 卡片5：服务依赖 depends_on 🔗

**一句话：** `depends_on` 控制容器启动顺序，被依赖的服务先启动。

**举例：**
```yaml
standalone:
  depends_on:
    - etcd    # 先启动
    - minio   # 再启动
  # standalone 最后启动
```

**应用：** 确保 Milvus 启动时 etcd 和 MinIO 已经运行。

---

### 卡片6：数据持久化 Volumes 💾

**一句话：** Volumes 把容器数据保存到宿主机，删除容器后数据不丢失。

**举例：**
```yaml
volumes:
  milvus_data:        # 定义 volume

services:
  milvus:
    volumes:
      - milvus_data:/var/lib/milvus  # 挂载
```

**应用：** 向量数据是宝贵资产，必须持久化保存。

---

### 卡片7：常用命令 ⌨️

**一句话：** 四个核心命令：up（启动）、down（停止）、ps（状态）、logs（日志）。

**举例：**
```bash
docker-compose up -d      # 后台启动
docker-compose down       # 停止（保留数据）
docker-compose ps         # 查看状态
docker-compose logs -f    # 实时日志
```

**应用：** 开发时经常需要重启服务、查看日志排查问题。

---

### 卡片8：健康检查 healthcheck 🏥

**一句话：** 健康检查定期探测服务是否正常，用于判断服务真正就绪。

**举例：**
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
  interval: 30s       # 每30秒检查一次
  timeout: 20s        # 超时时间
  retries: 3          # 失败重试次数
```

**应用：** 配合 `depends_on: condition: service_healthy` 确保依赖服务真正可用。

---

### 卡片9：开发 vs 生产环境 🏭

**一句话：** Docker-Compose 适合开发测试，生产环境用 Kubernetes。

**举例：**
```
开发环境：
  Docker-Compose → 单机、快速、简单

生产环境：
  Kubernetes → 高可用、自动扩展、滚动更新
```

**应用：** 选择合适的部署方式，避免在生产环境踩坑。

---

### 卡片10：快速启动流程 🚀

**一句话：** 三步完成部署：下载配置 → 启动服务 → 验证连接。

**举例：**
```bash
# 1. 下载配置
wget https://github.com/milvus-io/milvus/releases/download/v2.3.3/milvus-standalone-docker-compose.yml -O docker-compose.yml

# 2. 启动服务
docker-compose up -d

# 3. 验证连接
curl http://localhost:9091/healthz
```

**应用：** 新项目快速搭建 Milvus 开发环境，5分钟内完成。

---

## 10. 【一句话总结】

**Docker-Compose 是容器编排工具，通过声明式 YAML 配置文件实现 Milvus 多组件（etcd、MinIO、Milvus）的一键部署，是开发测试环境的最佳选择。**

---

## 📚 学习检查清单

- [ ] 理解 Docker-Compose 的作用
- [ ] 知道 Milvus 的三个核心组件
- [ ] 能读懂 docker-compose.yml 配置
- [ ] 掌握 up/down/ps/logs 四个命令
- [ ] 理解端口映射和数据持久化
- [ ] 知道 Docker-Compose 的适用场景

## 🔗 下一步学习

学完部署，下一步学习 **pymilvus 客户端连接**，开始用 Python 操作 Milvus！

## 📖 参考资源

- [Milvus 官方安装文档](https://milvus.io/docs/install_standalone-docker.md)
- [Docker-Compose 官方文档](https://docs.docker.com/compose/)
- [Milvus GitHub](https://github.com/milvus-io/milvus)
