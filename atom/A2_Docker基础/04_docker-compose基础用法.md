# docker-compose 基础用法

---

## 1. 【30字核心】

**docker-compose 是多容器编排工具，用 YAML 文件定义服务、网络和卷，一条命令启动整个应用栈。**

---

## 2. 【第一性原理】

### 什么是第一性原理？

**第一性原理**：回到事物最基本的真理，从源头思考问题

### docker-compose 的第一性原理 🎯

#### 1. 最基础的定义

**docker-compose = 多个 docker run 命令的声明式配置文件**

仅此而已！

- 把多个 `docker run` 命令写成一个 YAML 文件
- 用 `docker-compose up` 一键启动所有容器

#### 2. 为什么需要 docker-compose？

**核心问题：一个完整应用需要多个容器，怎么管理？**

```bash
# 没有 docker-compose：手动启动每个服务
docker run -d --name etcd ...
docker run -d --name minio ...
docker run -d --name milvus --link etcd --link minio ...

# 问题：
# 1. 命令太长，容易出错
# 2. 服务间依赖关系不清晰
# 3. 启动顺序需要手动控制
# 4. 无法一键启停
```

#### 3. docker-compose 的三层价值

##### 价值1：声明式配置
```yaml
# 所有配置一目了然
services:
  web:
    image: nginx
    ports:
      - "80:80"
```

##### 价值2：一键管理
```bash
docker-compose up -d   # 启动所有服务
docker-compose down    # 停止并清理
docker-compose logs    # 查看所有日志
```

##### 价值3：环境可复制
```bash
# 发给同事，同样环境一键启动
git clone project
cd project
docker-compose up -d
```

#### 4. 从第一性原理推导向量数据库部署

**推理链：**
```
1. 向量数据库（如 Milvus）需要多个组件
   ↓
2. etcd（元数据存储）+ MinIO（对象存储）+ Milvus（核心服务）
   ↓
3. 手动启动三个容器太复杂
   ↓
4. 用 docker-compose.yml 定义整个栈
   ↓
5. 一条命令启动完整的向量数据库环境
   ↓
6. 团队成员都能快速搭建一致的开发环境
```

#### 5. 一句话总结第一性原理

**docker-compose 将多容器应用的复杂部署简化为一个配置文件和一条命令，是团队协作和环境一致性的保障。**

---

## 3. 【3个核心概念】

### 核心概念1：services - 服务定义 🔧

**services 定义应用包含的所有容器，每个服务就是一个容器。**

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 服务1：Web 服务器
  web:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
  
  # 服务2：数据库
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

**常用配置项：**
| 配置项 | 作用 | 示例 |
|--------|------|------|
| `image` | 使用的镜像 | `nginx:latest` |
| `ports` | 端口映射 | `"8080:80"` |
| `volumes` | 数据卷挂载 | `./data:/app/data` |
| `environment` | 环境变量 | `DB_HOST: localhost` |
| `depends_on` | 依赖关系 | `[db, redis]` |
| `restart` | 重启策略 | `unless-stopped` |

**在向量数据库中的应用：**
```yaml
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant-data:/qdrant/storage
```

---

### 核心概念2：networks - 网络配置 🌐

**networks 定义服务间的网络通信，同一网络内的容器可以用服务名互相访问。**

```yaml
version: '3.8'

services:
  app:
    image: my-app
    networks:
      - backend
    depends_on:
      - db
  
  db:
    image: postgres
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

**网络通信示例：**
```python
# 在 app 容器内，可以直接用服务名访问 db
import psycopg2
conn = psycopg2.connect(
    host="db",  # 使用服务名作为主机名
    database="mydb",
    user="postgres"
)
```

**默认网络行为：**
- docker-compose 自动创建一个默认网络
- 同一 compose 文件中的服务自动加入默认网络
- 服务名自动成为 DNS 名称

**在向量数据库中的应用：**
```yaml
services:
  app:
    image: my-rag-app
    environment:
      - QDRANT_HOST=qdrant  # 用服务名访问
  
  qdrant:
    image: qdrant/qdrant
```

---

### 核心概念3：volumes - 数据卷管理 💾

**volumes 定义持久化存储，有命名卷和绑定挂载两种方式。**

```yaml
version: '3.8'

