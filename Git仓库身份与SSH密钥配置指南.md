# Git仓库身份与SSH密钥配置指南

本指南适用于Windows上的Git Bash。除生成、查看密钥和克隆仓库的命令外，其余命令均在需要配置的本地Git仓库目录中执行。克隆命令在希望存放项目的父目录中执行。

文中的`<用户名>`、`<仓库名>`、`<个人邮箱>`、`<提交署名>`和`<本地目录>`均为占位符，执行前替换为自己的信息。密钥文件名也是示例，已有个人密钥时使用实际文件名，无需重新生成。

按当前情况选择入口：

- 本地已有Git仓库：按第1节配置远程地址。使用SSH时，按需完成第2、3节，再按第4.1节指定私钥。
- 要克隆远程已有仓库并指定密钥：按需完成第2、3节，再按第4.2节克隆，不需要先执行第1节的添加远程命令。

两种入口完成后，都按第5至7节设置提交署名、查看配置并测试连接。

## 1. 配置远程仓库地址

GitHub远程地址有两种常见形式，选择其中一种：

| 连接方式 | 地址格式 | 认证方式 |
| --- | --- | --- |
| HTTPS | `https://github.com/<用户名>/<仓库名>.git` | 凭据管理器、访问令牌等 |
| SSH | `git@github.com:<用户名>/<仓库名>.git` | SSH密钥 |

首次添加名为`origin`的远程仓库时，选择下面一条命令执行：

```bash
# HTTPS
git remote add origin "https://github.com/<用户名>/<仓库名>.git"

# SSH
git remote add origin "git@github.com:<用户名>/<仓库名>.git"
```

如果已经存在`origin`，使用`set-url`修改地址，同样只执行所需的一条：

```bash
# 改成HTTPS
git remote set-url origin "https://github.com/<用户名>/<仓库名>.git"

# 改成SSH
git remote set-url origin "git@github.com:<用户名>/<仓库名>.git"
```

本文后续的SSH私钥配置仅在使用SSH远程地址时生效。

## 2. 生成个人SSH密钥

已有个人密钥时跳过本步骤。生成新密钥时，选择未被使用的文件名，避免覆盖已有密钥。

```bash
mkdir -p ~/.ssh
ssh-keygen -t ed25519 -C "<个人邮箱>" -f ~/.ssh/id_ed25519_personal
```

根据提示设置保护私钥的口令。

| 参数 | 含义 |
| --- | --- |
| `-t ed25519` | 使用Ed25519数字签名算法 |
| `-C "<个人邮箱>"` | 添加识别用的注释，不决定GitHub登录账号或提交署名 |
| `-f 路径` | 指定密钥文件的位置和名称 |

通常生成两个文件：

```text
~/.ssh/id_ed25519_personal       私钥，保留在本机
~/.ssh/id_ed25519_personal.pub   公钥，添加到GitHub
```

不要将私钥放进仓库或提交到GitHub。

## 3. 添加公钥到GitHub账号

查看公钥内容：

```bash
cat ~/.ssh/id_ed25519_personal.pub
```

如果没有对应的`.pub`文件，可以从私钥导出公钥并显示。这个命令输出的是公钥，不会输出私钥：

```bash
ssh-keygen -y -f ~/.ssh/id_ed25519_personal
```

复制公钥内容，然后在GitHub操作：

1. 点击头像，进入Settings。
2. 打开SSH and GPG keys，点击New SSH key。
3. 在Title中填写设备名称，例如“工作电脑”或“家里电脑”。
4. 将Key type设为Authentication Key。
5. 在Key中粘贴完整公钥，保存。

这里将公钥添加到个人账号。对应私钥可以以该账号的身份访问其有权限的仓库，不需要为每个仓库重复添加公钥。

仓库设置中的Deploy keys是另一种机制，通常用于只需访问某个仓库的服务器或自动化程序。

## 4. 为仓库指定个人私钥

可以复用同一GitHub账号已登记的个人密钥，不需要为每个仓库重新生成。前提是私钥文件已存在，对应公钥已添加到有权访问目标仓库的GitHub账号。

其他仓库中的`--local`配置不会自动应用到当前仓库。下面按已有本地仓库、尚未克隆两种情况，选择对应方式。

- `-i`指定私钥文件。
- `IdentitiesOnly=yes`限制SSH使用配置指定的身份，减少已加载的其他密钥造成的干扰。
- `core.sshCommand`记录SSH连接命令，不包含私钥内容。
- 这个仓库级设置不修改全局配置、其他仓库配置或任何密钥文件。

