# Docker 基础入门：镜像、容器、Dockerfile，以及用 compose 起一个 Postgres

- 目标读者：会写 Python，但**从没用过容器**，在 Windows 11 上开发（Docker Desktop + WSL2）
- 前置知识：只要会用命令行跑 `python xxx.py`。容器相关的词一个都不用提前会，第 3 节会把镜像、容器、端口映射、数据卷这些从零讲一遍
- 学习时长：通读约 60 分钟，跟着敲完约 2 小时 / 可跳过：第 2 节是纯背景推演，急着起 Postgres 可以先跳到第 4 节示例 2 和第 5 节

---

## 1. 它是什么（一句话 + 一句类比）

Docker 是一个**把「一个软件 + 它需要的整套运行环境」打成一个包，然后在任何装了 Docker 的机器上一条命令跑起来**的工具。

类比：

> **镜像（image）像一张装好了系统和软件的 U 盘只读母盘；容器（container）是拿这张母盘启动起来的一台机器。**
> 母盘本身永远不变，机器可以随便删了再开一台，开出来的还是同样干净的样子。

这个类比后面会一直对上真实命令：

- 「拿到母盘」= `docker pull postgres:16`
- 「用母盘开一台机器」= `docker run postgres:16`
- 「机器删了」= `docker rm 容器名`（母盘还在，下次还能再开）
- 「自己刻一张母盘」= 写 `Dockerfile` + `docker build`
- 「机器删了但资料要留下」= 挂 **数据卷（volume）**，相当于给机器插一块外置硬盘，机器删了硬盘还在

---

## 2. 它解决什么问题

### 先反面推演：不用 Docker，怎么在自己的 Windows 上装一个 Postgres？

朴素做法大概是这样：

1. 去 postgresql.org 找 Windows 安装包，下载几百 MB 的 installer
2. 一路点下一步，装成 Windows 服务（开机自启，常驻后台）
3. 安装过程中设一个 superuser 密码，记在某个地方（下周就忘了）
4. 装完去改 `postgresql.conf`、`pg_hba.conf`，调端口、调允许哪些地址连
5. 再把连接串写进项目的环境变量里，`DATABASE_URL=postgresql://...`

能跑通，但痛点是能被真实感知到的：

- **装完卸不干净。** 卸载器跑完，注册表、遗留的 data 目录、Windows 服务项经常还在。下次重装报「端口已占用」或「data 目录已存在」，你得手动去删。
- **两个版本会打架。** 老项目跑在 Postgres 14，新的 RAG 项目要 16 带 pgvector。同一台机器上装两套，两个都想占 5432 端口，两个都注册成开机服务，你得手动改端口、手动停服务。
- **换台机器全部重来。** 换新电脑、或者同事要跑你的项目，上面 5 步得原样再做一遍，而且很可能装出一个版本不一样的环境。
- **「在我电脑上是好的」。** 你的机器上是 Postgres 16.2 + 某个扩展装过了，同事是 15 且没装扩展，同一份代码你能跑他报错，排查半天发现是环境差异，不是代码问题。

### 根源一句话

**软件的行为不只取决于代码，还取决于它周围那一整套东西（系统、依赖库、版本、配置）；而这套东西过去只能靠人手在每台机器上重复搭建，所以每台机器都不一样。** Docker 把这一整套东西也变成了可以下载、可以版本化、可以一条命令重建的东西。

### 经典应用场景

- **本地开发起依赖服务**：数据库、Redis、消息队列，用一条命令起，用完删干净。这是你这周要做的事。
- **打包自己的服务去部署**：把 FastAPI 服务连 Python 版本、依赖一起打成镜像，服务器上 `docker run` 就跑，不用在服务器上装 Python 和 pip。
- **同一台机器跑多个版本**：Postgres 14 和 16 同时在跑，互不干扰，各自映射到不同端口。
- **CI 里跑测试**：每次测试都在一个全新的干净容器里跑，不受上次残留影响。

不适合的场景也说一句：图形界面程序、需要极致硬件性能的场景，容器化收益不大，本篇不涉及。

---

## 3. 读下去之前，先搞懂这些概念

每个概念给「一行定义 + 一句大白话 + 一条最小命令」。不用背，看一遍有个印象，第 4 节敲命令时会自然对上。

### 3.1 镜像 image

- **定义**：一个只读的打包产物，里面装着一个精简系统 + 你要跑的软件 + 它的依赖。
- **大白话**：装好了系统和软件的 U 盘母盘，只读，不会被改。
- **最小命令**：

```bash
docker pull python:3.12-slim   # 从网上把这张"母盘"下载到本机
```

### 3.2 容器 container

- **定义**：由镜像启动起来的一个正在运行（或已停止）的实例，有自己的文件系统、进程、网络。
- **大白话**：拿母盘开起来的那台机器。同一张母盘可以同时开 10 台，互不干扰。
- **最小命令**：

```bash
docker run python:3.12-slim python -c "print('hi')"   # 用这张母盘开一台机器，跑一句 Python，跑完就停
```

### 3.3 镜像仓库 registry / Docker Hub

- **定义**：存放镜像、供人下载的服务器。Docker Hub 是官方的公共仓库，默认从这里拉。
- **大白话**：母盘的应用商店。`postgres`、`python` 这些官方镜像都在上面，不用注册就能下载。
- **最小命令**：

```bash
docker search postgres   # 在 Docker Hub 上搜镜像（也可以直接浏览器打开 hub.docker.com 搜）
```

