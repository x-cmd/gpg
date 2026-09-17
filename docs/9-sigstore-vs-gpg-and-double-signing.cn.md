---
x-title: Sigstore vs GPG，以及双重签名策略
x-desc: Sigstore（无密钥签名 + 透明日志）与传统 GPG（长期密钥 + 可信通道）的并排对比 —— 覆盖信任根源、审计属性、生态适用场景，以及何时双重签名是合理的混合策略。**项目中立；纯探讨。**
x-sidebar: Sigstore vs GPG，以及双重签名
x-keywords: sigstore, gpg, 无密钥签名, 透明日志, rekor, fulcio, 双重签名, 双签, 供应链, slsa, cosign, rpm
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Sigstore vs GPG，以及双重签名策略'
      inLanguage: 'zh-CN'
      about: 'Sigstore vs GPG 对比与双重签名策略'
---

# Sigstore vs GPG，以及双重签名策略

2026 年软件供应链安全由两种截然不同的签名范式主导。
**GPG**（及其前身 PGP）自 1990 年代以来一直是软件签名的
事实标准；**Sigstore** 是较新的（2021+）生态，核心是基
于 OIDC 身份 + 公开透明日志的无密钥签名。它们解决不同
的问题；当团队交付的制品需要同时被旧与新生态消费时，
通常会同时考虑两者。

本文从机制、信任根源、审计属性、生态适用场景四个维度对
比二者。以 *双重签名* 在何时是合理策略、何时是过度设计的
分枝收尾。

> **状态：探讨。** 截至本文撰写时，x-cmd 团队尚未在实
> 际发布中采用任一方案。下文是分析，而非对当前已实施内
> 容的描述。

## 两种范式，两种威胁模型

| 维度                 | GPG（传统）                                                  | Sigstore（无密钥）                                                  |
| ---                   | ---                                                          | ---                                                                  |
| **核心机制**         | 用长期私钥签名；用对应公钥校验。                              | 用绑定到 OIDC 身份的短期证书（≈10 分钟）签名；通过证书链与透明日志条目校验。 |
| **凭证负担**         | 必须生成、分发、轮换、保护（离线笔记本、HSM、CI secret 等）长期私钥。泄露 = 私钥全生命周期内历史伪造风险。 | 每次签名事件动态生成新密钥对，签完立即销毁私钥半。无长期秘密要保护。 |
| **信任根源**         | 公钥 + 公钥的传递通道（团队自控域名 + HTTPS）。                | OIDC 身份提供方（GitHub、Google 等）+ 透明日志（Rekor）。            |
| **审计 / 不可否认性** | 弱：签名只说"这把密钥签了这个制品"，不能公开可验证地说在哪天哪台机器上签的。 | 强：每次签名与 OIDC 身份、制品哈希、时间戳一起写入 Rekor。任何人都能审计"谁在何时签了什么"。 |
| **生态适用**          | RPM / DEB / Apt / Pacman 等 OS 级包管理器原生内置 OpenPGP 校验。 | OCI 容器镜像、Kubernetes admission controller、GitHub Actions attestation 等云原生原生支持。OS 级包管理器 2026 年仍支持有限。 |
| **吊销**             | 可通过 OpenPGP 吊销证书完成，但吊销证书本身的分发是难题。   | 隐式：证书短期设计，无新证书即无新签名，吊销自动。 |
| **信任基底**         | 点对点 —— 任何人都能签、任何人都能 pin；适合个人与"无制度"项目。 | 制度性 —— 锚定 OIDC 身份提供方（GitHub、Google 等），需要先有"已知实体"。 |

两种范式防范不同的事。GPG 的安全声明本质上是"数学成立且
私钥保持私密" —— 一种长期 *能力* 担保，依赖运营纪律。
Sigstore 的安全声明是"这次签名由这个 OIDC 身份产生，且
整件事在公开日志里" —— 一种短期 *事件* 担保，依赖 OIDC
IdP 与日志保持可信。

## 信任根源分析

两套范式建立信任的方式在根本上不同。

**GPG 的信任链**经三层：
1. 数学：签名有效性（密钥对未被篡改时永远成立）。
2. 密钥：消费者导入密钥环的**具体字节**。Fingerprint 是
   对这些字节的 160 位承诺；只要字节对，就密码学锁定到
   那个生产者。
3. 通道：消费者怎么拿到这些字节。这才是真正的信任所
   在 —— `https://x-cmd.com/gpg/`、已签名 release
   tarball、已验证的 USB curl 等。只要通道未被攻破，链条
   成立。

最弱环节是第 3 层：控制传递通道的攻击者可替换为不同公
钥，消费者 `gpg --verify` 会通过。防御是 *信任通道* +
*fingerprint pinning*（把导入的 fingerprint 与独立参考源
交叉比对）。

