# x-cmd/gpg — x-cmd 团队公钥

x-cmd 核心团队的 GPG 公钥集合（权威发布、经团队另行交叉签名）。
本仓库 `keyring/` 目录下以 ASCII-armored 格式（`keyring/<handle>.asc`）
逐一发布；同时合并为 `keyring/keyring.asc` 串联 keyring，方便一次性
导入。

> 🇬🇧 **English: [README.md](./README.md)** — same catalog,
> English front matter.

> 📌 **维护政策。** 与我们其它仓库一样，我们虚心聆听各位
> 朋友的指导与建议 —— 如有想法，请开
> [issue](https://github.com/x-cmd/gpg/issues)。但出于安全
> 考虑，**本仓库不接受外部 pull request**，只能由维护者
> 团队维护。哪些反馈在范围内、哪些会被直接关闭，见
> [CONTRIBUTING.md](./CONTRIBUTING.md)。
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
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import
```

或单个密钥导入：

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/<handle>.asc \
  | gpg --import
```

## 如何验证密钥

裸 fingerprint 不是证据 —— 任何人都能敲出 40 个十六进制字符。
验证需要三步：

1. **获取**：通过你已信任的传输通道拉取 keyring。
   `https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc`
   在你信任 GitHub 的前提下即可；若需更高保证，可同时从
   团队官网或签名 release tarball 拉取同一文件并对比
   fingerprint。
2. **导入**：执行 `gpg --import keyring/keyring.asc`。
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

1. 旧 `keyring/<handle>.asc` 移入
   `keyring/archive/<handle>.<created-date>.asc`。
2. 新密钥占据 `keyring/<handle>.asc` 槽位。
3. `index.tsv` 在同一次提交中更新。

README 的 "Retired keys" 段落由 `keyring/archive/` 自动生成。
轮换要求与密码学过渡声明见 [`CONTRIBUTING.md`](./CONTRIBUTING.md)。

退役密钥永久保留在 `keyring/archive/`，并由团队主密钥签名，
以便已持有团队主密钥的消费者可验证托管链。退役密钥的
fingerprint 永远不会以新 handle 形式再次出现。

## 安全策略

> **仅从 x-cmd 官方渠道直接拉取。** 本 GitHub 仓库与团队官网
> 是 `keyring/` 下密钥的唯二授权来源。**未经授权的代理分发**
> —— 第三方公开镜像、通过 CDN / 反向代理 / 缓存代理等第三方
> 服务（jsdelivr、gcore、statically 等会自动代理
> raw.githubusercontent.com 的服务均属此列）重新分发、上传至
> 公开 keyserver、捆绑至其他软件包 —— 均**未经 LICENSE 授权**。
> 见 [`LICENSE`](./LICENSE)。如果你发现这些密钥由其他域名
> 提供，请视为不可信。

> **本仓库仅由 x-cmd 核心团队维护。** 与其它仓库一样，
> 我们虚心聆听各位朋友的指导 —— 反馈与建议欢迎走
> [issue](https://github.com/x-cmd/gpg/issues) —— 但 **任何
> 文件的外部 PR 都将被直接关闭、不合并**（文档、行文、
> `keyring/`、`index.tsv`，全部）。每次 commit 都由团队自
> 己落笔。详见 [`CONTRIBUTING.md`](./CONTRIBUTING.md)。

理由：每一个消费者（`x gpg`、包镜像、release tarball）都
把 fingerprint 当作信任锚点。一条把真 fingerprint 换成视觉
近似值的恶意 PR 就是供应链攻击，不是贡献。即便只是文档
PR，团队也倾向于自己持有对外密钥文档的单一声音，且"这
条 PR 是良性的"只有在 merge 之后才能验证 —— 到那时字节
已经出去了。

## License

**Copyright 2026 x-cmd —— 版权所有，保留所有权利。** 完整文本见
[`LICENSE`](./LICENSE)。本仓库公开发布，仅供查看、获取与使用
`keyring/` 下的 GPG 公钥进行签名验证；所有使用均限于从 x-cmd
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

**可以自建镜像、或者信任 CDN / 代理 / 公开 keyserver 来
分发这些密钥吗？**
这个判断由你下，不由我们定。

我们 *确实* 说的，归根到底是一件： **`github.com/x-cmd/gpg`
是真源 —— 团队实际维护、实际推送的地方。**
**`https://x-cmd.com/gpg/` 是在部署时 *从这个 GitHub 仓库*
构建出的呈现层：** 团队构建管线从 GitHub 拉 keyring 字节，
然后把它们挂在团队域名下，这样像 `rpm --import
https://x-cmd.com` 这样的短 URL 才不会有拼写风险。一个真
源，两种呈现。你从这两处任一处看到密钥，拿到就是我们发布
出去的字节。

我们 *不* 主张的是：不存在对任何人副本的控制权。GPG 文化
本来就是分布式的 —— 任何拿到字节的人都可以把它们镜像到
第三方服务器、上传至 `keys.openpgp.org`、经 CDN（如
jsdelivr）分发、捆绑至其他软件包等。我们一旦发布，密钥就
进入了公共世界；这是协议的设计。

我们 *不* 做的事是：告诉你该信任谁。如果你的威胁模型是
"只信任团队官方渠道"，就 pin 在 GitHub + `x-cmd.com`，
其他全部拒绝。如果你的威胁模型是"知名公开 keyserver 就
行"，就从 `keys.openpgp.org` 取。如果你的威胁模型是"CDN
更快、字节也够用"，就从 jsdelivr 取。三种都是合理选择 ——
字节都一样。我们只标出 *我们* 在哪里发布；你自行决定以何
处为锚。

## FAQ —— 供应链安全与企业级商业化部署

以下问答覆盖 x-cmd 的供应链密钥策略，以及本仓库
（`x-cmd/gpg`）在整个策略中承担的角色。读者既包括终端用户，
也包括金融 / 政企甲方的合规与安全评审团队。

### Q1：GPG 和 GPG Key 有什么区别？

定论：两者是软件程序与数据凭证的关系。GPG 是那把复杂的智
能锁（软件工具），负责执行加密、解密和签名校验等动作；而
GPG Key 则是开锁的钥匙（数据文件），是包含公私钥对的数字
凭证。在 x-cmd 生态中，功能代码由 [`x gpg`](https://x-cmd.com/mod/gpg)
模块实现，而密钥和签名历史则独立托管在 x-cmd/gpg 仓库中。

### Q2：既然有了现代化的 Sigstore（无密钥签名），为什么 x-cmd 的 RPM 和 DEB 包还要用 GPG？

定论：因为操作系统的原生包管理器只认 GPG。虽然 Sigstore
正在成为云原生和容器安全的新标杆，但对于 Linux 底层包管理
器（如 apt、dnf、yum）来说，GPG 依然是绝对统治、不可替代
的底层信任标准。为了确保企业用户能通过官方渠道绿灯安装，
必须支持原生的 GPG 文件或仓库签名。

### Q3：为什么 x-cmd 决定彻底抛弃 unsigned（未签名）的裸包发布？

定论：单独保留未签名包会暴露软件供应链的脆弱性，严重影响
商业推广（Promotion），且无法通过企业内网的安全扫描。x-cmd
的未签名状态仅存在于 CI 流水线的内存中。发布时通过"梯队式
信任架构"，用"受限"与"不受限"的两把官方钥匙对其进行签
名，用高强度的密码学防护封死黑客中途投毒的路径。

### Q4：针对不同级别的用户，x-cmd 最终提供哪几种 RPM/DEB 制品形态？

定论：在 GitHub Release 中只提供以下两种已签名的物理分身，
完美兼顾丝滑体验与变态合规：

1. **`x-cmd.rpm` / `x-cmd.deb`（社区标准版）**：使用不受
   限的官方主密钥签署。面向 99% 的开源极客和普通容器构建
   环境，用户一辈子只需导入一次公钥，后续升级终身零干预、
   零摩擦。
2. **`x-cmd-annual-<年份>.rpm` / `x-cmd-annual-<年份>.deb`
   （企业合规版）**：使用当年专属的年度隔离密钥
   （如 `key-2026`）签署。专为金融、政企等需要严格年度资产
   隔离审计的高付费大户准备。

### Q5：针对企业合规版的年度隔离密钥，为什么技术上采用"永不过期"，而操作上实施"一年一换"？

定论：这是为了在满足企业合规审计的同时，坚守生产环境的极
端连续性（Extreme Business Continuity）。

商业痛点：如果在密码学上将密钥硬编码设置为 365 天过期，
一旦跨年交替时甲方运维团队放假未能及时更新，他们的服务器
就会因为"密钥过期"而直接报警崩溃，引发严重的商务索赔
（SLA 纠纷）。

大厂 Practice：将密钥在技术上设置为永不过期，但在发布流
程上执行年度隔离（Annual Isolation）—— `key-2026` 仅且只
用来签署 2026 年内发布的版本，到了 2027 年这把私钥将被严
格物理封存，启用全新的 `key-2027`。

### Q6：为什么不采用 398 天、397 天、380 天这种带有缓冲期的密钥寿命设计？

定论：在真实的商务拉扯和自动化合规扫描中，非整数的天数反
而会增加沟通成本并导致审计误报。

商业风险：国际 Web 证书（SSL/TLS）标准在 2026 年刚刚将红
线寿命缩短至 200 天以内。代码签名虽然不受此限制，但如果
文案还死咬着旧 SSL 的 398 天概念，会被挑剔的甲方安全官判
定为"技术信息滞后"。因此，直接采用"1 年（365 天）商业
边界 + 永不过期密码学防护"，在商务上最干净利落。

### Q7：既然"一年一换"在跨年时会引入新密钥，老系统的旧钥匙要不要删除？

定论：这个决定由你自己做。旧钥匙是留还是删，是个 *用户侧*
的决定，发布方不强制。工具链（`gpg`、`apt`、`dnf`、`rpm`）
把密钥环完全交给运营方 —— `gpg --import` 之后，公钥就躺在
你自己的 `~/.gnupg/` 里，要删除得显式 `gpg --delete-keys`。
这把供应链的杠杆放在你手里：你愿意让本地接受哪些发布方的
哪些版本，由你说了算。

"删除旧钥匙"在实操中的含义：该主机上所有由旧密钥签名的制
品，在下一次日常扫描或 `dnf check` 跑起来时都会失败 —— 因
为本地密钥环不再认签名。这正是合规制度想要的杠杆（"如果不是
今年的密钥签的就拒"），但它同时也是把日常扫描变成寻呼事件
的杠杆（如果边界撞在装有老制品的主机上）。

"保留旧钥匙"在实操中的含义：所有历史制品继续校验，但你需要
无限期承担每把保留下来的密钥的信任面。

原则：谁控制本地密钥环，谁就控制本地校验什么。这是 GPG 的
设计 —— 控制权在用户手里，不在发布方手里。发布方唯一要做
的是把字节发出来；策略由运营方定。

### Q8：当进入新的一年，付费企业要求使用新密钥安装历史旧版本软件，x-cmd 如何处理？

定论：采用"版本重签发（Repackage / Resign Lifecycle）"机
制，且重签后的老包依然保持全网完全公开。

极客偷懒流：RPM 重新签名不需要重新编译老代码，也不需要动
历史的 GitHub Release 资产。在 CI/CD 中只需通过
`rpmsign --addsign` 刷一下二进制文件的签名 Header 即可。

收费卡点：虽然重签包公开托管在官网上以彰显供应链透明度，
但"让哪些历史版本获得重签、以及多久重签一次"由服务期决
定。官方不为免费用户维护旧版的重签，这完美变现为企业级
长期技术支持（LTS）的高额订阅特权。

### Q9：为什么最终决定将数据源仓库命名为 x-cmd/gpg，而不是 x-cmd/gpgkeyring？

单纯个人偏好 —— 我觉得 `gpg/keyring` 比 `gpgkeyring` 读起来
更顺。

公钥住在 `x-cmd/gpg` 仓库下的 `keyring/` 子目录里，自然读
作"x-cmd 团队的 GPG keyring，就在这"。`keyring` 也是
GnuPG 原生术语中对"公钥集合"的称呼（`gpg --keyring <path>`、
`~/.gnupg/pubring.kbx` 都用这个词），所以子目录名与上游词
汇对齐。把两个词硬拼成 `gpgkeyring` 会同时丢掉结构提示与
GnuPG 术语一致性。

### Q10：最终用户或甲方的自动化运维脚本，如何通过最简短的 URL 一键导入 x-cmd 的官方信任锚点？

定论：完美利用 GitHub Pages 的域名顶级路由机制（主官网
占领 `x-cmd.com` 根域名后，任何独立子仓库如 `gpg` 开启
Pages，会自动无缝映射到主域名的同名子目录下）。

黄金一键导入命令：

```sh
# 红帽系统（RHEL/CentOS/Rocky Linux）的一键合规导入
sudo rpm --import https://x-cmd.com
```

该架构不仅使主官网项目零污染（专心做 UI 和宣发），同时
让 `x-cmd/gpg` 仓库成为一个极其清爽、完全公开透明、随时
供全球安全专家与甲方内网离线脱水镜像（Mirroring）的"密码
学信任中心"。

## 相关

- [`x-cmd/x-cmd`](https://github.com/x-cmd/x-cmd) — 模块源码（`mod/gpg/`）
- [`x gpg` 模块文档](https://x-cmd.com/mod/gpg) — shell 端消费者
- [`CONTRIBUTING.md`](./CONTRIBUTING.md) — 维护者文档
- [`SKILL.md`](./SKILL.md) — 使用说明