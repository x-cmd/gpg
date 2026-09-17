---
x-title: x-cmd/gpg —— 总览
x-desc: x-cmd 团队 GPG 公钥串链 —— 一页摘要，链接到下方深度文章。
x-sidebar: x-cmd/gpg 总览
x-keywords: x-cmd, gpg, 公钥, 信任锚点, 供应链, 签名验证, keyring
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'x-cmd/gpg —— 团队 keyring'
      inLanguage: 'zh-CN'
      about: 'x-cmd 团队 GPG 公钥串链'
---

# x-cmd/gpg —— 团队的 keyring

> x-cmd 核心团队 GPG 公钥集合的权威、单一可信来源。**请
> 从本仓库直接拉取** —— 由其他域名提供的每一个字节都视
> 为不可信。代理分发禁止条款与信任锚点理由见
> [`LICENSE`](../LICENSE)。

本页为一页版摘要。下方文章会深入 *为什么* 与 *怎么做*：
什么是 GPG 信任锚点、keyring 如何发布、如何读
`index.tsv`、为什么团队同时发布社区版密钥与年度企业版
密钥、如何验证 fingerprint，以及消费 keyring 的三种
方式。

## 当前密钥

| Handle | UID | Fingerprint | Created | Purpose |
| --- | --- | --- | --- | --- |
| _(暂未发布密钥 —— 见_ [`CONTRIBUTING.md`](../CONTRIBUTING.md)_)_ | | | | |

正式表格由团队在每次发布提交时从
[`index.tsv`](../index.tsv) 自动生成。**以 fingerprint 为
锚，不要以 handle 为锚** —— handle 在轮换后可能被复用，
fingerprint 不会。

## 团队发布的两把密钥（FAQ Q4）

团队供应链密钥策略发布两把签名密钥，各有不同运营定位：

- **社区版密钥** —— 签名标准社区包（`x-cmd.rpm` /
  `x-cmd.deb`）。一次导入，未来升级静默验证。
- **年度密钥（`key-<year>`）** —— 签名年度企业合规包
  （`x-cmd-annual-<year>.rpm`）。严格的逐年隔离，满足
  金融 / 政企采购审计。

两把密钥都发布在本仓库 [`keyring/`](../keyring/) 下，
汇总为 [`keyring/keyring.asc`](../keyring/keyring.asc)。
下方文章会展开：为何这样拆分、为何密码学上"永不过期"
但运营上"年度轮换"、付费 LTS 客户依赖的重签发机制。

## 三种消费方式

```sh
# 1. 直接拉取 —— 从 GitHub 拉 keyring.asc
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import

# 2. 单把密钥导入
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/<handle>.asc \
  | gpg --import

# 3. x gpg shell 模块（自动缓存 + fingerprint 三向交叉验证）
x gpg import
x gpg info <handle>
x gpg verify <sig> <file>
```

**直接从 GitHub 拉取。** 不要走任何 CDN、反向代理或缓存
服务 —— 见 [`LICENSE`](../LICENSE) 的代理分发条款。

## 延伸阅读

- [1. x-cmd/gpg 为何存在](./1-why-x-cmd-gpg-exists.cn.md) ——
  本仓库解决的供应链问题
- [2. keyring 如何发布](./2-how-the-keyring-is-published.cn.md) ——
  从 `gpg --export` 到 `index.tsv` 中的一行，团队内部流程
- [3. 解读密钥目录](./3-reading-the-key-catalog.cn.md) ——
  `index.tsv` 的每一列、fingerprint 数学、为何以
  fingerprint 为锚
- [4. 年度密钥策略详解](./4-annual-key-strategy-explained.cn.md) ——
  FAQ Q4–Q8 的长文版
- [5. 验证一把密钥](./5-verifying-a-key.cn.md) ——
  拉取 → 导入 → 比对 的三步法
- [6. 消费 keyring 的三种方式](./6-three-ways-to-consume.cn.md) ——
  原生 curl、`x gpg`、GitHub Pages → x-cmd.com 跳转

技术参考（文件布局、schema、CI）见
[`CONTRIBUTING.md`](../CONTRIBUTING.md)。AI agent 用法与
命令行速查见 [`SKILL.md`](../SKILL.md)。

## FAQ —— 软件分发与代码签名密码学

以下问答具有科普性质，与具体项目无关。它们客观陈述
GPG 密钥管理、寿命设计与软件分发供应链安全在工业界的
主流现状、各方案优劣与权衡点，不偏向任何一种方案。文
章 1–6 在各专题上分别深入；本文是完整 8 问的权威参考。

### Q1：GPG 软件与 GPG Key（密钥）在技术上是什么关系？

是软件程序与数据凭证的关系。

- **GPG (GNU Privacy Guard)** 是实现了 OpenPGP 国际标准
  的开源加密软件。它执行加密、解密、生成数字签名与校
  验签名等具体的 *计算动作*。
- **GPG Key（密钥对）** 是该软件运行所需的数据凭证，包
  含一个可公开的 *公钥*（他人用来向你加密或验证你的签
  名）与一个必须严格保密的 *私钥*（你用来解密或创建签
  名）。

两者在实践中不可分割 —— GPG 无密钥无可用材料，密钥无
GPG 无可执行的操作。