**Sigstore 的信任链**经不同的三层：
1. 数学：签名对短期证书的有效性。
2. 证书：Fulcio 颁发的 x.509 证书，把一个 OIDC 身份
   （如 `x-cmd/x-cmd` 仓库 on GitHub）绑定到为本次签
   名事件生成的公钥。
3. 日志条目：Rekor 记录，把证书、制品哈希、OIDC 身份
   提交到一棵 Merkle 树，其根周期性发布。

最弱环节是第 2 层：Sigstore 的信任依赖于 OIDC IdP
（GitHub、Google 等）诚实地向谁颁发身份令牌。如果你的
GitHub 组织被攻陷，攻击者可以冒充你签发签名证书。防御
是 *加固 OIDC 身份*（GitHub 组织安全、分支保护、维护者 2FA
等） —— 与 GPG 要求的运营纪律相同，只是落在不同表面上。

## 主观信任：GPG 作为"前制度"机制

GPG 的设计有一个在技术对比中容易被忽视的属性：它是 *
前制度* 信任机制。两方可以在任一方还不是人 / 法人 / 已注
册组织之前，建立密码学可证、法院可采信的"身份与协议"证明。

之所以能这样，是因为 GPG 的信任链是 *点对点* 的 —— 你
选择信任一把具体密钥，是因为你决定信任它，而不是因为某
个机构替它背书。链条最后落在你本人的判断（主观信任）上
，记录后（你把密钥导入了密钥环），数学接手续上。

Sigstore 的设计则最后落在 *OIDC IdP* 上 —— 通常是
GitHub、Google、Microsoft。这些是有法人资格的机构行为方。
Sigstore 中的"签名身份"是 *IdP 同意为之背书的人*。这在多数
商业场景中有用，但有一个前提：IdP 愿意向你颁发身份令牌，
而这通常需要你拥有一个账户、付款方式、组织、或其他制度
性锚点。

具体含义：

- **匿名 OSS 贡献者**可以用 GPG 签自己的代码，下游消费
  者把他的指纹 pin 上，不需要在他处是人或公司。信任是纯
  密码学的 —— 没有机构中介。
- 同样的贡献者大概率不能用同样的方式用 Sigstore，因为他
  没有 GitHub 组织或等同的制度性锚点来绑签名身份。

这同时是一个有意义的 *法律* 属性，不只是技术上的。指纹
在许多司法管辖区是 *可采信证据*，证明特定文档或制品由
匹配私钥的持有者签署。信任链是直接的而非中介的，这在一
些法律框架下被视为比依赖第三方保持可信与可问责的"制度
性 attestation"更强的作者身份证据。（视司法管辖区而定；
这非法律建议，但对部分合规制度而言是真实考量。）

实际推论：

- **未注册公司前的项目。** 还没完成公司注册的项目可以
  用 GPG 发签名制品。同一个项目若用 Sigstore，通常需
  要在 GitHub 或其他 IdP 上有组织账户，而那通常要求是
  法人或有一位法人作为担保。
- **跨司法管辖区场景。** GPG 密钥跨境流转无注册责任。
  签名的法律效力适用哪个法域的法律，可通过合同约定，
  而无需第三方机构做中介。
- **法院可采信的证据。** 在某些司法管辖区，正确 pin
  的 GPG 签名比"制度性 attestation"更强的作者证据 —
  — 因为密码学链接是直接的而非中介的。
- **Sigstore 的制度性前提。** Sigstore 适合 *由* 公司或
  组织 *生产* 的软件，这些组织有制度性身份；对个人生
  产的 OSS 长尾不那么适用 —— 个人可能没有或不想有制
  度性身份。

这不是说 GPG 一般优于 Sigstore —— Sigstore 的制度性锚
点在多数商业场景下是特性。这是要你在选定前想清楚 *你实
际需要的是哪种信任*，并认识到两套范式优化的是不同的
场景。

## 适用场景边界

**GPG 是对的选，当：**
- 制品被 OS 级包管理器（`dnf`、`yum`、`apt`、`zypper`、
  `pacman`）消费。这些内置原生 OpenPGP 校验，默认拒绝
  安装未签名 RPM / DEB。
- 生态保守 —— RHEL、CentOS、Rocky、Debian、Ubuntu LTS、
  SUSE、Alpine、嵌入式 Linux 发行版。
- 消费者是跑遗留基础设施的运维，没有 Sigstore 验证工具。

**Sigstore 是对的选，当：**
- 制品是容器镜像、Kubernetes 资源、或软件供应链
  attestation（SLSA / in-toto）。
- 消费者已有 Kubernetes admission controller 或 CI/CD
  集成 Sigstore 验证（`cosign verify`、`kyverno`、
  policy controller 等）。
- 团队想要"谁在何时签了什么"的可审计公开追踪 —— 不
  想承诺长期密钥管理的运营纪律。

