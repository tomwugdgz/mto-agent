# 部署与本地预览

## 一、只想看演示（零安装）

双击 `mto.html`，浏览器打开即可。

没有后端、没有依赖、没有 API Key。所有数据存在浏览器 localStorage，刷新不丢，点「重置演示数据」可恢复。

## 二、跑通可信链的三个动作

| 动作 | 位置 | 你会看到 |
| --- | --- | --- |
| 等 10 秒，或点「注入」 | 监听器页 | 事件命中 → 门禁裁决 → 签发 Mandate → 开独立会话 |
| 点「批准」 | 需要人页 | 证据更新，链上新增 2 块 |
| 点「模拟篡改」再「校验全链」 | 可信链页 | `校验失败：第 N 块 —— 块哈希与内容不符` |

## 三、接真实 Agent 运行时（dsh-superagent）

`runtime/` 目录是 [dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) 的完整源码（MIT），已剔除 `node_modules` 与 `lib`。

### 1. 装运行时

必须是 Node **22.19+** 或 **24+**。22.13 这类早期版本会让 `dsh` 命令静默无输出。

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
dsh --version     # 应打印 0.2.1-rc.3
```

### 2. 构建

```sh
cd runtime
pnpm install
pnpm i -D unrun      # 必需：tsdown 的可选依赖 pnpm 不会自动装
pnpm build           # 产出 lib/
```

### 3. 建 profile 并装插件

```sh
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

### 4. 写机器配置

`~/.dsh/profiles/superagent/cordis.patch.yml`：

```yaml
- id: webserver
  config:
    host: 127.0.0.1
    port: 3082
- id: dsh-superagent
  config:
    stateDir: /home/<你>/.dsh/state/superagent
```

### 5. 启动

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

## 四、三个必须知道的坑

这三条都是在真实环境里踩出来的，不是文档抄的。

| 坑 | 现象 | 原因与解法 |
| --- | --- | --- |
| **`host` 写 `0.0.0.0` 起不来** | `webserver: host "0.0.0.0" is an unspecified (wildcard) address` | webserver 拒绝通配绑定，理由是把 RCE 暴露到网络。必须写具体 IP，如 `127.0.0.1` |
| **`pnpm build` 报 `Failed to import module "unrun"`** | 构建直接失败 | `unrun` 是 tsdown 的可选依赖，pnpm 默认不装。`pnpm i -D unrun` 后重跑 |
| **`--from-default-profile` 会覆盖配置** | 机器配置丢失 | 这条命令只能用一次。profile 已存在时改 `cordis.patch.yml`，不要重跑 |

## 五、还需要四样凭证

服务起来后，管理 API 会报 `repositoryUnset`，runtime 停在 `paused`。面板能开，Agent 不动手。

| 缺什么 | 怎么补 |
| --- | --- |
| 模型 API Key | 面板 → Settings → Model |
| Agent 的 GitHub 账号 | 新建专用账号，给仓库写权限。**不能用 GitHub App**，因为要读该账号的 notifications |
| 该账号的 gh CLI | 见下方命令 |
| 目标仓库与允许名单 | 面板 → Sources 填 `owner/name`，Policy 填 GitHub login |

```sh
mkdir -p ~/.local/bin
printf '#!/bin/sh\nGH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"\n' > ~/.local/bin/gh-agent
chmod +x ~/.local/bin/gh-agent
~/.local/bin/gh-agent auth login
~/.local/bin/gh-agent api user --jq .login
```

验证：在目标仓库开一个 Issue，评论 `@<agent login> <需求>`，约一分钟后面板出现工单。

## 六、实测结果

| 项 | 结果 |
| --- | --- |
| 类型检查 | 通过 |
| 单元测试 | 1196 / 1197（唯一失败为飞书事件消费的时序断言，与 GitHub 主流程无关） |
| 构建 | 通过，`lib/` 约 693 kB |
| 服务启动 | 通过，`dsh web: http://127.0.0.1:3082` |
| 面板响应 | HTTP 200 |

## 七、安全边界

上游 README 自己写明的三条，转述时请保留：

1. 研究原型，非官方产品，未做安全审计。
2. 只有一个 commit，整理后一次性开源，不是长期迭代项目。
3. 配套设计文章里的部分设计**没有在原型里实现**。

还有一条更实际：两个 preset 以 `danger-full-access` 运行，Worker 能按路径执行程序、能读 home 目录。**生产使用必须用独立操作系统账号隔离**，只放 Agent 自己的凭证。
