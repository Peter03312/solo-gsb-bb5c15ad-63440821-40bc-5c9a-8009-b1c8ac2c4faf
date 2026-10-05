# 离线基因组区间迁移 Web 应用（BED Liftover）

供实验室**内网/离线环境**使用的基因组坐标区间迁移工具。用户在浏览器中
上传 BED 文件与对应的 UCSC `chain` 文件，指定 BED 区间属于链的
**target 侧**还是**query 侧**，先在页面上逐条预览转换状态，确认后再
下载 TSV 结果。

- 后端：Python 3.11+ / FastAPI / Uvicorn
- 前端：原生 HTML + CSS + JavaScript（无任何 CDN / 外网依赖）
- 运行时数据不落盘：上传内容与转换结果只保存在进程内存中；校验失败时
  **不会**生成任何可下载文件

---

## 目录结构

```
.                            # 交付外层目录（Compose 项目目录之一）
├── .env                     # 外层启动时定位服务片段（已随仓库提交）
└── liftover-web/
    ├── app/                  # 后端
    │   ├── bed.py            # BED 解析（零基左闭右开，错误带行号）
    │   ├── chain.py          # UCSC chain 解析（严格校验，错误带行号）
    │   ├── engine.py         # 区间转换引擎（缺口拆分 / 歧义 / 负链）
    │   ├── report.py         # TSV 导出与预览 JSON
    │   └── main.py           # FastAPI 入口
    ├── static/               # 前端（纯本地资源）
    ├── examples/             # 可直接试用的 BED / chain 示例
    ├── tests/                # pytest 测试（28 个用例）
    ├── scripts/init.sh       # 本地开发依赖初始化
    ├── Dockerfile            # 镜像构建（含必需文件自检）
    ├── docker-compose.yml    # Compose 入口（include 服务片段）
    ├── service.yml           # 完整服务定义（build 上下文锚定本目录）
    ├── .env.example
    ├── requirements.txt      # 运行依赖
    └── requirements-dev.txt  # 运行 + 测试依赖
```

---

## 一、Docker / Compose 启动（推荐，完全离线）

镜像在构建时安装全部 Python 依赖，运行阶段不需要访问外网。需要
**Docker Compose v2.20+**（`docker compose version` 可查看；服务定义通过
`include` 片段组织，以保证两种入口构建的是同一份应用）。

### 方式 A：进入 `liftover-web/` 后直接启动（一键启动，保持不变）

```bash
cd liftover-web

# 构建并后台启动
docker compose up -d --build

# 查看状态 / 日志
docker compose ps
docker compose logs -f
```

### 方式 B：在交付外层目录指定 compose 文件启动

不必 `cd` 进应用目录，在**外层目录**（即 `liftover-web/` 的上一级）直接运行：

```bash
# 在外层目录；同目录下已随仓库提供 .env，用于把服务片段定位到 liftover-web/
docker compose -f liftover-web/docker-compose.yml up -d --build

# 也可显式声明项目目录（等价）
docker compose --project-directory . \
  -f liftover-web/docker-compose.yml up -d --build

docker compose -f liftover-web/docker-compose.yml ps
docker compose -f liftover-web/docker-compose.yml logs -f
```

> 为什么需要外层 `.env`？Compose 对相对构建路径的解析以**项目目录**为基准。
> 方式 A 的项目目录是 `liftover-web/`，方式 B 的项目目录是外层目录。
> `docker-compose.yml` 通过 `include` 引入锚定在 `liftover-web/` 的
> `service.yml`（build 上下文相对片段自身解析），而 include 路径默认取
> `service.yml`（方式 A）；外层 `.env` 将其覆盖为
> `liftover-web/service.yml`（方式 B）。两种方式最终都以 `liftover-web/`
> 为构建上下文、使用同一份 `Dockerfile`，产出镜像、容器端口与行为完全一致。

两种方式启动后访问地址相同：

- **http://localhost:8080** （应用页面）
- 健康检查：`http://localhost:8080/healthz`

停止（在启动时所在的目录执行；方式 B 需带上同样的 `-f`）：

```bash
# 方式 A
docker compose down

# 方式 B
docker compose -f liftover-web/docker-compose.yml down
```

### 端口与配置

复制环境变量模板并按需修改（仅方式 A 会读取 `liftover-web/.env`）：