### Q2：既然现代供应链出现了 Sigstore（无密钥签名），为什么 RPM 和 DEB 包分发依然高度依赖 GPG？

由操作系统的历史兼容性与原生工具链生态决定。

- **Sigstore** 广泛应用于现代云原生环境（Docker 镜像、
  Kubernetes 组件、npm / PyPI 依赖包）。其核心是通过短
  期临时证书与公开透明日志（Rekor）消除对长期私钥的管
  理依赖。
- **RPM (dnf / yum)** 与 **DEB (apt)** 是主流 Linux 发行
  版的原生基础包管理器。其底层安全校验引擎在设计之初就
  深度绑定了 OpenPGP (GPG) 标准，至今 100% 依赖 GPG 密
  钥对包或源索引文件进行数字签名。

两者并存；OS 级包分发的事实渠道仍是 GPG。

### Q3：为什么软件发布时通常不直接分发未签名的裸包？

**Pro（优点）**

- 开发者完全没有密钥管理开销，发布流程极简；
- 用户或企业可完全按自身内网安全策略离线重新给包签名。

**Con（缺点）**

- 网络传输与 CDN 节点缺乏密码学防篡改保护，MITM 与投毒
  极易；
- 多数现代 Linux 发行版的包管理器遇到未签名包时默认弹窗
  报错并拒绝安装，增加用户运维摩擦。

### Q4：硬编码 GPG 密钥为"永不过期（Never Expire）"有哪些优缺点？

**Pro（优点）**

- **极致的业务连续性**：多年前部署的服务器在任何未来时刻
  重新执行检查或环境恢复时，永远不会因"发布者密钥到期"
  而引发自动化运维脚本崩溃。
- **极低的维护成本**：发布者无需在每年特定时间点更新
  CI/CD 流水线密钥，也无需高频发布公告提醒全球用户更新
  公钥。

**Con（缺点）**

- **爆炸半径无限大**：一旦发布者的本地开发机或构建服务
  器中毒、私钥泄露，黑客可无限期伪造任意未来新版本。恢
  复依赖极难事后分发的"吊销证书（Revocation Certificate）"
  机制。

### Q5：历史上许多证书 / 密钥的有效期为什么设定为"398 天"或"397 天？

源于 CA/Browser Forum、Apple 与 Google 对公开信任的 Web
证书（SSL/TLS）施加的强制寿命限制。

- **398 天（密码学设计）**：自 2020 年起，国际标准强制规
  定一年期 Web 证书最大生命周期不得超过 398 天 —— 365
  天基线 + 33 天跨年假期与多时区缓冲。
- **397 天（工程实践）**：全球服务器存在时区漂移，部分自
  动化合规扫描器在卡点计算 398 天时可能因几小时时差误
  报"证书超期"。审慎的工程师在实践中硬编码 397 天，主动
  让出 1 天的防御性退让，换取全球扫描器的 100% 绿灯通过
  率。

### Q6：2026 年网络数字证书（SSL/TLS）标准发生了什么重大变化？代码签名受此影响吗？

- **Web 证书暴跌**：自 2026 年 3 月起，全球公开信任的 Web
  证书最大有效期已被压缩到 100 天左右（部分标准 200 天以
  内），旨在通过自动化轮替消灭长期密钥。
- **代码签名合规豁免**：国际主流根证书计划与 OS 底层安全
  审计规范明确规定，包签名与代码签名属于基础设施锚点，
  不参与 Web 证书的寿命缩减计划。在 Linux 包分发与企业
  合规审计领域，1 到 2 年长期密钥轮替仍是当前的行业主流
  实践。

### Q7：商业软件采用"一年一换（Annual Key）"密钥隔离有哪些优缺点？

**Pro（优点）**

- **高安全性与合规性**：完美对齐绝大多数金融、政企甲方要
  求的"年度 IT 资产审计（Annual Security Audit）"指标。
  即使某年私钥泄露，风险也被完美阻断在该年度版本内。
- **商业黏性**：按年强制更新信任源，可作为企业级客户续订
  "技术支持与安全服务保障合同"的天然技术纽带。

**Con（缺点）**

- **老系统兼容摩擦**：老系统在跨年时若直接删除旧钥匙，历
  史版本软件在例行依赖扫描时会因找不到签名源而报错。
- **双重签名（Dual Signing）困境**：若尝试同时签入新老两
  把钥匙，不同 Linux 发行版校验引擎（旧版 CentOS 与新版
  Rocky Linux）解析逻辑不一，易在生产环境引发未知故障。

### Q8：采用"一年一换"密钥模式，工业大厂如何在技术上解决跨年过渡与历史版本回滚？

标准模式是 **Trust Anchor Registry（信任锚点注册表）**：

- **公钥库常驻**：在官网开辟专门的凭证路径（公开数据仓
  库或专用 CDN 路径），将所有历年年度公钥（`key-2025.gpg`、
  `key-2026.gpg` …）合并为单个 keyring。
- **控制权交还用户**：要求企业用户的系统同时导入今年与明
  年的公钥。系统于是同时拥有历史与未来密钥；无论老系统
  升级新版本，还是干净系统回滚安装历史旧包，包管理器都
  能本地开锁；将"是否强制让旧密钥失效"的最终审计决定权
  完整交还给企业自己的运维策略。