### 3.4 tag（标签 / 版本号）

- **定义**：镜像名后面用冒号跟的那一段，标明版本或变体，格式 `镜像名:tag`。
- **大白话**：母盘的版本号。`postgres:16` 和 `postgres:14` 是两张不同的母盘。不写 tag 默认是 `:latest`，而 `latest` 今天和明天可能指向不同版本，**所以正式使用一定写死版本号**。
- **最小命令**：

```bash
docker pull postgres:16   # 明确要 16 这个版本，而不是含糊的 latest
```

### 3.5 层 layer（镜像分层）

- **定义**：镜像不是一个整体大文件，而是由多个只读层叠起来的；构建时的每一步生成一层。
- **大白话**：母盘是一叠透明胶片摞起来看到的效果。改最上面一张，下面几张不用重做——这就是后面「构建缓存」能省时间的原因。
- **最小命令**：

```bash
docker history python:3.12-slim   # 看这个镜像是由哪些层叠出来的，每层多大
```

### 3.6 Dockerfile

- **定义**：一个纯文本文件，一行一条指令，描述「怎么从一个基础镜像造出你要的镜像」。
- **大白话**：刻母盘的配方单。你写好配方，`docker build` 照着做出母盘。
- **最小命令**：

```bash
docker build -t 我的服务:v1 .   # 读当前目录的 Dockerfile，造出一张叫"我的服务:v1"的母盘
```

### 3.7 宿主机 host

- **定义**：运行 Docker 的那台真实机器。对你来说就是你的 Windows 11 笔记本本身。
- **大白话**：你手上这台电脑，相对于「电脑里跑着的那些容器」而言，就叫宿主机。后面说「宿主机的 5432 端口」，意思就是「你 Windows 上的 5432 端口」。

### 3.8 端口映射 port mapping

- **定义**：把宿主机的某个端口和容器内的某个端口接通，格式 `-p 宿主机端口:容器端口`。
- **大白话**：容器默认是关着门的，外面连不进去。端口映射相当于在你电脑的墙上开一个洞，接到容器内部的某个端口上。**冒号左边是你电脑，右边是容器里**，方向别记反。
- **最小命令**：

```bash
docker run -p 5432:5432 postgres:16   # 你电脑的 5432 → 容器里的 5432，这样本机的客户端才连得上
```

### 3.9 数据卷 volume

- **定义**：由 Docker 管理的一块独立存储，挂到容器内某个目录上，生命周期和容器分开。
- **大白话**：给容器插一块外置硬盘。容器删了，硬盘还在，下次新容器挂上同一块盘，数据还在。
- **最小命令**：

```bash
docker volume ls   # 列出本机所有数据卷
```

### 3.10 容器的「无状态」与数据会丢

- **定义**：容器内部写在自己文件系统里的改动，随 `docker rm` 一起消失。
- **大白话**：容器要当成「用完就扔」的东西。默认情况下，容器里数据库写的数据，容器一删就没了。**所以只要容器里有你不想丢的数据，就必须挂 volume。** 这是第 4 节示例 2 会实际演示一遍的坑。

### 3.11 Docker Desktop 与 WSL2 的关系

- **定义**：容器技术依赖 Linux 内核特性，Windows 上没有。Docker Desktop 借 WSL2（Windows 里的一个轻量 Linux 环境）提供 Linux 内核，容器实际跑在 WSL2 里；你在 PowerShell 或 Git Bash 里敲的 `docker` 命令只是客户端，把指令转发给它。
- **大白话**：Docker Desktop 是「窗口 + 引擎管理器」，WSL2 是「真正干活的地方」，`docker` 命令是「遥控器」。
- **实际影响两点**：① **Docker Desktop 没启动，任何 docker 命令都会失败**（第 6 节第一个坑）；② 容器里看到的文件系统是 Linux 的，路径是 `/var/lib/...` 这种，不是 `D:\...`，所以挂载 Windows 目录时写法有讲究（第 6 节最后一个坑）。

### 3.12 镜像 vs 虚拟机的区别

- **虚拟机**：模拟一整台电脑，里面装一个完整操作系统，有自己的内核，开机要几十秒，一个占几 GB 到几十 GB。
- **容器**：不装完整系统，**共用宿主机（在 Windows 上是 WSL2）的内核**，只打包应用和它的依赖库，启动通常一两秒，一个几十到几百 MB。
- **大白话**：虚拟机是租了一整套房；容器是在同一套房里各用一个上了锁的房间，共用水电（内核）。所以容器轻、快，但也因此容器里只能跑和宿主机内核兼容的东西（Windows 上就是 Linux 程序）。

### 术语速查表

| 术语 | 大白话 | 正文哪节 |
| --- | --- | --- |
| 镜像 image | 装好软件的只读 U 盘母盘 | 3.1 / 4.1 |
| 容器 container | 拿母盘开起来的一台机器 | 3.2 / 4.1 |
| registry / Docker Hub | 母盘的应用商店 | 3.3 |
| tag | 母盘的版本号，别用 latest | 3.4 |
| 层 layer | 摞起来的透明胶片，改上层不动下层 | 3.5 / 4.3 原理展开 |
| Dockerfile | 刻母盘的配方单 | 3.6 / 4.3 |
| 宿主机 host | 你手上这台 Windows 电脑 | 3.7 / 6.4 |
| 端口映射 `-p` | 在你电脑墙上开洞接到容器里，左=你电脑 右=容器 | 3.8 / 4.2 原理展开 |
| 数据卷 volume | 给容器插的外置硬盘，容器删了盘还在 | 3.9 / 4.2 |
| 无状态 | 容器用完就扔，不挂 volume 数据就丢 | 3.10 / 4.2 |
| Docker Desktop + WSL2 | 遥控器 + 真正干活的 Linux | 3.11 / 6.1 |

