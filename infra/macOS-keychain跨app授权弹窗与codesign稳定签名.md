# macOS 钥匙串跨 app 授权弹窗:两层根因与 codesign 稳定签名

## 背景

自建的本地小工具(菜单栏 app)需要读取 Claude Code 存在 macOS 登录钥匙串里的 OAuth token,去调用 usage API。因为是**跨 app** 读别人写的钥匙串条目("Claude Code-credentials"),系统反复弹出"XX 想访问钥匙串中的密钥"的授权窗——即使点了「始终允许」,过一阵又弹。

排查下来发现"反复弹窗"其实是**两个独立机制**叠加,容易误以为修好一个就全好了。把它们拆开是这次最关键的认知。

---

## 第一层:每次重编译都弹

### 根因

钥匙串的「始终允许」记录的**不是 app 的路径,而是它的 designated requirement**(签名指纹规则)。

app 当时用 ad-hoc 签名(`codesign --sign -`),这种签名的 designated requirement 是:

```
designated => cdhash H"f954...(二进制哈希)"
```

`cdhash` 直接绑二进制内容。开发迭代时每次重新编译,二进制变了 → cdhash 变了 → 跟钥匙串 ACL 里记录的对不上 → 系统认为是"另一个 app",重新弹窗。

**只要还在频繁重编译,ad-hoc 签名就必然反复弹。**

### 解法:换成稳定的自签代码签名证书

用一个固定证书签名后,designated requirement 变成:

```
designated => identifier "com.example.app" and certificate leaf = H"<cert-sha1>"
```

只绑 **bundle identifier + 证书指纹**,跟二进制内容无关 → 重编译不变 → 授权一次永久生效。

### 操作要点(踩过的坑)

```bash
# 1. openssl 自签一个带 codeSigning 用途的证书
#    关键扩展:extendedKeyUsage = critical,codeSigning
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
  -days 3650 -nodes -config cert.cnf

# 2. 导入登录钥匙串
#    坑:macOS 自带的 LibreSSL 生成的 .p12,security import 会报
#    "MAC verification failed" —— 加密算法不认。
#    绕法:不打包 p12,直接分开导入 pem 私钥和证书,钥匙串会自动关联成 identity
security import key.pem  -k ~/Library/Keychains/login.keychain-db -T /usr/bin/codesign -A
security import cert.pem -k ~/Library/Keychains/login.keychain-db -T /usr/bin/codesign -A

# 3. 自签证书默认不被信任,find-identity -p codesigning 不会列出它
#    必须显式设为代码签名可信(会弹一次系统授权,输登录密码)
security add-trusted-cert -p codeSign cert.pem

# 4. 构建脚本里把签名行从 ad-hoc 换成证书名
#    codesign --force --sign "Shima Dev" App.app
```

验证:`codesign -d -r- App.app` 看 designated requirement 是否已变成 `identifier ... and certificate leaf`。

> 注意:必须运行 `.app` bundle,不能跑裸二进制。裸二进制没有 bundle identifier,requirement 退回基于 cdhash,又不稳定。

---

## 第二层:每天还弹一次(这层才是真正的死结)

签名稳定后,重编译不再弹,但**每天仍会弹一次**。这说明还有第二个机制。

### 诊断手法:用 cdat / mdat 区分 update vs delete+add

```bash
security find-generic-password -s "Claude Code-credentials" | grep -Ei "cdat|mdat"
# cdat(创建时间)= 两个月前,固定不变
# mdat(修改时间)= 每天变,今天凌晨
```

- cdat 固定 → 这个条目**一直是同一个**,不是被删了重建(否则 cdat 会跟着变新)。
- mdat 每天变 → 属主 app 每天对它做了一次 **update**(改写数据)。

对应现实:Claude Code 每天刷新 OAuth token,把新 token 写回这个钥匙串条目。

### 根因:属主改写 item 会重置 ACL

macOS 钥匙串条目的 ACL(允许哪些 app 免确认访问)会在**属主 app 改写该条目时被重置**——之前你给"外来 app"点的「始终允许」被一并清掉。

于是每天 Claude Code 刷新 token 后,外来 app(我的工具)当天第一次去读,授权已失效 → 又弹。

### 关键结论

> **只要 A app 去读 B app 拥有、且 B 会频繁改写的钥匙串条目,纯靠「始终允许」就根治不了——你无法阻止 B 每天把授权清掉。**

而且这一层跟签名**完全无关**。签名稳定只解决"同一个 app 重编译后身份还认不认得出"(第一层),解决不了"条目被属主改写导致 ACL 重置"(第二层)。**两个机制别混为一谈**,这是最容易踩的误判。

---

## 根治方向

核心思路只有一个:**别去读别人拥有、且会被频繁改写的钥匙串条目**。绕开它。

- ❌ **让属主落磁盘文件、我读文件**:在 macOS 上不可行。Claude Code 的凭据存储在 macOS 上是**硬编码用 Keychain 的**,没有任何环境变量/设置能改成文件(只有 Linux/Windows 才用 `~/.claude/.credentials.json`)。
- ✅ **给外来 app 一个独立 token**:用 `claude setup-token` 生成一个**一年有效期**的独立 OAuth token,存进自己管理的文件,外来 app 只读这个文件。彻底不碰那个每天被改写的钥匙串条目 → 永不弹窗,一年才需重生成一次。
- 🅱️ **降级(零风险)**:外来 app 读钥匙串时把交互 UI 关掉(`kSecUseAuthenticationUI = ...Fail`),读不到就静默用缓存/占位,绝不主动弹窗打扰;代价是数据偶尔短暂不准。

---

## 关键知识点速记

- 钥匙串「始终允许」绑的是 **designated requirement**,不是文件路径。
- **ad-hoc 签名** → requirement = `cdhash`(随二进制变,不稳定);**证书签名** → requirement = `identifier + certificate leaf`(稳定)。开发期要稳定授权,必须用固定证书,不能用 ad-hoc。
- LibreSSL 生成的 `.p12`,`security import` 常报 MAC verification failed;**分开导入 .pem 私钥和证书**可绕过。
- 自签证书要 `security add-trusted-cert -p codeSign` 才会被 `find-identity -p codesigning` 认作有效签名身份。
- **cdat 固定 + mdat 变化 = 同一条目被 update**;cdat 也跟着变 = delete+add 重建。这是判断钥匙串条目"改写方式"的实用技巧。
- 钥匙串条目的 ACL 会在**属主 app 改写它时被重置**——跨 app 去读一个被频繁改写的条目,是无解的死结,只能绕开。
- macOS 上 Claude Code 凭据**硬编码 Keychain**,不可配置为文件;跨进程共享 token 用 `claude setup-token`(一年期独立 token)。
