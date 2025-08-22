本文主要实现笔记上传github仓库，保证数据完整性

涉及服务相关信息

| 服务名称 | 地址                                | 版本         |
| -------- | ----------------------------------- | ------------ |
| git      | https://git-scm.com/downloads/win   | 2.50.1       |
| typora   | https://typoraio.cn                 | 官网最新     |
| github   | https://github.com/settings/profile | 官网地址注册 |

### **环境验证**

确认 Git 已正确安装：

```bash
git --version  # 显示当前 Git 版本，验证安装成功
```

### **SSH 密钥对生成（用于与 GitHub 加密通信）**

```bash
# 生成 RSA 算法的密钥对，长度 4096 位，添加邮箱作为注释
ssh-keygen.exe -t rsa -b 4096 -C "你的邮箱地址"
```

- 执行后按提示操作（可直接回车使用默认路径和空密码），生成的密钥对默认存放在 `C:\Users\用户名\.ssh\` 目录（`id_rsa` 私钥，`id_rsa.pub` 公钥）。

### **GitHub 准备工作**

```bash
1.注册并登录 GitHub 账号
2.创建新仓库（Repository），记录仓库地址（如 git@github.com:用户名/仓库名.git）
3.上传公钥：
  打开 id_rsa.pub 文件，复制全部内容
  在 GitHub 点击头像 → Settings → SSH and GPG keys → New SSH key，粘贴公钥并保存
```

### **Git 全局配置（首次使用需配置）**

```bash
# 设置提交代码时显示的用户名（建议与 GitHub 用户名一致）
git config --global user.name "你的用户名"

# 设置提交代码时的关联邮箱（建议与 GitHub 注册邮箱一致）
git config --global user.email "你的邮箱地址"
```

### **本地仓库初始化与关联远程仓库**

```bash
# 进入项目文件夹后，初始化本地 Git 仓库
git init

# 关联远程仓库（替换为你的仓库地址）
git remote add origin git@github.com:121202011/typora.git

# 查看已关联的远程仓库列表（验证是否添加成功）
git remote -v
```

### **分支配置**

```bash
# 将当前分支重命名为 main（GitHub 仓库默认分支通常为 main）
git branch -M main

# 设置本地 main 分支与远程 origin/main 分支关联（后续可直接用 git push/pull）
git branch --set-upstream-to=origin/main main
```

### **代码提交流程**

```bash
# 查看工作区文件变更状态（哪些文件被修改/新增/删除）
git status  # 可简写为 git st

# 将所有变更（新增、修改、删除）添加到暂存区
git add -A  # -A 等价于 --all，包含工作区所有变更

# 将暂存区的变更提交到本地版本库，添加提交说明（必填，清晰描述变更内容）
git commit -m "提交说明：例如 初始化项目 / 添加登录功能 / 修复XXbug"

# 将本地提交推送到远程 GitHub 仓库（首次推送可能需要输入密码或 SSH 密钥密码）
git push origin main  # 由于已配置关联，可简化为 git push
```