---

## 4. 从最简单的代码示例开始

开始前先确认一件事：**打开 Docker Desktop，等左下角的鲸鱼图标变成稳定状态（不再转圈）。** 没启动的话所有命令都会报错，见第 6.1。

命令在 Git Bash 和 PowerShell 里都能跑，唯一有差别的是第 6.5 讲的路径挂载。

### 4.1 示例一：最小可运行版本，看清容器的生命周期

```bash
docker run hello-world   # 拉一个官方的最小测试镜像并跑起来，它只做一件事：打印一段话然后退出
```

第一次跑会看到 `Unable to find image 'hello-world:latest' locally`，然后开始下载——这说明本机没这张母盘，Docker 自动去 Docker Hub 拉。之后打印一段 `Hello from Docker!`，程序结束，容器停止。

再来一个能交互的：

```bash
docker run -it python:3.12-slim python
# -it  两个参数的组合：-i 保持标准输入打开（你能往里打字），-t 分配一个终端（提示符、换行显示正常）
#      不加 -it，交互式程序会立刻因为读不到输入而退出
# python:3.12-slim  母盘名:版本号。slim 是精简变体，比完整版小很多，够跑 Python
# 最后那个 python  是要在容器里执行的命令，即启动 Python 交互式解释器
```

你会看到熟悉的 `>>>`。这个解释器跑在容器里，不是你 Windows 上的 Python：

```python
>>> import sys; sys.platform      # 输出 'linux'，证明你正身处一个 Linux 环境里
>>> exit()                        # 退出解释器 → 容器里主进程结束 → 容器随之停止
```

**关键一点：容器退出了，但它还在。** 验证：

```bash
docker ps        # 只列出"正在运行"的容器 —— 刚退出的那个不在这里
docker ps -a     # -a = all，列出所有容器，包括已停止的 —— 刚退出的那个在这里，STATUS 显示 Exited
```

已停止的容器还占着磁盘。清掉它：

```bash
docker rm 容器ID或名字   # 删掉这个容器（那台"机器"）；镜像（母盘）不受影响，下次还能再开
docker images            # 确认母盘还在
```

小技巧，加一个参数就不用手动删：

```bash
docker run --rm -it python:3.12-slim python   # --rm = 容器一停止就自动删除，适合这种跑完就不要的临时容器
```

**逐段解读**

- `docker run` 做的事其实是两步合一：**创建**一个容器 + **启动**它。所以每敲一次 `docker run` 就多一台新机器，不是重启原来那台。
- 容器的寿命 = 它里面**主进程**的寿命。主进程（这里是 `python`）一结束，容器就停。这也解释了为什么后面起 Postgres 时容器能一直活着——Postgres 服务进程不会自己退出。
- `docker ps` 和 `docker ps -a` 的差别是新手最容易困惑的点：机器「关机了」不等于「拆掉了」，关机的机器还在，得 `docker rm` 才是拆掉。

**这一步你得到了什么**

你能拉镜像、起容器、进容器里跑 Python、看容器列表、删容器。你也理解了「镜像 / 运行中容器 / 已停止容器」是三种不同的东西。

### 4.2 示例二：用 `docker run` 起一个真实的 Postgres

这就是你这周的任务。先用 `docker run` 手动跑通，理解每个参数，第 5 节再换成 compose。

```bash
docker run -d \
  --name my-pg \
  -e POSTGRES_PASSWORD=devpass \
  -e POSTGRES_DB=appdb \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
```

一行行解释（**做什么 + 为什么**）：

- `-d`：detached，后台运行。**为什么**：数据库要一直在后台跑，不加这个参数你的终端会被日志占住，Ctrl+C 还会把它停掉。
- `--name my-pg`：给容器起个名。**为什么**：不起名 Docker 会随机分配一个（比如 `nervous_tesla`），后面每次操作都得先去查 ID；起了名就能直接 `docker logs my-pg`。
- `-e POSTGRES_PASSWORD=devpass`：`-e` 是设一个环境变量传进容器。**为什么**：官方 postgres 镜像靠环境变量来做首次初始化，`POSTGRES_PASSWORD` 是**必填**的，不给它容器会启动失败并在日志里明确要求你设置。
- `-e POSTGRES_DB=appdb`：可选。**为什么**：让它初始化时顺手建一个叫 `appdb` 的数据库，省得你自己进去 `CREATE DATABASE`。不写的话会建一个和用户名同名的库（默认用户是 `postgres`）。
- `-p 5432:5432`：端口映射。**为什么**：容器默认与外界隔离，不开这个洞，你 Windows 上的 DBeaver / psql / Python 代码都连不进去。
- `-v pgdata:/var/lib/postgresql/data`：把一个叫 `pgdata` 的数据卷挂到容器里 Postgres 存数据的目录。**为什么**：Postgres 把数据文件写在 `/var/lib/postgresql/data`，这个路径在容器内部，容器一删就没；挂上卷之后数据实际存在 Docker 管理的卷里，容器删了数据还在。**这是这条命令里最不能省的参数。**
- `postgres:16`：用哪张母盘。写死 16，不用 `latest`。

看它有没有起来：

