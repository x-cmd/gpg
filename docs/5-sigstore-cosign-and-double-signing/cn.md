---
x-title: Sigstore、Cosign 与双重签名
x-desc: Sigstore 是什么（项目）、Cosign 是什么（CLI）、Fulcio（CA）与 Rekor（透明日志）如何嵌合 —— 以及与 GPG 的对比、主观信任 / 前制度的差异、双重签名策略、以及何时双重签名值得。**项目中立；纯探讨。**
x-sidebar: Sigstore、Cosign 与双重签名
x-keywords: sigstore, cosign, fulcio, rekor, 透明日志, 无密钥签名, oci, 容器, slsa, 供应链, gpg vs sigstore, 双重签名, 双签, cncf
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Sigstore、Cosign 与双重签名'
      inLanguage: 'zh-CN'
      about: 'Sigstore / Cosign 生态与 GPG 对比'
---

# Sigstore、Cosign 与双重签名

2026 年软件供应链安全由两种截然不同的签名范式主导。
**GPG** 自 1990 年代以来一直是软件签名的事实标准；
**Sigstore** 是较新的（2021+）生态，核心是 *无密钥* 签名
+ OIDC 身份 + 公开透明日志。本文深入讲解 Sigstore 与
Cosign，再对比两种范式，分析 *双重签名* 何时是合理的混
合策略。

> **状态：探讨。** 截至本文撰写时，x-cmd 团队尚未在实际
> 发布签名中采用任一方案。

## 终端用户场景：验证 Sigstore 签名制品

如果你作为终端用户要校验 Sigstore 签名制品（OCI 容器镜像、
跟 `cosign sign-blob` 输出并列的 blob 等），典型流程如下：

### 场景 A：校验容器镜像

```sh
# 先装 cosign（https://docs.sigstore.dev/cosign/installation/）
cosign verify --keyless \
  ghcr.io/example/app:v1.2.3 \
  --certificate-identity-regexp '^https://github.com/example/.*$' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com'
```

这一步校验：

1. 签名由 Fulcio 颁发的证书产生，证书的 OIDC 颁发者匹配
   预期（这里是 GitHub Actions）。
2. 证书中的身份匹配预期 pattern（这里是
   `github.com/example/` 下任何东西）。
3. 签名已记入 Rekor 且 inclusion proof 可用。
4. 镜像摘要与签名的摘要匹配。

预期输出以 `Verified OK` 结尾。任何其他情况 —— `FAILED
to verify`、OIDC 身份不符 —— 都意味着停下来排查。

### 场景 B：用 detached sig + cert + bundle 校验 blob

对非容器镜像的制品（RPM、release tarball、二进制），发布方
通常并列发出三个独立文件：

```text
package.rpm        ← 制品
package.rpm.sig    ← 签名（二进制或 base64）
package.rpm.cert   ← Fulcio 证书
package.rpm.bundle ← Rekor inclusion proof
```

```sh
cosign verify-blob \
  --signature package.rpm.sig \
  --certificate package.rpm.cert \
  --bundle package.rpm.bundle \
  --certificate-identity your-identity \
  --certificate-oidc-issuer https://your-idp/ \
  package.rpm
```

预期输出以 `Verified OK` 结尾。

### 场景 C：交叉比对 OIDC 身份

OIDC 身份是最值得校验的东西 —— 它是 Sigstore 中"发布方"
的密码学等价物。一个典型 CI 签名身份形如：

```text
https://github.com/example/app/.github/workflows/release.yml@refs/tags/v1.2.3
```

发布方告诉你期望签什么；你在 `cosign verify` 里字面 pin。
如果证书的身份字段不匹配，签名就是别的东西产生的 ——
可能是当时恰好控制 OIDC 账户的攻击者。

### 终端用户常见坑

- **OIDC 身份 regex 太松。** 类似 `.*example.*` 这样的
  pattern 接受范围超出你的预期。精确 pin 到你信任的
  workflow / 仓库。
- **陈旧的 `--certificate-identity-regexp`。** 如果发布方
  改了 workflow 文件路径，regex 不再匹配。每次新 release
  都要更新。
