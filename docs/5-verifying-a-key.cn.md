---
x-title: 验证一把密钥
x-desc: 拉取 → 导入 → 比对 的三步法，含视觉近似攻击、CDN 缓存陷阱、自签 vs 团队签 的区分。
x-sidebar: 验证一把密钥
x-keywords: 验证, fingerprint, 三方交叉比对, 视觉近似攻击, cdn 缓存, 自签, 首次信任
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '验证一把密钥'
      inLanguage: 'zh-CN'
      about: '密钥验证实操'
---

# 验证一把密钥

README 里的 fingerprint **不是** 证据 —— 任何人都能敲出 40
个十六进制字符。验证是一个三步流程，把你导入的字节与团队
意图绑在一起，使用的多个通道不全依赖同一来源。本文逐
步展开方法、陷阱，以及每一步实际证明什么（与不证明什么）。

## 三步法

### 1. 拉取

通过你已信任的传输通道拉取密钥。

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc
```

如果你信任 GitHub，`https://raw.githubusercontent.com` 就够。
要更高保证，可以同时从第二个来源拉同一文件 —— 团队官网、
已签名 release tarball、同事已校验的 clone —— 验证字节相同。

**不要** 走任何 CDN、反向代理或缓存服务。见
[`LICENSE`](../LICENSE) 与 FAQ 的信任锚点推理。

### 2. 导入

把拉到的字节管道给 GnuPG：

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import
```

GnuPG 解析 OpenPGP packet 流，验证每把密钥的自签（详见下
文），把密钥加入本地密钥环。

### 3. 比对

GnuPG 给你的 fingerprint 必须与 `index.tsv` *以及* 团队
官网上的值一致。三方必须吻合。

```sh
# 本地密钥环里的 fingerprint（导入后）
gpg --list-keys --with-colons keyring/keyring.asc \
  | awk -F: '/^fpr:/{print $10}'

# 本仓库 index.tsv 里的 fingerprint
awk -F'\t' 'NR > 0 { print $3 }' index.tsv