```bash
docker ps                # STATUS 应该是 Up ... 说明在运行
docker logs my-pg        # 看启动日志，找到最后一行 "database system is ready to accept connections" 才算真的能连了
docker logs -f my-pg     # -f = follow，持续刷新跟着看，Ctrl+C 退出（只是退出看日志，不会停容器）
```

**注意启动有几秒延迟**：容器 `Up` 了不代表 Postgres 已经准备好接连接。首次启动它要初始化数据目录，之后才 ready。这个「容器起来了但服务还没好」的时间差，就是第 5 节要用**健康检查**解决的问题。

连进去建张表：

```bash
docker exec -it my-pg psql -U postgres -d appdb
# docker exec  在一个"已经在运行"的容器里额外执行一条命令（区别于 docker run 是新开一个容器）
# -it          同 4.1，psql 是交互式的，必须有这两个
# my-pg        操作哪个容器
# psql -U postgres -d appdb   容器里自带的 psql 客户端，用 postgres 用户连 appdb 库
```

进去之后：

```sql
CREATE TABLE notes (id serial PRIMARY KEY, body text);   -- 建张表
INSERT INTO notes (body) VALUES ('第一条数据');           -- 塞一条数据
SELECT * FROM notes;                                     -- 确认查得到
\q                                                       -- 退出 psql（\q 是 psql 自己的命令，不是 SQL）
```

从你 Windows 本机连也是通的，因为映射了端口。用 Python 试（宿主机上跑，host 写 `localhost`）：

```python
# 需要先 uv add psycopg[binary]
import psycopg
with psycopg.connect("postgresql://postgres:devpass@localhost:5432/appdb") as conn:
    print(conn.execute("SELECT body FROM notes").fetchall())   # 应该能打印出刚才插入的那条
```

#### 演示一遍「不挂 volume 就删容器 → 数据全没」

先证明**挂了卷**的情况数据能活：

```bash
docker rm -f my-pg     # -f = force，运行中的容器也直接删掉（等于先 stop 再 rm）
docker volume ls       # pgdata 这个卷还在，容器删了不影响它
```

用同样的卷再开一台新容器：

```bash
docker run -d --name my-pg2 -e POSTGRES_PASSWORD=devpass -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data postgres:16
docker exec -it my-pg2 psql -U postgres -d appdb -c "SELECT * FROM notes;"
# -c 直接执行一条 SQL 就退出，不进交互界面
# 结果：刚才那条"第一条数据"还在 —— 数据存在卷里，跟容器无关
```

> 注意：`POSTGRES_DB=appdb` 这次不用再写了。那些 `POSTGRES_*` 环境变量只在**数据目录为空的首次初始化**时起作用；卷里已经有数据了，它们会被忽略，库和密码沿用第一次设定的。

再看**没挂卷**会怎样：

```bash
docker run -d --name tmp-pg -e POSTGRES_PASSWORD=x -p 5433:5432 postgres:16   # 注意没有 -v；端口用 5433 避开上面那个
docker exec -it tmp-pg psql -U postgres -c "CREATE TABLE t (id int); INSERT INTO t VALUES (1);"
docker rm -f tmp-pg                                                          # 删掉
docker run -d --name tmp-pg -e POSTGRES_PASSWORD=x -p 5433:5432 postgres:16   # 同样的命令再开一台
docker exec -it tmp-pg psql -U postgres -c "SELECT * FROM t;"
# 报错：relation "t" does not exist —— 表和数据全没了，这是一台全新的机器
docker rm -f tmp-pg                                                          # 清理
```

#### 原理展开：端口映射 `主机端口:容器端口` 的方向

`-p 5432:5432` 两边数字一样，看不出方向，容易记混。改一下就清楚了：

```bash
docker run -d --name pg16 -e POSTGRES_PASSWORD=x -p 15432:5432 postgres:16
```

- **右边 5432** 是容器**内部**的端口。这个数字由容器里的软件决定，Postgres 就是听 5432，你不能随便改（改了 Postgres 也不会听）。
- **左边 15432** 是你 **Windows 电脑上**的端口，你随便挑一个没被占用的。
- 于是本机连接串要写 `localhost:15432`，Docker 会把流量转到容器里的 5432。

这条规则的用处：**本机已经装了 Postgres 占着 5432 时，把左边改掉就能共存**，比如 `-p 15432:5432`。也可以同时起 14 和 16：`-p 15432:5432` 给 16、`-p 14432:5432` 给 14，两台各自安好。左边冲突了就是第 6.3 那个 `port is already allocated` 报错。

#### 原理展开：数据卷为什么必要

容器的文件系统是在**只读镜像层**之上加了一个**可写层**。你在容器里写的任何文件都落在这个可写层里，而这个可写层是**属于这个容器**的——`docker rm` 把容器删掉，可写层一起没了。

数据库的数据显然不能这样。数据卷（volume）是一块**放在容器之外、由 Docker 单独管理**的存储；`-v pgdata:/var/lib/postgresql/data` 的意思是「容器里访问这个目录时，实际读写的是外面那块 `pgdata`」。所以容器是「用完即弃」的，卷是「要留下的」，两者寿命解绑。

判断要不要挂卷的简单标准：**这个容器里有没有我删了会心疼的数据？** 数据库有，一次性跑脚本的容器没有。

**这一步你得到了什么**

你能起一个带持久化的 Postgres、看它的启动日志、用 `docker exec` 进去执行 SQL、从 Windows 本机的 Python 连上它；你亲手验证了挂卷和不挂卷的区别，也知道端口映射左右两边分别是谁。

