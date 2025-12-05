---
title: 环境配置：Github GPG签名
date: 2025-12-05 14:04:26
tags: 环境配置
---

## 1. 安装 GPG 工具（适配 Apple 芯片）

```bash
# 安装核心工具：gnupg（GPG主程序）+ pinentry-mac（Mac友好的密码输入弹窗）
brew install gnupg pinentry-mac

# 创建GPG配置目录（权限必须700，否则GPG会报错）
mkdir -m 700 -p ~/.gnupg

# 配置pinentry-mac为默认密码输入程序（Apple Silicon路径固定为/opt/homebrew）
echo "pinentry-program /opt/homebrew/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf

# 配置环境变量（确保GPG Agent正常工作，zsh是Mac默认终端）
echo 'export GPG_TTY=$(tty)' >> ~/.zshrc
echo 'gpgconf --launch gpg-agent' >> ~/.zshrc
source ~/.zshrc

# 重启GPG Agent使配置生效
killall gpg-agent
```

## 2. 生成 GPG 密钥对

```bash
# 启动密钥生成向导（全程按提示操作）
gpg --full-generate-key
```

配置指引开始后, 一路回车即可, 直到要求填 github 的 id 和邮箱, comment 可以直接回车跳过
填完后设置密码(低强度密码会弹出警告, 忽略即可)

## 3. 提取公钥并添加到 Github 上

### 获取密钥 ID

```bash
# 列出密钥列表，获取密钥ID（重点看sec行后/后的字符串）
gpg --list-secret-keys --keyid-format=long
```

输出示例（7D9F898F12345678 就是密钥 ID，复制下来）：

```bash
sec   rsa4096/7D9F898F12345678 2025-12-05 [SC]
      1234567890ABCDEF1234567890ABCDEF12345678
uid                 [ 绝对信任 ] 你的姓名 <你的邮箱>
ssb   rsa4096/87654321FEDCBA09 2025-12-05 [E]
```

### 导出并上传公钥

导出公钥并复制到剪贴板（替换为你的密钥 ID）

```bash
gpg --armor --export 7D9F898F12345678 | pbcopy
```

之后登录 Github, 上传 GPG 公钥

## 4. 配置 Git 关联 GPG, 并开启自动签名

```bash
# 关联GPG密钥（替换为你的密钥ID）
git config --global user.signingkey 7D9F898F12345678

# 开启全局自动签名（所有commit都用GPG签名，无需手动加-S参数）
git config --global commit.gpgsign true

# 指定Git使用的GPG程序路径（Apple Silicon必配）
git config --global gpg.program gpg
```

## 5. 使用 Mac 钥匙串配置永久免密提交

### 前置条件

确认已安装 pinentry-mac（前置条件）

```bash
# 检查是否安装，Apple Silicon 路径应为/opt/homebrew/bin/pinentry-mac

which pinentry-mac

# 若输出为空，先安装：brew install pinentry-mac
```

### 配置 GPG Agent

编辑 GPG Agent 配置文件

```bash
vim ~/.gnupg/gpg-agent.conf
```

写入下面的内容

```bash
pinentry-program /opt/homebrew/bin/pinentry-mac
default-cache-ttl 0
max-cache-ttl 0
```

重启 GPG Agent 使配置生效

```bash
killall gpg-agent
gpgconf --launch gpg-agent
```

### 首次使用

首次提交代码时会弹出 pinentry-mac 的密码输入窗口，需完成 2 个操作：

- 输入你生成 GPG 密钥时设置的密码短语；
- 勾选「Save in Keychain」（存入钥匙串）→ 点击「OK」。