# 团队官网的 fingerprint
curl -fsSL https://x-cmd.com/gpg/ | grep -oE '[0-9A-F]{40}'
```

三方 fingerprint 必须完全一致。任一不一致，**停下来**排查
后再决定是否信任密钥。

## 每一步证明什么

| 步骤 | 证明                                                       | 不证明                                       |
| ---  | ---                                                       | ---                                         |
| 拉取 | GitHub 给的字节就是你拿到的字节                           | 这些字节是团队发布的                       |
| 导入 | 这些字节是合法 OpenPGP 密钥，且自签有效                  | 这把密钥是 *团队* 的，不是冒充者的          |
| 三方比对 | 你拿到的字节与团队在独立来源发布的字节密码学一致          | *团队* 本身是他们声称的那个人（不在本文范围） |

"比对"是承重步骤。少了它，导入一把密钥只能证明你拿到了语法
合法的 OpenPGP 二进制 —— 冒充者轻松就能满足这一档。多了它，
你就在三个来源上达成密码学一致。

## 自签 vs 团队签

GnuPG 的 `gpg --verify <key>.asc`（或在干净密钥环上直接
`gpg --import`）验证的是密钥的 *自签* —— 由密钥自身的私
钥半盖在自身公钥包上的签名。每个 OpenPGP 密钥都有自签 ——
GnuPG 借此知道密钥结构上没坏。

这 *不是* 团队对密钥的背书。自签说"这把密钥结构合规"，不
说"这把密钥属于 x-cmd"。要团队背书，需要：

- 本仓库 `keyring/<handle>.asc` 中的字节与团队 commit 历史
  对得上。
- fingerprint 与 `index.tsv` 与团队官网一致。
- （更高保证）另一位团队成员密钥的交叉签名 —— 这是
  [2. keyring 如何发布](./2-how-the-keyring-is-published.cn.md#密钥轮换)
  所述轮换流程中团队发布的。

首次验证单把密钥时，上面三条一般过度 —— 三方比对就够。
在关键生产环境上线或合规审计时，再叠加交叉签名检查。

## 常见陷阱

### CDN 缓存

最常见的"在不知不觉中导入 *陈旧* 密钥"的途径。你与
GitHub 之间的 CDN 或反向代理可能返回几秒前、几分钟前、
几小时前的字节 —— 包括早于当前轮换的字节。fingerprint 看
起来仍合法（字节仍是真 OpenPGP 密钥），但不与团队当前的
`index.tsv` 匹配。

**防御**：每次直接从 GitHub 拉，不走中间。比对独立来源
（团队官网）。见 [`LICENSE`](../LICENSE) 显式的代理分发
条款。

### 视觉近似密钥

攻击者构造一把 UID 与团队一致（同名、同邮箱）但
fingerprint 不同的密钥。你在公开 keyserver 上
`gpg --search-keys <邮箱>`，看到匹配的 UID，导入 —— 没
核 fingerprint。

**防御**：永远不要去 keyserver 搜已知团队的密钥。直接
从本仓库拉。以 fingerprint 为锚。

### handle 复用

团队轮换一把密钥；新密钥取相同的 handle。你的工具链以
"official key" 为锚，匹配上新的 fingerprint —— 但你的审
计日志现在把旧 fingerprint 与新 fingerprint 混着用。

**防御**：以 fingerprint 为锚，不要以 handle 为锚。回顾
自己的审计日志时，永远用 fingerprint 作标识。

### 短 key ID

某些老配方以 32 位短 key ID（`0xDEADBEEF`）为锚。把 160 位
截断到 32 位在实践中可逆 —— 攻击者可构造一把截断后短 ID
与目标匹配的密钥。别用短 key ID 作锚。

**防御**：永远用完整的 40 字符 fingerprint。

## 首次信任（TOFU）

如果你无法建立三方比对（比如没办法访问团队官网），你就被
退回"信任 GitHub 给的字节"。多数消费者场景这就够了 ——
`https://raw.githubusercontent.com` 是 GitHub 的 HTTPS 端
点，由 GitHub 证书锁定，由你操作系统的信任库验证。剩下的
攻击面是"GitHub 给错字节"，GitHub 在平台层面把它当安全
事件对待。

如果你的威胁模型包含"GitHub 给错字节"，你需要带外验证
—— 通常是团队其他渠道发布的一段已签名消息，确认新
fingerprint。团队官网在密钥轮换时发布这类消息。

## 延伸阅读

- [6. 消费 keyring 的三种方式](./6-three-ways-to-consume.cn.md) ——
  原生 curl、`x gpg`、GitHub Pages 经 x-cmd.com 跳转。
- [3. 解读密钥目录](./3-reading-the-key-catalog.cn.md) ——
  fingerprint 作为密码学承诺的细节。

## FAQ

本节从
[文章 0 的中心 FAQ](./0-x-cmd-gpg-overview.cn.md#faq--软件分发与代码签名密码学)
里挑出与本文最相关的子集。完整的 8 问在文章 0，
答案以行业普遍视角书写，与具体项目无关。

### Q1：GPG 软件与 GPG Key 在技术上是什么关系？

GPG 是执行密码学操作的程序；keypair 是持有密码学材料的
数据凭证。校验只使用密钥对的 *公钥* 半 —— 对应的私钥半
永远不需要离开发布方的基础设施。完整答案见文章 0。

### Q8：采用"一年一换"密钥模型，工业界如何解决跨年过渡与历史回滚？

标准模式是 **Trust Anchor Registry（信任锚点注册表）**，
杠杆点在用户侧的密钥环：发布方把所有历年年度公钥合并进
单个 keyring，消费者把 keyring 导入自己的 `gpg`。由去年
密钥签名的制品之所以在只导入了今年密钥的主机上仍能校验，
正是因为该主机的用户自己选择同时保留历史密钥 —— 这就是
密钥环归用户所有的全部意义。完整答案见文章 0。