### 4.3 示例三：写一个 Dockerfile，把 FastAPI 服务打包成镜像

假设项目结构是这样（用 uv 管依赖）：

```
my-api/
├── pyproject.toml      # 依赖声明
├── uv.lock             # 锁定的精确版本
├── .dockerignore       # 待会儿要建
├── Dockerfile          # 待会儿要建
└── app/
    └── main.py         # FastAPI 应用
```

`app/main.py` 就用最小的：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}
```

**Dockerfile**（文件名就叫 `Dockerfile`，没有扩展名）：

```dockerfile
FROM python:3.12-slim
# 从哪张母盘开始造。slim 是精简版，比完整 python 镜像小几百 MB
# 版本写死 3.12，别写 latest，否则某天基础镜像升到 3.13 你的代码可能就崩了

WORKDIR /app
# 设定容器内的工作目录，后续 COPY / RUN / CMD 都以这里为当前目录
# 为什么：避免把文件散落在容器根目录，也省得后面每条命令都写全路径

COPY pyproject.toml uv.lock ./
# 先只拷依赖声明文件，不拷代码。这个顺序是刻意的，原因见下面"原理展开"

RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev
# 装 uv，再按 uv.lock 里锁定的版本装依赖
# --no-cache-dir  不保留 pip 下载缓存，否则这些缓存会被固化进镜像层，白白增大体积
# --frozen        严格照 uv.lock 装，不允许它自行更新锁文件（保证镜像里的版本和你本机一致）
# --no-dev        不装开发依赖（pytest、ruff 之类），生产镜像不需要

COPY app ./app
# 现在才拷业务代码。代码改动频繁，放在最后

EXPOSE 8000
# 声明"这个镜像里的服务听 8000 端口"。注意：这只是一行文档说明，不会自动开放端口
# 真正对外开放还是靠 docker run 的 -p，写它是为了让读镜像的人知道该映射哪个端口

CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
# 容器启动时默认执行的命令，即启动 uvicorn
# --host 0.0.0.0 是关键：默认 127.0.0.1 只接受"容器自己内部"的连接，外面 -p 映射进来的流量会被拒
#        0.0.0.0 表示接受来自任何网络接口的连接，容器化服务必须这么写
# 用 JSON 数组形式（exec form）而不是一整个字符串：这样 uvicorn 是容器的主进程，
#        能正常收到 docker stop 发来的停止信号，优雅退出
```

**`.dockerignore`**（和 Dockerfile 同级）：

```
.venv/          # 本机的虚拟环境，里面是 Windows/Linux 混装的东西，拷进去只会捣乱
__pycache__/    # 编译缓存，无意义
.git/           # 版本历史，可能几百 MB，镜像里完全不需要
.env            # 装着密码的文件，绝对不能进镜像
*.md            # 文档不参与运行
.pytest_cache/  # 测试缓存
```

**为什么必须有 `.dockerignore`**：`COPY` 时 Docker 会先把当前目录的内容打包发给构建引擎。没有这个文件，`.git` 和 `.venv` 会一起被打包，构建变慢、镜像变大，而且 `.env` 里的密码会被固化进镜像层——**即使你后面再删掉，历史层里还在，谁拿到镜像都能翻出来**。

构建并跑起来：

```bash
docker build -t my-api:v1 .
# build      按 Dockerfile 造镜像
# -t my-api:v1   给造出来的母盘起名字:版本号（t = tag）。不起名就只有一串 ID，很难用
# 最后那个 .     构建上下文 = 当前目录，即"把这个目录发给构建引擎，COPY 就从这里找文件"