### 4.1 已有本地仓库

在该仓库目录中执行，将示例密钥文件名替换为实际私钥文件名：

```bash
git config --local core.sshCommand "ssh -i ~/.ssh/id_ed25519_personal -o IdentitiesOnly=yes"
```

仓库已克隆完成、需要补充或更换密钥时，也使用这条命令。

### 4.2 克隆远程已有仓库

在希望存放项目的父目录执行，将示例密钥文件名替换为实际私钥文件名：

```bash
git clone -c core.sshCommand="ssh -i ~/.ssh/id_ed25519_personal -o IdentitiesOnly=yes" \
  "git@github.com:<用户名>/<仓库名>.git" "<本地目录>"
```

命令中的反斜杠用于Git Bash多行续写，反斜杠后不要添加空格。

这里的`-c`放在`clone`后面，会把配置写入新仓库的`.git/config`，并在首次获取远程内容之前生效。后续在该仓库执行fetch、pull或push时，也会使用这项SSH配置，无需再执行第4.1节的设置命令。

注意下面两种写法的区别：

| 写法 | 配置生效范围 |
| --- | --- |
| `git clone -c key=value ...` | 写入新仓库配置，本次克隆和后续操作均可使用 |
| `git -c key=value clone ...` | 仅对本次Git命令生效，不保存到新仓库配置 |

克隆完成后进入新仓库，继续第5节的署名设置：

```bash
cd "<本地目录>"
```

克隆通常已自动配置origin和默认分支的上游，不需要再重复执行`git remote add origin`。

### 4.3 SSH配置的适用范围

上述配置不会用于HTTPS远程地址；已有仓库使用HTTPS时，需要先按第1节将远程地址改为SSH。

`~`表示当前用户的主目录。这条命令由Git调用SSH时使用；如果另有SSH配置或`GIT_SSH_COMMAND`环境变量，也需要检查其影响。环境变量`GIT_SSH_COMMAND`会覆盖`core.sshCommand`。

## 5. 设置当前仓库的署名和邮箱

无论本地自行建库还是从远程克隆，进入对应仓库目录后，都使用下面的命令设置提交署名：

```bash
git config --local user.name "<提交署名>"
git config --local user.email "<个人邮箱>"
```

- 署名可以是中文，不必与GitHub用户名一致。
- 邮箱使用GitHub账号已验证的邮箱，或GitHub提供的`noreply`邮箱。
- `--local`将设置写入当前仓库的`.git/config`，不修改其他仓库或全局配置。
- 这些设置影响之后创建的提交，不会自动改写已有提交。

署名、邮箱记录提交作者。SSH密钥负责远程认证，两者分别配置。只下载、阅读代码而不创建提交时，可以不单独设置署名。

## 6. 查看配置结果

查看远程地址：

```bash
git remote -v
```

查看实际使用的署名、邮箱和SSH命令，并显示配置来源：

```bash
git config --show-origin --get user.name
git config --show-origin --get user.email
git config --show-origin --get core.sshCommand
```

仓库级配置的来源应显示`file:.git/config`。如果显示用户主目录中的`.gitconfig`，说明该项来自全局配置。

查看密钥指纹，用于核对GitHub上的密钥记录：

```bash
ssh-keygen -lf ~/.ssh/id_ed25519_personal
```

## 7. 只读测试远程连接

```bash
git ls-remote origin
```

该命令使用当前仓库的远程地址和认证配置，读取远程分支、标签信息，不会提交或推送文件。空仓库可能没有引用可显示，执行成功时可以没有输出。

首次SSH连接可能需要确认服务器主机密钥，应先核对GitHub官方公布的主机指纹。这是GitHub服务器的身份，与自己的登录密钥不同。

如果出现连接超时，先检查网络和SSH端口的连通性；超时本身不能证明个人密钥错误。

## 8. 理解upstream上游分支

远程仓库配置记录仓库地址，上游配置记录某个本地分支默认跟踪哪个分支。它们是两个不同层次的配置。

例如，本地main跟踪origin远程仓库上的main分支：

```text
本地main ──跟踪──→ origin/main
```

`origin/main`是本地记录的远程跟踪引用。upstream是上游分支的角色名称，不是要求创建一个名叫upstream的分支。

上游关联会用于默认的`git pull`、`git status`和`git branch -vv`。在常见默认配置下，它也参与决定不带参数的`git push`如何推送；具体行为还受推送相关配置影响。

