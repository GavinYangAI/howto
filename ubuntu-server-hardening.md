# Ubuntu 服务器基础安全加固 + tmux 持久会话配置参考

> 适用场景:给远程自动化任务(如 codex 自动修 bug 并推送 GitHub)准备的 Ubuntu 服务器。
> 系统基线:Ubuntu 24.04 LTS。其他版本命令大同小异。
>
> 约定:下文用 `<SSH_PORT>` 代表你实际使用的 SSH 端口(建议改成非 22 的高位端口)。
> 执行命令前请把它替换成真实端口。

---

## 0. 加固前先做的事

1. **全程保留一个已登录的会话**。改 SSH 配置后另开新连接验证能登录,再关旧连接,避免把自己锁死。
2. (若打算启用「仅密钥登录」)先把公钥放进 `~/.ssh/authorized_keys` 并验证可用,再关密码登录。

---

## 1. SSH 加固

用独立 drop-in 文件,便于回滚,不动主配置 `/etc/ssh/sshd_config`。

`/etc/ssh/sshd_config.d/99-hardening.conf`:
```
# PasswordAuthentication no        # 进阶项:仅密钥登录(见下方说明)。默认保留密码登录,不强制。
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password  # root 仅允许密钥登录(如不需 root 登录可设 no)
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no                   # headless 服务器无需
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 0              # 0 = 不主动断开空闲连接;按需调整
```

应用(注意先校验语法,再用 reload 不用 restart,避免断现有连接):
```bash
sshd -t                  # 校验语法,无输出即 OK
systemctl reload ssh
sshd -T | grep -E 'passwordauthentication|permitrootlogin|x11forwarding'  # 确认生效
```

> **是否关闭密码登录?**
> - 默认**保留密码登录**(上面那行注释着),普通用户更习惯,配合 fail2ban(第 3 节)已能挡住绝大多数暴力破解。
> - 想进一步加固、且已确认密钥可登录,再去掉 `PasswordAuthentication no` 前的 `#`,改成仅密钥登录(最安全,杜绝爆破)。**关之前务必先验证密钥能登录,否则会锁死。**
> - 若保留密码登录,请确保所有账号都用**强密码**。

> **关于空闲超时**:`ClientAliveInterval 0` 表示服务器不主动断开发呆的连接。
> 若想自动清理空闲会话(更安全),可设 `ClientAliveInterval 300` + `ClientAliveCountMax 2`(约 10 分钟无响应断开)。
> 注意它只管「服务器主动断」,管不了真实网络断线 —— 后者靠下文 tmux 解决。

---

## 2. 防火墙 ufw

默认拒绝入站、放行出站,只开 SSH 端口。**务必先放行 SSH 端口再 enable**,否则锁死。

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow <SSH_PORT>/tcp comment 'SSH'
ufw --force enable
ufw status verbose        # 确认
```

> 出站全放行,`git push`、`gh`、`npm/pip` 拉包都正常。
> 以后对外开新服务(如 web 端口)再手动 `ufw allow <port>/tcp`。

---

## 3. fail2ban(防暴力破解)

> 若保留了密码登录,这一节尤其重要 —— 它是挡暴力破解的主力。

```bash
apt-get update && apt-get install -y fail2ban
```

`/etc/fail2ban/jail.local`:
```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
backend  = systemd

[sshd]
enabled  = true
port     = <SSH_PORT>     # 必须和实际 SSH 端口一致
maxretry = 4              # 4 次失败
bantime  = 2h             # 封 2 小时
```

```bash
systemctl enable --now fail2ban
fail2ban-client status sshd     # 查看封禁情况
```

---

## 4. 自动安全更新

Ubuntu 一般默认启用 `unattended-upgrades`。确认 / 启用:
```bash
systemctl is-active unattended-upgrades
# 若未装:apt-get install -y unattended-upgrades && dpkg-reconfigure -plow unattended-upgrades
```

---

## 5. tmux:断线不丢任务

**问题**:普通 SSH 跑的进程,连接一断就收到 SIGHUP 被杀掉。
**方案**:任务都在 tmux 里跑。tmux 会话独立于 SSH 连接,断线后任务在服务器继续跑,重连即可接回。

### 5.1 自动进 tmux

在登录用户的 `~/.bashrc` 末尾追加。**三重防护**确保只在交互式 SSH 登录时触发,不影响非交互的自动化(`ssh host '命令'`、scp、git-over-ssh、cron):

```bash
# 交互式 SSH 登录时自动进入持久 tmux 会话
if [[ $- == *i* ]] && [[ -n "$SSH_TTY" ]] && [[ -z "$TMUX" ]]; then
    tmux attach -t main 2>/dev/null || tmux new -s main
