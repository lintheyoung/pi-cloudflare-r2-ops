# Cloudflare R2 DevOps SOP

本仓库为 PestGG 团队提供 Cloudflare R2 运维的标准操作流程（SOP），以 Pi Package 形式分发，团队成员安装后可通过自然语言指令让 Pi 安全地管理 R2 存储桶和对象。

---

## 1. 前置条件

### 1.1 Cloudflare 账号信息

| 项目 | 值 |
|------|-----|
| 登录邮箱 | `ldyisme@outlook.com` |
| Account ID | `bf7302689d0dd0a365e5199aee2d3192` |
| 套餐 | Free Website + R2（按量计费） |

### 1.3 API Token 权限要求

Token 必须包含以下权限：

- `Account:Read` – 读取账号信息
- `R2 Bucket:Read` – 读取存储桶列表
- `R2 Bucket:Edit` – 创建/删除存储桶（如需变更）
- `R2 Object:Read` – 读取对象列表
- `R2 Object:Edit` – 上传/删除对象（如需变更）

**注意**：现有的 DNS-only token 缺少 R2 权限，调用 R2 API 会返回 `Authentication error`。需要前往 Cloudflare Dashboard 生成新的 token。

Token 格式：`cfut_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

**安全提示**：Token 不要提交到 Git，不要暴露在聊天输出中。

---

## 2. 环境配置（一次性）

将以下环境变量添加到 shell profile（`~/.zshrc` 或 `~/.bashrc`）：

```bash
# Cloudflare
export CLOUDFLARE_API_TOKEN="cfut_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export CLOUDFLARE_ACCOUNT_ID="bf7302689d0dd0a365e5199aee2d3192"
```

然后激活：

```bash
source ~/.zshrc
```

验证 Token（通用权限）：

```bash
curl -s -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | python3 -m json.tool
```

验证 R2 权限：

```bash
curl -s -X GET "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/r2/buckets" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" | python3 -m json.tool
```

如果返回 `Authentication error`，说明 Token 缺少 R2 权限，需要重新生成。

---

## 3. 安装 Pi Package

### 3.1 方式 A：全局安装（个人开发机）

```bash
pi install git:github.com/lintheyoung/pi-cloudflare-r2-ops
```

### 3.2 方式 B：项目本地安装（团队共享，推荐）

在项目根目录执行：

```bash
pi install -l git:github.com/lintheyoung/pi-cloudflare-r2-ops
```

这会在 `.pi/settings.json` 中写入 package 引用，随 Git 仓库共享给团队。

### 3.3 验证安装

```bash
# 确认 settings.json 包含本 package
cat .pi/settings.json
```

预期：

```json
{
  "packages": [
    "git:github.com/lintheyoung/pi-cloudflare-r2-ops"
  ]
}
```

---

## 4. 使用方法

安装后，在 Pi 中直接使用自然语言指令。Pi 会自动加载本 package 的 **skill**，遵循安全 guardrails。

### 4.1 查询操作

> "列出所有 R2 存储桶"
> "查看 my-bucket 里有哪些文件"
> "my-bucket 的大小和对象数量是多少"

### 4.2 创建存储桶

> "创建一个名为 backups 的 R2 存储桶"

Pi 会在执行前展示变更内容并要求你确认（**强制确认门控**）。

### 4.3 删除存储桶

> "删除 test-bucket 存储桶"

**注意**：
- Pi 会强制要求你输入 `yes` 确认
- 存储桶必须为空才能删除
- 如果你输入其他内容，操作会被取消

### 4.4 删除对象

> "删除 my-bucket 里的 old-file.zip"

---

## 5. 安全规范

1. **Token 保密**：不在聊天、日志、Git 提交中暴露完整 Token。
2. **强制确认**：任何创建/删除存储桶、删除对象操作必须先展示变更摘要，获得显式 `yes` 确认后才执行。
3. **结果验证**：API 返回必须包含 `"success": true`，否则视为失败并报告错误。
4. **审计记录**：在对话中保留变更历史，便于追溯。
5. **权限最小化**：Token 仅授予必要的读取和编辑权限。
6. **空桶删除**：删除存储桶前确认其为空，否则 API 会失败。

---

## 6. 故障排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `Authentication error` (10000) | Token 缺少 R2 权限 | 前往 Cloudflare Dashboard > API Tokens，生成包含 R2 权限的新 token |
| `Bucket not empty` | 删除非空存储桶 | 先删除桶内所有对象，再删除存储桶 |
| `Bucket already exists` | 全局命名冲突 | R2 存储桶名全局唯一，换一个名字 |
| `Invalid bucket name` | 命名不符合规则 | 仅使用小写字母、数字和连字符 |
| Pi 未加载 skill | `.pi/settings.json` 未包含 package | 执行安装命令 |
| 环境变量为空 | 未 source shell profile | 执行 `source ~/.zshrc` |
| Pi 直接执行未确认 | Skill 文件缺少 HARD-GATE | 确保 `skills/*/SKILL.md` 包含确认门控 |

---

## 7. 资源文件说明

| 文件 | 给谁看 | 加载方式 | 作用 |
|------|--------|----------|------|
| `README.md` | 人类 | 手动阅读 | 完整的 SOP 文档，环境配置、使用方法、故障排查 |
| `skills/cloudflare-r2/SKILL.md` | **AI (Pi)** | **自动注入** | **核心文件**：R2 API 工作流、命令模板、**强制确认门控** |
| `prompts/r2-guardrails.md` | AI (Pi) | 手动触发 (`/r2-guardrails`) | 可选的行为约束 |
| `package.json` | Pi 系统 | 自动读取 | Package 元数据 |

### ⚠️ Skill vs Prompt 的关键区别

| | Skill (`skills/*/SKILL.md`) | Prompt (`prompts/*.md`) |
|--|------------------------------|-------------------------|
| **加载方式** | **自动注入**到 Pi 系统提示中 | 需要手动输入 `/name` 触发 |
| **触发条件** | 用户输入与 skill `description` 匹配时 | 用户输入 `/name` 时 |
| **用途** | 定义工作流、API 命令、**强制规则** | 可选的额外约束或快捷模板 |
| **确认门控** | **必须放在这里** | 不适合放强制规则 |

---

## 8. 更新维护

当 R2 API 变更或团队需求变化时：

1. 更新本仓库的 `skills/cloudflare-r2/SKILL.md`
2. 更新本 README.md 的相关章节
3. 提交并推送：`git push`
4. 团队成员的 Pi 会在下次启动时自动拉取更新

---

**仓库**: https://github.com/lintheyoung/pi-cloudflare-r2-ops
