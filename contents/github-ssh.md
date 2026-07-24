## 配置 GitHub SSH

### 1. 配置 Git 提交信息

```bash
git config --global user.name "Senyang Su"
git config --global user.email "susenyang@163.com"
```

其中：

* `user.name` 会显示在 Git 提交记录中；
* `user.email` 用于让 GitHub 将提交关联到对应账号；
* 建议使用已经在 GitHub 中验证过的邮箱。

检查配置：

```bash
git config --global user.name
git config --global user.email
```

### 2. 生成 SSH 密钥

```bash
ssh-keygen -t ed25519 -C "susenyang@163.com"
```

执行后一路按回车，使用默认配置。

默认会生成两个文件：

```text
/root/.ssh/id_ed25519
/root/.ssh/id_ed25519.pub
```

其中：

* `id_ed25519` 是私钥，不能泄露；
* `id_ed25519.pub` 是公钥，可以添加到 GitHub。

### 3. 查看并复制公钥

```bash
cat ~/.ssh/id_ed25519.pub
```

复制终端输出的完整一行公钥。

### 4. 将公钥添加到 GitHub

进入 GitHub：

```text
Settings
→ SSH and GPG keys
→ New SSH key
```

填写：

```text
Title: AutoDL
Key type: Authentication Key
Key: 粘贴刚才复制的公钥
```

然后点击：

```text
Add SSH key
```

### 5. 测试 SSH 连接

```bash
ssh -T git@github.com
```

第一次连接时会提示确认主机指纹，输入：

```text
yes
```

连接成功后会显示类似：

```text
Hi Sy-SU! You've successfully authenticated, but GitHub does not provide shell access.
```

出现这段内容说明 SSH 配置成功。

### 6. 使用 SSH 地址克隆仓库

```bash
git clone git@github.com:Sy-SU/仓库名.git
```

不要再使用 HTTPS 地址：

```text
https://github.com/Sy-SU/仓库名.git
```
