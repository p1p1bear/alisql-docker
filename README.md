# AliSQL Docker Images

AliSQL 是阿里巴巴基于 MySQL 的分支版本。本仓库提供 AliSQL 的 Docker 镜像构建配置。

- AliSQL 源码:<https://github.com/alibaba/AliSQL>
- 镜像仓库:<https://hub.docker.com/r/songhuaxiong/alisql>

## Supported tags

- `8.0.44-1`, `8.0.44`, `8.0`, `latest` — 对齐社区 MySQL 8.0.44

支持架构:`linux/amd64`, `linux/arm64`

## Quick start

```bash
docker run -d --name alisql \
  -e MYSQL_ROOT_PASSWORD=your-password \
  -p 3306:3306 \
  songhuaxiong/alisql:8.0
```

## 环境变量

跟社区 MySQL 镜像完全一致:

- `MYSQL_ROOT_PASSWORD`
- `MYSQL_DATABASE`
- `MYSQL_USER` / `MYSQL_PASSWORD`
- `MYSQL_ALLOW_EMPTY_PASSWORD`
- `MYSQL_RANDOM_ROOT_PASSWORD`
- 以及 `*_FILE` 形式(用于 Docker secrets)

## 数据持久化

```bash
docker run -d -v alisql_data:/var/lib/mysql songhuaxiong/alisql:8.0
```

## 自定义配置

挂 `.cnf` 文件到 `/etc/mysql/conf.d/`:

```bash
docker run -d -v $(pwd)/my.cnf:/etc/mysql/conf.d/custom.cnf songhuaxiong/alisql:8.0
```

## 初始化 SQL

`*.sql` / `*.sh` 放到 `/docker-entrypoint-initdb.d/`,首次启动时执行。

## License

- AliSQL: GPLv2 (<https://github.com/alibaba/AliSQL>)
- 本仓库的 entrypoint 基于 docker-library/mysql (MIT)
