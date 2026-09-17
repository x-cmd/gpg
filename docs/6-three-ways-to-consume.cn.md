---
x-title: 消费 keyring 的三种方式
x-desc: 拉取 keyring 的三种第一方路径 —— 直接从 GitHub curl、`x gpg` shell 模块、GitHub Pages 经 x-cmd.com 跳转。权衡、常见配方、离线 / 气隙环境注意事项。
x-sidebar: 消费 keyring 的三种方式
x-keywords: curl, x gpg, github pages, x-cmd.com, 气隙, 离线, 批量导入, 三方拉取
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '消费 keyring 的三种方式'
      inLanguage: 'zh-CN'
      about: 'keyring 消费路径与权衡'
---

# 消费 keyring 的三种方式

x-cmd 团队 keyring 字节有三条第一方拉取路径，每条针对不
同的消费者优化。本文逐条给出实操配方、点出权衡，并以金
融 / 政企环境通常需要的离线 / 气隙配方收尾。

## 路径 1 —— 直接从 GitHub curl

最简单的路径：对 `raw.githubusercontent.com` 用 `curl`。

```sh
# 汇总 —— 一次性拿全部密钥
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import

# 按 handle 单把导入
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/<handle>.asc \
  | gpg --import

# 只取清单（不带密钥字节）
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/index.tsv
```

**适用场景**：任何能直接访问 GitHub 的消费者。CI 流水线、
开发工作站、容器构建、包构建步骤。

**优点**：
- 零依赖。`curl` 与 `gpg` 在每个 Linux 发行版上都是预装
  的。
- 拉取在使用的那一刻发生 —— 无陈旧缓存，无共享状态要
  维护。
- 极容易脚本化，无需特殊工具。

**缺点**：
- 要求能直接访问 `raw.githubusercontent.com`。
- 每个消费者都要重复同样的样板（解析输出、fingerprint
  校验、更新 trust store）。
- 不感知团队供应链策略（社区版 vs 年度密钥）—— 由消费
  者自行编码。

## 路径 2 —— `x gpg` shell 模块

x-cmd 团队发布了一个 shell 模块，把直接 curl 的路径包装
成带缓存、fingerprint 交叉校验、和一组高层命令的形式。

```sh
# 一次性导入全部 keyring（带缓存）
x gpg import

# 单把密钥，信任写入前先显示 fingerprint
x gpg import <handle>

# 不导入，仅查询团队索引中的某把密钥
x gpg info <handle>

# 用 x-cmd 密钥校验一个 detached 签名
x gpg verify <sig> <file>

# 用团队密钥签一个文件（需要团队侧访问权限）
x gpg sign <file>

# 列出本地密钥环里已有的密钥
x gpg ls
```

**适用场景**：已装 `x` 且想跳过样板代码的终端用户。也适
用于一次性查询（"团队的 release-signing fingerprint 是
多少？"）而不变更本地密钥环。

**优点**：
- 缓存 —— 一次导入，本地持有字节。再跑 `x gpg import`
  便宜。
- 内置交叉校验 —— `x gpg info` 显示 fingerprint 并在写
  入 trust 数据库前提示你确认它与 `index.tsv` 一致。
- 知道团队的两密钥策略 —— `x gpg info official` 与
  `x gpg info key-2026` 返回不同的元数据（社区版 vs
  年度版）。
- 源码在 `x-cmd/x-cmd`（`mod/gpg/`）；可以像审计其他
  x-cmd 模块一样审计它。

**缺点**：
- 要求安装 `x`，引入对 `x-cmd/x-cmd` monorepo 的依赖。
- 缓存层意味着你可能在用稍陈旧的字节做校验 —— `x gpg
  update` 会重新拉取。

## 路径 3 —— GitHub Pages 经 x-cmd.com 跳转

团队官网 `https://x-cmd.com` 暴露一条子域路径
（`https://x-cmd.com/gpg/`），解析到本仓库的
`keyring/keyring.asc`。跳转通过 GitHub Pages 加上团队自
身的边缘配置完成，所以 `https://x-cmd.com/gpg/keyring.asc`
与 `https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc`
提供同一字节 —— 发布时由团队 CI 验证。

```sh
# 经团队自有域名
curl -fsSL https://x-cmd.com/gpg/keyring.asc \
  | gpg --import

# 或者 —— 用于 RHEL/CentOS 包校验的短 URL
sudo rpm --import https://x-cmd.com
```

