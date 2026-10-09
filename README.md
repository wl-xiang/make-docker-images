# make-docker-images

在 GitHub Actions 中构建或拉取 Docker 镜像，并打包为可离线部署的 TGZ 压缩包。

## 整体流程

```
手动触发 (workflow_dispatch)
        │
        ▼
┌─────────────────────┐
│  校验输入参数        │  repo_url 格式 / image_name 必须为 name:tag
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│  浅克隆目标仓库      │  git clone --depth 1（私有仓库支持 CLONE_PAT）
└─────────┬───────────┘
          ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  获取镜像（三选一）                                                          │
│  mode=docker build : docker buildx build --platform ... -t <image_name>    │
│  mode=compose pull : 解析 Compose，逐个 docker pull --platform 全部镜像       │
│  mode=compose auto : 声明 image: 的服务拉取；仅声明 build:（无 image:）的构建  │
└─────────┬─────────────────────────────────────────────────────────────────┘
          ▼
┌─────────────────────┐
│  打包上传 Artifact   │
└─────────────────────┘
          │
          ▼
  <repo-name>_docker-images_<platform>_<yyyymmdd-HHMMSS>.tgz（可下载，离线部署用）
```

## 使用方法

### 前置条件

- 本仓库已推送到 GitHub，workflow 文件位于 `.github/workflows/build-and-package-images.yml`
- 目标镜像仓库可公开访问；若为**私有仓库**，见 [私有仓库支持](#私有仓库支持)

### 运行步骤

1. 进入本仓库的 GitHub 页面 → **Actions** 标签页
2. 左侧选择 **Build and Package Docker Images** workflow
3. 点击 **Run workflow**，填写参数后点击绿色 **Run workflow** 按钮
4. 运行完成后，在该次运行页面底部的 **Artifacts** 区域下载产物（下载得到 zip，解压即为 TGZ 文件）
5. 运行页面的 **Summary** 区域会显示本次运行的摘要：输入参数、产出文件名/大小/SHA256、镜像清单（架构 + 大小）、目标服务器导入命令

### 输入参数说明

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `repo_url` | string | ✅ | 无 | 目标仓库克隆链接，如 `https://github.com/wl-xiang/code-server-ai.git` |
| `platform` | choice | ✅ | `linux/amd64` | 目标部署平台：`linux/amd64` 或 `linux/arm64` |
| `mode` | choice | ✅ | `docker build` | 镜像获取模式，三选一：`docker build` / `compose pull` / `compose auto` |
| `dockerfile_path` | string | ❌ | `Dockerfile` | Dockerfile 路径（相对目标仓库根目录），仅 `mode=docker build` 生效 |
| `compose_file_path` | string | ❌ | `docker-compose.yml` | Compose 文件路径，仅 `mode=compose pull` / `compose auto` 生效 |
| `env_file_dir` | string | ❌ | 空 | 环境变量文件所在目录，留空表示项目根目录（存在 `.env.example` 时自动复制为 `.env`） |
| `force_env` | boolean | ❌ | `false` | 是否强制在目标路径生成一个空的 `.env` 文件（仅当该位置不存在 `.env` 时创建） |
| `env_path` | string | ❌ | 空 | `.env` 文件生成目录（相对目标仓库根目录），留空表示项目根目录；仅 `force_env=true` 时生效 |
| `image_name` | string | ⚠️ 条件必填 | 空 | 镜像名称:tag，如 `myimg:latest`，仅 `mode=docker build` 时使用 |

**模式说明**：

| 模式 | 行为 | 适用场景 |
|---|---|---|
| `docker build` | 用 `docker buildx build` 从单个 Dockerfile 构建 1 个镜像 | 目标仓库有单一 Dockerfile |
| `compose pull` | 解析 Compose 文件，逐个 `docker pull` 其全部镜像；若存在 `build:` 型服务则报错 | Compose 中所有服务都已声明 `image:` |
| `compose auto` | 解析 Compose 文件：声明 `image:` 的服务拉取；仅声明 `build:`（无 `image:`）的服务用 Compose 自动构建 | Compose 中混合了「拉取」与「构建」型服务 |

> `compose auto` 中，构建型镜像由 Compose 按 `<project>-<service>` 命名；项目名优先取 Compose 文件顶层 `name:`，否则取目标仓库名（不会落到克隆目录名 `target-repo`）。

**参数校验规则**（校验不通过会立即终止）：

1. `image_name` 必须包含冒号 `:` 且仅一个（格式 `name:tag`）
2. `mode=docker build` 时 `image_name` 可选（留空默认 `<repo_name>:latest`），填写时须符合 `name:tag`；`compose pull` / `compose auto` 时忽略
3. `repo_url` 必须是合法的 `https://github.com/<owner>/<repo>` 地址
4. 所有路径参数（`dockerfile_path` / `compose_file_path` / `env_file_dir` / `env_path`）必须为相对路径且不得包含 `..`

> GitHub Actions 的 `workflow_dispatch` 不支持按条件隐藏输入框，因此 `force_env` 与 `env_path` 会始终显示；`env_path` 仅在 `force_env=true` 时生效。

### 示例一：docker build 模式（amd64）

在目标仓库根目录有 `Dockerfile`，构建为 `myimg:latest`：

```
repo_url         = https://github.com/user/some-repo.git
platform         = linux/amd64
mode             = docker build
dockerfile_path  = Dockerfile          （可留空使用默认值）
image_name       = myimg:latest
```

### 示例二：docker build 模式（arm64，Dockerfile 在子目录）

目标服务器为 ARM 架构（如树莓派、ARM 云主机），Dockerfile 位于 `docker/app/Dockerfile`：

```
repo_url         = https://github.com/user/some-repo.git
platform         = linux/arm64
mode             = docker build
dockerfile_path  = docker/app/Dockerfile
image_name       = myimg:1.0.0
```

Runner 本身是 x86，workflow 会通过 **QEMU** 自动模拟 ARM 环境完成构建，无需额外配置。

### 示例三：compose pull 模式（打包 Compose 全部镜像）

目标仓库有 `docker-compose.yml`（或 `compose.yaml`），其中所有服务都已声明 `image:`，需要把全部镜像打包离线分发：

```
repo_url            = https://github.com/user/some-repo.git
platform            = linux/amd64
mode                = compose pull
compose_file_path   = docker-compose.yml （可留空使用默认值）
```

workflow 会：

1. 解析 Compose 文件，列出全部 `services.*.image` 镜像（**不启动任何容器**）
2. 若 Compose 中存在 `build:` 型服务，会提前报错并给出处理建议（改用 `compose auto` 或补充 `image:`）
3. 逐个以 `docker pull --platform <platform>` 强制拉取目标架构
4. 将全部镜像一次性 `docker save` 到同一个 TGZ

### 示例四：compose auto 模式（自动区分构建 / 拉取）

目标仓库的 Compose 中既有声明 `image:` 的服务、又有仅声明 `build:` 的服务（典型如：`backend` / `worker` 由源码构建，`db` / `redis` 直接用官方镜像）：

```yaml
services:
  backend:
    build: ./backend          # 仅 build，无 image → 自动构建
  worker:
    build: ./worker           # 仅 build，无 image → 自动构建
  db:
    image: postgres:16        # 有 image → 直接拉取
  redis:
    image: redis:7            # 有 image → 直接拉取
```

```
repo_url            = https://github.com/user/some-repo.git
platform            = linux/amd64
mode                = compose auto
compose_file_path   = docker-compose.yml （可留空使用默认值）
```

workflow 会：

1. 解析 Compose 文件，把服务分为两类：声明 `image:` 的（拉取）与仅声明 `build:` 的（构建）
2. 对拉取型服务逐个 `docker pull --platform <platform>`
3. 对构建型服务执行 `docker compose build`（Compose 自动处理 `context` / `dockerfile` / `args` / `target`），并按 `<project>-<service>` 命名
4. 合并两类镜像，一次性 `docker save` 到同一个 TGZ

> 若目标服务器在 `docker compose up` 时要复用这些构建型镜像，请确保其 Compose 项目名与本 workflow 一致（本 workflow 在 Compose 文件未声明顶层 `name:` 时使用目标仓库名作为项目名）。

### 示例五：强制生成空 `.env`

某些项目在 `docker compose config` 解析时要求存在 `.env`（或构建阶段会读取它），而仓库里既没有 `.env` 也没有 `.env.example`。此时可强制生成一个空的 `.env`：

```
repo_url   = https://github.com/user/some-repo.git
platform   = linux/amd64
mode       = compose auto
force_env  = true
env_path   =            （留空表示目标仓库根目录；也可填子目录，如 app）
```

`force_env=true` 时，会在 `env_path`（留空为项目根目录）生成一个空的 `.env`；若该位置已存在 `.env` 则跳过、不覆盖。

## 产出物

每次运行产出**一个 TGZ 文件**，命名规范：

```
<repo-name>_docker-images_<platform>_<yyyymmdd-HHMMSS>.tgz
例如：code-server-ai_docker-images_linux-amd64_20261009-153045.tgz
```

- `<repo-name>`：取自 `repo_url` 的项目名（如 `https://github.com/wl-xiang/code-server-ai.git` → `code-server-ai`）
- `<platform>`：目标平台，`/` 替换为 `-`（如 `linux/amd64` → `linux-amd64`）
- `<yyyymmdd-HHMMSS>`：打包时的北京时间戳
- `docker build` 模式：包含 1 个镜像
- `compose pull` 模式：包含 Compose 文件中声明的全部 `image:` 镜像
- `compose auto` 模式：包含 Compose 中拉取的镜像 + 构建型服务（`<project>-<service>`）的镜像
- Artifact 保留 14 天；不同运行通过时间戳区分，互不覆盖
- 打包后会自动**校验每个镜像的架构**与目标平台一致，防止拉错/构建错架构的镜像混入离线包
- Summary 中会记录 TGZ 的 **SHA256** 哈希，供传输后校验完整性

### 下载后在目标服务器导入

从 run 页面 **Artifacts** 区域下载（例如 `code-server-ai_docker-images_linux-amd64_20261009-153045`，下载得到的是 zip，**解压后即为 TGZ 文件**）：

```bash
# 0. 校验完整性（可选，SHA256 值见 run 页面的 Summary）
echo "<summary 中的 sha256>  code-server-ai_docker-images_linux-amd64_20261009-153045.tgz" | sha256sum -c

# 方式一：gunzip 后导入
gunzip code-server-ai_docker-images_linux-amd64_20261009-153045.tgz
docker load -i code-server-ai_docker-images_linux-amd64_20261009-153045.tar

# 方式二：一步到位（推荐）
docker load -i <(gunzip -c code-server-ai_docker-images_linux-amd64_20261009-153045.tgz)

# 验证镜像已导入
docker images
```

## 私有仓库支持

### 私有 Git 仓库（克隆目标仓库）

若 `repo_url` 指向私有仓库，需配置 PAT：

1. 创建一个具有 `repo` 读权限的 Personal Access Token（GitHub → Settings → Developer settings → Personal access tokens）
2. 在本仓库：**Settings → Secrets and variables → Actions → New repository secret**
3. Name 填 `CLONE_PAT`，Value 填令牌内容

配置后 workflow 会自动检测并使用该令牌克隆，无需修改任何参数。令牌不会出现在日志中（GitHub 自动掩码）。

### 私有镜像仓库（拉取/构建私有镜像）

若 `compose pull` / `compose auto` 模式要拉取的镜像、或 `docker build` / `compose auto` 模式构建时 `FROM` 的基础镜像位于私有镜像仓库，配置以下 secrets（均在 **Settings → Secrets and variables → Actions**）：

| Secret 名 | 必填 | 说明 |
|---|---|---|
| `REGISTRY_SERVER` | 否 | 仓库地址，如 `ghcr.io`、`registry.example.com:5000`，默认 `docker.io` |
| `REGISTRY_USERNAME` | 是 | 仓库用户名 |
| `REGISTRY_PASSWORD` | 是 | 仓库密码或令牌 |

三者（用户名+密码）配置后 workflow 会在拉取/构建前自动 `docker login`，运行结束时自动登出。

## FAQ 与故障排查

**Q: 运行报错 `image_name 必须包含冒号`？**
A: `mode=docker build` 时若填写了 `image_name`，必须写成 `名称:tag` 的完整形式（如 `myimg:latest`），不能只写名称；留空则默认 `<repo_name>:latest`。

**Q: 报错 `克隆失败`？**
A: 依次检查：① 仓库地址拼写是否正确；② 仓库是否真实存在；③ 私有仓库是否已配置 `CLONE_PAT` secret。

**Q: 报错 `未找到 Dockerfile` / `未找到 Compose 文件`？**
A: 路径是相对目标仓库**根目录**的。子目录中的文件需写完整相对路径，如 `docker/app/Dockerfile`。

**Q: arm64 构建特别慢或失败？**
A: arm64 构建依赖 QEMU 模拟，速度约为原生的 1/5~1/10，大型镜像可能超时。可拆分 Dockerfile 层、利用缓存，或增大 workflow 的 `timeout-minutes`。

**Q: 拉到的镜像架构不对？**
A: 本 workflow 已用 `docker pull --platform` 强制拉取指定架构。若仍报架构错误，说明该镜像本身不支持目标架构（如仅提供 amd64 版本），需联系镜像作者。

**Q: 报错 `解析 Compose 文件失败`？**
A: Compose 文件中引用了未定义的环境变量（如 `${DB_PASSWORD}`）会导致解析失败。目标仓库需自带 `.env`、提供默认值，或开启 `force_env=true` 生成空 `.env`。

**Q: 报错 `存在 build 型服务`？**
A: 该报错来自 `compose pull` 模式：Compose 中有服务声明了 `build:` 字段，而 `compose pull` 只负责拉取镜像、不执行构建。请任选其一：① 改用 `compose auto` 模式（声明 `image:` 的服务拉取、仅声明 `build:` 的服务自动构建）；② 为这些服务补充 `image:` 字段（指向 registry 中已有的镜像）；③ 改用 `docker build` 模式（单一 Dockerfile）。

**Q: `compose auto` 构建出来的镜像叫什么名字？导入后能直接被 Compose 使用吗？**
A: 构建型镜像由 Compose 按 `<project>-<service>` 命名。若 Compose 文件声明了顶层 `name:` 则以它为准，否则本 workflow 使用目标仓库名作为项目名。请确保目标服务器上 `docker compose up` 的项目名与之相同，导入的镜像才能被直接复用（否则 Compose 会尝试重新构建/拉取）。

**Q: `force_env=true` 但目标目录已有 `.env`？**
A: 会跳过、不覆盖已有 `.env`，只在目标位置不存在 `.env` 时生成一个空文件。

**Q: 下载的 Artifact 是 `.zip` 而不是 `.tgz`？**
A: 正常现象。GitHub Actions 下载 Artifact 时会自动套一层 zip，解压 zip 即得到 TGZ 文件。

**Q: 报错 `镜像的架构为 ... 与目标平台不符`？**
A: 打包前会校验每个镜像的架构。若出现该报错，说明镜像本身不支持目标架构（如仅提供 amd64 版本却选择了 arm64），需联系镜像作者或更换镜像。

**Q: `compose pull` / `compose auto` 某个服务没有出现在镜像清单里？**
A: 该服务可能配置在 Compose `profiles` 下且未默认激活。`docker compose config` 默认只解析默认 profile 的服务。

**Q: Artifact 在哪下载？**
A: 运行详情页 → 底部 **Artifacts** 区域，点击文件名即可下载。注意保留期为 14 天，过期自动删除。

## 运行历史清理 Workflow

本仓库另提供一个**仅手动触发**的 workflow —— **Clean Workflow Run History**（[.github/workflows/cleanup-run-history.yml](.github/workflows/cleanup-run-history.yml)），用于按规则自动清理本仓库 workflow 的运行历史（删除整条 run，含其日志与 Artifact），释放 Actions 存储空间。

### 使用步骤

1. 进入本仓库的 GitHub 页面 → **Actions** 标签页
2. 左侧选择 **Clean Workflow Run History**
3. 点击 **Run workflow**，填写参数后运行
4. **默认处于「预演模式」**：只打印将被删除的记录、不真正删除。先看运行页面的 **Summary** 确认清单无误
5. 确认后再次触发，将 `dry_run` 设为 `false`，才会真正删除

### 输入参数说明

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `clean_by` | choice | `num` | 清理依据：`num` = 按保留条数；`date` = 按保留天数 |
| `keep_count` | string | `10` | 保留最近 N 条运行记录（仅 `clean_by=num` 生效），须为正整数 |
| `keep_days` | string | `30` | 保留最近 N 天内「已完成」的运行记录，更早的删除（仅 `clean_by=date` 生效），须为正整数 |
| `dry_run` | boolean | `true` | 预演模式：`true` 只打印将删除的记录、不删除；`false` 真正删除 |

### 清理规则

- **`clean_by=num`**：按创建时间倒序，保留最近 `keep_count` 条记录；更早且状态为 `completed` 的记录被删除
- **`clean_by=date`**：删除状态为 `completed` 且完成时间（`updated_at`）早于「当前时间 − `keep_days` 天」的记录

### 安全策略

- **永不删除** 处于 `queued` / `in_progress` 的记录（含当前运行）
- **永不删除本 workflow 自身**的运行记录（也不参与保留计数）
- 默认 `dry_run=true`，首次运行零风险
- 删除**不可恢复**：删除 run 会同时删除其日志与 Artifact

### 前置条件

- 需要 `GITHUB_TOKEN` 具备 `actions: write` 权限（workflow 已声明 `permissions: actions: write`）。若组织/仓库限制了默认令牌权限，需在 **Settings → Actions → General → Workflow permissions** 中允许写入，或改用具备 `actions: write` 的 PAT
- 依赖 runner 预装的 `gh` CLI（`ubuntu-latest` 自带，无需额外安装）

### 用法示例

**示例一：保留最近 10 条（默认，先预演）**

```
clean_by    = num
keep_count  = 10
dry_run     = true       # 预演：只输出将删除的清单
```

确认清单无误后，将 `dry_run` 改为 `false` 重新触发即可真正清理。

**示例二：只保留最近 30 天内完成的历史**

```
clean_by    = date
keep_days   = 30
dry_run     = false
```

## 本仓库结构

```
.
├── .github/
│   └── workflows/
│       ├── build-and-package-images.yml   # 镜像构建打包 Workflow 主文件
│       └── cleanup-run-history.yml        # 运行历史清理 Workflow（手动触发）
├── .gitignore
├── DESIGN.md                              # 设计文档
└── README.md
```
