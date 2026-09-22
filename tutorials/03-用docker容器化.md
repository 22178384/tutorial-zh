# 03 · 用 Docker 容器化一个应用

1. 写 `Dockerfile`：`FROM python:3.12-slim` → `COPY` 代码 → `CMD`。
2. 构建：`docker build -t myapp .`。
3. 运行：`docker run -p 8080:80 myapp`。
4. 多服务用 `docker compose`：在 `docker-compose.yml` 里定义 services。
5. 数据持久化用 volume，别把状态写进容器层。