第二种形式是 RPM 包校验的一键信任锚点导入：
`rpm --import <URL>` 把 URL 给的字节读入系统密钥环。

**适用场景**：任何偏好以团队自有域名而非
`raw.githubusercontent.com` 作锚的消费者。尤其适合
RHEL/CentOS 包导入命令 —— 短 URL 更易读。

**优点**：
- 一个 URL 同时覆盖两种用途（只读拉取与
  `rpm --import`）。
- 比 `raw.githubusercontent.com/...` 更易记。
- 团队以后可以把字节搬到不同后端仓库（比如团队官网
  改自托管），而不会破坏 pin 在 `https://x-cmd.com`
  的消费者脚本。

**缺点**：
- 多一跳（GitHub Pages 跳转），是一丁点可用性风险。
- 对消费者而言更难独立审计 —— pin 在
  `raw.githubusercontent.com` 让你可以直接对照 GitHub
  HTTPS 证书链校验字节源；pin 在 `x-cmd.com` 需要你
  信任团队的证书配置。

## 选一种 or 组合

生产中常见组合模式：

1. **引导**：在受控环境（你的笔记本、你控制的 CI
   runner）一次性 `x gpg import`。
2. **生产主机**：用 `x-cmd.com` 跳转做
   `rpm --import` 风格的包校验，因为 URL 短且稳定。
3. **CI 校验步骤**：每次直接对 `raw.githubusercontent.com`
   跑 `curl | gpg --import`，让生产 CI 对 `x gpg` 缓存陈旧
   性免疫。

三种都从同一字节源拉取。选择的是工程性与信任叠加，不是
   字节"对错"。

## 离线 / 气隙环境

对没有直接互联网访问的环境（金融 / 政企内网、机密
enclave），三条路径合并成一条：

```sh
# 在联网机器上：
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  > /media/usb/keyring.asc

# 在气隙机器上：
gpg --import /media/usb/keyring.asc
```

两台机器之间的传输你选 —— 摆渡、机密网络传输、安全制度
允许的任何通道。keyring 文件是纯 ASCII 文本；可走任何能
保留字节的通道。

关键提醒：气隙机器的 `index.tsv` 应在同一批中传输，这样
消费者不依赖气隙网络任何外部可达性就能确认 fingerprint
一致。

## 交叉引用：`x gpg` 自身如何消费本仓库

对 `x gpg` 模块（路径 2 与路径 3 的实现）自身，见
[`x-cmd/x-cmd` 的
`mod/gpg/`](https://github.com/x-cmd/x-cmd/tree/main/mod/gpg)。
模块很小（几百行 POSIX shell），可端到端审计。

## 延伸阅读

文档系列到此为止。本仓库剩余文件是技术参考：

- [`README.md`](../README.md) —— 首页摘要
- [`README.cn.md`](../README.cn.md) —— 中文首页
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) —— 维护者流
  程与政策
- [`SKILL.md`](../SKILL.md) —— AI agent 用法

## FAQ

本节从
[文章 0 的中心 FAQ](./0-x-cmd-gpg-overview.cn.md#faq--软件分发与代码签名密码学)
里挑出与本文最相关的子集。完整的 8 问在文章 0，
答案以行业普遍视角书写，与具体项目无关。

### Q2：既然有了 Sigstore，为什么 RPM / DEB 依然高度依赖 GPG？

`rpm --import` 与 `apt-key add` 是 GPG 原生命令 —— OS 级
包管理器没有 Sigstore 校验模式。所以即便 Sigstore 在技术
上有优势（无长期私钥管理、公开透明日志），OS 包分发的
事实渠道仍是 GPG。本生态系统的消费者必须持有一份 GPG
签名的 keyring 来引导信任。完整答案见文章 0。

### Q8：采用"一年一换"密钥模型，工业界如何解决跨年过渡与历史回滚？

标准模式是 **Trust Anchor Registry（信任锚点注册表）**：
把每个历年年度公钥合并进单个 keyring，消费者导入一次后
永久保留。这让"用哪种传输拉 keyring"（curl、shell 模
块、GitHub Pages 经跳转、气隙环境的摆渡）的选择与
"keyring 里有哪些密钥"正交 —— 无论怎么运输，历史密钥
都跟着走。完整答案见文章 0。