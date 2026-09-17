---
x-title: keyring 如何发布
x-desc: 团队内部流程 —— 从团队成员机器上的 `gpg --export` 到 `index.tsv` 的一行与 `keyring/keyring.asc` 里的字节。包含文件布局、相关 gpg 命令、每次发布前的 CI 校验。
x-sidebar: keyring 如何发布
x-keywords: gpg --export, index.tsv, keyring.asc, 发布流水线, ci 校验, 轮换, archive
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'keyring 如何发布'
      inLanguage: 'zh-CN'
      about: 'x-cmd/gpg 发布流水线'
---

# keyring 如何发布

从"团队成员笔记本上有了一把新 GPG 密钥对"到"`index.tsv`
里的一行 + `keyring.asc` 里的字节"的流水线刻意做得很小
—— 六条 gpg 命令、一行清单更新、一次 CI 走通。本文逐
步拆解，并解释每一步为何是这个形状。

## 两层文件布局

进入流水线之前，先了解仓里到底落地什么：

```text
keyring/
  ├── <handle>.asc       # 每个发布密钥一个文件
  └── keyring.asc        # 汇总；每次发布重新生成
index.tsv                # 5 列清单
README.md / README.cn.md # 首页目录，由 index.tsv 自动生成
```

`keyring/<handle>.asc` 是权威产物 —— 单密钥消费者
（`x gpg import <handle>`）拉的就是它。`keyring/keyring.asc`
是汇总 —— "一次导入全部"的消费者一次 HTTP GET 拿到它。
`index.tsv` 是清单，让 README 目录、`x gpg info` 以及
任何工具都能在不解析 OpenPGP packet 的前提下回答"有哪些
密钥、什么 fingerprint、什么用途、何时建的"。

## 添加一把新密钥

团队内部、维护者发布新密钥时的流程：

```sh
# 1. 维护者导出公钥（ASCII-armored）
gpg --armor --export <KEYID> > keyring/<handle>.asc

# 2. 维护者自签导出文件
gpg --detach-sign --armor \
    -u <signing-subkey> \
    keyring/<handle>.asc
# 生成 keyring/<handle>.asc.asc —— 不进仓库，但 commit
# message 会引用，供审计者离线核验链

# 3. 维护者往 index.tsv 追加一行
#    handle<TAB>uid<TAB>fingerprint<TAB>YYYY-MM-DD<TAB>purpose

# 4. 维护者开一个 PR（*内部* PR，在 x-cmd monorepo 流程
#    里，不在这个公开仓库）。第二位团队成员审查：
#    - gpg --list-packets keyring/<handle>.asc 能干净解析
#    - index.tsv 的 fingerprint 与 gpg --show-keys 一致
#    - .asc.asc detached signature 能用签名子钥验证
#  - 密钥的主 UID 上有合法自签

# 5. merge 后，发布流水线把所有 <handle>.asc 按 handle
#    字典序拼接，重新生成 keyring/keyring.asc
```

外部贡献者永远看不到步骤 4–5。整个要点就是：团队是
`keyring/` 与 `index.tsv` 上 commit 的唯一路径。触这两条
路径的外部 PR 直接关闭、不合并 —— 政策原文见
[`CONTRIBUTING.md`](../CONTRIBUTING.md)。

## 重新生成汇总

`keyring/keyring.asc` *不* 手工编辑。发布流水线重新生成：

```sh
{
  for f in keyring/<handle>.asc; do
    cat "$f"
    echo ""                  # 拼接密钥之间空一行
  done
} > keyring/keyring.asc
```

脚本按 `handle` 排序（与 `index.tsv` 顺序一致），保证多次
重生成文件字节级稳定。CI workflow 在每次 push 到 `main`
时重跑这个脚本；若重生成的 `keyring.asc` 与已提交的不一
致，push 失败。这就保证了汇总文件永远与逐密钥文件锁步。

## CI 校验（每次 push + 每次 PR）

`.github/workflows/verify.yml` 在每次 push 与每次 PR 上跑
（外部 PR 用只读 token，所以能校验而无需给写权限）。四项
校验按顺序：

### 1. 解析检查

```sh
gpg --list-packets keyring/keyring.asc
```

退出码 0 → 文件是良构 OpenPGP packet 流。损坏或截断的导
出会先在这里失败，不会跑到后面的检查。

### 2. Fingerprint 与 manifest 一致性检查