docker run -d --name api -p 8000:8000 my-api:v1
# 后台起一台机器，把本机 8000 接到容器里的 8000
```

验证：浏览器打开 <http://localhost:8000/health>，或者

```bash
curl http://localhost:8000/health   # 应返回 {"status":"ok"}
docker logs api                     # 出问题就先看日志，uvicorn 的启动信息和报错都在这里
docker rm -f api                    # 用完删掉
```

#### 原理展开：镜像分层与构建缓存（为什么 `COPY pyproject.toml` 要在 `COPY .` 之前）

Dockerfile 里**每一条指令生成一个层**，层是叠起来的。`docker build` 时 Docker 会逐条检查：这条指令和它依赖的输入跟上次比有没有变？

- 没变 → 直接复用上次的那一层（日志里显示 `CACHED`），跳过执行
- 变了 → 重新执行这条指令，**并且它之后的所有指令全部重新执行**（因为下层变了，上层不能再复用）

关键就在最后半句：**缓存一旦在某一层失效，后面全部失效。** 所以指令顺序要遵循一个原则——**越不容易变的放前面，越容易变的放后面。**

对比一下：

```dockerfile
# 顺序错误的写法
COPY . .                                   # 代码和 pyproject.toml 一起拷进来
RUN pip install uv && uv sync --frozen     # 装依赖
```

你改一行 `main.py` 里的字符串，`COPY . .` 这层就变了，于是**下面装依赖那层也失效，几十个包全部重新下载安装**，构建从几秒变成几分钟。而你的依赖其实一个都没变。

正确写法把它们拆开：

```dockerfile
COPY pyproject.toml uv.lock ./             # 这两个文件很少变
RUN pip install uv && uv sync --frozen     # 所以这层几乎总能命中缓存
COPY app ./app                             # 只有这层失效，重拷几个 py 文件，一秒的事
```

改代码 → 只有最后一层重建。改了依赖（`pyproject.toml` 变了）→ 依赖层才重装，这时候本来就该重装。这就是「用顺序换构建速度」，也是所有语言的 Dockerfile 都在用的通用套路（Node 里是先 `COPY package.json`，一个道理）。

顺带解释 `docker history python:3.12-slim` 为什么能看到一串层：官方镜像也是这么一层层造出来的。多个镜像共享同一个基础层时，磁盘上只存一份——这也是为什么第二次拉别的 Python 镜像会快很多。

**这一步你得到了什么**

你能写 Dockerfile 把自己的 FastAPI 服务打成镜像并跑起来；你知道 `EXPOSE` 只是声明、真正开放靠 `-p`，知道 `--host 0.0.0.0` 不写就连不上，也知道 `COPY` 的顺序为什么直接决定你每次改代码要等几秒还是几分钟。

---

## 5. <mark style="background: #BBFABBA6;">最小可用的实际用法：一份 `docker-compose.yml`</mark>

第 4 节那条 `docker run` 又长又要手敲，而且起两个服务（Postgres + API）就得敲两条、还要处理它们之间的网络。**Compose 就是把这些写进一个 YAML 文件，一条命令全起起来。**

> **`docker compose`（空格）和 `docker-compose`（连字符）的关系**：连字符的是早年独立的 Python 版工具，现在已不再是推荐方式；空格的是 v2，作为 Docker CLI 的插件内置在 Docker Desktop 里。功能基本对应，**本篇一律用空格版**。网上老教程里的 `docker-compose up` 换成 `docker compose up` 即可。

在项目根目录建 `docker-compose.yml`：

```yaml
services:                                    # 【必须】所有要起的服务都写在这下面

  db:                                        # 【必须】服务名，自定义。这个名字同时是容器间互访的主机名，很重要
    image: postgres:16                       # 【必须】用哪张母盘，写死版本
    environment:                             # 【必须】传给容器的环境变量，等价于 docker run 的 -e
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD} # 【必须】postgres 镜像强制要求；${} 表示从同目录的 .env 文件读，不硬编码密码
      POSTGRES_USER: ${POSTGRES_USER}         # 【可选】不写默认用户是 postgres
      POSTGRES_DB: ${POSTGRES_DB}             # 【可选】不写会建一个和用户名同名的库
    ports:                                   # 【可选，但你需要】只有你想从 Windows 本机直连数据库时才要
      - "5432:5432"                          #   左=你电脑的端口，右=容器里的端口。被占用就改成 "15432:5432"
    volumes:                                 # 【必须】不挂卷，down 之后数据就没了
      - pgdata:/var/lib/postgresql/data      #   把下面定义的 pgdata 卷挂到 Postgres 的数据目录
    healthcheck:                             # 【可选，但强烈建议】让 Docker 判断"数据库真的能连了吗"
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
                                             #   pg_isready 是 Postgres 自带的探活工具，能连上就返回成功
      interval: 5s                           #   每 5 秒探一次
      timeout: 5s                            #   单次探测超过 5 秒算失败
      retries: 5                             #   连续失败 5 次才判定为 unhealthy
    restart: unless-stopped                  # 【可选】容器异常挂掉时自动重启，除非你手动停的它

  api:                                       # 【可选】只起数据库的话，整个 api 段可以删掉
    build: .                                 # 【必须，如果要这个服务】用当前目录的 Dockerfile 现场构建，而不是拉现成镜像
    ports:
      - "8000:8000"                          # 【必须，如果要这个服务】不映射的话浏览器访问不到
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      # 【必须】注意 host 写的是 db —— 上面那个服务名，不是 localhost。原因见下面"服务名互访"
    depends_on:                              # 【可选，但建议】控制启动顺序
      db:
        condition: service_healthy           #   等 db 的 healthcheck 通过了才起 api
                                             #   只写 depends_on: [db] 的话只等"容器起来"，不等"数据库准备好"，
                                             #   api 很可能在数据库 ready 之前就去连，直接连接失败
    restart: unless-stopped                  # 【可选】

volumes:                                     # 【必须】顶层声明一下用到的卷，Docker 才会去创建管理它
  pgdata:                                    #   名字和上面 volumes 里引用的一致即可，内容留空
