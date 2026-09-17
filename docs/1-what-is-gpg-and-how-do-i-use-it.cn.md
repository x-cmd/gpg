---
x-title: x-cmd/gpg 如何保护用户 —— 以及你如何使用
x-desc: x-cmd 团队的 GPG keyring 如何保护终端用户免受供应链攻击，以及如何实际使用 —— 装签名包、校验 release、把 fingerprint pin 到独立参考源。
x-sidebar: x-cmd/gpg 如何保护用户
x-keywords: gpg, 什么是gpg, gpg 教程, 公钥, 私钥, gpg key, gpg 签名, gpg 校验, 导入, x gpg, 终端用户, 供应链, 信任锚
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'x-cmd/gpg 如何保护用户 —— 以及你如何使用'
      inLanguage: 'zh-CN'
      about: 'x-cmd/gpg 如何保护终端用户以及他们如何使用 keyring'
---

# x-cmd/gpg 如何保护用户 —— 以及你如何使用

x-cmd 团队发布这个 keyring 仓库的工作是：当你把 x-cmd 装
到机器上时，让你拿到密码学证据 —— 字节来自我们，而不是来自
冒充者。本文走查 *我们如何保护你*（信任模型是什么、团队
承诺是什么）与 *你实际怎么用*（装签名包、校验 release、
把 fingerprint pin 到独立参考源）。

本文假设你用的是 Linux/macOS，并已装 `gpg`（或
`gpg2`）。大多数发行版默认装；没装就
`dnf install gnupg2` / `apt install gnupg2`。

## 我们如何保护你

信任模型刻意做得小而显式。三件事一起让系统安全：

1. **本仓库是真源。** 这仓库是团队维护 keyring 的唯
   一场所。我们不另开 keyserver，不通过 CDN 分发。每
   一个公钥字节都来自本仓库。
2. **消费者通过团队自控域名的 HTTPS 拉取。**
   `https://raw.githubusercontent.com/x-cmd/gpg/...` 与
   `https://x-cmd.com/gpg/...` 是团队背书的两条唯二渠道。
   两者指向同一字节 —— 第一条是源，第二条是部署时从本仓
   库构建出的呈现层。
4. **消费者把 fingerprint pin 到独立参考源。** 导入密钥
   是不够的 —— 字节可能错。fingerprint 必须匹配 *独立*
   来源（你自己的带外知识、`index.tsv` 参考、团队官网）。
   三方比对 —— 本地密钥环 vs `index.tsv` vs 团队官网
   —— 是承重的一步。

团队承诺在实际中的样子：

- **只有团队能改 `keyring/` 或 `index.tsv`。** 外部 PR
  直接关闭、不合并。这是供应链原则，不是客气 —— 一条
  PR 把真 fingerprint 换成视觉近似的，是攻击，不是贡
  献。
- **密钥轮换是团队的责任。** 密钥到期或轮换时，团队
  在本仓库发布新密钥并更新 `index.tsv`。旧密钥保留在
  `keyring/archive/` 里，历史校验仍能通过。
- **用大白话写文档，不做营销。** 文档解释我们做什么
  和为什么。如果你发现空缺或可改进处，请发 issue ——
  我们珍视反馈。

我们 *不* 做的事：

- **不上传到 keys.openpgpg.org 或 keyserver.ubuntu.com。**
  密钥通过本仓库分发。如果你在公开 keyserver 上找到我们
  的密钥，那是别人放上去的。
- **不背书任何第三方镜像或 CDN 代理。** 直接从
  `raw.githubusercontent.com` 或 `x-cmd.com` 拉。代理分发
  条款见 LICENSE 页脚。
- **不表态你是否该在主机上保留旧密钥。** 那是用户侧的
  决定。哲学见 [5. Sigstore、Cosign 与双重签名](./5-sigstore-cosign-and-double-signing.cn.md)。

## GPG 是什么，一句话

**GPG**（GNU Privacy Guard）是一个实现了 **OpenPGP** 标
准的开源加密程序。它执行密码学动作：加密、解密、生成签名、
校验签名。它是程序。

**GPG 密钥**是该程序操作的数据凭证 —— 一对密钥，包含
**公钥**（可公开）与 **私钥**（必须保密）。公钥验证由匹
配私钥做出的签名；私钥是用来签东西的。

如果你只能记住本文一句话，那应该是：**fingerprint 是密
钥的密码学身份**。它是你跑 `gpg --list-keys --fingerprint`
时看到的 40 字符十六进制串（如 `4E1C1B9E5C5F0A2D7B3C...`）。
两把 fingerprint 相同的密钥按定义就是同一把 —— 没有其他身
份来源。如果你 pin 到正确的 fingerprint，你就 pin 到了正
确的密钥。

**GPG**（GNU Privacy Guard）是一个实现了 **OpenPGP** 标准
的开源加密程序。它执行密码学动作：加密、解密、生成签名、
校验签名。它是程序。

**GPG 密钥**是该程序操作的数据凭证 —— 一对密钥，包含
**公钥**（可公开）与 **私钥**（必须保密）。公钥验证由匹
配私钥做出的签名；私钥是用来签东西的。

两者在实践中不可分割：GPG 无密钥无可加密或签名的材料；
密钥无 GPG 无可执行密码学操作的程序。

如果你只能记住本文一件事，那应该是：**fingerprint 是密钥
的密码学身份**。它是你跑 `gpg --list-keys --fingerprint`
时看到的 40 字符十六进制串（如 `4E1C1B9E5C5F0A2D7B3C...`）。
两把 fingerprint 相同的密钥按定义就是同一把 —— 没有其他身
份来源。如果你 pin 到正确的 fingerprint，你就 pin 到了正
确的密钥。

## 终端用户常见场景

