---
name: github-mirror-fix
description: "Fix flaky GitHub (China no-proxy) and clean leaked tokens."
version: 1.0.0
author: Hermes Agent (user ops record)
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [github, china, mirror, proxy, git, credentials, security, update]
---

# GitHub 直连不稳修复与 Token 凭据清理

## When to Use

- 用户在中国大陆、无代理，`hermes update` 失败或更新说明/更新日志网页打不开，但普通上网（PowerShell/浏览器访问其它网站）正常
- 发现 GitHub token（`ghp_` 开头）被明文写入 git 凭据文件、笔记或聊天记录，需要清理
- 需要让 git 走镜像加速访问 GitHub（clone/fetch/pull）

## 诊断（先诊断再动手）

1. **确认安装方式与版本**：`hermes --version` — 看 `Install method: git`（git 安装才有此问题；pip/二进制安装的更新不依赖本地 git）
2. **看上次更新是否失败**：读 `$LOCALAPPDATA/hermes/logs/update_receipts/latest.json`（或 `$HERMES_HOME/logs/update_receipts/`），`outcome: "failed"` 或 `exit_code` 非 0 即失败
3. **测 GitHub 连通性**（网页 200 不代表 git/api 稳定）：
   ```bash
   curl -s -o /dev/null -m 12 -w "%{http_code}\n" https://api.github.com/rate_limit
   # 000/超时 = 被墙或瞬断（大陆无代理常见：api.github.com 偶发 000）
   git ls-remote https://github.com/NousResearch/hermes-agent.git HEAD
   # 成功 = git 协议通；配合 mirror 测试镜像可用性
   ```
4. **检查凭据泄露面**：
   ```bash
   git config --global --get-regexp 'credential\.'      # 是否有 credential.helper=store 等
   ls -la ~/.git-credentials                             # 明文凭据文件是否存在
   git config --global --get-regexp 'url\..*insteadOf'  # 镜像配置现状
   ```

## 修复 1：git 走镜像加速（无需代理）

```bash
git config --global url."https://ghfast.top/https://github.com/".insteadOf "https://github.com/"
```

- 原理：`insteadOf` 把 GitHub 地址透明改写为镜像站地址，git fetch/clone/pull 全部走镜像
- **验证**：`git ls-remote https://github.com/NousResearch/hermes-agent.git HEAD` 仍成功即生效；`git remote -v` 显示改写后的地址
- **撤销**：`git config --global --unset url."https://ghfast.top/https://github.com/".insteadOf`

### 备用镜像切换（ghfast.top 失效时）

```bash
git config --global --unset url."https://ghfast.top/https://github.com/".insteadOf
git config --global url."https://ghproxy.net/https://github.com/".insteadOf "https://github.com/"
```

> `insteadOf` 是单一路由、**无自动故障切换**，需手动切换。

### 镜像可用性速查（2026-08 实测）

- ✅ `ghfast.top`、`ghproxy.net`（网页与 git 协议均通过）
- ❌ `gh-proxy.com`（403）、`gh.llkk.cc`、`github.moeyy.xyz`（超时）
- 测试命令：`curl -s -o /dev/null -w "%{http_code}" -m 15 -L "https://<mirror>/https://github.com/NousResearch/hermes-agent/releases/latest"` 和 `git ls-remote "https://<mirror>/https://github.com/NousResearch/hermes-agent.git" HEAD`

## 修复 2：清理泄露的 GitHub Token 凭据

**场景**：`credential.helper=store` 被启用，`~/.git-credentials` 明文存有 `ghp_` token；或 token 被写进 IMA 笔记/聊天。

```bash
# 1. 删除全部 credential 相关配置
git config --global --unset credential.helper
git config --global --unset credential.interactive
git config --global --unset credential.guiprompt
git config --global --unset credential.https://ghfast.top.provider
# 兜底：遍历删除任何残留 credential.*
for key in $(git config --global --get-regexp 'credential\.' 2>/dev/null | cut -d' ' -f1); do
  git config --global --unset "$key"
done

# 2. 删除明文凭据文件
rm -f ~/.git-credentials

# 3. 保留镜像 insteadOf（拉公共仓库无需凭据），验证：
git ls-remote https://github.com/NousResearch/hermes-agent.git HEAD
```

**提醒用户**：
- 到 GitHub → Settings → Developer settings → Personal access tokens **吊销**泄露的 token
- 笔记/知识库里的明文 token 条目需手动删除（IMA OpenAPI 无删除接口，只能客户端删）

## 安全准则（写入笔记时强调）

1. token 绝不写入笔记、聊天记录、明文文件
2. 不用 `credential.helper store` 明文存凭据；临时认证用完即删
3. 公共仓库读操作（hermes update）不需要 token
4. 需要 token 的集成（如 Skills Hub `GITHUB_TOKEN`）放 `$HERMES_HOME/.env`，且不提交入库

## Pitfalls / 坑

- `api.github.com` 时好时坏 ≠ 完全不可用：先测两次再下结论；`curl` 一次 000 可能是瞬断
- `git describe --tags` 报 "No names found" = 本地缺 tag，会导致更新说明链接生成失败；`git fetch origin` 可补拉 tag（走镜像后 fetch 也走镜像）
- hermes update 的 behind-count 走 `api.github.com`，属 best-effort，失败不影响更新主流程
- Windows 上检查用户文件用 `$LOCALAPPDATA`，不是 `~/.hermes`（profile 安装时 home 在 `$LOCALAPPDATA/hermes`）
- 镜像站会变：ghfast.top/ghproxy.net 当前可用，但使用前应实测（见上）

## Verification / 收尾检查

```bash
git config --global --list | grep -vE 'user\.(name|email)'   # 只剩镜像配置 + 无关原有项
test ! -f ~/.git-credentials && echo "凭据文件已清除"
git ls-remote https://github.com/NousResearch/hermes-agent.git HEAD  # 走镜像成功
```
