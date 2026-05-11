# AliSQL Docker 镜像构建指南

> 目标:把 AliSQL 8.0 / 5.7 编译产物打成 Docker 镜像,发布到 Docker Hub `alibaba/alisql`。
> 用户拉取方式:`docker pull alibaba/alisql:8.0` 或 `docker pull alibaba/alisql:5.7`

---

## 你的仓库的关键特点(贯穿全文)

| 项 | 值 |
|---|---|
| Git tag 格式 | `AliSQL-8.0.44-1`(社区版本号 + AliSQL 修订号) |
| Release 产物名(amd64) | `alisql-8.0.44-1-linux-glibc2.17-x86_64.tar.xz` |
| Release 产物名(arm64) | `alisql-8.0.44-1-linux-glibc2.17-aarch64.tar.xz` |
| 压缩格式 | `.tar.xz`(不是 .tar.gz) |
| 架构命名 | `x86_64` / `aarch64`(注意:Docker buildx 用的是 `amd64` / `arm64`,**两套要做映射**) |
| 解压目标 | `/opt/alisql/{bin,lib,share,...}` |
| 要支持的分支 | 8.0 + 5.7 |

---

## 0. 在 Mac 上能做吗?

**能。** Docker Desktop for Mac 自带 `buildx` 和 QEMU emulation,可以直接构建 amd64 + arm64 多架构镜像。

但要注意:

| 场景 | 说明 |
|---|---|
| M 系列 Mac(arm64)构建 arm64 镜像 | native,快 |
| M 系列 Mac 构建 amd64 镜像 | QEMU emulation,慢 3~5 倍,但能跑通 |
| Intel Mac(amd64)反过来 | 同上 |
| 本地 smoke test | Mac 完全够用 |
| 正式多架构 push | **建议交给 GitHub Actions**(Ubuntu Runner 原生跑 amd64,arm64 用 emulation 或专用 runner,稳定且可重复) |

**结论**:Mac 用来开发 Dockerfile + 本地验证;CI 用来正式发布。

---

## 1. 准备工作

### 1.1 本地工具
```bash
# Mac
brew install --cask docker
# 或 Linux
curl -fsSL https://get.docker.com | sh

# 验证
docker --version
docker buildx version
```

