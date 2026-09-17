---
x-title: Cosign 与 Sigstore 详解
x-desc: Sigstore 是什么（项目）、Cosign 是什么（CLI）、Fulcio（CA）与 Rekor（透明日志）如何嵌进签名工作流，以及典型签名与验证流程长什么样。**项目中立；纯探讨。**
x-sidebar: Cosign 与 Sigstore 详解
x-keywords: sigstore, cosign, fulcio, rekor, 透明日志, 无密钥签名, oci, 容器, slsa, 供应链, cncf
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Cosign 与 Sigstore 详解'
      inLanguage: 'zh-CN'
      about: 'Sigstore 生态与 Cosign CLI'
---

# Cosign 与 Sigstore 详解

Sigstore 与 Cosign 是 2020 年代供应链签名转变中最显眼的两
块：从"长期私钥签名"转向"短期证书 + 透明日志"签名，锚
定到既有的制度性身份（GitHub、Google 等）。本文走查整套
生态 —— 每块是什么、做什么、以及典型签名 / 验证流程的形
状。

本文刻意不在 Sigstore 与 GPG 之间选边。两者对比见
[9. Sigstore vs GPG，以及双重签名策略](./9-sigstore-vs-gpg-and-double-signing.cn.md)。

> **状态：探讨。** 这是项目中立地对 Sigstore 与 Cosign
> 的解释。x-cmd 团队截至本文撰写时未在任何实际发布签名
> 中采用 Sigstore。

## 一段话总结

**Sigstore** 是一个项目（源自 Google，现为 CNCF 毕业项目，
归 Linux 基金会），提供 *无密钥* 软件签名 —— 短期证书绑
定到 OIDC 身份，所有事件记入公开 append-only 日志。

**Cosign** 是 Sigstore 的 CLI 工具，对容器镜像、二进制
blob、attestation 等制品做签名与验证。

**Fulcio** 是颁发短期签名证书的 CA。

**Rekor** 是公开记录每次签名事件的透明日志。

四者合起来，让团队发布签名制品而 *无需管理任何长期私钥*
 —— 代价是签名身份锚定到 OIDC IdP（通常 GitHub 或
Google），不是团队持有的密钥。

## 三块核心

Sigstore 生态有三块核心，外加若干辅助工具。

### Cosign —— CLI

Cosign 是面向用户的工具。做两件事：

1. **签名** —— 产出签名、证书、透明日志条目。
2. **验证** —— 检查签名确实由特定身份产生并已记入 Rekor。

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

Fulcio 颁发短期签名证书。你跑 `cosign sign --keyless` 时发
生以下步骤：

1. Cosign 向你的 OIDC IdP（典型为 GitHub Actions 或
   `gh auth login`）请求 OIDC 身份令牌 —— 一份已签 JWT，承
   诺"我是组织 Y 的用户 X"。
2. Cosign 把该令牌发给 Fulcio。
3. Fulcio 校验 OIDC 令牌（IdP 的签名链成立），然后颁发
   一份 X.509 签名证书，把 OIDC 身份（如
   `https://github.com/example/.github/workflows/release.yml@refs/tags/v1.2.3`）
   绑定到一份为本次签名事件新生成的公钥。
4. 证书过期时间约 10 分钟（Fulcio 默认配短期证书寿命）。
5. Cosign 用匹配的私钥签制品，然后把私钥扔掉。

签好的制品、签名、证书随后发给 Rekor（下一节）做透明日志
登记。

Fulcio 由 Sigstore 项目自身运营（同一家运营 Rekor）。Fulcio
自身的信任来自其根证书，由社区发布与审计。

### Rekor —— 透明日志

Rekor 是 Sigstore 网络处理过的每次签名事件的 append-only
哈希链接 Merkle 树。你用 Cosign 签名时，签名、证书、制品
哈希被包成一个 bundle 提交为 Rekor 条目；Rekor 返回一份
把该条目绑到 Merkle 根的 inclusion proof。

Rekor 树周期性发布为已签 checkpoint，任何人都能把它对
照更早的 checkpoint（最终对照已知 URL 发布的"信任根"）做
校验。验证方在本地重建同一棵树，检查该条目已被包含。

实际含义：

- 每次签名事件**公开**且**防篡改**。Fulcio 颁的证就在
  Rekor 里；Rekor 里有它，inclusion proof 就验得上。
- Rekor 永远增长。完整节点持有整棵树。Rekor 全节点的
  运营者承诺长期存储。
- Rekor 由 Sigstore 项目运营；客户端连公共实例
  `rekor.sigstore.dev`，也可自建。

### 辅助工具

另有一些周边工具：

- **gitsign** —— 用 Sigstore（Cosign + Fulcio + Rekor）签
  Git commit，替代 GPG。是 `git commit -S` 的直接替代。
