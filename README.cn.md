# x-cmd/gpg — x-cmd 团队公钥

x-cmd 核心团队的 GPG 公钥集合（权威发布、经团队另行交叉签名）。
本仓库 `pub/` 目录下以 ASCII-armored 格式（`pub/<handle>.asc`）
逐一发布；同时合并为 `pub/keys.asc` 串联 keyring，方便一次性
导入。

> 🇬🇧 **English: [README.md](./README.md)** — same catalog,
> English front matter.
>
> - **[当前密钥](#当前密钥)**
> - **[如何验证密钥](#如何验证密钥)**
> - **[密钥轮换](#密钥轮换)**
> - **[安全策略](#安全策略)** — `CONTRIBUTING.md` 摘要
> - **[常见问题](#常见问题)**
>
> 终端用户请看 [`SKILL.md`](./SKILL.md)；维护者请看
> [`CONTRIBUTING.md`](./CONTRIBUTING.md)。

## 当前密钥

下表由团队在每次发布提交时从 [`index.tsv`](./index.tsv) 自动
生成。**以 fingerprint 为锚，不要以 handle 为锚** —— handle
在轮换后可能被复用，fingerprint 不会。

<!-- BEGIN keys.md -->

| Handle | UID | Fingerprint | Created | Purpose |
| --- | --- | --- | --- | --- |
| _(暂未发布密钥 —— 见_ [`CONTRIBUTING.md`](./CONTRIBUTING.md)_)_ | | | | |

<!-- END keys.md -->

批量导入本仓库全部密钥：

```sh
# 下载串联 keyring（每个密钥约 3 KB）
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/pub/keys.asc \
  | gpg --import
```

或单个密钥导入：

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/pub/<handle>.asc \
  | gpg --import
```

## 如何验证密钥

裸 fingerprint 不是证据 —— 任何人都能敲出 40 个十六进制字符。
验证需要三步：

1. **获取**：通过你已信任的传输通道拉取 keyring。
   `https://raw.githubusercontent.com/x-cmd/gpg/main/pub/keys.asc`
   在你信任 GitHub 的前提下即可；若需更高保证，可同时从
   团队官网或签名 release tarball 拉取同一文件并对比
   fingerprint。
2. **导入**：执行 `gpg --import pub/keys.asc`。
3. **比对**：让导入后输出的 fingerprint 与 `index.tsv` 中
   的值 *以及* 团队官网上的值三方一致。三方一致即为验证
   通过。

[`x gpg`](https://x-cmd.com/mod/gpg) 模块会自动执行上述流程：
`x gpg import <handle>` 拉取、导入、打印 fingerprint，并在
写入 trustdb 之前要求你确认与 `index.tsv` 一致。

### Fingerprint 的数学含义

GPG v4 fingerprint 是公钥 packet 的 40 位十六进制 SHA-1。
两把 fingerprint 相同的密钥按定义就是同一把 —— 密钥身份
**就是**这些比特本身，没有第二个来源。这正是以 fingerprint
为锚等于以字节级承诺为锚的原因。

## 密钥轮换

密钥到期或被轮换时：

1. 旧 `pub/<handle>.asc` 移入
   `pub/archive/<handle>.<created-date>.asc`。
2. 新密钥占据 `pub/<handle>.asc` 槽位。
3. `index.tsv` 在同一次提交中更新。

README 的 "Retired keys" 段落由 `pub/archive/` 自动生成。
轮换要求与密码学过渡声明见 [`CONTRIBUTING.md`](./CONTRIBUTING.md)。

退役密钥永久保留在 `pub/archive/`，并由团队主密钥签名，
以便已持有团队主密钥的消费者可验证托管链。退役密钥的
fingerprint 永远不会以新 handle 形式再次出现。

## 安全策略

> **仅从 x-cmd 官方渠道直接拉取。** 本 GitHub 仓库与团队官网
> 是 `pub/` 下密钥的唯二授权来源。**未经授权的代理分发**
> —— 第三方公开镜像、通过 CDN / 反向代理 / 缓存代理等第三方
> 服务（jsdelivr、gcore、statically 等会自动代理
> raw.githubusercontent.com 的服务均属此列）重新分发、上传至
> 公开 keyserver、捆绑至其他软件包 —— 均**未经 LICENSE 授权**。
> 见 [`LICENSE`](./LICENSE)。如果你发现这些密钥由其他域名
> 提供，请视为不可信。

> **本仓库仅由 x-cmd 核心团队维护。** 修改 `pub/`、`index.tsv`
> 或其他承载信任信息的文件的外部 PR 将被直接关闭、不合并。

理由：每一个消费者（`x gpg`、包镜像、release tarball）都
把 fingerprint 当作信任锚点。一条把真 fingerprint 换成视觉
近似值的恶意 PR 就是供应链攻击，不是贡献。

> **对文章（README 中的密钥说明、文档细节、错别字）有建议？**
> 请发 [issue](https://github.com/x-cmd/gpg/issues)。Issue
> 公开、可评审，进入团队的文档 backlog。

非团队贡献者的三条具体规则见
[`CONTRIBUTING.md`](./CONTRIBUTING.md)。

## License

**Copyright 2026 x-cmd —— 版权所有，保留所有权利。** 完整文本见
[`LICENSE`](./LICENSE)。本仓库公开发布，仅供查看、获取与使用
`pub/` 下的 GPG 公钥进行签名验证；所有使用均限于从 x-cmd
官方渠道（GitHub: x-cmd/gpg）获取；第三方镜像、上传至公开
keyserver、再分发至其他软件包等行为未经授权；修改、商业使用
等其他权利亦需事先获得 x-cmd 的书面授权。

## 常见问题

**为什么单独建一个仓库，而不是放在 `x-cmd/x-cmd`？** 三个理由。
(1) Release 由这些密钥签名；`x-cmd/x-cmd` 的发布工作流需要从
一个稳定、公开、且不被自身签名（避免循环）的位置取签名密钥。
(2) 轮换频率低但安全关键；独立小型仓库让评审面更聚焦。
(3) `x gpg` 需要一个轻量、离线下可用的密钥来源，不依赖
`x-cmd/x-cmd` 全量 monorepo 克隆。

**为什么用 ASCII-armored `.asc` 而不用二进制 `.gpg`？**
armored 文件可安全地复制粘贴到邮件、聊天窗口和 GitHub Web
UI；diff 干净；`gpg --import` 两种格式都接受。

**我怀疑一把密钥被攻陷了怎么办？** 不要在本仓库报告。请通过
[x-cmd.com](https://x-cmd.com) 上列出的团队渠道直接联系。团队
会：用旧密钥签发密码学过渡声明、轮换到新密钥、在
[`index.tsv`](./index.tsv) 中新增一行。

**本仓库会发布团队的签名策略 / CPS 吗？** 不会。本仓库只发布
公钥侧，`index.tsv` 的 `purpose` 列说明每把密钥签名什么对象。
更广义的信任策略文档见团队官网。

**可以自建镜像吗？** **未经事先书面授权不可以。** 本仓库与
团队官网是唯二授权来源。第三方公开镜像、通过 CDN / 反向
代理 / 缓存代理等第三方服务重新分发（jsdelivr、gcore、
statically 等会自动代理 raw.githubusercontent.com 的服务均
属此列）、上传至公开 keyserver（keys.openpgp.org、
keyserver.ubuntu.com 等）、捆绑至其他软件包等行为均被
[`LICENSE`](./LICENSE) 显式禁止。理由是信任锚点问题：一旦
镜像或代理换掉 fingerprint，所有信任该来源的消费者都会
静默被攻破；即便是诚实的 CDN 缓存，在密钥轮换期间也会
返回陈旧字节。pin 你的工具链到 GitHub，每次都直接拉取。

## 相关

- [`x-cmd/x-cmd`](https://github.com/x-cmd/x-cmd) — 模块源码（`mod/gpg/`）
- [`x gpg` 模块文档](https://x-cmd.com/mod/gpg) — shell 端消费者
- [`CONTRIBUTING.md`](./CONTRIBUTING.md) — 维护者文档
- [`SKILL.md`](./SKILL.md) — 使用说明