**单独用哪个都不够，当：**
- 制品需要跨"旧 + 新"生态被消费。见下文"双重签名"。

## 什么是双重签名

**双重签名**（也叫 dual-signing）指对同一制品独立产生两
套签名：每个范式一套。消费者验证与自身工具链匹配的签名；
另一生态的消费者验证另一套；任何消费者都不用改工具来接受
该制品。

对一个 RPM，一条双重签名流水线通常如下：

1. 构建 `.rpm`（未签名）。
2. **GPG 层。** `rpmsign --addsign package.rpm` —— 把
   OpenPGP 签名头嵌入 RPM 元数据。此时 `rpm -K` 通过、
   `dnf install` 接受。
3. **Sigstore 层。** `cosign sign-blob package.rpm`（或
   对 OCI 镜像用 `cosign sign`） —— 产出 detached 签名
   （`sig`）与证书（`cert`），引用 Rekor 日志条目。
   此时 `cosign verify-blob` 通过。
4. 发布制品、cosign 的 `sig` + `cert` + bundle，以及公
   开的 GPG 公钥（在团队的 `keyring/keyring.asc` 中，
   供 OS 级消费者）。

两套签名各自独立可校验；任何一方都不依赖另一方；任一生态
的消费者拿到的是同一份带相同信任的制品。

## 何时双重签名值得

双重签名值得加 CI / 存储成本，当：

- 制品被**两个**生态同时消费（OS 级包管理器 *和* 云原生
  供应链工具）。
- 团队能消化运营开销：CI 签两次，发布两套制品（包内
  RPM 签名头 + 包外 cosign `sig` / `cert` / bundle），维护
  两套验证文档。
- 团队的威胁模型同时覆盖旧与新攻击面 —— 即既关心遗留
  运维流水线（GPG 必需），又关心现代云 attestation
  流水线（Sigstore 期望）。

具体的项目/团队做过这套：Kubernetes 上游（容器镜像用
Docker Content Trust 与 cosign 双签、源码 tarball 用
GPG）；多家 Linux 发行版在 OS 包之外签发云原生制品；
Sigstore 项目本身（用 cosign 与 GPG 双签自己的二进制）。

## 何时双重签名过度

双重签名**不**值得，当：

- 受众只在一个生态。如果消费者全是 `dnf`，加 cosign 是
  无收益的成本。如果消费者都在 Sigstore 验证的
  Kubernetes 集群里，GPG 层是死重。
- 制品不跨场景复用。一个只在 CI 里跑的脚本不需要两种
  签名。
- 团队负担不起翻倍的 CI 复杂度。两种签名意味着两套密
  钥管理故事、两种失败模式、两套文档。如果运营简单比
  生态覆盖更重要，选一种，发。

## 混合流水线 —— 实操要点

如果采用双重签名，几个实操考量：

- **顺序敏感。** 先签 GPG（会改 RPM 头），再对 *结果*
  签 cosign。反过来意味着 cosign 在 sig GPG 前的字节，Rekor
  不覆盖 GPG 签名。设计上是 cosign 覆盖 post-GPG 状态；
  仅用 cosign 校验的消费者据此作锚。
- **透明日志覆盖范围。** Rekor 记录签名那一刻的制品哈
  希。cosign 覆盖 post-GPG RPM。仅用 cosign 校验的消费
  者锁的是 post-GPG 状态；这是设计意图。
- **密钥轮换成本。** GPG 层仍需密钥轮换故事（见
  [4. 年度密钥策略 —— 设计探讨](./4-annual-key-strategy-explained.cn.md)）。
  Sigstore 层自轮换（证书短期）。两层有独立的运营节奏。
- **文档维护。** 团队需要维护两套验证路径的文档。
  README / docs 同时需要 `rpm -K` 与 `cosign verify-blob`
  的配方。再加第三方消费者（如 `slsa-verifier`）有时会
  拉入第三条验证路径，复杂度可能复利。

## 延伸阅读

- [4. 年度密钥策略 —— 设计探讨](./4-annual-key-strategy-explained.cn.md) ——
  长期 GPG 密钥轮换权衡；若采用双重签名，与 Sigstore
  层配对使用。
- [7. 用 GPG 给 RPM 包签名](./7-signing-an-rpm-with-gpg.cn.md) ——
  双重签名流水线中 GPG 层会调用的 `rpmsign --addsign`
  步骤。
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) —— 选定签名策略
  后的发布侧流水线约定。

## 来源

- [Sigstore 项目文档](https://docs.sigstore.dev/) ——
  Fulcio（CA）、Rekor（透明日志）、cosign（CLI）。
- [RFC 4880 — OpenPGP Message Format](https://datatracker.ietf.org/doc/html/rfc4880)
- [Sigstore 博客](https://blog.sigstore.dev/)