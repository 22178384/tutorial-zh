# 03 用 docker 容器化

我第一次听 docker 的时候完全没懂"容器"是个啥。后来想通了：**它就是一个轻量的
虚拟机，但只打包你的程序运行需要的那点东西**。好处是你本地跑通的，服务器上
大概率也能跑通，不用再对着"你机器上是 Python 3.9 我这是 3.11"扯皮。

## 0. 装 docker

macOS：装 Docker Desktop，官网下载 dmg，双击装。
Ubuntu：

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER    # 把当前用户加进 docker 组
# 然后注销重新登录，不然还是要 sudo
```

验证：

```bash
docker run --rm hello-world
```

**坑**：`usermod` 之后不重新登录，`docker` 命令还是会报
`permission denied while trying to connect to the Docker daemon socket`。
不是没装好，是权限没生效。

## 1. 三个概念

- **镜像（image）**：一个只读的模板。可以理解成"装了 Python 3.11 和一堆依赖的
  一个快照"。
- **容器（container）**：镜像跑起来的一个实例。一个镜像可以跑出十个容器。
- **Dockerfile**：描述怎么从基础镜像构建出你的镜像。

```bash
docker images            # 本地有哪些镜像
docker ps                # 正在跑的容器
docker ps -a             # 包括已经停掉的（排查问题时看这个）
docker logs 容器名        # 看日志
docker exec -it 容器名 bash   # 进容器里看看
```

`docker ps -a` 里状态是 `Exited (1)` 就说明启动就崩了，去 `docker logs` 看原因。

## 2. 写一个 Dockerfile

一个 Python 项目的例子：

```dockerfile
FROM python:3.11-slim

# 不在容器里生成 .pyc，也不缓冲 stdout（这样日志能实时打出来）
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# 先只拷贝依赖文件，再装依赖。
# 这样改业务代码时，依赖这层缓存还能用，不用每次重装。
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# 别用 root 跑
RUN useradd -m appuser
USER appuser

CMD ["python", "-m", "src.main"]
```

**两个我踩过的坑：**

**坑一：日志不输出。** Python 默认缓冲 stdout，容器里日志会卡住半天不出来。
`PYTHONUNBUFFERED=1` 解决。

**坑二：每次改代码都重装依赖。** 如果一开始就 `COPY . .` 再 `pip install`，
那任何一行代码改动都会让依赖层缓存失效，构建要等好几分钟。先 COPY 依赖文件
单独装，是利用了 Docker 的分层缓存。

构建：

```bash
docker build -t my-app:0.1 .
docker run --rm my-app:0.1
```

**注意最后的 `.`**，它表示构建上下文是当前目录。忘了写会报
`ERROR: "docker buildx build" requires exactly 1 argument.`

## 3. 数据怎么持久化

容器删了里面的数据就没了。要持久化，用 volume：

```bash
docker run -d \
  -v /host/data:/app/data \
  -p 8000:8000 \
  --name my-app \
  my-app:0.1
```

- `-v 宿主机路径:容器路径` 把宿主机目录挂进容器
- `-p 宿主机端口:容器端口` 端口映射，注意方向：**左边是外面的，右边是容器里的**

**坑**：`-p 8000:8000` 写成 `-p 8000` 的话，端口是随机的，你以为映射好了其实
连不上。永远写完整。

## 4. docker compose

跑一个容器还好，要是数据库、缓存、你的应用要一起跑，手敲 `docker run` 会疯。
用 compose：

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

注意 `DATABASE_URL` 里的主机名是 `db`，**不是 localhost**。compose 会自动建一个
网络，服务之间用服务名互相访问。这是新手最容易搞错的一点，连 localhost 永远连不上。

```bash
docker compose up -d        # 后台起
docker compose logs -f app  # 跟日志
docker compose down         # 停掉（数据还在，volume 保留）
docker compose down -v      # 连 volume 一起删（数据没了！）
```

**`down -v` 会删数据**，别顺手敲。

## 5. 镜像瘦身

`python:3.11` 大概 1GB，`python:3.11-slim` 大概 150MB，`python:3.11-alpine`
更小但用 musl libc，有些 C 扩展装不上。**我一般用 slim**，别为了那点体积去踩
alpine 的坑。

多阶段构建能再小一点，但除非你在意镜像大小，否则没必要，构建时间反而变长。

## 6. 常见报错对照

| 报错 | 原因 |
|------|------|
| `permission denied ... docker.sock` | 没加 docker 组，或加完没重新登录 |
| `port is already allocated` | 端口被占了，`lsof -i :8000` 看是谁 |
| `no space left on device` | 镜像太多，`docker system prune -a` 清一下 |
| 容器起来就 `Exited (1)` | 看 `docker logs`，通常是启动命令或环境变量错 |
| `exec format error` | 镜像架构不对，比如在 M1 Mac 上跑 amd64 镜像 |

最后一条现在很常见。Apple Silicon 上构建的镜像在 x86 服务器上跑不了。构建时指定：

```bash
docker build --platform linux/amd64 -t my-app:0.1 .
```

下一篇：[用 Python 写脚本](04-用python写脚本.md)。
