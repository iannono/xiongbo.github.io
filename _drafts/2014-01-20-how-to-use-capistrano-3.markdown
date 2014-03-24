经过前两篇的配置， 我们现在可以尝试执行一下capistrano3的检查任务：`cap staging deploy:check`。 

在执行之前， 将**log_level** 改为 **:debug**, 否则，可能会看不到git任务的失败信息。同时指定发布目录

```
# deploy.rb
set :log_level, :debug
set :deploy_to, '/var/www/test_app'
set :format, :pretty
set :keep_releases, 3 #指定在服务器上保存几个发布版本
```

这个任务会执行以下几步操作：

* 检查ruby版本， 如果你使用了rbenv gem的话
* 检查ssh的连接情况，是否能够正确上传脚本
* 检查能否正确读取git repo的信息（即执行`git ls-remote`， 如果提示`Permission denied`, 需要执行`ssh add`命令）
* 在你的发布目录下新建releases和shared文件夹。（**如果上面的任务失败，则不会执行该任务**）

这里需要说明的是： 每次运行deploy任务，都会在releases下新建一套应用代码的副本，以时间戳命名。

如果成功的建立了releases版本， 则会同时在发布目录下建立一个指向当前发布版本的系统连接， 并命名为current,  也是我们的http服务器应该指向的地址。

而shared文件夹中存放的是asset中静态文件以及用户上传的文件等。

## 发布

上面的命令成功后， 我们可以就执行`cap staging deploy`，看看默认发布的信息。

执行过程中可能会出现无法执行bundle命令的错误， 这是因为capstrano 默认使用的是non-login, non-interactive shell。 

简单来说，就是使用capistrano执行这些命令的时候， 是不会像你ssh到远程服务器那样， 为你建立loginshell， 并载入 .dotfiles (.bashrc, .bash_profile, etc)文件的， 所以环境变量中也就找不到对应的命令了， 特别是如果你是用rbenv或者rvm管理ROR的情况。 
    
详细解释看[这里](http://www.capistranorb.com/documentation/faq/why-does-something-work-in-my-ssh-session-but-not-in-capistrano/)
    
**对应的解决办法如下**：
    
1. 在gemfile中添加`gem 'capistrano-rbenv', '~> 2.0'`  
2. 取消capfile中`require 'capistrano/rbenv'` 的注释
3. 在deploy中为:rbenv_ruby 指定对应的版本：`set :rbenv_ruby, '2.0.0-p247'`

或者通过设置default_env 解决（目前未尝试）

`set :default_env, { path: "~/.rbenv/shims:~/.rbenv/bin:$PATH" }`

如果在输出中发现如下的信息， 不用在意， 这只是表示该文件夹已经存在， 对执行结果不会造成影响。

```
Finished in 1.456 seconds with exit status 1 (failed).
```

默认的deploy任务除了会执行check中的操作外， 还会执行以下几步操作：

* 在shared文件夹下建立public 和 assets文件夹
* 克隆git repo, 并执行update
* 为shared中的assets和current建立系统链接
* 执行bundle操作
* 执行`assets precompile` 和 `db:migrate` 操作， 如果require了assets和migrations模块的话
* 写入发布日志

在最后一部分中， 我们将编写针对nginx和unicorn的相关配置，以及了解一下capistrano3的执行flow.

## 参考：

http://www.capistranorb.com/documentation/faq/why-does-something-work-in-my-ssh-session-but-not-in-capistrano/
https://github.com/capistrano/sshkit
http://stackoverflow.com/questions/19716131/usr-bin-env-ruby-no-such-file-or-directory-using-capistrano-3-capistrano-rben

