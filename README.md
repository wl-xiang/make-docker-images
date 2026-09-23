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
4. 运行完成后，在该次运行页面底部的 **Artifacts** 区域下载 TGZ 文件

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

1. 解析 Compose 文件，列出全部 `services.*.image` 镜像
2. 逐个以 `docker pull --platform <platform>` 强制拉取目标架构
3. 将全部镜像一次性 `docker save` 到同一个 TGZ

## 产出物

每次运行产出**一个 TGZ 文件**，命名规范：

```
images-<platform>-<mode>-<run_number>.tgz
例如：images-linux-amd64-build-13.tgz
```

- `build` 模式：包含 1 个镜像
- `pull` 模式：包含 Compose 文件中的全部镜像
- Artifact 保留 14 天；不同运行通过 run_number 区分，互不覆盖

### 下载后在目标服务器导入

```bash
# 方式一：gunzip 后导入
gunzip images-linux-amd64-build-13.tgz
docker load -i images-linux-amd64-build-13.tar

# 方式二：一步到位（推荐）
docker load -i <(gunzip -c images-linux-amd64-build-13.tgz)

# 验证镜像已导入
docker images
```

## 私有仓库支持

若 `repo_url` 指向私有仓库，需配置 PAT：

1. 创建一个具有 `repo` 读权限的 Personal Access Token（GitHub → Settings → Developer settings → Personal access tokens）
2. 在本仓库：**Settings → Secrets and variables → Actions → New repository secret**
3. Name 填 `CLONE_PAT`，Value 填令牌内容

配置后 workflow 会自动检测并使用该令牌克隆，无需修改任何参数。令牌不会出现在日志中（GitHub 自动掩码）。

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
