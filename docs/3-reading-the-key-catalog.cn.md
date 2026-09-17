---
x-title: 解读密钥目录
x-desc: `index.tsv` schema 详解 —— 每一列、背后的 fingerprint 数学、handle 命名约定，以及为何"以 fingerprint 为锚、不要以 handle 为锚"是唯一安全的规则。
x-sidebar: 解读密钥目录
x-keywords: index.tsv, fingerprint, sha-1, openpgp packet, handle 命名, 以 fingerprint 为锚
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '解读密钥目录'
      inLanguage: 'zh-CN'
      about: 'index.tsv schema 与 fingerprint 数学'
---

# 解读密钥目录

仓库根的 `index.tsv` 文件是其他所有文档都会指向的清单。
本文逐列说明、解释 fingerprint 列背后的密码学数学、
handle 列的约定，以及让这一切安全的规则：**以 fingerprint
为锚，永远不以 handle 为锚**。

## Schema —— 5 列，tab 分隔

```tsv
handle	uid	fingerprint	created	purpose
```

文件本身没有表头（表头只是文档）。行按 `handle` 排序
（字典序、ASCII），所以裸 `sort` 就能产出稳定 diff ——
review 轮换 PR 时很实用。

## 列 1 —— `handle`

团队内部对该密钥的标识。小写 ASCII，无空格。与团队其他
仓库（`x-cmd` monorepo 中的 handle、团队官网、GitHub
commit attribution）中使用的标识一致。对应磁盘上的文件
路径 `keyring/<handle>.asc`。

**handle 可在轮换中复用。** 如果某维护者轮换密钥，*新*
密钥可以取相同的 handle —— 只有 fingerprint（列 3）能告
诉你这是不同的密钥。这正是"以 fingerprint 为锚"是规则、
"以 handle 为锚"不是规则的原因。

按约定保留的 handle（实际填入时）：

| Handle         | 用途                                                        |
| ---            | ---                                                        |
| `official`     | 团队不受限主密钥。签名社区版包。                            |
| `key-<year>`   | 当年（公历年度）的年度隔离密钥。                            |

这些命名反映
[4. 年度密钥策略详解](./4-annual-key-strategy-explained.cn.md)
中详述的团队供应链密钥策略。本仓库不强制这些命名 —— 是
团队约定，不是校验规则。

## 列 2 —— `uid`

密钥发布在 OpenPGP packet 中的主 UID —— 通常格式为
`Name (comment) <email>`。这是 *显示文本*，密钥持有者
可编辑。两把密钥 UID 冲突（比如冒充者上传一把声称相同邮
箱的密钥）**不** 是安全属性 —— 只有 fingerprint 是。

团队更新 UID（如换了邮箱）时，`index.tsv` 的 `uid` 列变
化，`fingerprint` 列不变。所以 `index.tsv` 的修订历史能
区分：仅 UID 变化、仅 fingerprint 变化、或两者都变。

## 列 3 —— `fingerprint` —— 信任锚点

40 字符大写十六进制串，无空格（某些工具按 4 字符插入空
格提升可读性 —— `4E1C 1B9E 5C5F 0A2D 7B3C …` —— 但
`index.tsv` 中的规范形式是无空格）。按 RFC 4880 §12.2，
是密钥 *公钥 packet* 的 SHA-1。

具体含义：fingerprint 是公钥精确字节的 160 位密码学承诺。
两把 fingerprint 相同的密钥，按定义就是同一把密钥。没有
其他身份来源 —— 没有"key id"、没有名字、没有邮箱 —— 能
钉到同一级别的密码学确定性。

### 为何以 fingerprint 为锚，而非 handle 或 UID

- **Handle** —— 团队选定，轮换时可复用。它是标签，不是
  身份。
- **UID**（`Name <email>`）—— 密钥持有者可编辑。视觉近
  似攻击者可以构造一把名字与邮箱相同但 fingerprint 不同
  的密钥，UID 列帮不了你区分。
- **Key ID**（短的 `0xDEADBEEF` 形式，`gpg --list-keys`
  常显示）—— 由 fingerprint 派生，但截断到 32 位。易受
  *key-ID 冲突攻击*：攻击者可构造一把 32 位 key ID 与
  目标相同的密钥。不是密码学承诺。
- **Fingerprint** —— 完整 160 位。抗冲突能力仅受 SHA-1
  原像空间（2^160）约束。现实无已知有效冲突攻击。