services:
  db:
    image: postgres
    volumes:
      # 命名卷（推荐用于持久化数据）
      - db-data:/var/lib/postgresql/data
      # 绑定挂载（适合配置文件、代码）
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

# 声明命名卷
volumes:
  db-data:
    driver: local
```

**两种挂载方式对比：**
```yaml
volumes:
  # 方式1：命名卷（Docker 管理）
  - mydata:/app/data
  
  # 方式2：绑定挂载（指定宿主机路径）
  - ./local/data:/app/data
  - /absolute/path:/app/config
```

**在向量数据库中的应用：**
```yaml
services:
  milvus:
    volumes:
      - milvus-data:/var/lib/milvus  # 向量数据
      - ./milvus.yaml:/milvus/configs/milvus.yaml  # 配置文件

volumes:
  milvus-data:
```

---

## 4. 【最小可用】

掌握以下内容，就能使用 docker-compose 管理多容器应用：

### 4.1 最小化配置模板

```yaml
# docker-compose.yml
version: '3.8'

services:
  服务名:
    image: 镜像名
    ports:
      - "宿主端口:容器端口"
    volumes:
      - ./本地目录:/容器目录
```

### 4.2 四个核心命令

```bash
# 启动所有服务（后台运行）
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f

# 停止并删除所有服务
docker-compose down
```

### 4.3 向量数据库实用模板

```yaml
# Qdrant 单机版
version: '3.8'

services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant-storage:/qdrant/storage
    restart: unless-stopped

volumes:
  qdrant-storage:
```

```yaml
# Milvus Standalone（完整版）
version: '3.8'

services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.5
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
    volumes:
      - etcd-data:/etcd

  minio:
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    volumes:
      - minio-data:/minio_data
    command: minio server /minio_data

  milvus:
    image: milvusdb/milvus:v2.3.0
    ports:
      - "19530:19530"
      - "9091:9091"
    depends_on:
      - etcd
      - minio
    volumes:
      - milvus-data:/var/lib/milvus
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000

volumes:
  etcd-data:
  minio-data:
  milvus-data:
```

### 4.4 常用命令速查

| 命令 | 作用 |
|------|------|
| `docker-compose up -d` | 后台启动所有服务 |
| `docker-compose down` | 停止并删除容器和网络 |
| `docker-compose down -v` | 同时删除数据卷 |
| `docker-compose ps` | 查看服务状态 |
| `docker-compose logs -f` | 实时查看日志 |
| `docker-compose restart` | 重启所有服务 |
| `docker-compose exec 服务名 命令` | 在容器内执行命令 |

**这些知识足以：**
- 编写基本的 docker-compose.yml 文件
- 一键启动向量数据库开发环境
- 管理多服务应用的生命周期

---

## 5. 【1个类比】

### 类比1：docker-compose.yml = package.json 📦

**相似性：** 都是项目的声明式配置文件

```json
// package.json - 定义 Node.js 项目
{
  "name": "my-app",
  "dependencies": {
    "express": "^4.18.0",
    "pg": "^8.11.0"
  },
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  }
}
```

```yaml
# docker-compose.yml - 定义容器化项目
version: '3.8'
services:
  app:
    image: node:18
    command: npm start
  db:
    image: postgres
```

| package.json | docker-compose.yml |
|--------------|-------------------|
| dependencies | services.*.image |
| scripts | services.*.command |
| npm install | docker-compose pull |
| npm start | docker-compose up |

---

### 类比2：services = React 组件 🧩

**相似性：** 都是可复用、可组合的独立单元

```jsx
// React：组件组合成应用
function App() {
  return (
    <>
      <Header />
      <MainContent />
      <Footer />
    </>
  );
}
```

```yaml
# docker-compose：服务组合成应用
services:
  frontend:
    image: nginx
  backend:
    image: node
  database:
    image: postgres
```

---

### 类比3：depends_on = import 依赖 📥

**相似性：** 都声明了模块间的依赖关系

```javascript
// JavaScript：模块依赖
import { db } from './database';  // 先加载 db
import { api } from './api';       // api 依赖 db

