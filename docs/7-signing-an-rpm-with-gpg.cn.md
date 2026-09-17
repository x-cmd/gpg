---
x-title: 用 GPG 给 RPM 包签名
x-desc: 如何给 RPM 包加 GPG 签名 —— 导入签名私钥、配置 rpmsign、单包 / 批量签名、验证结果、以及对老包的重签（付费 LTS 客户的 repackage / resign 生命周期）。
x-sidebar: 用 GPG 给 RPM 包签名
x-keywords: rpm, rpmsign, gpg, 包签名, createrepo, 重签发, lts, 批量签名, gpg-agent
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '用 GPG 给 RPM 包签名'
      inLanguage: 'zh-CN'
      about: 'RPM 包签名工作流'
---

# 用 GPG 给 RPM 包签名

团队发布流水线的实操教程：拿到一个构建出来的 `.rpm` 文
件，给它盖上团队的 GPG 签名，然后发到 yum/dnf 仓库。同一
份配方也用于付费 LTS 客户依赖的重签发生命周期（用今年
的密钥重签老制品 —— 见
[4. 年度密钥策略详解](./4-annual-key-strategy-explained.cn.md#重签发机制-lts-客户)）。

## 先决条件

签名 RPM 之前你需要三样东西：

1. **团队签名私钥**在本地 GPG 密钥环里。对应的公钥在本仓
   库的 [`keyring/<handle>.asc`](../keyring/)；私钥半由签
   名方持有（通常是发布 CI 的密文 secret store，或构建
   工程师的离线笔记本）。
2. **已装 `rpm-build` 与 `rpm-sign` 包**。RHEL 系发行版上：

   ```sh
   dnf install rpm-build rpm-sign gnupg2
   ```
3. **一个构建好的 `.rpm` 文件**。本文假设包已造好，只
   需要盖章。

## 导入签名密钥

私钥从团队保存 secret 的地方来 —— 通常一次性导出后加密存
放，或从 CI secret 拉。导入动作与导入公钥相同
（`gpg --import`），只是文件持有的是私钥半：

```sh
# 从已导出的私钥包导入
gpg --import /secure/path/to/team-signing-key.private.asc

# 确认导入
gpg --list-secret-keys --keyid-format long
```

输出应列出团队的密钥，同时带 `pub` 与 `sec` 行。注意长
key ID（`rsa4096/` 算法标签后的 16 位十六进制）—— 你要
把它传给 `rpmsign`。

## 配置 rpmsign

`rpmsign` 读 `~/.rpmmacros`（或 `%_topdir/.rpmmacros`）取默
认密钥与签名设置。对于 CI / 批量工作，在宏文件里显式设
置密钥，不要依赖 `gpg-agent` 的"默认密钥"猜测：

```text
# ~/.rpmmacros
%_gpg_name  Li Junhao (x-cmd 签名密钥) <l@x-cmd.com>
%_gpgbin    /usr/bin/gpg2
```

如果密钥环里有多把密钥，`%_gpg_name` 明确指定用哪把。
格式是密钥的主 UID。

CI 下的 passphrase 处理见下方
[Passphrase 与 gpg-agent](#passphrase-与-gpg-agent) 节。

## 给单个 RPM 签名

```sh
rpmsign --addsign /path/to/x-cmd-1.2.3-1.x86_64.rpm
```

`--addsign` *追加* 一个签名，不移除已有签名 —— 重签发生
命周期里有用：你希望制品同时带今年与去年的密钥签名。要
替换所有已有签名，用 `--resign`：

```sh
rpmsign --resign /path/to/x-cmd-1.2.3-1.x86_64.rpm
```

日常发布 `--addsign` 是更稳妥的默认；`--resign` 只在
你有意让所有旧签名失效时才合适。

## 批量签名（CI / 一批包）

构建产物目录里的所有 RPM：

```sh
# 遍历构建输出目录里所有 .rpm
for rpm in /path/to/build/RPMS/*/*.rpm; do
  echo "signing $rpm"
  rpmsign --addsign "$rpm"
done
```

多架构构建产物（`.x86_64.rpm`、`.aarch64.rpm`、`.noarch.rpm`）
同一个循环覆盖 —— 每个 RPM 独立签名。

## 验证签名

```sh
rpm -K /path/to/x-cmd-1.2.3-1.x86_64.rpm
```

输出：

```text
/path/to/x-cmd-1.2.3-1.x86_64.rpm: rsa4096 (sig1) OK (Full RSA)
```

`OK` 表示签名能对照系统 rpm 密钥数据库里的公钥校验通过。
要看更详细（签名密钥的 fingerprint 与 UID）：

```sh
rpm -Kv /path/to/x-cmd-1.2.3-1.x86_64.rpm
```

把签名密钥的 fingerprint 与 [`index.tsv`](../index.tsv) 第
3 列交叉比对 —— 这就是
[5. 验证一把密钥](./5-verifying-a-key.cn.md) 的"三方
fingerprint 比对"流程。

## 重签发生命周期（LTS 工作流）

付费 LTS 客户希望把旧包用今年（`key-2026` 或当前）密钥
重签时：

1. 从团队发布归档里取历史包（文件与当初 `key-2025` 签名
   时一模一样）。
2. 对当前年度密钥跑 `rpmsign --addsign`：

   ```sh
   rpmsign --addsign /path/to/x-cmd-1.2.0-1.x86_64.rpm
   ```

   包 *内部* 字节不变；只追加了签名头。
3. 把重签后的制品与原签版本一起重新发布到 release 归
   档。两个版本都可用；消费者按自己持有哪些密钥决定导入
   哪个。

这就是
[4. 年度密钥策略详解](./4-annual-key-strategy-explained.cn.md#重签发机制-lts-客户)
里描述的重签发 —— 对现有 release 资产跑
`rpmsign --addsign`，不重编译上游二进制。

## Passphrase 与 gpg-agent

`rpmsign` 内部调用 `gpg`，每次调用都会提示输入签名密钥
的 passphrase。人手签名没问题；CI / 批量时你需要：

- **`gpg-agent` 预加载缓存**。在 CI 里先启动
  `gpg-agent`，把 `default-cache-ttl` 设到足够覆盖批量
  任务，用 `gpg-preset-passphrase` 预设 passphrase。
- **`--passphrase-file <path>`** 传给 `rpmsign`（转发给
  `gpg`）。文件 `chmod 600`，放在 CI 的 secret store。

笔记本一次性签名的最简配方：

```sh
# 启动 gpg-agent、输一次 passphrase、跑 rpmsign
gpg-agent --daemon --max-cache-ttl 3600
rpmsign --addsign package.rpm
gpgconf --kill gpg-agent
```

agent 缓存 passphrase 一小时，期间把所有 RPM 都签了，然
后杀掉。多小时 CI 构建把 `--max-cache-ttl` 相应拉长。

## 常见坑

### `rpm -K` 报"Public key not found"

系统 rpm 数据库里没团队公钥。导入：

```sh
# 从本仓库拉
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/<handle>.asc \
  | sudo rpm --import -
```

### 选错密钥

`rpmsign` 用 `~/.rpmmacros` 里 `%_gpg_name` 匹配的那把签
名。密钥环里有多个密钥、宏文件缺或错时，可能签错。CI
里始终显式设 `%_gpg_name`；最终 RPM 的签名头会记录到底
是哪把签的。

### 多重签名

`--addsign` 把签名 *追加* 到已有签名之后。同一包用两把
不同密钥各签一次，会带两个有效签名 —— 两个都通过校验、
都不失效。这正是 LTS 重签想要的（老密钥、新密钥都能验
证）。要干净发布只带当前密钥的版本，用 `--resign`。

### CI 里卡在 passphrase 提示

`rpmsign` 不读 `~/.bashrc`，也不把 `GPG_TTY` 传给
`gpg-agent`。headless CI 要么用 `--passphrase-file`（`chmod
600`），要么预先用 `gpg-preset-passphrase` 喂
`gpg-agent`，并配好 `--max-cache-ttl`。

### "package is not signed" 重签失败

`--addsign` 要求包至少已有 *一个* 签名（构建时的原始签
名）。如果 `rpmbuild` 跑时没启用签名，包就是未签的；
`rpmsign --addsign` 会拒绝。两种解决：
- 重新构建时设 `%_gpg_name`，让 `rpmbuild` 自己签进原始
  签名（更干净的路径），或
- 对已经在构建时签过名的包用 `rpmsign --addsign`。

## 整体串起来（release CI）

典型的发布 job：

```sh
# 1. 从 CI secret store 导入团队签名密钥
echo "$TEAM_SIGNING_KEY_PRIVATE" | gpg --import --batch

# 2. 配 rpmmacros
cat > ~/.rpmmacros <<EOF
%_gpg_name  Li Junhao (x-cmd 签名密钥) <l@x-cmd.com>
EOF

# 3. 把 passphrase 预加载到 gpg-agent
echo "$TEAM_GPG_PASSPHRASE" \
  | gpg-preset-passphrase --preset $(gpg --list-secret-keys --with-colons \
                                       | awk -F: '/^sec/{print $5}')

# 4. 给构建输出里每个 RPM 签名
for rpm in /build/RPMS/*/*.rpm; do
  rpmsign --addsign "$rpm"
done

# 5. 验证
rpm -Kv /build/RPMS/*/*.rpm

# 6. 发布到 yum 仓库
createrepo --update /var/www/repo/x-cmd/
```

每一步独立可审计：哪把密钥签的（步骤 5 + 与 `index.tsv`
交叉比对），签了什么（文件列表），何时（release tag）。

## 延伸阅读

- [5. 验证一把密钥](./5-verifying-a-key.cn.md) ——
  拉取 → 导入 → 比对 的三步法；签完跑一遍确认是预期的
  密钥。
- [4. 年度密钥策略详解](./4-annual-key-strategy-explained.cn.md#重签发机制-lts-客户) ——
  本文签名工作流如何嵌入年度轮换与重签发生命周期。
- [2. keyring 如何发布](./2-how-the-keyring-is-published.cn.md) ——
  本文是其中"签名"这一步的展开。

## FAQ

本节从
[文章 0 的中心 FAQ](./0-x-cmd-gpg-overview.cn.md#faq--软件分发与代码签名密码学)
里挑出与本文最相关的子集。完整的 8 问在文章 0，
答案以行业普遍视角书写，与具体项目无关。

### Q1：GPG 软件与 GPG Key 在技术上是什么关系？

本文同时涉及这两方：**gpg** 程序（`rpmsign` 内部调用它
执行密码学操作）与 **keypair**（被程序操作的密码学材料
的数据凭证）。把团队签名私钥导入本地 GPG 密钥环是
`rpmsign` 能找到签名密钥的数据侧前提。完整答案见文章 0。

### Q3：为什么发布者通常不直接分发未签名的裸包？

因为 `dnf` / `yum` / `rpm` 默认拒绝安装未签名包，且未签
名分发在传输过程中没有任何密码学防篡改保护（MITM 与
包投毒变得轻而易举）。本文的签名工作流就是标准缓解方
案：构建出包，发布前用 `rpmsign --addsign` 盖章，消费者
端的 `rpm -K` 就返回 `OK`。`--addsign` 这步操作把密码
学身份（团队的密钥对）绑定到字节（RPM 文件）。完整答案
见文章 0。