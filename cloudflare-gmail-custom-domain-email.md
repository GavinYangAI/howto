# Cloudflare + Gmail 搭建个人免费域名信箱

> 适用场景:有一个自己的域名(托管在 Cloudflare),想用 `you@你的域名` 这样的地址**收发邮件**,
> 但不想花钱买 Google Workspace / 付费邮箱服务。
>
> 方案:**收信**靠 Cloudflare Email Routing(免费,把发到你域名的邮件转发进你的 Gmail);
> **发信**靠 Gmail 的「Send mail as / 用其他地址发信」,经 Gmail 自家 SMTP 发出,
> 收件人看到的发件人就是 `you@你的域名`。整套零成本。
>
> 约定:下文用 `<你的域名>` 代表你的域名(如 `example.com`),
> `<你的Gmail>` 代表你真实的 Gmail 地址(如 `me@gmail.com`),
> `<别名>` 代表你想对外用的前缀(如 `hi`、`me`、`contact`)。执行时替换成真实值。

---

## 0. 它是怎么工作的(先理解再动手)

```
别人发信 → you@<你的域名>
            │
            ▼
   Cloudflare Email Routing(收信网关,免费)
            │  转发
            ▼
   你的 Gmail 收件箱 ← 在这里读所有邮件

你回信 / 主动发信
   Gmail「用其他地址发信」选 you@<你的域名>
            │  走 Gmail SMTP(smtp.gmail.com)
            ▼
   收件人看到发件人 = you@<你的域名>
```

- **收**和**发**是两套独立机制,要分别配。
- Cloudflare 只负责「收 + 转发」,**它不能帮你发信**(没有外发 SMTP)。发信必须借道 Gmail。
- 全程在你已有的 Gmail 里读写,不用再开新邮箱客户端。

### 前置条件

1. 一个域名,且 **DNS 已托管在 Cloudflare**(NS 指向 Cloudflare)。
2. 一个 Gmail 账号,且**已开启两步验证(2FA)** —— 这是后面生成应用专用密码的前提。

---

## 1. 收信:配置 Cloudflare Email Routing

进入 Cloudflare 控制台 → 选中 `<你的域名>` → 左侧 **Email → Email Routing**。

1. 点 **Get started / 启用**,Cloudflare 会提示自动添加所需 DNS 记录(MX + 一条 SPF TXT),**点同意让它自动加**。
2. 在 **Routing rules → Custom addresses** 里新建转发规则:
   - **Custom address**:`<别名>@<你的域名>`(如 `hi@example.com`)
   - **Action**:Send to → 填 `<你的Gmail>`
3. Cloudflare 会给 `<你的Gmail>` 发一封**验证邮件**,去 Gmail 点确认链接,destination 才生效。
4. (可选)开 **Catch-all**:把发往 `*@<你的域名>` 的所有邮件都转进你的 Gmail,省得逐个建别名。

> Cloudflare 自动加的 DNS 记录长这样(它替你管,通常不用手改):
> ```
> MX   @   route1.mx.cloudflare.net   (优先级 多条)
> MX   @   route2.mx.cloudflare.net
> MX   @   route3.mx.cloudflare.net
> TXT  @   "v=spf1 include:_spf.mx.cloudflare.net ~all"
> ```

> **验证收信**:用别的邮箱往 `<别名>@<你的域名>` 发一封,看是否落进 Gmail 收件箱。
> 到这一步,「收」就通了。

---

## 2. 发信:Gmail「用其他地址发信」

让 Gmail 能以 `<别名>@<你的域名>` 的身份外发。

### 2.1 开启 2FA(若还没开)

打开 <https://myaccount.google.com/signinoptions/two-step-verification> 开启两步验证。
**不开 2FA 就生成不了下一步的应用专用密码。**

### 2.2 生成 Gmail 应用专用密码(App Password)

1. 打开 <https://myaccount.google.com/apppasswords>
2. 给它起个名字(如 `domain-mail`),点 **生成 / Create**
3. 复制那串 **16 位密码**(形如 `abcd efgh ijkl mnop`),只显示这一次,先存好。

> 这串密码**不是**你的 Gmail 登录密码,而是专给 SMTP 用的「一次性钥匙」,
> 泄露后可在同一页面单独吊销,不影响主账号。

### 2.3 在 Gmail 里添加发信地址

Gmail → 右上角齿轮 **查看所有设置** → **账号和导入(Accounts and Import)**
→ **用以下地址发送邮件(Send mail as)** → **添加其他电子邮件地址**。

1. **姓名**:对外显示的名字
2. **电子邮件地址**:`<别名>@<你的域名>`
3. **取消勾选**「视为别名 / Treat as an alias」
4. 点**下一步**

### 2.4 填 SMTP(关键一步)

| 字段 | 值 |
|---|---|
| SMTP 服务器 | `smtp.gmail.com` |
| 端口 | `587` |
| 用户名 | `<你的Gmail>`(完整地址,带 `@gmail.com`) |
| 密码 | 2.2 生成的**应用专用密码**(不是登录密码) |
| 加密 | **TLS**(端口 587 对应 TLS;选项里通常默认就是) |

