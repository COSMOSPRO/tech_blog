---
title: "GitHub SSH Key 备份与恢复方案"
date: 2026-08-22T16:41:56Z
tags: ["github", "ssh", "security", "macos"]
author: "COSMOSPRO"
draft: false
---

# GitHub SSH Key 备份与恢复方案

> **适用对象**：`~/.ssh/github_key` / `~/.ssh/github_key.pub`（ed25519）
> **绑定账号**：GitHub 用户各自的账号
> **用途**：本机 push `git@github.com:<owner>/<repo>.git` 等仓库

> ⚠️ 本文档是**公开版**，已脱敏：
> - 真实指纹用 `<YOUR_FINGERPRINT>` 替代
> - 私钥 sha256 已删除
> - 公钥原文已删除
> - 备份介质细节已留空
> 完整版（含私钥元信息）保留在你本机的 `~/Documents/github_key_metadata.md`。

<!--more-->

---

## 0. 一分钟确认现状

```bash
ls -la ~/.ssh/github_key ~/.ssh/github_key.pub
ssh-keygen -lf ~/.ssh/github_key.pub   # 指纹应与元信息里的一致
ssh -T git@github.com                  # 应看到：Hi <your-username>! You've successfully authenticated...
```

---

## 1. 备份（首次 + 每次重装 Mac 之前）

### 1.1 收集三样东西

| 文件 | 路径 | 备注 |
|---|---|---|
| 私钥 | `~/.ssh/github_key` | **绝对不能泄露**，必须加密或放受信介质 |
| 公钥 | `~/.ssh/github_key.pub` | 公开，可以随便贴 |
| SSH config | `~/.ssh/config` 中 `Host github.com` 段 | 决定 Git 用哪把 key、是否绕过代理 |

### 1.2 打包命令

```bash
mkdir -p ~/.ssh/_backups
TS=$(date +%Y%m%d_%H%M%S)

tar czf ~/.ssh/_backups/github_key_backup_${TS}.tar.gz \
  -C ~/.ssh github_key github_key.pub

awk '/^Host github.com/{p=1} /^Host /{if($2!~/github.com/&&NR>1)p=0} p' \
  ~/.ssh/config > ~/.ssh/_backups/github_ssh_config_${TS}.txt
```

解包后权限自动保留 `600 / 644`。

### 1.3 记录元信息（**这一步必做**，单独存放）

把下面这段存为 `KEY_METADATA.md`，放在**仓库外**（如 `~/Documents/`）：

```yaml
key_type: ed25519
fingerprint: <YOUR_FINGERPRINT>            # ssh-keygen -lf .pub 拿
label_on_github: <YOUR_GITHUB_KEY_TITLE>
created_on: <YYYY-MM-DD>
passphrase: <none | yes>
permissions_legacy: "600/644, owned by $(whoami)"
linked_account: <your-github-username>
proxy_bypass: ~/.ssh/config github.com 段加了 ProxyCommand none
# sha256_of_private_key: <sha256sum ~/.ssh/github_key>  ← 仅本地留存
```

### 1.4 加密并搬到至少两个异地

私钥若**无 passphrase**，任何人拿到备份文件 = 拿到你 GitHub 写权限。**必须加密**或放在受信环境。推荐三选二：

| 介质 | 加密方式 | 适合 |
|---|---|---|
| 1Password / Bitwarden 附件 | 自动加密 | 最方便，Mac/手机同步 |
| 加密 U 盘 / Time Machine 加密备份 | APFS / FileVault | 离线冷备 |
| iCloud Drive（启用高级数据保护） | Apple 端到端加密 | 自动同步 |

> **不要**裸传到普通云盘 / 邮箱附件。

加密示例（macOS + 一个强密码）：

```bash
openssl enc -aes-256-gcm -salt -pbkdf2 \
  -in ~/.ssh/_backups/github_key_backup_${TS}.tar.gz \
  -out ~/.ssh/_backups/github_key_backup_${TS}.tar.gz.enc
rm -P ~/.ssh/_backups/github_key_backup_${TS}.tar.gz   # macOS 三次覆写删除
```

---

## 2. 换机器 / 重新安装 macOS 后的恢复

### 2.1 恢复文件

```bash
# A. 加密备份
openssl enc -d -aes-256-gcm -pbkdf2 \
  -in ~/Downloads/github_key_backup_*.tar.gz.enc \
  -out /tmp/github_key_backup.tar.gz
tar xzf /tmp/github_key_backup.tar.gz -C ~/.ssh/

# B. 直接从密码管理器下载的私钥
# 把 github_key / github_key.pub 放进 ~/.ssh/

chmod 600 ~/.ssh/github_key
chmod 644 ~/.ssh/github_key.pub
cat ~/Downloads/github_ssh_config_*.txt >> ~/.ssh/config
```

### 2.2 让 ssh 找到这把 key

```bash
# 走 ssh-agent + macOS keychain
ssh-add --apple-use-keychain ~/.ssh/github_key
ssh-add -l

# 或确认 ~/.ssh/config 的 Host github.com 段：
#   IdentityFile ~/.ssh/github_key
#   IdentitiesOnly yes
#   ProxyCommand none
```

### 2.3 验证

```bash
ssh-keygen -lf ~/.ssh/github_key.pub   # 指纹应与元信息一致
ssh -T git@github.com                  # 应看到 Hi <username>!
```

### 2.4 把 key 注册到 GitHub（如尚未注册）

**关键步骤**：私钥能恢复 ≠ GitHub 还认你。把 `github_key.pub` 内容加到 https://github.com/settings/keys：

```bash
cat ~/.ssh/github_key.pub
# 整行 ssh-ed25519 AAA... 复制粘贴；Title 与元信息里的 label_on_github 一致
```

---

## 3. 故障排查速查

| 现象 | 原因 | 解决 |
|---|---|---|
| `Permission denied (publickey)` | ssh 找不到这把 key | `ssh-add ~/.ssh/github_key` 或检查 `~/.ssh/config` |
| `Could not resolve hostname github.com` | 开了系统代理但没绕过 | 确认 `~/.ssh/config` `ProxyCommand none` |
| `Permission to X denied` | key 在 GitHub 已注册，但本机没装 | 把 pub key 加到 https://github.com/settings/keys |
| `Load key "github_key": invalid format` | 文件被加了 BOM 或改了换行 | `xxd ~/.ssh/github_key \| head` 确认首字节是 `0x2d 0x2d 0x2d 0x2d` |
| `Permissions 0644 for 'github_key' are too open` | 私钥权限过宽 | `chmod 600 ~/.ssh/github_key` |
| macOS 每次开机都问 passphrase | keychain 没存 | `ssh-add --apple-use-keychain ~/.ssh/github_key` |

---

## 4. 长期建议

1. **加 passphrase**：`ssh-keygen -p -f ~/.ssh/github_key`。改后指纹变，必须把新公钥重新注册到 GitHub，旧 key 至少保留 7 天再删。
2. **每台机器一把 key**：不要同一把私钥拷到多台设备。每台机器用独立 key，单台失窃可以单独撤销。
3. **保留旧 key 至少 7 天**：换 key 前先用新 key 验证通过再删旧 key。
4. **每年演练一次恢复**：解包 → 改权限 → `ssh -T`，跑通说明备份是新鲜的。