- **policy-controller / kyverno 集成** —— Kubernetes
  admission webhook，拒绝任何 Rekor/Cosign 签名不符合配
  置身份的镜像。
- **SLSA provenance** —— Sigstore 工具常用于签名与验证
  SLSA provenance attestation（in-toto），与制品签名并列。
- **The Update Framework (TUF) 集成** —— TUF 签名的元数据
  可用 Cosign 签。

## 典型签名 / 验证流程

把三块合起来，典型流程：

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
4. Cosign 验证书里的 OIDC 身份：匹配预期身份（如
   `https://github.com/example/.github/workflows/release.yml@refs/tags/v1.2.3`）
5. Cosign 验 Rekor inclusion proof：签名已记录，bundle 的
   Merkle 根匹配已签 checkpoint
6. Cosign 用证书的公钥验制品签名
```

每一步独立可查。消费者除信任已发布的 Merk 树真实（根植
于公钥签 checkpoint）外，不需信任 Sigstore 服务器。

## OIDC 身份契约

Sigstore 最具后果的设计抉择是 *签名证书绑定到哪个身份*。
OIDC 令牌含类似以下 claim：

- `email` —— 用户邮箱（人类签名者）
- `sub` —— OIDC 主体标识
- URL 形"identity" claim，唯一标识 workflow、仓库、ref 等
  —— 用于 CI 签名者

GitHub Actions workflow 的典型签名身份形如：

```
https://github.com/example/app/.github/workflows/release.yml@refs/tags/v1.2.3
```

验证方 pin 那个身份。"谁签了这个制品"的指纹就成了"在仓库 R
的提交 Z 上跑 workflow Y 的 GitHub 用户 X" —— 制度意义上的
*事件*，而非长期密钥。

发布方与验证方之间的契约：
- 发布方承诺 *谁* 来签（OIDC 身份）。
- 验证方 pin 该身份。
- 数学与透明日志保证签名在记录时间点由该身份产生。

这正好是 GPG 契约的反面 —— GPG 里发布方承诺 *哪把密钥* 签，
验证方 pin 密钥字节。

## 采用现状（2026）

Sigstore 2023 年从 CNCF 毕业，目前在以下场景生产使用：

- **Kubernetes** 上游 —— 签 release 制品与容器镜像。
- **GitHub** —— `gh` CLI 可用 Cosign 签 release。
- **npm** —— 基于 sigstore 的 attestation。
- **多家 Linux 发行版** —— 与 OS 包 GPG 签名并列。
- **Sigstore 项目自身** —— 用 GPG 与 Cosign 双签自己的二进制。

Cosign UI 在 2023–2026 间已稳定；透明日志与 Fulcio 基础设施公开可审计。大部分工具集成已沉淀在 `cosign sign` / `cosign verify` 两条规范命令上。

## 局限与坑

几点提醒：

- **OIDC IdP 可靠性。** Sigstore 信任链的根落在 OIDC IdP
  上。如果你 GitHub 组织被攻陷，攻击者就能冒充你签发证
  书。防御是 *加固 OIDC 身份*（组织安全、分支保护、强制
  2FA 等）。
- **Rekor 的膨胀。** Rekor 树 append-only、永远增长。全节
  点持有整棵树；轻量客户端对照已发布 checkpoint 验证
  inclusion proof。
- **时间窗校验。** Fulcio 证书约 10 分钟过期。意味着签名
  在签名那一刻是 *自证* 的，但你无法在事后问"这个签名还
  有效吗？"而是不查透明日志与证书信任链。
- **OS 级包管理器尚未原生支持。** 2026 年的 `dnf`、`apt`
  等不原生校验 Cosign 签名。OS 级分发仍是 GPG 的场子。
- **无前制度身份。** Sigstore 需要制度性锚点（通常是
  GitHub 组织）。没有 GitHub 账号的匿名 OSS 贡献者，不能
  以与 GPG 相同的方式用 Sigstore 签名。

## 延伸阅读

- [9. Sigstore vs GPG，以及双重签名策略](./9-sigstore-vs-gpg-and-double-signing.cn.md) ——
  与 GPG 的对比与混合策略。
- [7. 用 GPG 给 RPM 包签名](./7-signing-an-rpm-with-gpg.cn.md) ——
  典型双重签名流水线中的 GPG 侧对应物。
- [`CONTRIBUTING.md`](../CONTRIBUTING.md) —— 选定签名策略
  后，密钥 / 身份如何发布。

## 来源

- [Sigstore 项目主页](https://www.sigstore.dev/)
- [Cosign 文档](https://docs.sigstore.dev/policy-controller/overview)
- [Fulcio —— Sigstore 的 CA](https://github.com/sigstore/Fulcio)
- [Rekor —— Sigstore 的透明日志](https://github.com/sigstore/Rekor)
- [CNCF Sigstore 毕业公告](https://www.cncf.io/announcements/2023)