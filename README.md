# mrs — Microsoft Rewards Script 中国适配版

在低内存 ARM 设备上跑微软奖励脚本，面向国内账号（CN 区域）。

## 架构

```
GitHub Actions  ──build──>  ghcr.io/owlshift/mrs-build:china
                                     │
                          docker compose pull
                                     ↓
                    部署机 (2 containers)
                          ├─ microsoft-rewards-script     账号1
                          └─ microsoft-rewards-script-2   账号2
```

镜像在 CI 构建（runner 有充足临时盘），部署机只 pull —— 目标设备磁盘通常吃紧，
本地 build 会耗尽空间。

**任务只在部署机上跑。** GitHub Actions 只负责构建镜像：会话 cookie
（`sessions/sessions.db`）和账号密码（`.env`）都不进 git，Actions 里也不需要这些凭据。

## 内存约束

目标设备物理内存很小（<1GiB 量级），所以：

| 项 | 值 | 说明 |
|---|---|---|
| `mem_limit` | 700m | 略高于物理内存，超出部分靠 swap |
| `memswap_limit` | 900m | swap 上限 |
| `--max-old-space-size` | 480 | V8 老生代上限，留 ~220MB 给 Chromium |
| `STUCK_PROCESS_TIMEOUT_HOURS` | 4 | 上游默认 8h，会让早档任务活过下一档，两个容器各 700m 撞车 OOMKill |
| `ulimits.core` | 0 | OOM 时曾生成 700MB+ core dump 撑爆系统盘 |

两个账号错峰（见 `compose.yaml` 的 `CRON_SCHEDULE`）也是为了不同时抢内存。

## 改代码的流程

源码在 GitHub，改完推 main，Actions 自动重建镜像：

```bash
git add -A && git commit -m "..." && git push
```

然后在部署机上：

```bash
docker compose pull && docker compose up -d
```

仓库和镜像都是公开的，不需要 ghcr 登录。

## 镜像源

Dockerfile 里 npm 和 apt 的源做成了 build arg，因为两边的最优选择相反：

- 部署机在国内 → 走 `registry.npmmirror.com` + `mirrors.aliyun.com`（Dockerfile 默认值）
- CI runner 在境外 → 官方源更快更稳

Actions 通过仓库变量 `NPM_REGISTRY` / `APT_MIRROR` 传空值覆盖成官方源。
留空即不设置，沿用基础镜像自带的官方源。

## 本地构建

部署机上不 build。真要本地 build 得显式开 profile：

```bash
docker compose --profile build build
```

## 上游

基于 [TheNetsky/microsoft-rewards-script](https://github.com/TheNetsky/microsoft-rewards-script)
v4.3.2.4。国内适配（热搜词源、PushPlus、Server酱、ClawBot 推送）见
`src/functions/QueryEngine.ts` 和 `src/logging/`。GPL-3.0。

## 常见问题

**跑一半 V8 heap OOM。** 上游有个疑似内存泄漏，长跑（数小时）会撞上 `--max-old-space-size`
上限。表现为 `FATAL ERROR: Ineffective mark-compacts near heap limit`。
放宽 `--max-old-space-size` 只能延缓，不能根治；要定位需开
`--heapsnapshot-near-heap-limit=2` 抓快照。

**core dump 撑爆磁盘。** 确认 `ulimits.core: 0` 生效，否则一次 OOM 就能写掉几百 MB。

**两个容器同时跑被 OOMKill。** 错峰被打破时会发生（常见于前一档任务卡死）。
`STUCK_PROCESS_TIMEOUT_HOURS` 就是为此设的。
