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