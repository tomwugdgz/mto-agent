# dsh-superagent 部署记录

仓库：`tomwugdgz/dsh-superagent-prototype`（fork 自 deepseek-ai，MIT，1 commit）
部署时间：2026-11-02
执行环境：Ubuntu 22.04 容器 / Node 24.11.1（`/opt/node-v24.11.1-linux-x64`）

## 一、已完成

| 步骤 | 结果 | 说明 |
| --- | --- | --- |
| 克隆仓库 | 完成 | `/workspace/dsh-superagent-prototype` |
| 安装依赖 | 完成 | pnpm 11.7.0，锁定版本 |
| 类型检查 | 通过 | `tsc --noEmit` 无错误 |
| 单元测试 | 1196 / 1197 通过 | 唯一失败：`tests/sources.spec.ts` 飞书事件消费的时序断言，与 GitHub 主流程无关 |
| 构建 | 完成 | `lib/`，ESM 约 693 kB |
| 安装 DSH 运行时 | 完成 | `@deepseek-ai/dsh@0.2.1-rc.3` 全局安装 |
| 创建 profile | 完成 | `~/.dsh/profiles/superagent`（从 web 模板复制） |
| 装入 bundle | 完成 | `dsh plugin --profile superagent add link:$PWD` |
| 写机器配置 | 完成 | `stateDir=/root/.dsh/state/superagent`，端口 3082 |
| 启动服务 | 运行中 | PID 578332，`dsh web: http://127.0.0.1:3082` |
| 面板可访问 | 200 | 带 token 打开，标题 `DeepSeek Harness` |

构建期有一个环境坑：`tsdown` 依赖的可选包 `unrun` 未随 pnpm 一起装上，导致 `pnpm build` 报 `Failed to import module "unrun"`。补装后构建正常，`package.json` 已还原，工作区 `git status` 干净。

## 二、当前阻塞

管理 API 的 setup 状态：

```json
{"place":"github","code":"repositoryUnset","detail":"The GitHub Source names no repository."}
```

runtime 因此处于 `paused`。面板能开，Agent 不会动手。缺的不是代码，是四类凭证：

| 缺什么 | 为什么必须有 | 你要做的 |
| --- | --- | --- |
| 模型 API Key | DSH 的所有 Session 都要一个 model provider | 面板 → Settings → Model，填 DeepSeek（或其他）Key |
| Agent 的 GitHub 账号 | Agent 以这个身份评论、推分支、开 PR。不能用 GitHub App，因为要读该账号的 notifications | 新建一个专用账号，给它仓库写权限 |
| 该账号的 gh CLI | 源码只认 `gh` / `lark-cli` 的输出格式 | 按下方命令登录，作为 Source 的 `cli` 字段 |
| 目标仓库与允许名单 | `owner/name` 与「谁可以派活」 | 面板 → Sources 填仓库，Policy 填 GitHub login |

## 三、剩余三步（你在自己机器上执行）

### 1. Agent 账号的 gh CLI

```sh
mkdir -p ~/.local/bin
printf '#!/bin/sh\nGH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"\n' > ~/.local/bin/gh-agent
chmod +x ~/.local/bin/gh-agent
~/.local/bin/gh-agent auth login          # 登录 agent 账号，不是你的账号
~/.local/bin/gh-agent api user --jq .login
```

### 2. 面板里填三项

| 页面 | 字段 | 填什么 |
| --- | --- | --- |
| Sources → GitHub | repository | `owner/name` |
| Sources → GitHub | cli | `/home/<你>/.local/bin/gh-agent` 绝对路径 |
| Policy | 允许名单 | 可以派活的 GitHub login，先只填你自己 |

保存会触发布置检查。检查通过后 runtime 从 `paused` 转 `ready`。

### 3. 验证

在目标仓库开一个 Issue，或评论一句：

```
@<agent login> 把 README 的安装段落改成 Node 24
```

约一分钟后 Overview 出现工单。Agent 在自己的 Session 和目录里干活，回到原对话交付 PR。

## 四、从零复现（换机器时照抄）

```sh
# 1. 运行时：必须是 Node 22.19+ 或 24+
npm install -g @deepseek-ai/dsh@0.2.1-rc.3

# 2. 取代码并构建
git clone https://github.com/tomwugdgz/dsh-superagent-prototype.git dsh-superagent
cd dsh-superagent && pnpm install && pnpm i -D unrun && pnpm build

# 3. 建 profile 并装插件
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"

# 4. 写机器配置
cat > ~/.dsh/profiles/superagent/cordis.patch.yml <<'YAML'
- id: webserver
  config:
    host: 127.0.0.1
    port: 3082
- id: dsh-superagent
  config:
    stateDir: /home/<你>/.dsh/state/superagent
YAML

# 5. 起服务
scripts/dsh-superagent start
scripts/dsh-superagent status
```

三个注意点：

1. `host` 不能写 `0.0.0.0`。webserver 会拒绝通配地址，理由是「绑定所有网卡等于把远程代码执行暴露到网络」。写具体 IP。
2. `scripts/dsh-superagent start` 依赖 tmux。没有 tmux 就直接 `dsh --profile superagent --no-open &`。
3. `dsh superagent --from-default-profile web` 只能用一次。profile 已存在时再跑会覆盖 `cordis.patch.yml`，机器配置全丢。

## 五、README 自己划的三条线

部署之前先认这三条，能省掉误判：

1. 它是研究原型，不是 DeepSeek 官方产品，没有安全审计。
2. 只有一个 commit，是整理后一次性开源，不是长期迭代的项目。
3. 配套设计文章里的 Router + Worker、门禁、Effects 队列，有一部分**没有在这个原型里实现**。读架构图时别当产品说明书。

还有一条更实际：两个 preset 都以 `danger-full-access` 运行，Worker 能按路径执行程序、能读你的 home 目录。官方建议用独立操作系统账号隔离，只放 Agent 自己的凭证。