- **缺 bundle。** 没传 `--bundle`，cosign 会从公共 Rekor
  拉，对终端用户能用但慢。发离线制品的发布方应把
  inclusion proof 打包进 bundle。
- **混淆 Cosign 与 Docker Content Trust。** 是不同的生态；
  `cosign verify` 不校验 `DOCKER_CONTENT_TRUST` 签名，反之
  亦然。

## 一段话总结

**Sigstore** 是一个项目（源自 Google，现为 CNCF 毕业项
目，归 Linux 基金会），提供 *无密钥* 软件签名 —— 短期证
书绑定到 OIDC 身份、所有事件记入公开 append-only 日志。

**Cosign** 是 Sigstore 的 CLI 工具，对容器镜像、二进制
blob、attestation 等制品做签名与验证。

**Fulcio** 是颁发短期签名证书的 CA。

**Rekor** 是公开记录每次签名事件的透明日志。

四者合起来，让团队发布签名制品而 *无需管理任何长期私钥*
 —— 代价是签名身份锚定到 OIDC IdP（通常 GitHub 或
Google），不是团队持有的密钥。

## 三块核心

![cosign](https://repo.x-cmd.io/cosign.svg)

### Cosign —— CLI

Cosign 是面向用户的工具。做两件事：**签名**（产出签名、
证书、透明日志条目）与 **验证**（检查签名确实由特定身份
产生并已记入 Rekor）。

典型命令：

```sh
# 签名容器镜像（最常见用法）
cosign sign --keyless ghcr.io/example/app:v1.2.3

# 签名任意 blob（如 RPM）
cosign sign-blob --output-signature sig \
  --output-certificate cert \
  package.rpm

# 验证容器镜像
cosign verify --keyless ghcr.io/example/app:v1.2.3

# 验证 blob
cosign verify-blob --signature sig \
  --certificate cert \
  --certificate-identity github \
  --certificate-oidc-issuer https://github.com/login/oauth \
  package.rpm
```

2026 年的默认就是 `--keyless` —— Sigstore 的全部意义就在
于不需要长期私钥。

### Fulcio —— CA

Fulcio 颁发短期签名证书。你跑 `cosign sign --keyless` 时
发生以下步骤：

1. Cosign 向你的 OIDC IdP（典型为 GitHub Actions 或
   `gh auth login`）请求 OIDC 身份令牌 —— 一份已签 JWT，
   承诺"我是组织 Y 的用户 X"。
2. Cosign 把该令牌发给 Fulcio。
3. Fulcio 校验 OIDC 令牌，然后颁发一份 X.509 签名证书，
   把 OIDC 身份（如
   `https://github.com/example/.github/workflows/release.yml@refs/tags/v1.2.3`）
   绑定到一份为本次签名事件新生成的公钥。
4. 证书过期时间约 10 分钟（Fulcio 默认配短期证书寿命）。
5. Cosign 用匹配的私钥签制品，然后把私钥扔掉。

签好的制品、签名、证书随后发给 Rekor 做透明日志登记。

### Rekor —— 透明日志

Rekor 是 Sigstore 网络处理过的每次签名事件的
append-only 哈希链接 Merkle 树。每次签名事件把签名、证
书、制品哈希打包提交为 Rekor 条目；Rekor 返回一份把该
条目绑到 Merkle 根的 inclusion proof。

实际含义：

- 每次签名事件**公开**且**防篡改**。
- Rekor 永远增长；完整节点持有整棵树。
- 客户端连公共实例 `rekor.sigstore.dev`，也可自建。

### 辅助工具

- **gitsign** —— 用 Sigstore（Cosign + Fulcio + Rekor）
  签 Git commit，替代 GPG；`git commit -S` 的直接替代。
- **policy-controller / kyverno 集成** —— admission
  webhook，拒绝任何 Cosign 签名不符合配置身份的镜像。
- **SLSA provenance** —— Sigstore 工具常用于签名与验证
  SLSA provenance attestation（in-toto），与制品签名并列。
- **TUF 集成** —— TUF 签名元数据可用 Cosign 签。

## 典型签名 / 验证流程

### 签名（发布方）

```
1. CI 构建跑 `cosign sign-blob package.rpm`
2. Cosign 从 IdP 取 OIDC 令牌（如 GitHub Actions 暴露
   `$ACTIONS_ID_TOKEN_REQUEST_TOKEN`）
3. Cosign 把令牌发给 Fulcio
4. Fulcio 返回绑定到 OIDC 身份的短期 X.509 证书
5. Cosign 生成新密钥对，用私钥半签制品，把私钥半扔掉
6. Cosign 把 {签名, 证书, 制品哈希} 提交到 Rekor
7. Rekor 返回 inclusion proof
8. Cosign 把签名、证书、Rekor bundle 与制品并列写出
   （或附着在容器注册表，对 OCI 镜像而言）
```

### 验证（消费方）

```
1. 消费者拉取制品 + 签名 + 证书 + bundle
2. 消费者跑 `cosign verify-blob`（OCI 用 `cosign verify`）
3. Cosign 验证书链：Fulcio 签发，在 ~10 分钟有效期内
4. Cosign 验证书里的 OIDC 身份：匹配预期身份
5. Cosign 验 Rekor inclusion proof：签名已记录，bundle 的
   Merkle 根匹配已签 checkpoint
6. Cosign 用证书的公钥验制品签名
```

每一步独立可查。

## Sigstore vs GPG

| 维度                 | GPG（传统）                                                  | Sigstore（无密钥）                                                  |
| ---                   | ---                                                          | ---                                                                  |
| **核心机制**         | 用长期私钥签名；用对应公钥校验。                              | 用绑定到 OIDC 身份的短期证书（≈10 分钟）签名；通过证书链与透明日志条目校验。 |
| **凭证负担**         | 必须生成、分发、轮换、保护长期私钥。泄露 = 私钥全生命周期内历史伪造风险。 | 每次签名事件动态生成新密钥对，签完立即销毁私钥半。无长期秘密要保护。 |
| **信任根源**         | 公钥 + 公钥的传递通道（团队自控域名 + HTTPS）。                | OIDC 身份提供方（GitHub、Google 等）+ 透明日志（Rekor）。            |
| **审计 / 不可否认性** | 弱：签名只说"这把密钥签了这个制品"，不能公开可验证地说在哪天哪台机器上签的。 | 强：每次签名与 OIDC 身份、制品哈希、时间戳一起写入 Rekor。 |
| **生态适用**          | RPM / DEB / Apt / Pacman 等 OS 级包管理器原生内置 OpenPGP 校验。 | OCI 容器镜像、Kubernetes admission controller、GitHub Actions attestation 等云原生支持。OS 级包管理器 2026 年仍支持有限。 |
| **吊销**             | 可通过 OpenPGP 吊销证书完成，但吊销证书本身的分发是难题。   | 隐式：证书短期设计，无新证书即无新签名，吊销自动。 |
| **信任基底**         | 点对点 —— 任何人都能签、任何人都能 pin；适合个人与"无制度"项目。 | 制度性 —— 锚定 OIDC 身份提供方（GitHub、Google 等），需要先有"已知实体"。 |

### 信任根源分析

**GPG 的信任链**经三层：
1. 数学：签名有效性（密钥对未被篡改时永远成立）。
2. 密钥：消费者导入密钥环的**具体字节**。Fingerprint 是
   对这些字节的 160 位承诺。
3. 通道：消费者怎么拿到这些字节。这才是真正的信任所
   在 —— `https://x-cmd.com/gpg/`、已签名 release
   tarball、已验证的 USB curl 等。

**Sigstore 的信任链**经：
1. 数学：签名对短期证书的有效性。
2. 证书：Fulcio 颁发的 x.509 证书，把 OIDC 身份绑定到
   为本次签名事件生成的公钥。
3. 日志条目：Rekor 记录，把证书、制品哈希、OIDC 身份提
   交到一棵 Merkle 树，其根周期性发布。

### 主观信任：GPG 作为"前制度"机制

GPG 的设计有一个在技术对比中容易被忽视的属性：它是 *
前制度* 信任机制。两方可以在任一方还不是人 / 法人 / 已注
册组织之前，建立密码学可证、法院可采信的"身份与协议"证明。

之所以能这样，是因为 GPG 的信任链是 *点对点* 的 —— 你
选择信任一把具体密钥，是因为你决定信任它，而不是因为某
个机构替它背书。链条最后落在你本人的判断（主观信任）上，
记录后（你把密钥导入了密钥环），数学接手续上。

Sigstore 的设计则最后落在 *OIDC IdP* 上 —— 通常是
GitHub、Google、Microsoft。这些是有法人资格的机构行为方。
Sigstore 中的"签名身份"是 *IdP 同意为之背书的人*。这在多
数商业场景中有用，但有一个前提：IdP 愿意向你颁发身份令
牌，而这通常需要你拥有一个账户、付款方式、组织、或其他
制度性锚点。

具体含义：

- **匿名 OSS 贡献者**可以用 GPG 签自己的代码，下游消费
  者把他的指纹 pin 上，不需要在他处是人或公司。
- 同样的贡献者大概率不能用同样的方式用 Sigstore，因为他
  没有 GitHub 组织或等同的制度性锚点来绑签名身份。

这同时是一个有意义的 *法律* 属性，不只是技术上的。指纹
在许多司法管辖区是 *可采信证据*，证明特定文档或制品由
匹配私钥的持有者签署。信任链是直接的而非中介的。实际
推论：

- **未注册公司前的项目。** 还没完成公司注册的项目可以
  用 GPG 发签名制品；用 Sigstore 通常需要 IdP 组织账户。
- **跨司法管辖区场景。** GPG 密钥跨境无注册责任。
- **法院可采信的证据。** 正确 pin 的 GPG 签名在某些司法
  管辖区比"制度性 attestation"更强的作者证据。
- **Sigstore 的制度性前提。** Sigstore 适合 *由* 公司或
  组织 *生产* 的软件；对个人生产的 OSS 长尾不那么适用。

这不是说 GPG 一般优于 Sigstore —— Sigstore 的制度性锚点
在多数商业场景下是特性。这是要你在选定前想清楚 *你实际需
要的是哪种信任*，并认识到两套范式优化的是不同的场景。

## 适用场景边界

**GPG 是对的选，当：**
- 制品被 OS 级包管理器（`dnf`、`yum`、`apt`、`zypper`、
  `pacman`）消费。
- 生态保守 —— RHEL、CentOS、Rocky、Debian、Ubuntu LTS、
  SUSE、Alpine、嵌入式 Linux 发行版。
- 消费者是跑遗留基础设施的运维，没有 Sigstore 验证工具。
- 签名者没有（或不想要）公司 / 组织身份。

**Sigstore 是对的选，当：**
- 制品是容器镜像、Kubernetes 资源、或软件供应链
  attestation（SLSA / in-toto）。
- 消费者已有 Kubernetes admission controller 或 CI/CD
  集成 Sigstore 验证。
- 团队想要"谁在何时签了什么"的可审计公开追踪。
- 团队有制度性身份（GitHub 组织等）来锚签名身份。

**单独用哪个都不够，当：** 制品需要跨"旧 + 新"生态被消费。
见下文"双重签名"。

## 双重签名策略

**双重签名**（也叫 dual-signing）指对同一制品独立产生两
套签名：每个范式一套。消费者验证与自身工具链匹配的签名；
另一生态的消费者验证另一套。

对一个 RPM，典型流水线：

1. 构建 `.rpm`（未签名）。
2. **GPG 层。** `rpmsign --addsign package.rpm` —— 把
   OpenPGP 签名头嵌入 RPM 元数据。
3. **Sigstore 层。** `cosign sign-blob package.rpm` —— 产出
   detached 签名（`sig`）与证书（`cert`），引用 Rekor 日
   志条目。
5. 发布制品、cosign 的 `sig` + `cert` + bundle，以及公开
   的 GPG 公钥。

### 何时双重签名值得

双重签名值得加 CI / 存储成本，当：

- 制品被**两个**生态同时消费（OS 级包管理器 *和* 云原生
  供应链工具）。
- 团队能消化运营开销：CI 签两次，发布两套制品，维护两
  套验证文档。
- 团队的威胁模型同时覆盖旧与新攻击面。

具体做过的项目：Kubernetes 上游（容器镜像用 cosign、
源码 tarball 用 GPG）；多家 Linux 发行版在 OS 包之外签发
云原生制品；Sigstore 项目自身。

### 何时双重签名过度

双重签名**不**值得，当：

- 受众只在一个生态。
- 制品不跨场景复用。
- 团队负担不起翻倍的 CI 复杂度。

### 混合流水线要点

如果采用双重签名，几个实操考量：

- **顺序敏感。** 先签 GPG（改 RPM 头），再对结果签
  cosign。反过来意味着 cosign 在 sig GPG 前的字节，Rekor
  不覆盖 GPG 签名。设计上是 cosign 覆盖 post-GPG 状态。
- **透明日志覆盖范围。** Rekor 记录签名那一刻的制品哈
  希。cosign 覆盖 post-GPG RPM。仅用 cosign 校验的消费
  者锁的是 post-GPG 状态。
- **密钥轮换成本。** GPG 层仍需密钥轮换故事；Sigstore
  层自轮换。两层有独立的运营节奏。
- **文档维护。** 团队需要维护两套验证路径的文档。

## Sigstore 采用现状（2026）

Sigstore 2023 年从 CNCF 毕业，目前在以下场景生产使用：

- **Kubernetes** 上游 —— 签 release 制品与容器镜像。
- **GitHub** —— `gh` CLI 可用 Cosign 签 release。
- **npm** —— 基于 sigstore 的 attestation。
- **多家 Linux 发行版** —— 与 OS 包 GPG 签名并列。
- **Sigstore 项目自身** —— 用 GPG 与 Cosign 双签自己的
  二进制。

## Sigstore 的局限

- **OIDC IdP 可靠性。** Sigstore 信任链的根落在 OIDC IdP
  上。如果你 GitHub 组织被攻陷，攻击者就能冒充你签发证
  书。
- **Rekor 的膨胀。** Rekor 树 append-only、永远增长。
- **时间窗校验。** Fulcio 证书约 10 分钟过期。事后只能
  对照透明日志与证书信任链验证。
- **OS 级包管理器尚未原生支持。** 2026 年的 `dnf`、`apt`
  等不原生校验 Cosign 签名。
- **无前制度身份。** Sigstore 需要制度性锚点（通常是
  GitHub 组织）。匿名 OSS 贡献者不能以与 GPG 相同的方
  式用 Sigstore 签名。

## 延伸阅读

- [0. x-cmd/gpg overview](./0-x-cmd-gpg-overview.en.md) ——
  本仓库是什么与维护政策。
- [1. 什么是 GPG，怎么用？](./1-what-is-gpg-and-how-do-i-use-it.cn.md) ——
  终端用户入门；fingerprint 是信任锚。
- [3. 年度密钥策略](./3-annual-key-strategy-explained.cn.md) ——
  长期 GPG 密钥轮换权衡。
- [4. GPG UID 命名约定](./4-gpg-uid-naming-conventions.cn.md) ——
  UID 里的 ™/®；品牌防御工作放别处。
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) —— 选定签名策略
  后的发布侧流水线约定。

## 来源

- [Sigstore 项目主页](https://www.sigstore.dev/)
- [Cosign 文档](https://docs.sigstore.dev/policy-controller/overview)
- [Fulcio —— Sigstore 的 CA](https://github.com/sigstore/Fulcio)
- [Rekor —— Sigstore 的透明日志](https://github.com/sigstore/Rekor)
- [RFC 4880 — OpenPGP Message Format](https://datatracker.ietf.org/doc/html/rfc4880)