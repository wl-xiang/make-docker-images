以下是优化后的提示词，已整理为专业的项目执行计划书格式，可直接复制使用：

```
# 项目执行计划书：GitHub Actions 镜像构建与打包 Workflow

## 一、项目背景与目标

构建一个独立的 GitHub Actions Workflow，实现以下完整链路：

  Git Clone 用户仓库 → 构建或拉取 Docker 镜像 → 打包镜像 → 产出 TGZ Artifact

用户通过 workflow_dispatch 手动触发，输入参数化配置，最终产出可下载的镜像压缩包，用于离线部署到目标服务器。

---

## 二、触发方式

- 触发事件：`workflow_dispatch`（手动触发）
- 所有输入参数通过 `inputs` 定义，并配置必要的参数校验逻辑

---

## 三、用户输入参数定义

| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|
| `repo_url` | string | 是 | 无 | GitHub 仓库克隆链接，例如 `https://github.com/wl-xiang/code-server-ai.git` |
| `platform` | choice | 是 | 无 | 目标部署服务器平台，可选值：`linux/amd64` \| `linux/arm64` |
| `mode` | choice | 是 | 无 | 镜像获取模式，可选值：`build`（Build from Dockerfile）\| `pull`（Pull from Compose） |
| `dockerfile_path` | string | 否 | `Dockerfile` | Dockerfile 路径。根目录填 `Dockerfile`；子目录填 `path/to/Dockerfile`。仅 `mode=build` 时生效 |
| `compose_file_path` | string | 否 | `docker-compose.yml` | Docker Compose 文件路径。仅 `mode=pull` 时生效 |
| `image_name` | string | 条件必填 | 无 | 构建产出的镜像名称与 tag，例如 `myimg:latest`。仅 `mode=build` 时必填 |

### 参数校验规则（必须实现）

1. **镜像名格式校验**：`image_name` 必须包含冒号 `:`（即必须显式指定 tag，格式为 `name:tag`），否则立即报错终止 workflow。
2. **模式联动校验**：
   - `mode=build` 时：`image_name` 必填；`dockerfile_path` 为空时使用默认值 `Dockerfile`。
   - `mode=pull` 时：`image_name` 忽略；`compose_file_path` 为空时使用默认值 `docker-compose.yml`。
3. **仓库地址校验**：`repo_url` 必须符合合法的 GitHub 仓库 URL 格式，克隆失败时给出明确错误信息。

---

## 四、Workflow 执行流程

### Step 1：Checkout 本仓库
- 检出 workflow 所在仓库代码

### Step 2：克隆用户目标仓库
- 使用 `git clone --depth 1` 浅克隆 `repo_url` 到工作目录（如 `./target-repo`）
- 克隆失败则打印明确错误并终止

### Step 3：模式 A — Build from Dockerfile（`mode=build`）

1. 进入克隆下来的仓库目录
2. 根据 `platform` 设置 Docker Buildx（`docker/setup-buildx-action`），支持跨平台构建
3. 使用 `docker build` 构建镜像：
   - `-f` 指定 `dockerfile_path`
   - `-t` 指定校验后的 `image_name`
   - `--platform` 指定目标平台
4. 构建成功后通过 `docker save` 导出镜像为 tar 包

### Step 4：模式 B — Pull from Compose（`mode=pull`）

1. 进入克隆下来的仓库目录
2. 根据 `compose_file_path` 启动 Compose 服务（`docker compose up -d` 或 `docker-compose up -d`），使镜像被拉取到本地
   - 若镜像本地不存在，Compose 会自动从 registry 拉取
   - 需处理平台差异：`pull` 模式下平台由镜像 registry 决定；如镜像支持多架构，需配置 `--platform` 强制拉取目标架构
3. 使用 `docker compose images` 或解析 compose 文件，列出本次涉及的所有镜像
4. 通过 `docker save` 将全部镜像一次性导出为单个 tar 包

### Step 5：打包产出 Artifact

1. 将镜像 tar 包压缩为 TGZ 格式，命名建议：`images-<platform>-<mode>-<run_number>.tgz`
2. 使用 `actions/upload-artifact` 上传该 TGZ 文件

---

## 五、Workflow 产出物

- **唯一产出**：一个 TGZ 文件，包含本次 workflow 涉及的全部镜像
  - `build` 模式：1 个镜像
  - `pull` 模式：Compose 文件中定义/拉取的全部镜像
- Artifact 命名规范：`images-<platform>-<mode>`（可附加 run 编号或 commit SHA 以区分版本）
- Artifact 保留期（retention-days）：建议 7~30 天

---

## 六、目标仓库文件结构

```

.
├── .github/
│ └── workflows/
│ └── build-and-package-images.yml # Action YML 主文件
├── .gitignore
└── README.md

```

### 各文件要求

1. **`.github/workflows/build-and-package-images.yml`**
   - 完整的 workflow 定义，包含全部输入参数、校验逻辑、构建/拉取/打包步骤
   - 关键步骤添加注释，逻辑清晰可读

2. **`README.md`**
   - 项目简介与整体流程图/说明
   - 使用教程：如何配置 workflow、各输入参数的详细说明与示例
   - 两种模式（Build / Pull）的使用示例
   - 产出 Artifact 的下载与使用方法（如 `docker load` 导入镜像的命令示例）
   - 常见问题（FAQ）与故障排查指引

3. **`.gitignore`**
   - 忽略常见的临时产物与本地开发文件（如 `*.tgz`、`*.tar`、`target-repo/`、`.DS_Store`、IDE 配置目录等）

---

## 七、技术要求与约束

1. **Runner 环境**：默认使用 `ubuntu-latest`
2. **跨平台支持**：通过 Docker Buildx + QEMU 实现 `linux/arm64` 在非 ARM Runner 上的构建/拉取
3. **错误处理**：每个关键步骤失败时给出清晰的中文/英文错误提示，并正确终止 workflow（非零退出码）
4. **安全性**：不在日志中泄露敏感信息；如仓库为私有仓库，预留 `secrets` 配置位（如 `GITHUB_TOKEN` 或 PAT）并说明用法
5. **幂等性**：workflow 可重复触发执行，产物互不覆盖（通过命名区分）
6. **性能**：克隆使用 `--depth 1` 浅克隆；镜像拉取/构建后及时清理无用缓存（可选 `docker system prune`）

---

## 八、验收标准

- [ ] 三种输入组合（amd64+build、arm64+build、amd64/arm64+pull）均可成功执行
- [ ] `image_name` 缺少冒号时，workflow 明确报错且不继续执行
- [ ] `dockerfile_path` / `compose_file_path` 使用默认值和自定义路径时均能正确工作
- [ ] 最终 Artifact 为单一 TGZ 文件，下载后可通过 `gunzip` + `docker load` 正确导入全部镜像
- [ ] README 内容完整，新用户可按文档独立完成首次配置与运行
```

主要优化点：

1. **结构化**：将原始零散描述整理为背景、参数表、执行流程、产出物、文件结构、验收标准六大模块，AI 执行时不易遗漏。
2. **参数校验显式化**：把"一定要有冒号否则报错"这类隐式要求写成明确的校验规则条目。
3. **补充了模糊地带的处理逻辑**：比如 Pull 模式下平台如何生效（QEMU + 强制 `--platform`）、镜像列表如何获取（`docker compose images`）、Artifact 命名规范等，减少 AI 自由发挥的空间。
4. **增加验收标准 checklist**：让产出结果可逐条验证，质量可控。
5. **明确文件清单与每个文件的内容要求**：确保 README 不只是占位符，而是包含使用教程和 FAQ。
