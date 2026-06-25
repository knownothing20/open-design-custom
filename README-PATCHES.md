# Open Design Custom (my-patches)

这是 [nexu-io/open-design](https://github.com/nexu-io/open-design) 的自定义分支，用于 192.168.31.183 服务器部署。

## 为什么 Fork

1. 需要本地化适配（中国网络环境、内网 API）
2. 需要修改提示词注入逻辑（让 agent 看到 Settings 里的真实配置）
3. 需要将默认图片模型改为 custom-image（避免每次都用 gpt-image-2）

## 补丁列表

### 1. 本地化适配 (`e398efd`)

- Dockerfile: 使用阿里云镜像 + npmmirror 加速
- docker-compose.yml: 使用 1panel 镜像代理，绑定 `0.0.0.0:7456`
- connectionTest.ts: 支持 `OPEN_DESIGN_ALLOW_INTERNAL_API_BASE_URLS` 环境变量

### 2. 提示词实时注入 media provider (`cefbac7`)

**问题**：agent 在提示词里看不到用户在 Settings 里配置的真实模型，总是用默认的 `gpt-image-2`。

**解决**：在组装系统提示词时，实时读取 `media-config.json`，注入 provider 信息。

**改动文件**：
- `apps/daemon/src/server.ts` — 调用 `composeSystemPrompt` 前解析配置
- `apps/daemon/src/prompts/system.ts` — 添加 `mediaProviderInfo` 字段和显示逻辑
- `packages/contracts/src/prompts/system.ts` — 同步修改接口定义

**效果**：

```
# 修改前
- **imageModel**: custom-image

# 修改后
- **imageModel**: custom-image
- **mediaProvider**: Custom Image API · http://192.168.31.183:8787/v1 · model: agnes-image-2.1-flash
```

改 Settings 后下次对话立即生效，不需要重启。

### 3. 默认图片模型改为 custom-image (`3a32767`)

**问题**：新建项目默认用 `gpt-image-2`，每次都要手动改成 custom-image。

**解决**：修改 `apps/web/src/media/models.ts`，将 `custom-image` 设为默认。

**改动**：
- 去掉 `gpt-image-2` 的 `default: true`
- 给 `custom-image` 加 `default: true`

**效果**：新建 image 项目时，默认选中 custom-image，agent 直接用用户在 Settings 里配的端点。

### 4. media-contract 默认模型修复 (`3ee736b`)

**问题**：`media-contract.ts` 第 378 行硬编码了 `gpt-image-2` 作为默认模型，导致 agent 的示例提示词总是用 `gpt-image-2`，即使 Settings 里配了 custom-image 也没用。

**解决**：修改 `apps/daemon/src/prompts/media-contract.ts`，将 `otherwise use \`gpt-image-2\`` 改为 `otherwise use \`custom-image\``。

**改动文件**：
- `apps/daemon/src/prompts/media-contract.ts` — 硬编码默认模型
- `apps/daemon/src/media/models.ts` — daemon 侧 models 默认值
- `apps/web/src/media/models.ts` — 前端侧 models 默认值

**效果**：agent 的示例提示词自动使用 custom-image，不再需要手动覆盖。

### 5. 项目删除按钮 (`d419c68`)

**问题**：OD 没有删除项目的功能，API 有 `DELETE /api/projects/:id` 但 UI 没有暴露。

**解决**：在 RecentProjectsStrip 项目卡片上添加删除按钮。

**改动文件**：
- `apps/web/src/components/RecentProjectsStrip.tsx` — 添加 `onDelete` prop 和删除按钮
- `apps/web/src/styles/home/recent-projects.css` — 删除按钮样式（hover 显示红色 X）
- `apps/web/src/components/HomeView.tsx` — 传递 onDelete 处理函数

**效果**：鼠标 hover 到项目卡片时，右上角出现红色 X 按钮，点击后确认删除。

## 查看补丁

```bash
# 列出所有补丁
git log my-patches --not upstream/main --oneline

# 查看某个补丁的详细改动
git show <commit-hash>
```

## 升级方法

```bash
# 1. 拉取上游最新代码
git fetch upstream

# 2. 把补丁嫁接到最新代码上
git checkout my-patches
git rebase upstream/main

# 3. 如果有冲突，手动解决后：
git add <解决的文件>
git rebase --continue

# 4. 推送到 fork
git push origin my-patches --force-with-lease

# 5. 重建 Docker
cd deploy
docker compose build
docker compose up -d
```

## 当前环境

- 服务器：192.168.31.183 (Debian)
- Docker 容器：`open-design` (端口 7456)
- 数据目录：docker volume `open_design_data`
- LLM Provider：OpenAI 兼容 → 192.168.31.124 (37 个模型)
- Image Provider：Custom Image API → http://192.168.31.183:8787/v1 (agnes)