### 场景一：装签名包

多数 Linux 包管理器（`dnf`、`apt`、`zypper`、`pacman`）
会自动校验 GPG 签名。典型流程：

```sh
# RHEL / Fedora / Rocky：导入团队签名密钥，再装
sudo rpm --import https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc
sudo dnf install x-cmd

# Debian / Ubuntu：导入团队签名密钥，再装
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | sudo gpg --dearmor \
  | sudo tee /etc/apt/keyrings/x-cmd.gpg > /dev/null
sudo apt update && sudo apt install x-cmd
```

跳过导入步骤，包管理器会拒绝安装或弹出未受信签名的警
告。这是设计使然 —— 未签名包是供应链风险。

### 场景二：手工校验下载的制品

对 tarball、源码 release、或者任何非包管理器管的制品：

```sh
# 拉团队的 keyring 并导入
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import

# 校验签名
gpg --verify x-cmd-1.2.3.tar.gz.asc x-cmd-1.2.3.tar.gz

# 输出应以 "Good signature from ..." 结尾
```

如果输出说 `BAD signature`，**停下来排查后再决定是否信任**
该文件 —— 字节与团队签的对不上。

### 场景三：把 fingerprint 与独立参考源比对

导入密钥还不够 —— 你还要确认 fingerprint 与独立来源一致。
x-cmd 仓库在 [`index.tsv`](../index.tsv) 里发布团队 fingerprint；
导入后做交叉比对：

```sh
# 导入后列 fingerprint
gpg --list-keys --with-colons x-cmd \
  | awk -F: '/^fpr:/{print $10}'
# 与 index.tsv 比对
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/index.tsv \
  | awk -F'\t' 'NR > 1 {print $3}'
```

一致 → 你拿到的字节就是团队发布的字节。不一致 → **别再查**
—— 你可能从与你以为不同的源下载了。

### 场景四：用 `x gpg` shell 模块

如果你装了 `x`（x-cmd shell 框架），`x gpg` 模块把上面
那套手工流程包装成带缓存与 fingerprint 交叉校验的命令：

```sh
# 一次性导入全部 keyring（带缓存）
x gpg import

# 不导入，查某把密钥的信息
x gpg info official

# 校验签名
x gpg verify x-cmd-1.2.3.tar.gz.asc x-cmd-1.2.3.tar.gz

# 列本地密钥环里已有的密钥
x gpg ls
```

`x gpg` 在 `x-cmd/x-cmd` 的 `mod/gpg/` 下 —— 模块小到你想审
计工具本身也能端到端跑一遍。

## 三种拉取 keyring 的方式

有三条第一方路径拿到团队 keyring 字节。三条都是同一字节
钥匙。

**路径一 —— 从 GitHub 直接 curl。** 最快、零依赖、哪儿都
行。

```sh
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/keyring.asc \
  | gpg --import
```

**路径二 —— `x gpg` shell 模块。** 加缓存与 fingerprint
交叉校验。

```sh
x gpg import
```

**路径三 —— `https://x-cmd.com/gpg/`。** 部署时从本
GitHub 仓库构建出的呈现层。短 URL 方便，比如
`rpm --import https://x-cmd.com`。

```sh
sudo rpm --import https://x-cmd.com
```

**不要**走任何第三方 CDN、反向代理或缓存服务（jsdelivr、
gcore、statically 等）。代理可能返回几分钟前的字节 —— 包
括早于当前轮换的字节 —— 你的 fingerprint 校验依然会过。
信任锚点推理见 LICENSE 页脚。

## 常见坑

### 装包时"Public key not found"

系统 rpm 数据库里没团队的公钥。导入一下（上面的场景一）。

### CDN 缓存

不知不觉中导入 *陈旧* 密钥的最常见方式。每次直接从 GitHub
拉，不走中间。

### 视觉近似密钥

攻击者构造一把 UID 与团队一致（同名同邮箱）但 fingerprint
不同的密钥。你在公开 keyserver 上 `gpg --search-keys <邮
箱> 看到了匹配的 UID，导入 —— 没核 fingerprint。**永远不要去
keyserver 搜已知团队的密钥。** 直接从本仓库拉。以 fingerprint
为锚。

### handle 复用

轮换后新密钥持着相同 handle。你的工具以"official key"为
锚 —— 但字节串变了。**永远以 fingerprint 为锚，不要以 handle
为锚。**

### 短 key ID

有些老配方以 32 位短 key ID（`0xDEADBEEF`）为锚。把 160
位截断到 32 位在实践中可逆 —— 攻击者可构造一把截断后短
ID 与目标匹配的密钥。别用短 key ID 作锚。**永远用完整的 40
字符 fingerprint。**

## 首次信任（TOFU）

如果你到不了独立参考源（比如上不去 `x-cmd.com` 或者比对
不了 `index.tsv`），你就被退回到"信任 GitHub 给的字节"。
多数消费者场景这就够了 —— `https://your.tld` 由 GitHub 证书锁定，
由你操作系统信任库验证。

如果你的威胁模型包含"GitHub 给错字节"，你需要带外验证
—— 通常是团队其他渠道发布的一段已签名消息确认新
fingerprint。团队在过渡时发布这类消息。

## 延伸阅读

- [3. 年度密钥策略](./3-annual-key-strategy-explained.cn.md)
  —— 团队的长期 GPG 密钥轮换策略（社区主密钥 + 年度隔离密钥）。
- [4. GPG UID 命名约定](./4-gpg-uid-naming-conventions.cn.md)
  —— UID 里的 ™/®，fingerprint 为锚。
- [5. Sigstore、Cosign 与双重签名](./5-sigstore-cosign-and-double-signing.cn.md)
  —— 云原生供应链的现代替代方案，以及双重签名的取舍。