// 加载顺序：database → api
```

```yaml
# docker-compose：服务依赖
services:
  api:
    depends_on:
      - db  # api 依赖 db
  db:
    image: postgres

# 启动顺序：db → api
```

---

### 类比4：networks = 局域网 🌐

**相似性：** 同一网络内的设备可以互相访问

```javascript
// 前端：同一个局域网内
// 手机访问电脑的开发服务器
// http://192.168.1.100:3000

// 不同局域网需要公网 IP 或端口转发
```

```yaml
# docker-compose：同一网络内
services:
  app:
    networks:
      - mynet
  db:
    networks:
      - mynet

networks:
  mynet:

# app 可以直接访问 db:5432
```

---

### 类比5：docker-compose up = npm run dev 🚀

**相似性：** 一条命令启动整个开发环境

```bash
# 前端项目
cd my-react-app
npm install
npm run dev
# 启动开发服务器、热重载、代理等

# Docker 项目
cd my-docker-project
docker-compose up -d
# 启动所有服务、配置网络、挂载卷等
```

---

### 类比总结表

| 前端概念 | docker-compose 概念 | 作用 |
|----------|---------------------|------|
| package.json | docker-compose.yml | 项目配置文件 |
| React 组件 | services | 可组合的独立单元 |
| import/依赖 | depends_on | 声明依赖关系 |
| 局域网 | networks | 内部网络通信 |
| npm run dev | docker-compose up | 一键启动环境 |
| npm install | docker-compose pull | 获取依赖 |
| .env 文件 | environment | 环境变量配置 |

---

## 6. 【反直觉点】

### 误区1：depends_on 保证服务完全就绪 ❌

**为什么错？**
- `depends_on` 只保证容器**启动顺序**
- 不保证服务**完全可用**（如数据库初始化完成）

**为什么人们容易这样错？**
名字叫"依赖"，自然以为依赖的服务已经完全准备好。

**正确理解：**
```yaml
# 问题场景
services:
  app:
    depends_on:
      - db
    # app 启动时，db 容器已启动
    # 但 PostgreSQL 可能还在初始化！

  db:
    image: postgres
```

**正确做法：**
```yaml
# 方法1：使用 healthcheck
services:
  app:
    depends_on:
      db:
        condition: service_healthy
  
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      timeout: 5s
      retries: 5

# 方法2：应用层重试
# 在 app 代码中实现连接重试逻辑
```

---

### 误区2：docker-compose down 会删除数据 ❌

**为什么错？**
- `docker-compose down` 只删除容器和网络
- 不会删除数据卷
- 需要 `docker-compose down -v` 才删除卷

**为什么人们容易这样错？**
认为 "down" 是完全清理的意思。

**正确理解：**
```bash
# 只停止和删除容器、网络
docker-compose down
# 数据卷还在！

# 查看数据卷
docker volume ls
# 输出：projectname_db-data

# 完全清理（包括数据卷）
docker-compose down -v
# 数据卷被删除

# 完全清理（包括镜像）
docker-compose down -v --rmi all
```

**命令对比：**
| 命令 | 删除容器 | 删除网络 | 删除卷 | 删除镜像 |
|------|---------|---------|--------|---------|
| `down` | ✅ | ✅ | ❌ | ❌ |
| `down -v` | ✅ | ✅ | ✅ | ❌ |
| `down --rmi all` | ✅ | ✅ | ❌ | ✅ |
| `down -v --rmi all` | ✅ | ✅ | ✅ | ✅ |

---

### 误区3：每次修改都要 down 再 up ❌

**为什么错？**
- 只修改了配置，可以用 `docker-compose up -d` 重新创建
- 只改了代码（绑定挂载），根本不需要重启
- 频繁 down 会丢失容器内的临时状态

**为什么人们容易这样错？**
来自传统部署经验，觉得修改配置必须重启。

**正确理解：**
```bash
# 场景1：修改了 docker-compose.yml
docker-compose up -d
# Docker 会自动检测变化，只重建有变化的容器

# 场景2：修改了绑定挂载的代码
# 不需要任何操作！代码自动同步

# 场景3：需要完全重建
docker-compose up -d --build --force-recreate

# 场景4：只重启某个服务
docker-compose restart 服务名
```

---

## 7. 【实战代码】

```yaml
# docker-compose.yml
# 场景：搭建完整的 RAG 开发环境
# 包含：向量数据库 + 后端 API + 前端界面

