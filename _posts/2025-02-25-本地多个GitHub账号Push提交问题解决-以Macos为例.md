---
title: 2025-02-25-本地多个GitHub账号Push提交问题解决-以MacOS为例
date: 2025-02-25 17:00:00 +0800
categories: [Git]
tags: [Git, GitHub]
---
### 1. 起因: 
在clone luxstar(也就是本GitHub账号) 项目下来之后修改完, push的时候出现以下错误
```
Error: Permission denied
```
### 2. 原因:
发现是GitHub登陆私钥混乱, 造成以账号a的身份 push 账号b的项目
### 3. 解决
1. 创建好GitHub的ssh的公私密钥, 参考命令, 此时会创建四个文件, 分别是:
   1. account1.github
   2. account1.github.pub
   3. account2.github
   4. account2.github.pub
```
cd ~/.ssh/
mkdir github && cd github
ssh-keygen -t ed25519 -C "account1@gmail.com"
ssh-keygen -t ed25519 -C "account2@gmail.com"
```
2. 登陆GitHub account1 进入 [SSH and GPG keys](https://github.com/settings/keys) 页面, 点击New SSH key 把account1.pub添加进去!
账号account2同理
![img-description](assets/img/2025-02-25-本地多个GitHub账号Push提交问题解决-以Macos为例-1.png)
_SSH and GPG keys_
3. 编写脚本到.zshrc 或者.bashrc 文件中

> 写完记得source .zshrc/.bashrc 一下
{: .prompt-warning }

```shell
hub.account1(){
    echo "start clone $1..." &&
    git clone "git@account1.github.com:account1/$1" &&
    cd "$1" &&
    git config --local user.name "account1" &&
    git config --local user.email "account1@gmail.com" &&
    cd - &&
    echo "==>success" || echo "==>fail!!"
}
hub.account2(){
    echo "start clone $1..." &&
    git clone "git@account2.github.com:account2/$1" &&
    cd "$1" &&
    git config --local user.name "account2" &&
    git config --local user.email "account2@gmail.com" &&
    cd - &&
    echo "==>success" || echo "==>fail!!"
}
```

### 4. 最后
完成上述操作之后即可通过命令 hub.account1/account2 + 项目名 完成clone, 修改完成之后, git push 即可正常提交了
```
# 例如, 克隆账号1 account1 的项目, 账号2 account2 同理
hub.account1 account1.github.io 
# 修改之后...
git add .
git commit -m 'change ...'
git push
```