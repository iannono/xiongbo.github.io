**本节的介绍将以ROR应用为例**

## 安装capistrano3


假设已经建好的rails应用为test_app（这个名字和我们上一章在服务器上建立的应用文件夹名称一致`/var/www/test_app`），将以下代码添加到Gemfile中，并执行`bundle install`

```
gem capistrano
gem 'capistrano-rails', '~> 1.1.0', group: :development
```

为了保证安全，最好将一些含有敏感数据的文件排除，比如database.yml中有可能记录了我们本机的数据库密码，我们通过如下方式，复制一个example文件，并将原文件从git库中排除。

```
cp config/database.yml{,.example}
echo config/database.yml >> .gitignore
```

然后，初始化capistrano

```
$ cd my_repo
$ cap install
```

生成的目录结构如下：

```
├── Capfile
├── config
│   ├── deploy
│   │   ├── production.rb
│   │   └── staging.rb
│   └── deploy.rb
└── lib
    └── capistrano
            └── tasks
```

## capfile


capfile 用于载入各种依赖， 以及用户所编写的各项任务， 对于rails应用， 我们需要取消下面几行代码的注释， 便于执行capistrano-rails gem为我们提供的任务。

```
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
```

这样我们就可以在任务中调用`deploy:migration`和`deploy:compile_assets`来执行对应的任务了。


## config file


capistrano3 默认会生成两个阶段配置， 用于定义你的部署环境， 分别是staging和production。 

我们先抛开production, 来看staging.rb文件。

capistrano3 使用角色来定义服务器环境。 在编写task的时候可以指定对应的角色， 从而使用特定的用户在特定的服务器上执行操作。 如果没有指定角色的话， 则该task会在所有角色所对应的服务器上执行一次。

```
role :all, %w{hello@world.com example.com:1234}
```

另外，还可以通过扩展的`server`语法来对服务器做进一步的定义。 下面的定义和上面是等价的。

```
server 'world.com' roles: [:web], user: 'hello'
server 'example.com', roles: [:web], port: 1234
```

根据我们的情况， 因为只有一台服务器， 只需做如下的配置：

```
# 如果远程服务器是ip地址的话， 将remote.com换成对应的ip
role :all, %w{deploy@remote.com}
```


## deploy file


deploy.rb主要用于指定对各个环境所通用的配置， 比如repo的地址以及执行部署的用户等。 以下是一个例子：

```
set :application, 'test_app'
set :repo_url, 'git@github.com:me/test_app.git'
set :branch, 'master'
```

通过以上的配置， capistrano 就知道配置服务器和托管代码各在什么地方了。

额外的，针对rails项目， 如果需要每次发布都更新对应的静态文件的话，可以通过如下设置实现：

`set :normalize_asset_timestamps, %{public/images public/javascripts public/stylesheets}`
    
    
## 编写第一个cap任务


在编写之前，我们可以执行以下命令， 用于检查我们是否能够顺利连接到远程服务器，并读取对应的信息。

```
me@localhost $ ssh deploy@remote 'ls -lR /var/www/test_app'
test_app:
total 8
drwxrwsr-x 2 deploy deploy 4096 Jun 24 20:55 releases
drwxrwsr-x 2 deploy deploy 4096 Jun 24 20:55 shared

test_app/releases:
total 0

test_app/shared:
total 0
```

然后， 在tasks文件夹下新建一个文件access_check.cap， 并写入如下的代码：

```
desc "Check that we can access everything"
task :check_write_permissions do
  on roles(:all) do |host|
    if test("[ -w #{fetch(:deploy_to)} ]")
      info "#{fetch(:deploy_to)} is writable on #{host}"
    else
      error "#{fetch(:deploy_to)} is not writable on #{host}"
    end
  end
end
```

然后在终端中执行`cap -T`检查我们的任务是否在任务列表中：

```
me@localhost $ bundle exec cap -T
# ... lots of other tasks ...
cap check_write_permissions  # Check that we can access everything
# ... lots of other tasks ...
```

接下来，我们就可以通过如下的方式进行调用：

```
me@localhost $ bundle exec cap staging check_write_permissions
DEBUG [82c92144] Running /usr/bin/env [ -w /var/www/test-app ] on remote.com
DEBUG [82c92144] Command: [ -w /var/www/test_app ]
DEBUG [82c92144] Finished in 0.456 seconds command successful.
INFO /var/www/test_app is writable on remote.com
```

capistrano会根据你在deploy.rb中所设置的日志级别， 输出对应的日志信息。

最后， 通过执行`cap staging git:check`， 来检查是否能够正确的上传git脚本， 并执行， 可能的输出如下：

```
INFO [f6bde26c] Running /usr/bin/env mkdir -p /tmp/test_app/ on remote.com
DEBUG [f6bde26c] Command: /usr/bin/env mkdir -p /tmp/test_app/
 INFO [f6bde26c] Finished in 1.013 seconds with exit status 0 (successful).
DEBUG Uploading /tmp/test_app/git-ssh.sh 0.0%
 INFO Uploading /tmp/test_app/git-ssh.sh 100.0%
 INFO [74768e44] Running /usr/bin/env chmod +x /tmp/test_app/git-ssh.sh on remote.com
DEBUG [74768e44] Command: /usr/bin/env chmod +x /tmp/test_app/git-ssh.sh
 INFO [74768e44] Finished in 0.442 seconds with exit status 0 (successful).
DEBUG [fd6d945f] Running /usr/bin/env git ls-remote git@github.com:xiongbo/test_app.git on remote.com
DEBUG [fd6d945f] Command: ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/test_app/git-ssh.sh /usr/bin/env git ls-remote git@github.com:xiongbo/test_app.git )
DEBUG [fd6d945f] 	9476b1643a9dfd0a5741d6c753f491e8a5079fa8
DEBUG [fd6d945f] 		
DEBUG [fd6d945f] 	HEAD
DEBUG [fd6d945f] 	
DEBUG [fd6d945f] 	9476b1643a9dfd0a5741d6c753f491e8a5079fa8
DEBUG [fd6d945f] 		
DEBUG [fd6d945f] 	refs/heads/master
DEBUG [fd6d945f] 	
DEBUG [fd6d945f] Finished in 5.516 seconds with exit status 0 (successful).
```

如果输出信息提示`Permission denied (publickey)`, 可以通过执行`ssh-add`， 来确保SSH Agent正确的载入了你的密匙。

OK， 在第三部分中， 会针对我们的应用来编写对应的task！