version: '3.8'

services:
  # ===== 向量数据库 =====
  qdrant:
    image: qdrant/qdrant:latest
    container_name: rag-qdrant
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant-storage:/qdrant/storage
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/readyz"]
      interval: 10s
      timeout: 5s
      retries: 3

  # ===== 后端 API =====
  backend:
    image: python:3.11-slim
    container_name: rag-backend
    working_dir: /app
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
      - pip-cache:/root/.cache/pip
    environment:
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
    command: >
      bash -c "
        pip install fastapi uvicorn qdrant-client &&
        uvicorn main:app --host 0.0.0.0 --port 8000 --reload
      "
    depends_on:
      qdrant:
        condition: service_healthy
    restart: unless-stopped

  # ===== 前端界面 =====
  frontend:
    image: nginx:alpine
    container_name: rag-frontend
    ports:
      - "80:80"
    volumes:
      - ./frontend:/usr/share/nginx/html:ro
    depends_on:
      - backend
    restart: unless-stopped

volumes:
  qdrant-storage:
  pip-cache:

networks:
  default:
    name: rag-network
```

```bash
#!/bin/bash
# docker-compose 实战演示脚本

echo "===== docker-compose 基础实战 ====="

# ===== 1. 准备项目结构 =====
echo ""
echo "=== 1. 创建项目结构 ==="
mkdir -p compose-demo/backend
mkdir -p compose-demo/frontend

# 创建后端代码
cat > compose-demo/backend/main.py << 'EOF'
from fastapi import FastAPI
from qdrant_client import QdrantClient
import os

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "RAG Backend is running"}

@app.get("/health")
def health_check():
    try:
        client = QdrantClient(
            host=os.getenv("QDRANT_HOST", "localhost"),
            port=int(os.getenv("QDRANT_PORT", 6333))
        )
        collections = client.get_collections()
        return {
            "status": "healthy",
            "qdrant": "connected",
            "collections": len(collections.collections)
        }
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
EOF

# 创建前端页面
cat > compose-demo/frontend/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>RAG Demo</title></head>
<body>
    <h1>RAG System Frontend</h1>
    <p>Backend API: <a href="http://localhost:8000">http://localhost:8000</a></p>
    <p>Qdrant UI: <a href="http://localhost:6333/dashboard">http://localhost:6333/dashboard</a></p>
</body>
</html>
EOF

# 创建 docker-compose.yml
cat > compose-demo/docker-compose.yml << 'EOF'
version: '3.8'

services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: demo-qdrant
    ports:
      - "6333:6333"
    volumes:
      - qdrant-data:/qdrant/storage
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/readyz"]
      interval: 5s
      timeout: 3s
      retries: 5

  backend:
    image: python:3.11-slim
    container_name: demo-backend
    working_dir: /app
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
    environment:
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
    command: >
      bash -c "
        pip install -q fastapi uvicorn qdrant-client &&
        uvicorn main:app --host 0.0.0.0 --port 8000
      "
    depends_on:
      qdrant:
        condition: service_healthy

  frontend:
    image: nginx:alpine
    container_name: demo-frontend
    ports:
      - "80:80"
    volumes:
      - ./frontend:/usr/share/nginx/html:ro

volumes:
  qdrant-data:
EOF

echo "项目结构："
ls -la compose-demo/

# ===== 2. 启动服务 =====
echo ""
echo "=== 2. 启动所有服务 ==="
cd compose-demo
docker-compose up -d
echo "等待服务启动..."
sleep 15

# ===== 3. 查看服务状态 =====
echo ""
echo "=== 3. 查看服务状态 ==="
docker-compose ps

# ===== 4. 查看日志 =====
echo ""
echo "=== 4. 查看服务日志 ==="
docker-compose logs --tail 5

# ===== 5. 测试服务 =====
echo ""
echo "=== 5. 测试各服务 ==="

echo "测试前端 (localhost:80):"
curl -s http://localhost:80 | head -c 100
echo ""

echo "测试后端 (localhost:8000):"
curl -s http://localhost:8000 | python3 -m json.tool 2>/dev/null