`index.tsv` 中每一行所列 fingerprint 都出现在 `keyring/"
keyring.asc` 中。漂移（manifest 行的 fingerprint 不在 keyring
中，或 keyring 中存在 fingerprint 没有 manifest 行）即失
败。

### 3. 自签检查

`keyring/keyring.asc` 中每把密钥的主 UID 上都带合法自签。
这能抓到"半导出密钥"（比如有人 `gpg --export` 时密钥
环处于半导入态，结果只导出部分）。

### 4. 唯一性检查

`index.tsv` 行之间不重复 fingerprint；`keyring/*.asc` 文
件之间不重复 fingerprint。重复即失败。这是最简单可行的
"两把近似密钥，一把是冒充"攻击的拦截。

CI **不** 验证 trust 签名或 web-of-trust 路径 —— 那是消
费者的责任，用 `x gpg` 或上游 `gpg --check-sigs` 即可。

## 密钥轮换

密钥到期或团队成员轮换时：

1. 团队成员发布**密码学过渡声明** —— 一段由 *旧* 密钥
   签名的消息，声明"这把密钥退役，新密钥为 `<fingerprint>`"。
   声明链接在团队官网上，不进本仓库（让仓库保持轻量）。
2. 旧 `keyring/<handle>.asc` 在下一次发布 commit 中移入
   `keyring/archive/<handle>.<created-date>.asc`。当前的
   `keyring/<handle>.asc` 被新密钥替换。
3. `index.tsv` 在同一次 commit 中更新以反映轮换。README
   的"Retired keys"表由 `keyring/archive/` 在每次发布时
   自动生成。

退役密钥在 archive 中永久保留，并**由团队主密钥签名**，
以便已导入团队主密钥的消费者验证托管链。退役密钥的
fingerprint 永远不会以新 handle 形式再次出现。

## 你（外部读者）能验证什么

如果你想确认团队没在某个 release 里偷偷塞恶意 commit：

1. `git log` 看 `keyring/<handle>.asc` 的历史。每条 commit
   message 都引用添加密钥的团队成员，并（理想情况下）引用
   detached-signature 文件名供离线核验。
2. `.github/workflows/` 里的 CI workflow 文件本身就在 git
   里 —— 你可以读它们，看到团队对每次 commit 跑了哪些
   校验。
3. 团队的 release tag 是 GitHub 签名过的（`gh:x-cmd/gpg`
   仓库的 tag 应携带团队 GPG 签名）。用
   `git tag --verify <tag>` 验证。

这些都不能替代直接导入密钥并把 fingerprint 与独立来源对
比 —— 但它们给了你审计轨迹。

## 延伸阅读

- [3. 解读密钥目录](./3-reading-the-key-catalog.cn.md) ——
  `index.tsv` schema 详解与 fingerprint 数学。
- [4. 年度密钥策略详解](./4-annual-key-strategy-explained.cn.md) ——
  年度隔离密钥工作流如何接入同一条流水线。
- [5. 验证一把密钥](./5-verifying-a-key.cn.md) —— 拉取 →
  导入 → 比对 的三步法。

## FAQ

本节从
[文章 0 的中心 FAQ](./0-x-cmd-gpg-overview.cn.md#faq--软件分发与代码签名密码学)
里挑出与本文最相关的子集。完整的 8 问在文章 0，
答案以行业普遍视角书写，与具体项目无关。

### Q4：硬编码 GPG 密钥为"永不过期"有哪些优缺点？

权衡点是业务连续性与爆炸半径。永不过期意味着历史制品永
远能校验，CI/CD 永远不用轮换密钥 —— 零维护、未值守主机
不会因"密钥过期"出现故障；代价是一旦私钥被攻陷泄露，攻
击者获得无限期的伪造能力，恢复依赖发行后极难分发的"吊
销证书"机制。完整答案见文章 0。

### Q8：采用"一年一换"密钥模型，工业界如何解决跨年过渡与历史版本回滚？

标准模式是 **Trust Anchor Registry（信任锚点注册表）**，
杠杆点在用户侧的密钥环：发布方维护的数据路径把所有历年
年度公钥合并进单个 keyring；消费者把 keyring 导入自己的
`gpg`，于是在本地同时持有历史与未来密钥。从此供应链归
用户 —— "保留旧密钥、跨年时删除、还是自定节奏轮换"的策
略由用户在自己的 `gpg` 密钥环上定。完整答案见文章 0。