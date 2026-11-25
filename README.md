# GitHub MCP

为 [DeepSeek Harness](https://deepseek.com) 接入 **GitHub 官方远程 MCP 服务器**(`https://api.githubcopilot.com/mcp/`),让助手用你的 GitHub 账号完成日常开发工作:创建和管理 Issue(提问题)、查看与评审 Pull Request、搜索仓库与代码、读取文件内容等,共 **46 个工具**。

配置迁移自 Claude Code 的 `mcpServers.github`,工具命名与 Claude Code 完全一致(`mcp__github__<tool>`),权限规则和使用习惯可以直接延续。

## 功能一览

| 能力 | 示例工具 |
|---|---|
| Issue(提问题) | `create_issue`、`issue_read`、`search_issues`、`add_issue_comment`、`sub_issue_write` |
| Pull Request | `list_pull_requests`、`pull_request_read`、`merge_pull_request`、`create_pull_request`、`update_pull_request_branch` |
| 仓库与代码 | `search_repositories`、`search_code`、`get_file_contents`、`list_commits`、`get_commit` |
| 发布与标签 | `list_releases`、`get_latest_release`、`list_tags`、`get_tag` |
| 协作 | `get_me`、`search_users`、`list_repository_collaborators`、`get_teams` |
| 安全 | `run_secret_scanning` |

安装后,在对话里直接说「帮我看一下 `owner/repo` 的 open issue」即可使用。

## 工作原理

本插件是一个 DSH bundle:其 `cordis.patch.yml` 向配置中插入一条 `@deepseek-ai/dsh-mcp-client` 插件行,以 **streamable-http** 传输连接 GitHub 远程 MCP 服务器。服务器上的工具以 `mcp__github__<tool>` 的稳定名称注册到助手,例如 `mcp__github__search_issues`。

GitHub 访问令牌**不写在本插件里**:补丁使用 `${credential:GITHUB_PAT}` 模板,运行时从 DSH 凭据存储解析,并自动从诊断信息中遮蔽。

## 前置要求

- DeepSeek Harness `>=0.2.2-rc.0`(Desktop 或其他带配置文件的 Host)。
- 一个 GitHub 账号,以及一个 **Personal Access Token(PAT)**。
- 网络可达 `https://api.githubcopilot.com/mcp/`(国内网络可直连,无需加速)。

## 安装

### 方式一:插件广场(推荐)

1. 打开侧边栏 **插件 → 发现**。
2. 首次使用点击 **添加索引源**,粘贴本仓库的索引地址:
   `https://raw.githubusercontent.com/allon2/dsh-github-mcp/main/index.json`
3. 找到 **GitHub MCP**,点击 **安装**,确认后完成。

### 方式二:本地目录安装

把本仓库克隆到本地,然后在 DSH 的插件页或 `plugin_manager` 工具中以目录安装:

```
<E:\你的路径>\dsh-github-mcp
```

`plugin_manager`(`install_bundle`,target 填上面的绝对路径)会把它登记进 profile 并立即启用。

## 配置 GitHub 凭据

1. 在 GitHub 上创建 PAT:**Settings → Developer settings → Personal access tokens**;按需选择 classic(勾 `repo` 等)或 fine-grained(选择仓库并授予 Issues、Pull requests、Contents 等读权限,需要写操作再放宽)。
2. 把它存入 DSH 凭据存储,引用名必须是 **`GITHUB_PAT`**,两种方式任选:
   - **DSH 设置 → 凭据**,添加 `GITHUB_PAT`;
   - 直接编辑 `~/.dsh/.credentials.yaml`:
     ```yaml
     version: 1
     refs:
       GITHUB_PAT: ghp_xxxxxxxxxxxxxxxxxxxxxxxx
     ```
     保存后自动生效。

没存凭据时,MCP 连接会以明确的错误失败(`credential "GITHUB_PAT" is not configured`),补上即可恢复。

## 配置参考

`cordis.patch.yml` 中可调整的字段(`@deepseek-ai/dsh-mcp-client` 的配置):

| 字段 | 本插件默认 | 说明 |
|---|---|---|
| `serverName` | `github` | 工具命名空间,决定 `mcp__github__<tool>`;改名会变更所有工具名 |
| `transport` | `streamable-http` | GitHub 远程服务器的传输方式 |
| `url` | `https://api.githubcopilot.com/mcp/` | 服务器端点 |
| `headers` | `Authorization: Bearer ${credential:GITHUB_PAT}` | 鉴权头,凭据运行时解析 |
| `expandTemplates` | `true` | 允许 `${credential:…}` 模板 |
| `startupTimeoutMs` | `60000` | 首次连接与工具同步的预算;官方服务器生成完整工具列表约需 10–15 秒(网络波动时更久),默认 30 秒偏紧,故设 60 秒 |
| `failOnStartupError` | `true` | 连接失败时拒绝插件激活(尽早暴露凭据/网络问题) |
| `toolCallTimeoutMs` | `60000`(客户端默认) | 单次工具调用超时 |
| `reconnect.*` | 客户端默认 | 断线自动重连(最多 10 次,指数退避) |

## 验证安装

安装并配置凭据后,在对话中问一句:

> 用 GitHub 工具查一下我的账号信息。

助手会调用 `mcp__github__get_me` 并返回你的 GitHub 登录名。

## 与 Claude Code 的关系

- 本插件是 **MCP 服务器插件**(Issue、PR、仓库等 GitHub 工具),不是网络加速工具;与 `github-cn` 之类的访问加速**技能**互不冲突、互不替代。
- 配置来源:Claude Code 的 `C:\Users\<你>\.claude.json` 中 `mcpServers.github` 的 `url` 与鉴权头;工具名(`mcp__github__…`)与 Claude Code 相同。

## 故障排查

| 现象 | 处理 |
|---|---|
| `credential "GITHUB_PAT" is not configured` | 按「配置 GitHub 凭据」一节保存凭据 |
| 401 / Unauthorized | PAT 失效或权限不足;重新生成并更新凭据存储 |
| 启动即报错、没有 `mcp__github__*` 工具 | 本插件 `failOnStartupError: true`:检查网络能否直连 `api.githubcopilot.com`,凭据是否已保存 |
| 日志出现 `startup timed out` | 网络到 GitHub 较慢,工具列表同步没在预算内完成;把 `startupTimeoutMs` 调大(如 `120000`)后重启会话重试 |
| 想临时禁用 | 插件页中关掉 GitHub MCP 的 `mcp-github` 行即可,不影响其他插件 |

## 安全说明

- 令牌只保存在本机凭据存储(`~/.dsh/.credentials.yaml`),除发往 GitHub 的请求外不外传;本仓库与已发布的索引均**不含**任何令牌。
- 更换 PAT 后,凭据存储立即生效,无需重装插件。

## 许可证

[MIT](LICENSE)