fi
```

> 三个条件缺一不可:`$-` 含 `i`(交互式)、`$SSH_TTY` 非空(真实 SSH 终端)、`$TMUX` 为空(未嵌套)。
> 这样普通 `ssh host '命令'` 这类自动化绝不会被困进 tmux。

### 5.2 tmux 配置

`~/.tmux.conf`:
```bash
# ---- 基础体验 ----
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",*256col*:Tc"   # 真彩色
set -g history-limit 100000                  # 滚动历史 10 万行
set -g mouse on                              # 鼠标:滚轮翻页/点击切窗格/拖拽调整
set -g base-index 1
setw -g pane-base-index 1
set -g renumber-windows on
set -sg escape-time 10                        # 减少 ESC 延迟

# ---- 状态栏:高亮会话名 ----
set -g status-interval 5
set -g status-style 'bg=colour236 fg=colour250'
set -g status-left '#[bg=colour33,fg=white,bold] #S #[default] '
set -g status-left-length 30
set -g status-right '#[fg=colour245]#{?client_prefix,#[bg=colour202]#[fg=white] PREFIX ,}#[fg=colour245] %Y-%m-%d %H:%M '
set -g status-right-length 60
setw -g window-status-current-style 'fg=colour33,bold'

# Ctrl+b 然后 r 重载配置
bind r source-file ~/.tmux.conf \; display "tmux 配置已重载"
```

### 5.3 日常用法

登录后自动在 tmux `main` 会话里,直接跑任务即可。

| 操作 | 结果 |
|---|---|
| 网络突然断开 | 任务继续跑,重新登录自动接回现场 |
| `Ctrl+b` 然后 `d`(detach) | 主动离开,任务后台继续,下次接回 |
| 在 tmux 里敲 `exit` | **结束会话**(任务也停) |

常用命令:`tmux ls`(列会话)、`tmux attach -t main`(手动接回)、`tmux new -s 名字`(开新会话跑并行任务)。

> 记住:**只有 `exit` 会结束会话**。断网 / detach / 关终端都只是断开连接,任务照跑 —— 这正是要的效果。

---

## 6. 回滚备忘

所有改动都在独立文件,删掉即恢复默认:

| 改动 | 回滚方式 |
|---|---|
| SSH 加固 | `rm /etc/ssh/sshd_config.d/99-hardening.conf && systemctl reload ssh` |
| fail2ban | `systemctl disable --now fail2ban && apt remove fail2ban` |
| 防火墙 | `ufw disable` |
| 自动进 tmux | 删 `~/.bashrc` 末尾的 tmux 代码块 |
| tmux 配置 | `rm ~/.tmux.conf` |

> 改 SSH 配置后务必 `sshd -t` 校验语法,再 `systemctl reload ssh`(reload 不断现有连接),并**另开新连接确认能登录**后再关旧连接,避免锁死。

---

## 7. 自动化任务的凭据建议

给服务器配 GitHub 等凭据时:
- 用**细粒度 PAT 或 deploy key,仅授权目标仓库**,不要用账号全权 token。
- 自动 push 建议用**单独的受限身份**,缩小误操作 / 泄露的影响面。

---

## 附:其他注意点

- 云服务器厂商常预装监控 / 安全 agent。装包时 `needrestart` 可能顺带重启这些服务,偶发报错多为预存问题,与本加固无关,可单独排查。
- 区分两层防护:`ufw` 是入站访问控制,`fail2ban` 是针对暴力破解的动态封禁,二者互补。