### 1.2 Docker Hub 账号 + 仓库
1. 注册账号(<https://hub.docker.com>),假设组织名 `alibaba`
2. 在 hub 网页上手动建一个 public repository:`alibaba/alisql`
3. **Account Settings → Personal Access Tokens** 生成一个 token,权限 `Read, Write, Delete`,记下来留给 CI 用

### 1.3 GitHub 仓库
新建 `alibaba/alisql-docker`(独立仓库,**不要混在 AliSQL 主仓库里**)。

---

## 2. 仓库结构

```
alisql-docker/
├── README.md                    # 给 GitHub + Docker Hub 共用
├── .gitignore                   # 忽略 *.tar.xz
├── .github/workflows/build.yml
├── 8.0/
│   ├── Dockerfile
│   └── docker-entrypoint.sh
└── 5.7/
    ├── Dockerfile
    └── docker-entrypoint.sh
```

`.gitignore`:
```
*.tar.xz
*.tar.gz
```
(tarball 不入 git,CI 里下载或本地手动放)

---

## 3. Dockerfile(8.0)

`8.0/Dockerfile`:

```dockerfile
FROM oraclelinux:8-slim

RUN groupadd --system --gid 999 mysql \
 && useradd --system --uid 999 --gid 999 --home-dir /var/lib/mysql --no-create-home mysql

# gosu:用 root 启动后 drop 到 mysql 用户跑 mysqld
ENV GOSU_VERSION=1.17
RUN set -eux; \
    arch="$(uname -m)"; \
    case "$arch" in \
      aarch64) gosuArch=arm64 ;; \
      x86_64)  gosuArch=amd64 ;; \
      *) echo "unsupported $arch"; exit 1 ;; \
    esac; \
    curl -fL -o /usr/local/bin/gosu \
      "https://github.com/tianon/gosu/releases/download/${GOSU_VERSION}/gosu-${gosuArch}"; \
    chmod +x /usr/local/bin/gosu; gosu nobody true

# xz 用来解 .tar.xz;libaio/numactl/ncurses 是 mysqld 的运行依赖
RUN microdnf install -y bzip2 gzip openssl xz zstd findutils \
      libaio numactl-libs ncurses-compat-libs \
 && microdnf clean all

# AliSQL 完整版本号(含 -N 修订号)
ENV ALISQL_VERSION=8.0.44-1
ENV MYSQL_MAJOR=8.0

# Docker buildx 注入 TARGETARCH = amd64 | arm64
# 但 AliSQL tarball 用 x86_64 / aarch64 命名 —— CI 下载时就重命名好,
# 让 COPY 这一行可以直接拼出文件名。
ARG TARGETARCH
COPY alisql-${ALISQL_VERSION}-${TARGETARCH}.tar.xz /tmp/alisql.tar.xz

RUN set -eux; \
    tar -xJf /tmp/alisql.tar.xz -C /opt; \
    # tarball 顶层目录可能带版本号(如 alisql-8.0.44-1-linux-...),
    # 统一重命名为 /opt/alisql,方便配置文件写死路径
    if [ ! -d /opt/alisql ]; then \
      mv /opt/alisql-* /opt/alisql; \
    fi; \
    test -x /opt/alisql/bin/mysqld; \
    rm /tmp/alisql.tar.xz

ENV PATH=/opt/alisql/bin:$PATH
RUN mysqld --version

RUN mkdir -p /etc/mysql/conf.d /var/lib/mysql /var/run/mysqld /docker-entrypoint-initdb.d \
 && chown -R mysql:mysql /var/lib/mysql /var/run/mysqld \
 && chmod 1777 /var/lib/mysql /var/run/mysqld \
 && printf '%s\n' \
      '[mysqld]' \
      'basedir=/opt/alisql' \
      'datadir=/var/lib/mysql' \
      'socket=/var/run/mysqld/mysqld.sock' \
      'lc-messages-dir=/opt/alisql/share' \
      '' \
      '[client]' \
      'socket=/var/run/mysqld/mysqld.sock' \
      '' \
      '!includedir /etc/mysql/conf.d/' \
      > /etc/my.cnf

VOLUME /var/lib/mysql
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

EXPOSE 3306 33060
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["mysqld"]
```

### Dockerfile 关键决策说明

| 决策 | 为什么 |
|---|---|
| `FROM oraclelinux:8-slim` | 跟 MySQL 官方 8.x 镜像同源,`microdnf` 装包快;如果 AliSQL 编译用了更新的 glibc,可能要换成 `oraclelinux:9-slim` |
| `ARG TARGETARCH` + 文件名拼接 | buildx 自动注入,实现一份 Dockerfile 同时构建 amd64/arm64 |
| **CI 下载时重命名** tarball | 把 `x86_64`→`amd64`、`aarch64`→`arm64`,跟 TARGETARCH 对齐 |
| `tar -xJf` | `.tar.xz` 必须用 `-J`(`-z` 是 gzip,`-j` 是 bzip2) |
| `mv /opt/alisql-* /opt/alisql` | tarball 顶层目录如果带版本号,统一改名,后面配置就能写死 `/opt/alisql` |
| `lc-messages-dir=/opt/alisql/share` | **不配会启动失败**,报 `Can't find error-message file '/usr/share/mysql/english/errmsg.sys'` |
| `basedir=/opt/alisql` | 让 mysqld 知道 plugin/share 在哪 |
| `gosu` | entrypoint 以 root 启动好处理权限,真正跑 mysqld 时降到 mysql 用户 |

---

## 4. Dockerfile(5.7)

`5.7/Dockerfile` 跟 8.0 几乎一样,只改两处:

```dockerfile
ENV ALISQL_VERSION=5.7.xx-y     # 替换成你们 5.7 的实际版本
ENV MYSQL_MAJOR=5.7
```

> ⚠️ 如果 AliSQL 5.7 是用更老的 glibc 编译的,可能在 `oraclelinux:8-slim` 上跑不起来。先试,跑不了再换 `oraclelinux:7-slim`(注意:7 用的是 `yum`,不是 `microdnf`,装包命令要改)。

---

## 5. entrypoint 脚本

### 5.1 8.0 用上游 8.x 的 entrypoint(已下载)

文件已在 `~/alisql-docker/8.0/docker-entrypoint.sh`(419 行,从 `docker-library/mysql/master/8.4/` 拿的)。

如果要重新下载:
```bash
curl -fsSL -o 8.0/docker-entrypoint.sh \
  https://raw.githubusercontent.com/docker-library/mysql/master/8.4/docker-entrypoint.sh
chmod +x 8.0/docker-entrypoint.sh
```

它处理了:
- 环境变量:`MYSQL_ROOT_PASSWORD` / `MYSQL_DATABASE` / `MYSQL_USER` / `MYSQL_PASSWORD` / `MYSQL_ROOT_HOST` / `MYSQL_ALLOW_EMPTY_PASSWORD` / `MYSQL_RANDOM_ROOT_PASSWORD`
- `*_FILE` 形式(读 Docker secrets)
- 首次启动:`mysqld --initialize-insecure` → 临时实例 → 建库建用户 → 跑 `/docker-entrypoint-initdb.d/*.{sh,sql,sql.gz,sql.xz,sql.zst,sql.bz2}` → 关掉
- 后续启动:数据目录已在,直接 `exec mysqld`
- `gosu mysql` 降权

### 5.2 5.7 entrypoint 的获取方式

5.7 不能直接抄 8.x 的(8.x 的 ALTER USER 用了 caching_sha2_password 等 5.7 不支持的语法)。**最稳的办法:从社区 mysql:5.7 镜像里拷一份**:

```bash
docker pull mysql:5.7
id=$(docker create mysql:5.7)
docker cp "$id:/usr/local/bin/docker-entrypoint.sh" 5.7/docker-entrypoint.sh
docker rm "$id"
chmod +x 5.7/docker-entrypoint.sh
```

这是经过实战验证的版本,直接能用。

### 5.3 LICENSE

两份 entrypoint 都来自 `docker-library/mysql`(MIT),**README 里要注明出处**。

---

## 6. 本地构建 + 冒烟测试(Mac/Linux 都行)

```bash
cd ~/alisql-docker

# 1) 把产物拿到位 —— 注意:文件名按 Dockerfile 期望的来重命名
#    原始文件名:alisql-8.0.44-1-linux-glibc2.17-x86_64.tar.xz
#    Dockerfile 期望:alisql-8.0.44-1-amd64.tar.xz
cp /path/to/alisql-8.0.44-1-linux-glibc2.17-x86_64.tar.xz \
   8.0/alisql-8.0.44-1-amd64.tar.xz

# 2) 单架构构建(本机架构,先验证 Dockerfile 正确)
#    Mac M 系列上如果只放了 amd64 tarball,要加 --platform linux/amd64
docker build \
  --platform linux/amd64 \
  --build-arg TARGETARCH=amd64 \
  -t alisql-test:8.0.44-1 \
  8.0/

# 3) 启动验证
docker run -d --name alisql-test \
  -e MYSQL_ROOT_PASSWORD=test123 \
  -e MYSQL_DATABASE=demo \
  -p 3306:3306 \
  alisql-test:8.0.44-1

# 4) 看初始化日志(等 ~10s 看到 "ready for connections")
docker logs -f alisql-test

# 5) 进容器验证 banner
docker exec -it alisql-test mysql -uroot -ptest123 -e 'select version(); show databases;'

# 6) 清理
docker rm -f alisql-test
docker rmi alisql-test:8.0.44-1
```

`select version();` 输出形如 `8.0.44-AliSQL-...` 就成了。

### 常见失败排查

| 现象 | 可能原因 | 解决 |
|---|---|---|
| `Can't find error-message file` | `lc-messages-dir` 没配 | 检查 my.cnf 里这一行 |
| `error while loading shared libraries: libxxx.so` | 镜像里少依赖 | `docker run --rm alisql-test:xxx ldd /opt/alisql/bin/mysqld` 查,加进 `microdnf install` |
| `unsupported architecture` | tarball 跟 platform 不匹配 | 检查 `--platform` 和实际 tarball 架构 |
| 启动卡住,日志空白 | 数据目录权限问题 | 确认 `/var/lib/mysql` 是 mysql:mysql owner |

---

## 7. 多架构构建(本地手动)

如果想本地直接 push 两架构:

```bash
# 一次性创建 buildx builder
docker buildx create --name multiarch --use --bootstrap

# 两个架构的 tarball 都准备好(已重命名)
ls 8.0/
# alisql-8.0.44-1-amd64.tar.xz
# alisql-8.0.44-1-arm64.tar.xz

# 登录
docker login -u alibaba

# build + push(一条命令同时产 amd64 + arm64,自动 manifest list)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t alibaba/alisql:8.0.44-1 \
  -t alibaba/alisql:8.0.44 \
  -t alibaba/alisql:8.0 \
  -t alibaba/alisql:latest \
  --push \
  8.0/
```

> Mac M 系列上构建 amd64 走 emulation,大概 5~15 分钟一个架构;CI 上更快。

---

## 8. GitHub Actions 自动化

`.github/workflows/build.yml`:

```yaml
name: build-and-push

on:
  push:
    tags:
      - 'AliSQL-8.0.*'
      - 'AliSQL-5.7.*'
  workflow_dispatch:
    inputs:
      branch:
        description: '分支:8.0 或 5.7'
        required: true
        default: '8.0'
      version:
        description: '完整版本号,如 8.0.44-1'
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      # ---- 解析 tag → branch + version ----
      - name: Parse tag / inputs
        id: parse
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            BRANCH="${{ github.event.inputs.branch }}"
            VERSION="${{ github.event.inputs.version }}"
          else
            # GITHUB_REF_NAME 形如 AliSQL-8.0.44-1
            TAG="${GITHUB_REF_NAME}"
            VERSION="${TAG#AliSQL-}"               # 8.0.44-1
            BRANCH=$(echo "$VERSION" | cut -d. -f1-2)  # 8.0
          fi
          echo "branch=$BRANCH"   >> $GITHUB_OUTPUT
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          # 还需要"无修订号版本",用于打 8.0.44 这个滚动 tag
          BASE=$(echo "$VERSION" | sed 's/-[0-9]*$//')   # 8.0.44
          echo "base=$BASE"       >> $GITHUB_OUTPUT
          echo "Parsed: branch=$BRANCH version=$VERSION base=$BASE"

      # ---- 下载 tarball 并重命名(关键:x86_64→amd64, aarch64→arm64) ----
      - name: Download AliSQL tarballs
        env:
          BRANCH:  ${{ steps.parse.outputs.branch }}
          VERSION: ${{ steps.parse.outputs.version }}
          # ⚠️ 改成你们真实的发布物存放地址
          RELEASE_BASE: https://github.com/alibaba/AliSQL/releases/download/AliSQL-${{ steps.parse.outputs.version }}
        run: |
          set -eux
          for pair in "amd64:x86_64" "arm64:aarch64"; do
            DOCKER_ARCH="${pair%:*}"
            ALISQL_ARCH="${pair#*:}"
            curl -fL -o "${BRANCH}/alisql-${VERSION}-${DOCKER_ARCH}.tar.xz" \
              "${RELEASE_BASE}/alisql-${VERSION}-linux-glibc2.17-${ALISQL_ARCH}.tar.xz"
          done
          ls -la "${BRANCH}/"

      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # ---- 计算 tag 列表 ----
      - name: Compute tags
        id: tags
        env:
          BRANCH:  ${{ steps.parse.outputs.branch }}
          VERSION: ${{ steps.parse.outputs.version }}
          BASE:    ${{ steps.parse.outputs.base }}
        run: |
          TAGS=(
            "alibaba/alisql:${VERSION}"   # 8.0.44-1   精确版本
            "alibaba/alisql:${BASE}"      # 8.0.44     最新修订
            "alibaba/alisql:${BRANCH}"    # 8.0        系列最新
          )
          # latest 只跟 8.0
          if [[ "${BRANCH}" == "8.0" ]]; then
            TAGS+=("alibaba/alisql:latest")
          fi
          T=$(IFS=,; echo "${TAGS[*]}")
          echo "list=$T" >> $GITHUB_OUTPUT
          echo "Tags: $T"

      - uses: docker/build-push-action@v5
        with:
          context: ${{ steps.parse.outputs.branch }}
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.tags.outputs.list }}
          cache-from: type=gha,scope=${{ steps.parse.outputs.branch }}
          cache-to:   type=gha,scope=${{ steps.parse.outputs.branch }},mode=max

  sync-readme:
    needs: build
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: peter-evans/dockerhub-description@v4
        with:
          username:          ${{ secrets.DOCKERHUB_USERNAME }}
          password:          ${{ secrets.DOCKERHUB_TOKEN }}
          repository:        alibaba/alisql
          short-description: "AliSQL — Alibaba's branch of MySQL"
          readme-filepath:   ./README.md
```

### Secrets 配置(在 GitHub repo Settings → Secrets and variables → Actions)
- `DOCKERHUB_USERNAME` = `alibaba`
- `DOCKERHUB_TOKEN`    = 第 1.2 步生成的 token

### 触发方式
```bash
# 自动触发(打 tag 即发布)
git tag AliSQL-8.0.44-1
git push origin AliSQL-8.0.44-1

# 手动触发(在 GitHub Actions 页面填表单)
```

---

## 9. Tag 策略

| Docker tag | 含义 | 何时更新 |
|---|---|---|
| `8.0.44-1` | 精确版本(对应你的 git tag) | 发版时一次性 |
| `8.0.44` | 8.0.44 系列最新修订(8.0.44-1, -2, -3...) | 每次 8.0.44-N 发布 |
| `8.0` | 8.0 系列最新 | 每次 8.0.x-N 发布 |
| `latest` | 当前主推 = 最新 8.0 | 跟随 8.0 |
| `5.7.xx-y` | 5.7 精确版本 | 5.7 发版时 |
| `5.7` | 5.7 系列最新 | 每次 5.7.x-N 发布 |

**`latest` 只跟 8.0,不跟 5.7**——避免新用户拉到老版本。

---

## 10. README.md(根目录)

参考骨架(根据需要扩充):

```markdown
# AliSQL Docker Images

AliSQL 是阿里巴巴基于 MySQL 的分支版本。本仓库提供 AliSQL 的官方 Docker 镜像。

源码:https://github.com/alibaba/AliSQL

## Supported tags

- `8.0.44-1`, `8.0.44`, `8.0`, `latest` — 对齐社区 MySQL 8.0.44
- `5.7.xx-y`, `5.7`                     — 对齐社区 MySQL 5.7.xx

支持架构:`linux/amd64`, `linux/arm64`

## Quick start

    docker run -d --name alisql \
      -e MYSQL_ROOT_PASSWORD=your-password \
      -p 3306:3306 \
      alibaba/alisql:8.0

5.7:

    docker run -d --name alisql \
      -e MYSQL_ROOT_PASSWORD=your-password \
      -p 3306:3306 \
      alibaba/alisql:5.7

## 环境变量

跟社区 MySQL 镜像完全一致:

- `MYSQL_ROOT_PASSWORD`
- `MYSQL_DATABASE`
- `MYSQL_USER` / `MYSQL_PASSWORD`
- `MYSQL_ALLOW_EMPTY_PASSWORD`
- `MYSQL_RANDOM_ROOT_PASSWORD`
- 以及 `*_FILE` 形式(用于 Docker secrets)

## 数据持久化

    docker run -d -v alisql_data:/var/lib/mysql alibaba/alisql:8.0

## 自定义配置

挂 `.cnf` 文件到 `/etc/mysql/conf.d/`:

    docker run -d -v $(pwd)/my.cnf:/etc/mysql/conf.d/custom.cnf alibaba/alisql:8.0

## 初始化 SQL

`*.sql` / `*.sh` 放到 `/docker-entrypoint-initdb.d/`,首次启动时执行。

## License

- AliSQL: GPLv2 (https://github.com/alibaba/AliSQL)
- 本仓库的 entrypoint 基于 docker-library/mysql (MIT)
```

---

## 11. 端到端发布一次的完整 checklist

**一次性准备**(只做一次):
- [ ] Docker Hub 注册 + 创建 `alibaba/alisql` 仓库 + 生成 token
- [ ] GitHub 创建 `alibaba/alisql-docker` 仓库
- [ ] 在 GitHub Settings → Secrets 加 `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`
- [ ] 把本指南里的 Dockerfile / entrypoint / workflow / README 提交到仓库
- [ ] 确认 AliSQL release 产物有公网可访问的 URL(GitHub Releases / OSS / 自建)

**每次发版**:
- [ ] AliSQL 主仓库打 tag `AliSQL-8.0.44-1` 并产出 tarball
- [ ] tarball 上传到 release 渠道
- [ ] 在 `alisql-docker` 仓库打同名 tag 并 push:
      ```
      git tag AliSQL-8.0.44-1
      git push origin AliSQL-8.0.44-1
      ```
- [ ] 等 GitHub Actions 跑完(~10~30 分钟,arm64 emulation 是大头)
- [ ] 在 <https://hub.docker.com/r/alibaba/alisql/tags> 确认新 tag 出现
- [ ] 拉一下验证:`docker pull alibaba/alisql:8.0.44-1 && docker run --rm -e MYSQL_ROOT_PASSWORD=t alibaba/alisql:8.0.44-1 mysqld --version`

---

## 12. 常见坑速查

| 坑 | 表现 | 解决 |
|---|---|---|
| .tar.xz 解压失败 | `tar: -z: Cannot ...` | 用 `tar -xJf`,装 xz |
| `Can't find error-message file` | mysqld 起不来 | my.cnf 加 `lc-messages-dir=/opt/alisql/share` |
| `caching_sha2_password cannot be loaded` | 客户端连不上 | 5.7 的客户端,启动参数加 `--default-authentication-plugin=mysql_native_password`(8.0 才有此问题) |
| Mac 构建慢 | M 芯片跑 amd64 走 QEMU | 本地只测一个架构,push 交给 CI |
| arm64 跑得慢 | 同上 | CI 上自购 arm64 runner 或用 `linux/arm64` 专用 builder |
| GitHub Actions 拉不到 release | 404 | 检查 release 是否 public、URL 是否对、`gh auth` token 权限(私有源要加 `Authorization` header) |
| 5.7 entrypoint 报 SQL 语法错 | 用了 8.x 版本 | 用 §5.2 的方法从 `mysql:5.7` 镜像拷出 |

---

## 附录:目录里已有的文件

```
~/alisql-docker/
├── 8.0/
│   └── docker-entrypoint.sh    ← 已下载(419 行,from docker-library/mysql 8.4)
├── 5.7/                         ← 空,按 §5.2 从 mysql:5.7 镜像 cp 出来
└── BUILD-GUIDE.md               ← 本文档
```

下一步建议:
1. 通读本文档,标注疑问
2. 先跑 §6(本地冒烟),确认 Dockerfile 正确
3. 再上 §8(CI 自动化)
