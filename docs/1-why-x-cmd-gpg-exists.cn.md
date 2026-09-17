---
x-title: x-cmd/gpg 为何存在
x-desc: 给新人 —— 什么是 GPG 信任锚点，本仓库解决的供应链问题，以及 x-cmd 为何维护自有 keyring 而非使用 keys.openpgp.org 等。
x-sidebar: x-cmd/gpg 为何存在
x-keywords: gpg 入门, 信任锚点, 供应链, 签名验证, 为何独立仓库, x-cmd
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'x-cmd/gpg 为何存在'
      inLanguage: 'zh-CN'
      about: 'x-cmd/gpg 用途与理由'
---

# x-cmd/gpg 为何存在

为 GPG 新手、供应链密钥新手或 x-cmd 新人准备的入门。
读完后你应该能向同事（或采购方）解释：什么是 GPG 信任锚
点、x-cmd 为何需要自己的 keyring 仓库、如果改用通用公钥
服务器会出什么问题。

## "信任锚点"的实操含义

GPG 公钥是一串字节，导入 GnuPG 后，本地安装就**愿意校验**
由对应私钥生成的签名。一旦导入并标记为受信任，你下载的
每一个带有该私钥有效签名的字节都能通过本地的
`gpg --verify` 校验。

整个游戏规则就是这些。没有 CA、没有吊销服务器、没有
"什么是合法 x-cmd 密钥"的中央注册机构 —— 你导入一把密
钥的决定，以及你相信"导入的字节就是团队发布的字节"，就
是整个信任模型。

所以我们说"信任锚点"，意思是：公钥的精确字节、由你独立
验证过的团队自有渠道发布。其他一切 —— fingerprint 格式、
密钥过期、web-of-trust 签名 —— 都是"我有没有拿对字节"
的下游问题。

## 本仓库解决的供应链问题

x-cmd 发布的制品都带签名：RPM、DEB、容器镜像、release
tarball。每个签名制品都带签名头，意思是"该私钥的持有者
签署了这个制品"。消费者的任务是：用一把他们信任的公钥
校验该签名。

要让这件事成立，两个前提必须成立：

1. 制品上的签名确实来自 *x-cmd 的* 私钥（不是视觉近似
   的冒充者）。
3. 消费者用来校验的公钥确实 *是* x-cmd 的公钥（不是被
   替换的冒充副本）。

前提 1 由 GPG 数学保证 —— 签名在密码学上绑定到具体那把
私钥。前提 2 由 *本仓库* 保证 —— 一个由团队控制的、单一
权威、公钥管理位发布字节的地方，并以团队自身的 commit
历史与签名 release 作为审计追踪。

如果前提 2 不成立，整条链就崩了 —— 你把签名校验降级成了
"信任把密钥发给你的人没有撒谎"。这不是安全属性，只是
希望。

## 为什么要单独的仓库

x-cmd 代码库在 `x-cmd/x-cmd`，是个大型 monorepo，每个
`x gpg` 用户、每个包构建、每个 release pipeline 都要 clone。
把团队 GPG 公钥放进 `x-cmd/x-cmd` 会带来三个问题：

1. **循环签名。** `x-cmd/x-cmd` 的发布工作流拉取签名密钥
   去签 release 制品。如果这些密钥住在它们自己签名的仓
   库里，你就把密钥分发降级到了"信任未签名的引导链" ——
   这是一类真实的供应链弱点。
2. **仓库体积与变更。** monorepo 每周变更上千次。从这里
   拉密钥意味着每个消费者也要拉取并校验那些变更 —— 你
   的密钥校验就要依赖于对 monorepo 全量历史的信任。信任
   面缩小了。

   单一职责的 `x-cmd/gpg` 仓库只在添加或轮换密钥时变更
   —— 一年也就几次 commit。
3. **独立的评审面。** 轮换频率低但安全关键。小的、聚焦
   的仓库配上自己的 diff 历史，让第二位团队成员评审"这
   真的是新密钥？"无需扫数千条无关变更。

所以结论是：把信任锚点放在自己的仓库里，字节稳定、评审
面小、伪造密钥的唯一路径就是"说服团队 commit 进去" ——
团队没经过内部评审是不会这样做的。

## 为什么不直接用 keys.openpgp.org / keyserver.ubuntu.com

公开 keyserver 是个伟大的通用工具：如果你不知道某人的密
钥在哪，去 keyserver 搜。但对 *团队* 向 *自有消费者* 分
发自有密钥而言是错工具，四个理由：

1. **谁都能上传。** keys.openpgp.org 的上传是自证的 ——
   任何拿到字节的用户都能放到服务器上。服务器不验证上
   传是否来自团队。
2. **替换攻击。** 一旦上传，攻击者可以再上传一把 *视觉
   近似* 的密钥，UID 几乎相同但 fingerprint 不同。按名
   字 / 邮箱 `gpg --search-keys` 的消费者可能选错。
3. **没有第一方审计追踪。** keyserver 不以映射回团队
   commit 历史的方式保留 *谁* 在 *何时* 上传的。本仓库
   的 `git log` 做到了。
4. **LICENSE 冲突。** 公开 keyserver 是通用基础设施；
   它们从任何上传者重新分发字节。本仓库的 LICENSE 显
   式禁止第三方再分发。用 keyserver 作为分发渠道违反
   LICENSE。

对 99% 已经知道要 x-cmd 团队密钥的用户，答案就是：从
https://github.com/x-cmd/gpg（或经由 GitHub-Pages-via-
x-cmd.com 的跳转）拉取。不要去 keyserver 搜。

## 本仓库 *不是* 什么

- *不是* 通用 keyserver。我们不接受上传，不镜像
  keys.openpgp.org，不从 keyserver.ubuntu.com 拉取。
- *不是* 吊销权威。被攻陷密钥的报告走团队的私有渠道，
  不走本仓库。
- *不是* 签名存档。签名 commit 与签名 release 制品住在
  产生它们的上游仓库。本仓库只发布密钥对的 *公钥半*。

## 延伸阅读

- [2. keyring 如何发布](./2-how-the-keyring-is-published.cn.md) ——
  从 `gpg --export` 到 `index.tsv` 中的一行，团队内部
  流程。
- [3. 解读密钥目录](./3-reading-the-key-catalog.cn.md) ——
  `index.tsv` 的每一列与背后的 fingerprint 数学。
- [4. 年度密钥策略详解](./4-annual-key-strategy-explained.cn.md) ——
  FAQ Q4–Q8 的长文版，含"密码学永不过期 / 运营年度轮换"
  的决策。
