# mrs — Microsoft Rewards Script 中国适配版

在 785MiB 内存的 ARM 板子（armbian）上跑微软奖励脚本，国内账号（QQ 邮箱 + CN 区域）。

## 架构

```
GitHub Actions  ──build──>  ghcr.io/owlshift/mrs-build:china
                                     │
                          docker compose pull
                                     ↓
                        armbian (2 containers)
                          ├─ microsoft-rewards-script    07:00  账号1
                          └─ microsoft-rewards-script-2  14:00  账号2
```

镜像在 CI 构建（runner 有 14GB 临时盘），板子只 pull —— 板子磁盘常年 80%+，
本地 build 会耗尽空间。

**任务只在 armbian 上跑。** GitHub Actions 只负责构建镜像：会话 cookie
（`sessions/sessions.db`）和账号密码（`.env`）都不进 git，Actions 里也没有这两个账号。

## 内存约束

板子物理内存 785MiB，所以：

| 项 | 值 | 说明 |
|---|---|---|
| `mem_limit` | 700m | 略高于物理内存，超出部分靠 swap |
| `memswap_limit` | 900m | swap 上限 |
| `--max-old-space-size` | 480 | V8 老生代上限，留 ~220MB 给 Chromium |
| `STUCK_PROCESS_TIMEOUT_HOURS` | 4 | 上游默认 8h，会让 07:00 的任务活过 14:00，两个容器各 700m 撞车 OOMKill |
| `ulimits.core` | 0 | OOM 时曾生成 734MB core dump 撑爆 6.4G 系统盘 |

两个账号错峰（07:00 / 14:00）也是为了不同时抢内存。

## 改代码的流程

源码在 GitHub，改完推 main，Actions 自动重建镜像：

```bash
git add -A && git commit -m "..." && git push
```

然后在 armbian 上：

```bash
cd /root/microsoft-rewards-script
docker compose pull && docker compose up -d
```

需要登录 ghcr（仓库是私有的）：

```bash
echo "$GHCR_PAT" | docker login ghcr.io -u owlshift --password-stdin
```

PAT 需要 `write:packages` scope。

## 镜像源

Dockerfile 里 npm 和 apt 的源做成了 build arg，因为两边的最优选择相反：

- armbian 在国内 → 必须走 `registry.npmmirror.com` + `mirrors.aliyun.com`
- Actions runner 在美国 → 官方源更快更稳

所以仓库变量里设了 `NPM_REGISTRY` / `APT_MIRROR` 覆盖默认值。

## 本地构建

板子上不 build。真要本地 build 得显式开 profile：

```bash
docker compose --profile build build
```

## 上游

基于 [TheNetsky/microsoft-rewards-script](https://github.com/TheNetsky/microsoft-rewards-script)
v4.3.2.4，国内适配（热搜源、PushPlus、Server酱、ClawBot）见 `src/functions/QueryEngine.ts`
和 `src/logging/`。GPL-3.0。