```bash
cp .env.example .env
```

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `LIFTOVER_PORT` | `8080` | 宿主机映射端口（容器内固定监听 8080） |
| `LIFTOVER_MAX_UPLOAD_MB` | `50` | 单个上传文件大小上限（MB） |
| `LIFTOVER_JOB_TTL_SECONDS` | `3600` | 转换结果在内存中的保留时长（秒） |
| `LIFTOVER_MAX_JOBS` | `100` | 内存中同时保留的结果任务数量上限 |

容器内**始终监听 8080**；上表的 `LIFTOVER_PORT` 只改变宿主机映射端口。
改端口示例（随后访问 `http://localhost:9000`）：

```bash
# 方式 A
LIFTOVER_PORT=9000 docker compose up -d

# 方式 B
LIFTOVER_PORT=9000 docker compose -f liftover-web/docker-compose.yml up -d
```

不用 Compose 时（在 `liftover-web/` 目录内执行）：

```bash
docker build -t liftover-web:1.0.0 .
docker run -d --name liftover-web -p 8080:8080 \
  -e LIFTOVER_MAX_UPLOAD_MB=100 \
  liftover-web:1.0.0
```

### 构建必需文件缺失即失败

`Dockerfile` 在安装依赖前会校验 `requirements.txt`、`app/` 全部模块与
`static/` 三个静态文件是否都在构建上下文中，并在安装后执行一次
`import app.main` 自检。缺少任一构建必需文件时 **build 直接失败**，
不会产出“容器能启动、健康检查却访问不到页面/资源”的残缺镜像。

---

## 二、本地 Python 直接启动（开发 / 调试）

需要 Python 3.11+。

```bash
cd liftover-web

# 1) 初始化：创建虚拟环境并安装依赖（含测试依赖）
bash scripts/init.sh

# 2) 启动
.venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8080
```

访问 **http://localhost:8080**。

运行测试：

```bash
.venv/bin/python -m pytest -q
```

---

## 三、使用方法

1. 选择 BED 文件（至少前 3 列 `chrom start end`，坐标为**零基左闭右开**，
   支持第 4 列起的额外列但不参与转换；允许 `#`、`browser`、`track` 行）。
2. 选择对应的 UCSC `chain` 文件。
3. 选择 BED 区间位于 chain 的哪一侧：
   - **target 侧**：target → query；
   - **query 侧**：query → target。
4. 点击「校验并转换」。页面显示每条源区间的状态；确认无误后点击
   「下载完整结果 TSV」。

### 状态定义

| 状态 | 含义 |
| --- | --- |
| 完全映射 `mapped` | 区间全部落在对齐块内 |
| 部分映射 `partial` | 一部分落在链内缺口/未对齐区域；命中部分被拆成**按源坐标有序的片段**，未映射部分丢弃 |
| 歧义 `ambiguous` | 同一源区间命中**多个 chain / 多个目标位置**；系统列出全部候选，**绝不自动挑选** |
| 无映射 `unmapped` | 没有任何碱基可映射；结果中保留源行号、源区间与原因 |

### 坐标与负链

- 输入与输出一律是**零基、左闭右开**（`start < end`）。
- chain 负链侧的存储坐标按 UCSC 语义（以染色体长度为枢轴）归一化后
  再做区间求交，因此**负链转换后的坐标仍然是 start < end 的零基左闭
  右开区间**；结果的 `dest_strand` 列标明该候选在目标坐标系中的方向。

### 下载文件格式（TSV）

UTF-8（带 BOM，Excel 友好），列：

```
source_line, status, source_chrom, source_start, source_end,
candidate_index, chain_id, dest_chrom, dest_strand,
fragment_index, src_start, src_end, dest_start, dest_end, reason
```

- `source_line` 是源文件**物理行号**（从 1 开始）；
- 跨缺口区间每个片段占一行；歧义区间的每个候选、每个片段各占一行；
- `unmapped` 行保留源行号与原因，候选/片段列写 `.`。

### 失败即阻止导出

- BED 坏行（列数不足、坐标非整数、`start >= end` 等）或 chain 格式错误
  （头列数/字段非法、块越界、缺少终止行、gap 越界等）都会在页面上
  **带行号列出全部问题**，接口返回 400，且**不会生成或保留任何可下载
  文件**；修正后需重新提交。

---

## 四、HTTP API 调用示例

页面用到的全部接口均可直接用 `curl` 调用。默认地址 `http://localhost:8080`
（若用 `LIFTOVER_PORT` 改了宿主机端口，把端口换成对应值；路径不变）。

### 1. 健康检查 `GET /healthz`

```bash
curl -i http://localhost:8080/healthz
# HTTP/1.1 200 OK
# {"status":"ok"}
```