```

同目录建 `.env`（**加进 `.gitignore`，不要提交**）：

```
POSTGRES_USER=appuser
POSTGRES_PASSWORD=devpass_change_me
POSTGRES_DB=appdb
```

### 关于「服务名互访」

这是 compose 最省事的地方，也是最容易踩的认知坑：

- **同一个 compose 文件里的服务之间，直接用服务名当主机名互相访问。** `api` 连数据库写 `db:5432`，Docker 内置的 DNS 会把 `db` 解析到数据库容器。注意这里用的是**容器端口 5432**，跟 `ports` 那行映射的宿主机端口无关——容器间通信根本不经过宿主机。
- **从你 Windows 本机（宿主机）连，写 `localhost:5432`**，走的是 `ports` 映射出来的那个洞。
- 这两个连接串不一样，是同一个数据库的两条不同进门路径。搞混了就是第 6.4 那个坑。

### 四条常用命令

```bash
docker compose up -d          # 起：按 yml 把所有服务建好并后台运行（第一次会自动 build 和 pull）
docker compose logs -f        # 看：跟踪所有服务的日志。看单个服务加服务名：docker compose logs -f db
docker compose down           # 停：停止并删除容器和网络。【卷保留，数据还在】日常收工用这个
docker compose down -v        # 危险：连数据卷一起删。数据库里所有数据立刻永久消失，没有回收站、没有确认提示
```

> **`down -v` 请当成危险命令对待。** 它和 `down` 只差两个字符，后果差别是「明天接着用」和「表和数据全部重建」。只在你明确想推倒重来（比如改了初始化脚本要重新初始化）时才用。日常一律 `down`。

常用的还有几条：

```bash
docker compose ps                          # 看这个项目下各服务的状态，能看到 healthy / unhealthy
docker compose exec db psql -U appuser -d appdb   # 进数据库容器执行 psql，等价于 4.2 的 docker exec
docker compose up -d --build               # 改了代码/Dockerfile 后强制重新构建镜像再起
docker compose restart api                 # 只重启某一个服务
```

### 实际项目通常还会配什么（进阶，可先跳过）

先跑通上面那份，这些等有需要再回来看：

- **多阶段构建瘦身镜像**：在一个临时阶段里装编译依赖、构建产物，再只把产物拷进一个干净的最终镜像，去掉编译工具链，镜像能小一大截。
- **非 root 用户运行**：默认容器里是 root 身份，一旦服务被攻破风险更大。Dockerfile 里建个普通用户再 `USER` 切过去。
- **`.env` 不入库**：`.env` 加进 `.gitignore`，另外提交一份 `.env.example` 只写键名不写值，让别人知道要配哪些变量。
- **pgvector**：你后面做 RAG 要向量检索，把 `image: postgres:16` 换成带 pgvector 的镜像（如 `pgvector/pgvector:pg16`）就行，compose 的其他部分不用动——这也说明了容器化的好处：换个数据库变体只是改一行。

---

## 6. 常见坑与易错点

### 6.1 Docker Desktop 没启动（Windows 上出现频率第一）

**报错**（Windows 上大致长这样，措辞随版本略有差异）：

```
docker: error during connect: ... open //./pipe/dockerDesktopLinuxEngine:
The system cannot find the file specified.
```

**现象**：任何 docker 命令都失败，包括 `docker ps`。刚开机、或者 Docker Desktop 被你关掉之后必然遇到。

**原因**：你敲的 `docker` 只是个**客户端**（遥控器），真正干活的引擎在 Docker Desktop 管理的 WSL2 里（见 3.11）。引擎没起来，客户端连不上那个通信管道，就报「找不到指定的文件」——这里的「文件」指的是通信管道，不是你的代码文件，所以这条报错信息容易误导。

**解决**：启动 Docker Desktop，等鲸鱼图标不再转圈（首次启动可能要几十秒），再 `docker ps` 确认能返回表头。想省事就在 Docker Desktop 设置里勾上开机自启。

### 6.2 端口已被占用

**报错**：

```
Error response from daemon: driver failed programming external connectivity on endpoint ...
Bind for 0.0.0.0:5432 failed: port is already allocated
```

（如果占用方是 Windows 本机程序而非另一个容器，也可能是 `bind: address already in use`。）

**现象**：起 Postgres 容器时失败，`docker ps -a` 里能看到这个容器状态是 Created 或 Exited。

**原因**：`-p 5432:5432` 左边那个宿主机端口已经有人在听了。最常见两种：① 你以前用安装包在 Windows 上装过 Postgres，它注册成了开机自启的 Windows 服务，一直占着 5432；② 你之前起的另一个 Postgres 容器还在跑。

**解决**（三选一）：

```bash
docker ps        # 先看是不是自己之前的容器占着，是就 docker rm -f 掉
```

要查是哪个进程占的，PowerShell 里：

```powershell
netstat -ano | findstr :5432     # 最后一列是 PID，再去任务管理器按 PID 找是谁
```

最省事的办法是**换宿主机端口，别去动本机已有服务**：

```bash
-p 15432:5432    # 改左边即可，容器里还是 5432。之后本机连接串写 localhost:15432
```

compose 里同理，改成 `- "15432:5432"`。注意：**这只影响你从 Windows 本机连**，compose 内部 `api` 连 `db:5432` 完全不受影响。

### 6.3 `docker compose down -v` 误删数据卷

**现象**：昨天建的表和数据全没了，服务起来像全新安装，日志里又出现了首次初始化的内容。

**原因**：`down -v` 的 `-v` 会删除该 compose 项目创建的**具名数据卷**。数据卷是数据库数据真正待的地方（见 4.2 原理展开），删了就是删库。这个命令没有二次确认，也没有回收站。

**解决**：删了就是真没了，**只能靠备份恢复**。所以重点是预防：

- 日常停服务只用 `docker compose down`，甚至 `docker compose stop`（连容器都保留，起得更快）
- 想清空数据重来时，才有意识地用 `down -v`
- 有真需要留的本地数据，用 `pg_dump` 定期导一份到项目外的目录

同类要小心的还有 `docker system prune -a --volumes`，它会清理全机未使用的卷，比 `down -v` 影响面更大。

### 6.4 容器里的 `localhost` 指的是容器自己，不是你的电脑

这是最反直觉的一个，值得单独记住。

**现象**：FastAPI 代码里连接串写 `postgresql://...@localhost:5432/appdb`，在 Windows 上直接 `uv run uvicorn` 跑好好的；一放进容器就报连接被拒绝（`Connection refused` / `could not connect to server`）。

