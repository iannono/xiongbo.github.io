---
layout: post
title:  "如何使用capistrano部署服务器.1!"
date:   2014-01-10 17:06:25
categories: capistrano ruby deploy
---

这个主题将分为四部分内容，第一部分是介绍服务器的配置，第二部分将介绍部署代码的编写。

## 配置SSH验证

因为我们的目的是实现远程发布， 所以需要两个方面的授权。

一方面是从我们本地机器到服务器的验证， 这种情况可以通过SSH Key Agent实现。 

另一方面是从服务器访问我们的代码托管服务器。 这种情况可以通过SSH agent forwarding, HTTP authentication, 或者deploy keys三种方式实现。 这里， 我们以GitHub为例， 采用SSH agent forwarding的方式。 其他方式参考[这里](http://www.capistranorb.com/documentation/getting-started/authentication-and-authorisation/)


### 从本地到服务器

首先， 通过SSH登陆远程服务器， 并建立如下的用户( -l 参数是为了锁定该用户， 这样就无法通过输入密码的方式进行登陆了 )：

```
root@remote $ adduser deploy
root@remote $ passwd -l deploy
```

然后， 我们需要在本地机器上生成我们的ssh密匙， 并根据提示输入安全密码：

```
me@localhost $ ssh-keygen -t rsa -C 'me@my_email_address.com'
```
    
接下来， 我们需要将本地的公钥拷贝到远程服务器上的`.ssh/authorized_keys`文件中， 这样我们再次通过SSH访问远程服务器的时候， 就不需要输入安全密码了。

通常， 我们可以直接通过以下命令进行拷贝， 并自动配置好目录的权限：

```bash
me@localhost $ ssh-copy-id -i ~/.ssh/id_rsa.pub deploy@remote
```
    
该命令实际进行的是如下操作：

```bash
me@localhost $ ssh root@remote.com
root@remote $ su - deploy
deploy@remote $ cd ~
deploy@remote $ mkdir .ssh
deploy@remote $ echo "ssh-rsa jccXJ/JRfGxnkh/8iL........dbfCH/9cDiKa0Dw8XGAo01mU/w== /Users/me/.ssh/id_rsa" >> .ssh/authorized_keys
deploy@remote $ chmod 700 .ssh
deploy@remote $ chmod 600 .ssh/authorized_keys
```
    
现在， 已经可以不用输入密码， 即能通过SSH命令登录远程主机了， 通过如下的命令能够验证操作是否正确：

```bash
me@localhost $ ssh deploy@remote.com 'hostname; uptime' 
remote.com
19:23:32 up 62 days, 44 min, 1 user, load average: 0.00, 0.01, 0.05
```
    

### 从服务器到托管服务器

简单来说， 通过SSH Agent Forwarding 我们可以使用本地的ssh key来进行远程服务器到托管服务器的认证。

首先通过以下命令确保你本地的SSH Agent 已经载入了你的ssh key：
    
```bash
me@localhost $ ssh-add
me@localhost $ ssh-add -l
# 这里会显示你的ssh key信息
```
    
然后执行如下命令来测试远程服务器是否能够正常的读取你的应用：

```bash
me@localhost $ ssh -A deploy@remote.com 'git ls-remote git@github.com:me/my-repo.git'
```
    
如果出现`The remote end hung up unexpectedly`或者以下的问题，需登录到远程服务器， 使用你添加的发布用户执行一次`ssh git@github.com`，并根据提示执行操作。 原因在[这里](https://help.github.com/articles/deploying-with-capistrano)

```bash
The authenticity of host 'github.com (207.97.227.239)' can't be established.
# RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
# Are you sure you want to continue connecting (yes/no)?
```


## 配置用户授权

为了让服务器（nginx、apache）能够顺利的访问我们部署的应用， 需要对deploy用户进行授权。

```bash
me@localhost $ ssh root@remote
# Capistrano will use /var/www/....... where ... is the value set in
# :application, you can override this by setting the ':deploy_to' variable
root@remote $ deploy_to=/var/www/my-repo
root@remote $ mkdir ${deploy_to}
root@remote $ chown deploy:deploy ${deploy_to}
root@remote $ umask 0002
root@remote $ chmod g+s ${deploy_to}
root@remote $ mkdir ${deploy_to}/{releases,shared}
```

需要注意的是`chmod g+s`这个命令，**s**代表在文件执行时，将进程的属主或组ID置为该文件的文件属主， 在这里即代表所有在`deploy_to`下新建的文件，都会继承相同的组权限，即deploy。

至此，我们就可以开始编写部署代码了！


## 参考

http://www.capistranorb.com/
http://railscasts.com/episodes/335-deploying-to-a-vps?view=asciicast

