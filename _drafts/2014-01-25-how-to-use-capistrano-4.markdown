这一节将介绍针对nginx和unicorn的配置策略， 以及基本的配置脚本。

在服务器上安装好nginx 和 unicorn后， 需要对三个配置文件进行配置， 分别是nginx.conf, unicorn.rb, unicorn_init.sh.

首先, nginx会默认载入/etc/nginx/conf.d/下所有以.conf结尾的文件， 同时还会载入/etc/nginx/sites-enabled/目录下的所有文件， 所以， 一般会删除sites-enabled下的default文件， 然后加入我们自己编写的配置文件。

**需要注意的是root目录下添加的current**， 具体原因可以参考[第三节](http://xiongbo.me/ghost/6)

```
# /test_app/config/nginx_unicorn.conf

server {
  listen 80 default;
  root /var/www/test_app/current/public;
  
  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  try_files $uri/index.html $uri @unicorn;
  location @unicorn {
    proxy_pass http://127.0.0.1:8888;
  }

  error_page 500 502 503 504 /500.html;
  keepalive_timeout 10;
}
```

然后是unicorn的配置文件：

```
# /test_app/config/unicorn.rb

root = "/var/www/test_app/current"
appname = "test_app"
user 'deploy', 'deploy'

working_directory root
pid "#{root}/tmp/pids/unicorn.pid"
stderr_path "#{root}/log/unicorn.log"
stdout_path "#{root}/log/unicorn.log"

listen 8888 # 对应nginx的端口
worker_processes 2
timeout 30
```

最后是unicorn_init.sh， 该文件用于通过脚本的方式执行unicorn的各项操作，unicorn的样例代码中提供了[范本](https://raw.github.com/defunkt/unicorn/master/examples/init.sh)。 我们需要修改的主要是其中的 `APP_ROOT` 和 `CMD` 变量。

```
# /etc/init.d/unicorn_init.sh 

......
# Feel free to change any of the following variables for your app:
TIMEOUT=${TIMEOUT-60}
APP_ROOT=/var/www/test_app/current
PID=$APP_ROOT/tmp/pids/unicorn.pid
CMD="cd $APP_ROOT; bundle exec unicorn -D -c $APP_ROOT/config/unicorn.rb -E production"
AS_USER=deploy
......
```

OK， 完成了以上的配置， 我们可以开始扩展我们的deploy任务了。

整个deploy任务的执行流程如下， 我们可以根据需要在其中插入我们自己定义的任务。


```
deploy
  deploy:starting
    [before]
      deploy:ensure_stage
      deploy:set_shared_assets
    deploy:check
  deploy:started
  deploy:updating
    git:create_release
    deploy:symlink:shared
  deploy:updated
    [before]
      deploy:bundle
    [after]
      deploy:migrate
      deploy:compile_assets
      deploy:cleanup_assets
      deploy:normalise_assets
  deploy:publishing
    deploy:symlink:release
    deploy:restart
  deploy:published
  deploy:finishing
    deploy:cleanup
  deploy:finished
    deploy:log_revision
```

首先在deploy中添加三个用于操作unicorn server的任务。 分别为unicorn_start, unicorn_stop, unicorn_restart。

```
namespace :unicorn do
  %w[start stop restart].each do |command|
    desc "#{command} unicorn server"
    task "unicorn_#{command}", roles: :app, except: {no_release: true} do
      exec "/etc/init.d/unicorn_#{application} #{command}"
    end
  end
end
```

然后， 是针对数据库文件的配置。 我们需要将database.yml.example改为database.yml， 因此可以将我们的任务安排在`published`后面。

```

```

接下来， 是建立nginx和unicorn的系统链接， 这里需要注意的是你是否拥有对应目录的权限， 否则任务会失败。

```
```

最后就是执行`cap staging deploy`, 看看胜利的果实吧！

如果执行完毕后， 没有任何的错误， 我们就可以启动nginx以及unicorn服务器， 然后通过浏览器查看应用是否正确执行了。

## 参考
https://github.com/capistrano/capistrano/blob/dfe25c9436461c395cb7425f8c452ab87ca6b5e0/lib/capistrano/tasks/deploy.rake
http://railscasts.com/episodes/335-deploying-to-a-vps
http://www.capistranorb.com/documentation/getting-started/flow/

