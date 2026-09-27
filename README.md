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
┌─────────────────────┐   mode=build                mode=pull
│  获取镜像            │ ┌────────────────────┐  ┌──────────────────────┐
└─────────┬───────────┘ │  docker build       │  │  解析 Compose 文件    │
          ▼             │  --platform ...     │  │  docker pull --platform│
┌─────────────────────┐ │  -t <image_name>    │  │  逐个拉取全部镜像      │
│  打包上传 Artifact   │ └────────────────────┘  └──────────────────────┘
└─────────────────────┘
          │
          ▼
  images-<platform>-<mode>-<run_number>.tgz（可下载，离线部署用）
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
| `mode` | choice | ✅ | `build` | `build`：从 Dockerfile 构建；`pull`：从 Compose 拉取 |
| `dockerfile_path` | string | ❌ | `Dockerfile` | Dockerfile 路径（相对目标仓库根目录），仅 `mode=build` 生效 |
| `compose_file_path` | string | ❌ | `docker-compose.yml` | Compose 文件路径，仅 `mode=pull` 生效 |
| `image_name` | string | ⚠️ 条件必填 | 空 | 镜像名称:tag，如 `myimg:latest`，仅 `mode=build` 时必填 |

**参数校验规则**（校验不通过会立即终止）：

1. `image_name` 必须包含冒号 `:` 且仅一个（格式 `name:tag`）
2. `mode=build` 时 `image_name` 必填；`mode=pull` 时忽略
3. `repo_url` 必须是合法的 `https://github.com/<owner>/<repo>` 地址

### 示例一：Build 模式（amd64）

在目标仓库根目录有 `Dockerfile`，构建为 `myimg:latest`：

```
repo_url         = https://github.com/user/some-repo.git
platform         = linux/amd64
mode             = build
dockerfile_path  = Dockerfile          （可留空使用默认值）
image_name       = myimg:latest
```

### 示例二：Build 模式（arm64，Dockerfile 在子目录）

目标服务器为 ARM 架构（如树莓派、ARM 云主机），Dockerfile 位于 `docker/app/Dockerfile`：

```
repo_url         = https://github.com/user/some-repo.git
platform         = linux/arm64
mode             = build
dockerfile_path  = docker/app/Dockerfile
image_name       = myimg:1.0.0
```

Runner 本身是 x86，workflow 会通过 **QEMU** 自动模拟 ARM 环境完成构建，无需额外配置。

### 示例三：Pull 模式（打包 Compose 全部镜像）

目标仓库有 `docker-compose.yml`（或 `compose.yaml`），需要把其中全部镜像打包离线分发：

```
repo_url            = https://github.com/user/some-repo.git
platform            = linux/amd64
mode                = pull
compose_file_path   = docker-compose.yml （可留空使用默认值）
```

workflow 会：

1. 解析 Compose 文件，列出全部 `services.*.image` 镜像（**不启动任何容器**）
2. 若 Compose 中存在 `build:` 型服务，会提前报错并给出处理建议（pull 模式只负责拉取镜像）
3. 逐个以 `docker pull --platform <platform>` 强制拉取目标架构
4. 将全部镜像一次性 `docker save` 到同一个 TGZ

## 产出物

每次运行产出**一个 TGZ 文件**，命名规范：

```
images-<platform>-<mode>-<run_number>.tgz
例如：images-linux-amd64-build-13.tgz
```

- `build` 模式：包含 1 个镜像
- `pull` 模式：包含 Compose 文件中的全部镜像
- Artifact 保留 14 天；不同运行通过 run_number 区分，互不覆盖
- 打包后会自动**校验每个镜像的架构**与目标平台一致，防止拉错/构建错架构的镜像混入离线包
- Summary 中会记录 TGZ 的 **SHA256** 哈希，供传输后校验完整性

### 下载后在目标服务器导入

从 run 页面 **Artifacts** 区域下载（例如 `images-linux-amd64-build-13`，下载得到的是 zip，**解压后即为 TGZ 文件**）：

