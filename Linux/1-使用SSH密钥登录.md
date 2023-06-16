# 使用SSH密钥登录并设置禁用密码登录

## 配置Linux服务器使用SSH密钥登录

### 生成密钥

```bash
#以下 ssh-keygen 命令默认在 ~/.ssh 目录中生成 4096 位 SSH RSA 公钥和私钥文件。 如果当前位置存在 SSH 密钥对，这些文件将被覆盖
ssh-keygen -m PEM -t rsa -b 4096
```

以下示例显示可用于创建 SSH RSA 密钥对的其他命令选项。 如果当前位置存在 SSH 密钥对，这些文件将被覆盖。

```bash
ssh-keygen \
    -m PEM \    # 将密钥的格式设为 PEM
    -t rsa \    # 要创建的密钥类型，本例中为 RSA 格式
    -b 4096 \   # 密钥的位数，本例中为 4096
    -C "azureuser@myserver" \   # 追加到公钥文件末尾以便于识别的注释。 通常以电子邮件地址用作注释，但也可以使用任何最适合你基础结构的事物。
    -f ~/.ssh/mykeys/myprivatekey \     # 私钥文件的文件名（如果选择不使用默认名称）。 追加了 .pub 的相应公钥文件在相同目录中生成。 该目录必须存在。
    -N mypassphrase     # 用于访问私钥文件的其他密码
```

### 使用 ssh-copy-id 将密钥复制到服务器

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@myserver
```

### 使用 ssh-agent 来存储私钥密码

为了避免在每次 SSH 登录时键入私钥文件密码，可以使用 ssh-agent 来缓存私钥文件密码。 如果使用 Mac，macOS Keychain 在用户调用 ssh-agent 时会安全存储私钥密码。

验证并使用 ssh-agent 和 ssh-add 将密钥文件的情况通知给 SSH 系统，这样就无需交互使用密码。

```bash
eval "$(ssh-agent -s)"
```

现在，使用命令 ssh-add 将私钥添加到 ssh-agent。

```bash
ssh-add ~/.ssh/id_rsa
```

### 禁用服务器密码登录

```bash
# 修改配置文件
sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
# 重启ssh服务
systemctl restart sshd
```