每个本地分支分别记录自己的上游。main已有上游，不代表新建的feat-xxx也已有合适的上游。从哪个分支创建新分支，与默认跟踪哪个分支是两件事；具体是否自动建立跟踪，还取决于创建命令和配置。

## 9. 首次推送与git push -u

先确认当前本地分支名：

```bash
git branch --show-current
```

假设本地分支为main，并且已经至少有一次提交，首次推送可以执行：

```bash
git push -u origin main
```

这条命令推送本地main，并为成功推送或已同步的分支保存上游关联。`-u`是`--set-upstream`的简写。

远程不需要预先存在main。对没有任何分支的空远程仓库，首次推送会创建远程分支。空仓库在推送前执行`git fetch --all`和`git branch -r`没有分支输出，是正常现象。

如果本地分支叫master，命令中的分支名相应改为master。

对新建的feat-xxx分支，提交修改后首次推送：

```bash
git push -u origin feat-xxx
```

建立的关系是：

```text
本地main     ──跟踪──→ origin/main
本地feat-xxx ──跟踪──→ origin/feat-xxx
```

关联正确后，在对应分支通常直接执行：

```bash
git push
git pull
```

远程分支的创建不依赖`-u`。下面的命令也可以推送并创建远程feat-xxx，只是不顺便保存上游关联：

```bash
git push origin feat-xxx
```

上游通常设置一次即可，不需要每次推送都加`-u`。通过`git clone`获取的默认分支，通常已经自动设置上游。

## 10. 查看、设置和修改上游关联

查看本地分支及其跟踪关系：

```bash
git branch -vv
```

输出示例：

```text
* feat-xxx abc1234 [origin/feat-xxx] 某次提交说明
```

方括号中的`origin/feat-xxx`表示上游。领先、落后的提交数量比较的是本地已记录的上游状态；需要最新信息时先执行`git fetch origin`。

如果远程已有feat-xxx，本地分支却没有关联，或关联到了错误的分支，可以设置或修改：

```bash
git fetch origin
git branch --set-upstream-to=origin/feat-xxx feat-xxx
```

其中，`origin/feat-xxx`是目标上游，最后的`feat-xxx`是要修改配置的本地分支。目标远程跟踪引用需要已经存在，例如通过fetch获取。

这条命令也可以简写为：

```bash
git branch -u origin/feat-xxx feat-xxx
```

| 命令 | 作用 |
| --- | --- |
| `git push -u origin feat-xxx` | 推送分支，并保存上游关联 |
| `git branch -u origin/feat-xxx feat-xxx` | 仅设置或修改上游关联，不推送、不合并、不改写提交 |

取消关联：

```bash
git branch --unset-upstream feat-xxx
```

取消关联不会删除本地分支或远程分支。更换远程分支名称、改用另一个remote，或修正错误关联时，可能需要重新设置上游。日常提交不需要反复调整。

## 11. 查看远程分支

查看本地已记录的远程分支：

```bash
git branch -r
```

先更新已配置的远程仓库信息，再查看：

```bash
git fetch --all
git branch -r
```

`git fetch`更新远程跟踪信息，不会自动把这些修改合并进当前工作分支。

直接查询origin服务器上实际存在的所有分支：

```bash
git ls-remote --heads origin
```

这条命令显示提交ID和`refs/heads/分支名`，不会替代fetch更新本地的远程跟踪引用。命令成功但没有输出时，远程可能确实没有分支。

## 12. 换一台电脑继续使用

推荐每台电脑分别生成密钥，并将各自的公钥添加到同一个GitHub账号。私钥分别留在对应电脑。

仓库中的文件通过提交、推送和拉取同步；`.git/config`中的仓库级配置不会随推送同步。另一台电脑需要重新设置署名、邮箱，以及适合该电脑的私钥路径。

在GitHub的SSH and GPG keys页面，可以按设备名称管理或移除公钥。

## 官方参考

- [Git配置说明：core.sshCommand](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coresshCommand)
- [Git：克隆仓库并保存配置](https://git-scm.com/docs/git-clone)
- [Git：推送与设置上游](https://git-scm.com/docs/git-push)
- [Git：查看分支与修改跟踪关系](https://git-scm.com/docs/git-branch)
- [GitHub：生成SSH密钥](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- [GitHub：添加SSH公钥](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
- [GitHub：设置提交邮箱](https://docs.github.com/en/account-and-profile/how-tos/email-preferences/setting-your-commit-email-address)
- [GitHub：SSH服务器主机指纹](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)