```bash
# 0. 校验完整性（可选，SHA256 值见 run 页面的 Summary）
echo "<summary 中的 sha256>  images-linux-amd64-build-13.tgz" | sha256sum -c

# 方式一：gunzip 后导入
gunzip images-linux-amd64-build-13.tgz
docker load -i images-linux-amd64-build-13.tar

# 方式二：一步到位（推荐）
docker load -i <(gunzip -c images-linux-amd64-build-13.tgz)

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

若 pull 模式要拉取的镜像、或 build 模式 Dockerfile 的 `FROM` 基础镜像位于私有镜像仓库，配置以下 secrets（均在 **Settings → Secrets and variables → Actions**）：

| Secret 名 | 必填 | 说明 |
|---|---|---|
| `REGISTRY_SERVER` | 否 | 仓库地址，如 `ghcr.io`、`registry.example.com:5000`，默认 `docker.io` |
| `REGISTRY_USERNAME` | 是 | 仓库用户名 |
| `REGISTRY_PASSWORD` | 是 | 仓库密码或令牌 |

三者（用户名+密码）配置后 workflow 会在拉取/构建前自动 `docker login`，运行结束时自动登出。

## FAQ 与故障排查

**Q: 运行报错 `image_name 必须包含冒号`？**
A: `mode=build` 时 `image_name` 必须写成 `名称:tag` 的完整形式（如 `myimg:latest`），不能只写名称。

**Q: 报错 `克隆失败`？**
A: 依次检查：① 仓库地址拼写是否正确；② 仓库是否真实存在；③ 私有仓库是否已配置 `CLONE_PAT` secret。

**Q: 报错 `未找到 Dockerfile` / `未找到 Compose 文件`？**
A: 路径是相对目标仓库**根目录**的。子目录中的文件需写完整相对路径，如 `docker/app/Dockerfile`。

**Q: arm64 构建特别慢或失败？**
A: arm64 构建依赖 QEMU 模拟，速度约为原生的 1/5~1/10，大型镜像可能超时。可拆分 Dockerfile 层、利用缓存，或增大 workflow 的 `timeout-minutes`。

**Q: pull 模式拉到的镜像架构不对？**
A: 本 workflow 已用 `docker pull --platform` 强制拉取指定架构。若仍报架构错误，说明该镜像本身不支持目标架构（如仅提供 amd64 版本），需联系镜像作者。

**Q: 报错 `解析 Compose 文件失败`？**
A: Compose 文件中引用了未定义的环境变量（如 `${DB_PASSWORD}`）会导致解析失败。目标仓库需自带 `.env` 或提供默认值。

**Q: 报错 `存在 build 型服务`？**
A: Compose 中有服务声明了 `build:` 字段，pull 模式只负责拉取镜像、不执行构建。请在目标仓库为该服务补充 `image:` 字段（指向 registry 中已有的镜像），或改用 build 模式。

**Q: 下载的 Artifact 是 `.zip` 而不是 `.tgz`？**
A: 正常现象。GitHub Actions 下载 Artifact 时会自动套一层 zip，解压 zip 即得到 TGZ 文件。

**Q: 报错 `镜像的架构为 ... 与目标平台不符`？**
A: 打包前会校验每个镜像的架构。若出现该报错，说明镜像本身不支持目标架构（如仅提供 amd64 版本却选择了 arm64），需联系镜像作者或更换镜像。

**Q: pull 模式某个服务没有出现在镜像清单里？**
A: 该服务可能配置在 Compose `profiles` 下且未默认激活。`docker compose config` 默认只解析默认 profile 的服务。

**Q: Artifact 在哪下载？**
A: 运行详情页 → 底部 **Artifacts** 区域，点击文件名即可下载。注意保留期为 14 天，过期自动删除。

## 本仓库结构

```
.
├── .github/
│   └── workflows/
│       └── build-and-package-images.yml   # Workflow 主文件
├── .gitignore
├── DESIGN.md                              # 设计文档
└── README.md
```