**原因**：每个容器有自己独立的网络，`localhost` 在容器里指的是**这个容器自身**。API 容器里没有 Postgres，所以它连自己的 5432 当然连不上。它跟宿主机的 `localhost` 是两个完全不同的东西。

**解决**，按谁访问谁分三种情况：

| 谁访问谁 | host 应该写什么 | 说明 |
| --- | --- | --- |
| Windows 本机 → 容器里的 Postgres | `localhost:5432` | 走 `ports` 映射出来的洞（5.节 ports 那行） |
| 容器 → 同一个 compose 里的另一个容器 | `db:5432`（服务名） | Docker 内置 DNS 解析服务名；用容器端口，不经过宿主机 |
| 容器 → 宿主机上跑的服务 | `host.docker.internal` | Docker Desktop 提供的特殊主机名，指向你的 Windows。比如容器里的服务要连你本机装的某个工具 |

所以第 5 节 compose 里 `DATABASE_URL` 的 host 是 `db`。一个实用做法：**把连接串放进环境变量**，本机跑时用 `localhost`，容器里跑时由 compose 注入 `db`，代码一行不用改。

### 6.5 Windows 路径挂载 `-v` 的写法坑

**现象**：想把本机目录挂进容器（比如挂代码目录做热重载、或挂初始化 SQL 脚本），结果报路径无效，或者容器里那个目录是空的。

**原因**：容器里是 Linux，路径是 `/app` 这种；Windows 是 `D:\project` 这种。反斜杠、盘符、还有 Git Bash 的路径自动转换三件事凑在一起，容易出问题。特别是 **Git Bash 会把命令里看起来像路径的 `/app` 自动转换成 `D:/Git/app`**，导致挂载到一个莫名其妙的位置。

**解决**：

PowerShell 里用正斜杠，或者用变量表示当前目录：

```powershell
docker run -v D:/freelancer/workspace/my-api:/app my-api:v1   # 盘符 + 正斜杠，冒号左边是 Windows 路径
docker run -v ${PWD}:/app my-api:v1                           # PowerShell 用 ${PWD} 表示当前目录
```

Git Bash 里当前目录用 `$(pwd)`，并在冒号右边的容器路径前**多加一个斜杠**来阻止自动转换：

```bash
docker run -v "$(pwd)":/app my-api:v1      # 有时可用，但 /app 可能被 Git Bash 改写
docker run -v "$(pwd)"://app my-api:v1     # 双斜杠告诉 Git Bash 别转换这个路径
MSYS_NO_PATHCONV=1 docker run -v "$(pwd)":/app my-api:v1   # 更彻底：这次调用整体关闭路径转换
```

**规避这个坑最省事的办法是用 compose**，写在 yml 里的相对路径不受 shell 影响：

```yaml
    volumes:
      - ./app:/app          # 相对于 yml 文件所在目录，跨平台一致，不用管反斜杠和路径转换
```

另外提一句性能：挂载 Windows 磁盘上的目录（`D:\...`）进容器，读写会明显比 Linux 原生慢，文件多的时候能感觉到。项目放在 WSL2 内部的文件系统里会快很多，但那是另一套工作流，起步阶段不必折腾。

---

## 7. 验证你学会了（自测题）

不看正文做，卡住了再回去翻标注的小节。

### 第 1 题（概念复述）

用自己的话回答，别用术语解释术语：

1. 镜像和容器分别是什么，两者什么关系？为什么「删掉容器，镜像还在」？
2. Docker 到底解决了什么问题？举一个你自己电脑上真实遇到过、Docker 能解决的麻烦。
3. 容器和虚拟机差在哪？为什么容器能一两秒起来？

> 答案在：第 1 节、第 2 节、第 3.12 小节

### 第 2 题（默写命令）

关掉这份文档，默写一条 `docker run` 命令，要求满足：

- 起 Postgres 16，后台运行
- 容器叫 `ragdb`
- 密码设成 `secret`，顺手建一个叫 `ragapp` 的库
- **数据要持久化**，容器删了重建数据还在
- 你本机已经装了 Postgres 占着 5432，所以宿主机这边要用 15432

写完自己检查三件事：`-d` 有没有？`-v` 挂的容器内路径对不对？`-p` 的左右两个数字有没有写反？

再写出：① 怎么看它的启动日志确认 ready ② 怎么进去执行一条 SQL。

> 答案在：第 4.2 小节

### 第 3 题（组合应用）

你在一个 `docker-compose.yml` 里同时起了 Postgres（服务名 `db`）和 FastAPI 服务（服务名 `api`），Postgres 那边配了 `ports: - "15432:5432"`。

1. `api` 容器里的 `DATABASE_URL`，host 和端口该写什么？为什么不是 `localhost:15432`？
2. 你要用 Windows 上的 DBeaver 连这个库，连接串的 host 和端口写什么？
3. 为什么要给 `db` 配 `healthcheck`，`api` 要配 `depends_on: condition: service_healthy`？只写 `depends_on: [db]` 会出什么问题？
4. 收工了想停掉所有服务，但明天还要用今天建的表——敲哪条命令？哪条命令绝对不能敲？

> 答案在：第 5 节（服务名互访、depends_on、四条常用命令）、第 6.4 小节

---

## 附：本篇没讲什么

以下超出入门范围，等真的要部署到多机环境时再学：Kubernetes、把镜像推送到镜像仓库、Docker Swarm。本篇只覆盖「本地起依赖服务」和「把自己的服务打成镜像跑起来」这两件事，这两件事够你把 RAG 项目的本地环境搭起来。