### 2. 页面与静态资源

```bash
curl -i http://localhost:8080/                 # 200，返回 static/index.html
curl -o /dev/null -w "%{http_code}\n" http://localhost:8080/static/app.js
curl -o /dev/null -w "%{http_code}\n" http://localhost:8080/static/style.css
```

### 3. 上传并转换 `POST /api/liftover`

`multipart/form-data`，字段固定为：

| 字段 | 类型 | 取值 |
| --- | --- | --- |
| `bed` | 文件 | BED 文件（至少 `chrom start end` 三列） |
| `chain` | 文件 | UCSC chain 文件 |
| `side` | 表单字段 | `target`（target → query）或 `query`（query → target） |

```bash
curl -s -X POST http://localhost:8080/api/liftover \
  -F "bed=@examples/example_target.bed" \
  -F "chain=@examples/demo.chain" \
  -F "side=target" | python3 -m json.tool
```

成功返回 `200`：

```json
{
  "download_token": "8DYGSVPHDaFs...（用于第 4 步下载）",
  "download_name": "liftover_target_to_query_20261005-032133.tsv",
  "preview": {
    "total": 6,
    "counts": {"mapped": 1, "partial": 1, "ambiguous": 1, "unmapped": 3},
    "rows": [ /* 每条源区间一行的预览（状态、候选、片段、原因） */ ]
  }
}
```

`side` 非法时返回 `400`，`{"errors": [...]}`：

```bash
curl -i -X POST http://localhost:8080/api/liftover \
  -F "bed=@examples/example_target.bed" \
  -F "chain=@examples/demo.chain" \
  -F "side=sideways"        # HTTP/1.1 400 Bad Request
```

文件内容校验失败（如 `examples/example_bad.bed`）同样返回 `400`，
`errors` 数组带行号列出全部问题，且**不会**产生下载 token：

```bash
curl -s -X POST http://localhost:8080/api/liftover \
  -F "bed=@examples/example_bad.bed" \
  -F "chain=@examples/demo.chain" \
  -F "side=target"
# {"errors": ["BED 第 3 行: ...", "BED 第 4 行: ...", "BED 第 5 行: ..."]}
```

query → target（含负链）只需把 `side` 换成 `query`：

```bash
curl -s -X POST http://localhost:8080/api/liftover \
  -F "bed=@examples/example_query.bed" \
  -F "chain=@examples/demo.chain" \
  -F "side=query"
```

### 4. 下载结果 `GET /api/download/{token}`

用上一步返回的 `download_token` 拼接路径。响应为带 UTF-8 BOM 的
`text/tab-separated-values`，并通过 `Content-Disposition` 给出文件名
（即第 3 步返回的 `download_name`）：

```bash
TOKEN="上一步返回的 download_token"

curl -i "http://localhost:8080/api/download/${TOKEN}" -o result.tsv
# HTTP/1.1 200 OK
# content-type: text/tab-separated-values; charset=utf-8
# content-disposition: attachment; filename="liftover_target_to_query_....tsv"

head -1 result.tsv   # 首行 TSV 表头
```

token 不存在或结果已过期（受 `LIFTOVER_JOB_TTL_SECONDS` 控制）时
返回 `404`：

```bash
curl -i http://localhost:8080/api/download/not-a-real-token
# HTTP/1.1 404 Not Found
# {"errors": ["结果不存在或已过期，请重新提交转换"]}
```

> 仅当第 3 步返回 `200` 时才会存在可下载结果；任何 `400` 校验失败都不会
> 生成或保留文件。

---

## 五、快速试用

```bash
# 页面上分别选择：
#   BED:   examples/example_target.bed
#   chain: examples/demo.chain
#   侧别:  target 侧
```

该示例覆盖全部四种状态（完全映射 / 跨缺口拆分的部分映射 / 歧义 /
无映射）。`examples/example_query.bed` 演示 query → target，
包含负链 query 的反向转换；`examples/example_bad.bed` 演示错误定位
与导出拦截。

---

## 六、离线部署说明

- 前端资源全部由本服务在 `/static/` 下提供，页面不含任何外部字体、
  脚本或样式表；
- Docker 构建需在有网络的机器上 `docker build`（或导出镜像），之后把
  镜像 `docker save | gzip > liftover-web.tgz` 拷贝到内网，
  `docker load` 后即可运行；
- 容器以非 root 用户运行，只暴露 8080 端口，不挂载任何数据卷；
- 上传文件与结果仅存内存，进程重启即清除，适合处理受限数据。