echo ""
echo "测试后端健康检查:"
curl -s http://localhost:8000/health | python3 -m json.tool 2>/dev/null

echo ""
echo "测试 Qdrant (localhost:6333):"
curl -s http://localhost:6333/collections | python3 -m json.tool 2>/dev/null

# ===== 6. 在容器内执行命令 =====
echo ""
echo "=== 6. 在容器内执行命令 ==="
docker-compose exec qdrant ls /qdrant

# ===== 7. 查看网络 =====
echo ""
echo "=== 7. 查看 Docker 网络 ==="
docker network ls | grep compose

# ===== 8. 扩展服务 =====
echo ""
echo "=== 8. 服务伸缩（演示）==="
echo "docker-compose up -d --scale backend=3"
echo "(注意：需要移除 container_name 和使用不同端口)"

# ===== 9. 清理环境 =====
echo ""
echo "=== 9. 停止服务 ==="
docker-compose down
echo "服务已停止，数据卷保留"

echo ""
echo "查看数据卷："
docker volume ls | grep compose

echo ""
echo "完全清理（包括数据卷）："
echo "docker-compose down -v"

# 返回原目录
cd ..

echo ""
echo "===== 实战演示完成 ====="
echo ""
echo "项目目录: ./compose-demo"
echo "启动方式: cd compose-demo && docker-compose up -d"
```

**运行输出示例：**
```
===== docker-compose 基础实战 =====

=== 1. 创建项目结构 ===
项目结构：
-rw-r--r--  backend
-rw-r--r--  frontend
-rw-r--r--  docker-compose.yml

=== 2. 启动所有服务 ===
Creating network "compose-demo_default" with the default driver
Creating volume "compose-demo_qdrant-data" with default driver
Creating demo-qdrant ... done
Creating demo-backend ... done
Creating demo-frontend ... done

=== 3. 查看服务状态 ===
     Name           Command                  State           Ports
-------------------------------------------------------------------------
demo-backend    bash -c pip install ...      Up      0.0.0.0:8000->8000/tcp
demo-frontend   nginx -g daemon off;         Up      0.0.0.0:80->80/tcp
demo-qdrant     ./qdrant                     Up      0.0.0.0:6333->6333/tcp

=== 5. 测试各服务 ===
测试后端健康检查:
{
    "status": "healthy",
    "qdrant": "connected",
    "collections": 0
}

===== 实战演示完成 =====
```

---

## 8. 【面试必问】

### 问题："什么是 docker-compose？它解决了什么问题？在项目中如何使用？"

**普通回答（❌ 不出彩）：**
"docker-compose 是管理多个容器的工具，用 YAML 文件定义，一条命令启动。"

**出彩回答（✅ 推荐）：**

> **docker-compose 是多容器应用的编排工具，解决了三个核心问题：**
>
> 1. **复杂性问题**：将多个 `docker run` 命令简化为一个 YAML 配置文件
>    - 服务定义、端口、卷、环境变量一目了然
>    - 版本控制友好，配置变更可追踪
>
> 2. **依赖管理问题**：通过 `depends_on` 和 healthcheck 处理服务启动顺序
>    ```yaml
>    services:
>      app:
>        depends_on:
>          db:
>            condition: service_healthy
>    ```
>
> 3. **环境一致性问题**：团队成员共享同一份配置，一键启动相同环境
>
> **在我们的向量数据库项目中的应用：**
> ```yaml
> services:
>   qdrant:      # 向量数据库
>   backend:     # RAG 后端 API
>   frontend:    # 用户界面
> ```
> 
> 新成员入职，只需 `git clone && docker-compose up -d`，5分钟就能启动完整开发环境，不用花一天配置依赖。
>
> **注意事项：**
> - 生产环境通常用 Kubernetes，docker-compose 更适合开发和测试
> - `depends_on` 只保证启动顺序，不保证服务就绪
> - 使用 `docker-compose down` 不会删除数据卷，数据是安全的

**为什么这个回答出彩？**
1. ✅ 清晰说明解决的三个核心问题
2. ✅ 给出实际代码示例
3. ✅ 结合真实项目场景
4. ✅ 指出生产环境的注意事项

---

## 9. 【化骨绵掌】

### 卡片1：docker-compose 是什么 📋

**一句话：** 把多个 docker run 写成一个 YAML 文件。

**举例：**
```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"
```

**应用：** 一个文件定义整个向量数据库环境。

---

### 卡片2：基本文件结构 📄

**一句话：** version + services + volumes + networks。

**举例：**
```yaml
version: '3.8'
services:
  ...