填好点**添加账号**。

> 注意用户名填的是你**真实的 Gmail 地址**,不是 `<别名>@<你的域名>`。
> 信是借你的 Gmail 账号发出去的,只是「显示」成域名地址。

### 2.5 验证归属

Google 会往 `<别名>@<你的域名>` 发一封**验证码邮件** —— 因为第 1 节已配好收信,
这封信会转进你的 Gmail。复制验证码填回去(或点确认链接)即可。

> 验证通过后,Gmail 写信时「发件人」下拉就能选 `<别名>@<你的域名>` 了。
> 建议同时把它设为**默认发件地址**(Send mail as 里点 "make default")。

---

## 3. 补 DNS 记录:让发出去的信不进垃圾箱(SPF / DMARC)

收发能通后,为了**送达率**(避免被对方判成垃圾邮件 / 伪造),建议在 Cloudflare DNS 里补两条记录。

### 3.1 SPF:授权 Gmail 也能代发你的域名

第 1 节 Cloudflare 自动加的 SPF 只授权了它自己的转发服务。
既然现在还用 Gmail SMTP 外发,要把 `_spf.google.com` 也加进去。
**SPF 一个域名只能有一条 TXT**,所以是**改**那条已有的,不是新建:

```
类型: TXT
名称: @
内容: v=spf1 include:_spf.mx.cloudflare.net include:_spf.google.com ~all
```

### 3.2 DMARC:声明校验策略

```
类型: TXT
名称: _dmarc
内容: v=DMARC1; p=none; rua=mailto:<你的Gmail>
```

> **`p=none` 是有意为之**。因为本方案借 Gmail 发信,DKIM 签名是 Gmail 的域(`gmail.com`),
> 跟你的发件域 `<你的域名>` 对不齐(DKIM alignment 不通过)。
> 若把策略设成 `p=quarantine` 或 `p=reject`,你自己发的信反而可能被对方拦掉。
> 先用 `p=none` 只监控不拦截最稳妥;`rua` 那个邮箱会定期收到送达报告。

> **关于 DKIM**:Cloudflare Email Routing 是「转发」不是「托管发件」,
> 加上发信走 Gmail,所以**你的域名这边没法给外发信加 DKIM 签名**,这是本免费方案的固有局限。
> 对个人 / 低频使用,SPF + `p=none` 的 DMARC 已经够用,正常不会进垃圾箱。
> 真要严格 DKIM 对齐 + `p=reject`,得上 Google Workspace 或专门的发信服务(如 Resend / Mailgun),那就不免费了。

---

## 4. 常见问题 / 坑

| 现象 | 原因 / 解法 |
|---|---|
| 收不到转发的信 | Cloudflare 那封 destination 验证邮件没点确认;或 MX 记录被别的服务覆盖了 |
| App Password 页面打不开 / 没入口 | 2FA 没开,先开两步验证 |
| 添加发信地址时 SMTP 报认证失败 | 密码填成了登录密码;应填 **16 位应用专用密码**,且用户名是真实 Gmail 地址 |
| 发出去的信进对方垃圾箱 | 补 3.1 的 SPF;DMARC 保持 `p=none`;别用 `p=reject` |
| 想要多个别名(如 `hi@`、`me@`) | Cloudflare 多建几条 Custom address,或直接开 Catch-all;Gmail 里逐个 "Send mail as" 添加 |
| 回复时发件人变回了 Gmail | Gmail 设置里把域名地址设为默认,或回信时手动在「发件人」下拉切换 |

---

## 5. 回滚备忘

| 改动 | 回滚方式 |
|---|---|
| Gmail 发信地址 | 设置 → 账号和导入 → Send mail as → 删除该地址 |
| 应用专用密码 | <https://myaccount.google.com/apppasswords> 里吊销对应那条 |
| Cloudflare 收信 | Email Routing → Routing rules 删规则,或整体 Disable Email Routing |
| DNS 记录 | Cloudflare DNS 里删掉/还原 SPF、DMARC、MX 记录 |

> Email Routing 关闭后,Cloudflare 自动加的 MX/SPF 不一定会自动撤,确认时去 DNS 列表手动核对一遍。

---

## 附:整体检查清单

- [ ] 域名 NS 已托管在 Cloudflare
- [ ] Gmail 开了 2FA
- [ ] Cloudflare Email Routing 启用,且 destination(你的 Gmail)已验证
- [ ] 测试:外部 → `<别名>@<你的域名>` 能收到
- [ ] Gmail 生成了应用专用密码
- [ ] Gmail "Send mail as" 添加并验证了域名地址(SMTP 587 + App Password)
- [ ] 测试:用域名地址发一封给外部邮箱,确认发件人显示正确
- [ ] SPF 含 `_spf.google.com`,DMARC `p=none` 已加