所以当 CI 流水线、`apt`/`dnf` 仓库或 `x gpg` 调用方要
确认"给制品 X 签名的密钥就是 `index.tsv` 中的那把？"时，
唯一答案是：从制品签名中取出 fingerprint，去 `index.tsv`
中查，fingerprint 对得上才信任该制品。比这弱一档的都面
临替换攻击。

## 列 4 —— `created`

`YYYY-MM-DD` —— 密钥创建日期，按 `gpg --list-keys --with-colons`
（`pub` 行的 `creation-date` 字段，ISO 8601 格式）。纯
展示用 —— 用来理解密钥集历史（"哪些密钥已经 2 年快到自
定轮换窗口？"），但不是安全属性。

如果一把密钥在同一个 handle 下被创建、轮换、再创建，那么
`index.tsv` 在不同时间会有同一 handle 的两行（fingerprint
不同）。`created` 列帮你消歧。

## 列 5 —— `purpose`

自由文本、小写、用连字符。团队用的值如
`release-signing`、`package-signing`、`mirror` 等。没
有形式枚举 —— 该列记录每把密钥 *做什么用*，让消费者的工
具能用单条 `awk` 过滤回答"显示 2026 年企业包用的密钥"。

团队计划用的 purpose 值示例：

| Purpose                       | 该密钥签署什么                                              |
| ---                           | ---                                                        |
| `release-signing`              | 在 GitHub Release 上发布的源码 tarball 签名。              |
| `package-signing`             | `x-cmd.<ext>` 的 RPM / DEB / 容器镜像签名。               |
| `package-signing-yearly`      | 年度企业包签名（`x-cmd-annual-<year>`）。                  |
| `mirror-signing`              | 团队镜像 bucket / CDN 上传的签名。                          |

再说一次：值是约定，不强制。CI 校验 `index.tsv` 能解析且
fingerprint 唯一 —— 不校验 purpose 字符串。

## 实用查询

```sh
# 团队的 release-signing 密钥的 fingerprint 是什么？
awk -F'\t' '$5 == "release-signing" { print $1, $3 }' index.tsv

# 2025 年创建的所有密钥
awk -F'\t' '$4 ~ /^2025-/' index.tsv

# 给定的 fingerprint 是否出现在团队的 index 中？
grep -F "<40 位十六进制 fingerprint>" index.tsv

# 三方交叉比对 fingerprint（GitHub、团队官网、本地副本）
fpr=$(awk -F'\t' '$1=="official"{print $3}' index.tsv | tr -d ' ')
curl -fsSL https://raw.githubusercontent.com/x-cmd/gpg/main/keyring/official.asc \
  | gpg --show-keys --with-colons \
  | awk -F: '/^fpr:/{print $10}'
diff <(echo "$fpr") <(curl ... | awk ...)
```

最后一段是最低门槛的自动化校验 —— 拉取 GitHub 上的同一
文件，从 GnuPG 输出解析 fingerprint，与 `index.tsv` 中
的值做 `diff`。匹配 → GitHub 显示的字节就是团队发布的字
节。不匹配 → 停下来排查再决定是否信任。

## 延伸阅读

- [4. 年度密钥策略详解](./4-annual-key-strategy-explained.cn.md) ——
  FAQ Q4–Q8 的长文版。
- [5. 验证一把密钥](./5-verifying-a-key.cn.md) —— 拉取 →
  导入 → 比对 的三步法。

## FAQ

本节从
[文章 0 的中心 FAQ](./0-x-cmd-gpg-overview.cn.md#faq--软件分发与代码签名密码学)
里挑出与本文最相关的子集。完整的 8 问在文章 0，
答案以行业普遍视角书写，与具体项目无关。

### Q1：GPG 软件与 GPG Key 在技术上是什么关系？

是软件程序与数据凭证的关系。GPG 是执行密码学操作的程序；
keypair 是持有密码学材料的数据凭证。类似 `index.tsv` 这
种清单只记录密钥对的 *公钥* 半 —— 安全可公开的那一半；
对应的私钥半由发布方持有。完整答案见文章 0。

### Q3：为什么发布者通常不直接分发未签名的裸包？

只列出 *已签名* 制品密钥的清单之所以有意义，正因为签名
是事实标准 —— 未签名分发在现实中很少见，因为包管理器默认
拒绝安装未签名包，且 MITM / 投毒的攻击面无法接受。清单
条目代表已签名制品，其存在即暗示有签名。完整答案见文章 0。