volumes:
  ...
networks:
  ...
```

**应用：** 掌握这个结构就能写配置文件。

---

### 卡片3：启动命令 🚀

**一句话：** docker-compose up -d 一键启动所有服务。

**举例：**
```bash
docker-compose up -d     # 启动
docker-compose down      # 停止
docker-compose ps        # 查看状态
```

**应用：** 三个命令覆盖日常 90% 场景。

---

### 卡片4：服务间通信 🔗

**一句话：** 同一 compose 的服务可以用服务名互访。

**举例：**
```yaml
services:
  app:
    environment:
      - DB_HOST=db  # 直接用服务名
  db:
    image: postgres
```

**应用：** RAG 应用直接用 `qdrant` 作为主机名。

---

### 卡片5：depends_on 依赖 ⏳

**一句话：** 控制启动顺序，但不保证服务就绪。

**举例：**
```yaml
services:
  app:
    depends_on:
      - db  # db 先启动
```

**应用：** 确保向量数据库先于应用启动。

---

### 卡片6：数据卷持久化 💾

**一句话：** 在顶层声明的卷会被 Docker 管理。

**举例：**
```yaml
services:
  db:
    volumes:
      - db-data:/var/lib/data
volumes:
  db-data:  # 声明命名卷
```

**应用：** 向量索引数据持久保存。

---

### 卡片7：环境变量 🔧

**一句话：** environment 设置容器内的环境变量。

**举例：**
```yaml
services:
  app:
    environment:
      - API_KEY=xxx
      - DEBUG=true
```

**应用：** 配置 API 密钥、数据库连接等。

---

### 卡片8：healthcheck 健康检查 ❤️

**一句话：** 检测服务是否真正可用。

**举例：**
```yaml
services:
  db:
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 10s
```

**应用：** 确保数据库初始化完成后再启动应用。

---

### 卡片9：常用命令速查 📚

**一句话：** 记住这些命令，日常够用了。

**举例：**
```bash
up -d      # 后台启动
down       # 停止删除
down -v    # 同时删除卷
logs -f    # 实时日志
exec       # 进入容器
```

**应用：** 快速排查和管理服务。

---

### 卡片10：开发 vs 生产 ⚖️

**一句话：** docker-compose 适合开发，生产用 K8s。

**举例：**
```
开发环境：docker-compose up -d
生产环境：kubectl apply -f deployment.yaml
```

**应用：** 开发用 compose 快速迭代，生产用 K8s 保证可用性。

---

## 10. 【一句话总结】

**docker-compose 是多容器应用的编排工具，通过 YAML 文件声明式定义服务、网络和数据卷，使用 up/down/ps/logs 四个核心命令管理生命周期，是搭建向量数据库开发环境和微服务测试环境的最佳选择。**

---

## 📚 学习检查清单

- [ ] 能写出基本的 docker-compose.yml 文件结构
- [ ] 理解 services、volumes、networks 三大部分
- [ ] 掌握 up -d、down、ps、logs 四个核心命令
- [ ] 知道 depends_on 的作用和局限性
- [ ] 能用 docker-compose 启动向量数据库环境

## 🔗 下一步学习

- Dockerfile 编写（自定义镜像）
- Docker 多阶段构建
- Kubernetes 基础

## 📖 配置速查表

```yaml
version: '3.8'

services:
  service_name:
    image: image:tag                    # 镜像
    container_name: my-container        # 容器名
    ports:
      - "host:container"                # 端口映射
    volumes:
      - ./local:/container              # 绑定挂载
      - named-vol:/container            # 命名卷
    environment:
      - KEY=value                       # 环境变量
    depends_on:
      - other_service                   # 依赖
    restart: unless-stopped             # 重启策略
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  named-vol:

networks:
  custom-net:
    driver: bridge
```

---

**版本：** v1.0  
**最后更新：** 2025-12-05
