---
layout: post
title: Deploy Rails with PostgreSQL Using Passenger and Nginx on CentOS
category: Tech
---

最近读完了 Michael Hartl 的《Ruby on Rails Tutorial》，于是想试着在自己的 Digital Ocean (CentOS 7) 上部署一下最终完成的 sample app。最简单的方法当然是：

``` sh
$ rails server -e production -b 0.0.0.0 -p 80
```

但 Rails 内建的 WEBrick + SQLite 性能较差，而且这样也没法配置虚拟主机。相比 Apache，Nginx 的异步模型能更好地处理高并发的场景，Passenger 是性能最好的 Rails servers 之一，而 PostgreSQL 比 MySQL / MariaDB 功能更强大，这三者差不多是 Web 开发的最佳实践，因此我决定折腾一下 Nginx + Passenger + PostgreSQL。

## 安装 Ruby

直接使用 yum 安装当然是可行的，但 [Ruby Version Manager (RVM)](https://rvm.io) 能够更方便地安装并管理 Ruby 版本，因此下面以 RVM 为例。安装 RVM：

``` sh
$ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
$ curl -sSL https://get.rvm.io | bash -s stable
```

接着就能轻松地安装任意版本的 Ruby 了（`rvm list known` 可以列出已知的版本）：

``` sh
$ rvm install X.X.X
$ rvm --default use X.X.X
```

<!--more-->

为了管理项目依赖，还需要安装一下 [Bundler](http://bundler.io)：

``` sh
$ gem install bundler --no-rdoc --no-ri
```

如果服务器在国内，由于众所周知的原因，也许会频繁地遭遇网络连接失败，那你可以考虑使用淘宝的 [Ruby 源代码镜像和 RubyGems 镜像](https://ruby.taobao.org)：

``` sh
$ sed -i .bak -E 's!https?://cache.ruby-lang.org/pub/ruby!https://ruby.taobao.org/mirrors/ruby!' $rvm_path/config/db
$ gem sources -a https://ruby.taobao.org/ -r https://rubygems.org/
```

## 安装 JavaScript 引擎

如果 Gemfile 直接或间接依赖了 [ExecJS](https://github.com/rails/execjs)，譬如 `coffee-rails` / `uglifier` / `turbolinks` / `bootstrap-sass` 等等，则还需要安装 JS 引擎，以在 Ruby 中执行 JS 代码。

- 一种方法是在 Gemfile 中加上 `gem 'therubyracer'`，这是一个嵌入式的 V8 引擎。
- 另一种方法是在服务器上安装 Node.js，这使用的也是 V8 引擎。安装 [Extra Packages for Enterprise Linux (EPEL)](https://fedoraproject.org/wiki/EPEL/zh-cn) 后可以直接通过 yum 安装 node，但版本会比较旧（0.10.X）；与 RVM 类似，Node.js 也有更好用的 [NVM](https://github.com/creationix/nvm) 来安装和管理版本，这里不再赘述。

## 安装 Nginx + Passenger

yum 原来的源（repo）中没有 Nginx 和 Passenger，可以导入 [Passenger 自己的源](https://www.phusionpassenger.com/library/install/nginx/install/oss/el7/)，安装已包含 Passenger 模块的 Nginx 版本（这样就不必去运行 `passenger-install-nginx-module` 了）和 Passenger 本体：

``` sh
$ curl --fail -sSLo /etc/yum.repos.d/passenger.repo https://oss-binaries.phusionpassenger.com/yum/definitions/el-passenger.repo
$ yum -y install nginx passenger
```

接着在 `/etc/nginx/conf.d/passenger.conf` 中解除下面三行的注释：

``` nginx
passenger_root /usr/share/ruby/vendor_ruby/phusion_passenger/locations.ini;
passenger_ruby /usr/bin/ruby;
passenger_instance_registry_dir /var/run/passenger-instreg;
```

## 安装 PostgreSQL

先从 yum 安装 PostgreSQL：

``` sh
$ yum -y install postgresql-server postgresql-devel
$ postgresql-setup initdb
```

接着用 [systemd](https://fedoraproject.org/wiki/Systemd/zh-cn)（在 CentOS 7 中它取代了 Upstart 和 SysVinit）启动数据库并设为开机启动项：

``` sh
$ systemctl start postgresql
$ systemctl enable postgresql
```

PostgreSQL 默认的超级用户是 `postgres`，使用同名的操作系统账户可以直接登录，这里为数据库新建一个用户，该用户**应与后文的 Linux 用户同名**以便免密码连接数据库：

``` sh
$ su - postgres
$ createuser sample-app
$ createdb -O sample-app sample-app
```

`Gemfile` 中也需要加入 PostgreSQL 的 Ruby 接口：

``` ruby
gem 'pg', group: :production
```

如果创建 app 的时候没有指定数据库（譬如 `rails new sample-app -d postgresql`），那么 Rails 默认使用的是 SQLite 3，我们还需要提前修改一下 Rails 的配置文件 `config/database.yml`：

``` yaml
production:
  adapter: postgresql
  encoding: unicode
  pool: 5
  database: sample-app
  username: sample-app
```

## 部署 Rails

为了安全考虑，最好为每个 app 创建一个同名的 Linux 用户，Passenger 也提供了相应的[用户账户沙盒](https://www.phusionpassenger.com/library/deploy/nginx/user_sandboxing.html)功能，默认会以 `config.ru` 的所有者运行 app。下面的部署工作都在该账户下完成（以下假设 app 在 `/srv/sample-app`）：

``` sh
$ adduser sample-app
$ chown -R sample-app: /srv/sample-app
$ su - sample-app
```

首先是通过 Bundler 安装 app 的依赖项，因为是部署模式所以 `Gemfile.lock` 应当提前已经生成好：

``` sh
$ cd /srv/sample-app
$ bundle install --deployment --without development test
```

接下来检查一下 `config/database.yml` 和 `config/secrets.yml`，前者即上文提到的数据库配置，后者存储的是用来加密 cookies / sessions 的密钥，生产环境默认从环境变量读取密钥，以防开发者手滑将其上传到 public repo：

``` yaml
# Do not keep production secrets in the repository, instead read values from the environment.
production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
```

密钥可通过 `bundle exec rake secret` 生成，然后在 `~/.bashrc` 或 `/srv/rails/.env.production`（使用了 [gem 'dotenv-rails'](https://github.com/bkeepers/dotenv) 的话）设置环境变量：

``` sh
export SECRET_KEY_BASE=...
```

我个人比较喜欢的做法是在后面提到的 Nginx 配置文件中加上：

``` nginx
passenger_env_var SECRET_KEY_BASE ...;
```

但千万不要直接在 shell 里输 export 命令，这只会在当前 shell 及其子进程中生效。也不要随意更改密钥，这会使原先加密的 cookies 失效。

最后让 [Asset Pipeline](http://guides.ruby-china.org/asset_pipeline.html) 对静态资源做预编译，让 Active Record 做数据库迁移：

``` sh
$ bundle exec rake assets:precompile db:migrate RAILS_ENV=production
```

## 启动 Nginx

回到管理员账户，创建 Nginx 配置文件 `/etc/nginx/conf.d/sample-app.conf`，为自己的 app 添加一个虚拟主机，其中 `passenger_ruby` 的路径可以通过 `passenger-config about ruby-command` 命令获知，`passenger_app_root` 默认是虚拟主机 `root` 的上级目录：

``` nginx
server {
    listen 80;
    server_name sample-app.example.com;
    root /srv/sample-app/public;
    passenger_enabled on;
    passenger_ruby /usr/local/rvm/gems/ruby-X.X.X/wrappers/ruby;
}
```

`/etc/nginx/nginx.conf` 里默认有一行 `include /etc/nginx/conf.d/*.conf;` 所以我们不需要去更改其他配置。于是现在可以正式启动 Nginx 了：

``` sh
$ systemctl start nginx
$ systemctl enable nginx
```

理论上你的 app 已经部署成功了～ 🎉

之后如果需要让 Passenger 重启 app，或是要重载 Nginx：

``` sh
$ touch /srv/sample-app/tmp/restart.txt
$ systemctl reload nginx
```

## Tips

- 如果在本地测试**生产环境**时，发现静态文件未能加载：
  - 其一可能是因为没有做预编译，Rails 认为生产环境的静态资源应该已经事先编译好了，所以 Asset Pipeline 的实时编译是关闭的；
  - 其二可能是因为 Rails 默认没有启用静态文件服务，你可以通过设置 `RAILS_SERVE_STATIC_FILES` 环境变量来开启它，但这在生产环境中低效且不安全，应该让 Nginx / Apache 来处理它们。
  - 上述默认配置均在 `config/environments/production.rb` 文件中：

``` ruby
# Do not fallback to assets pipeline if a precompiled asset is missed.
config.assets.compile = false

# Disable serving static files from the `/public` folder by default since Apache or NGINX already handles this.
config.serve_static_files = ENV['RAILS_SERVE_STATIC_FILES'].present?
```

- 访问网站时显示「Incomplete response received from application」一般是没有设置 `secret_key_base`，具体参见前文「部署 Rails」。
- 如果在 `gem install pg` 时报告找不到头文件 `libpq-fe.h`，那是因为没有安装 `postgresql-devel`，或者在其他操作系统中这个包叫 `libpq-dev`。

## 版本信息

CentOS 7 x86_64 (1511) / Nginx 1.8.1 / Passenger 5.0.24 / PostgreSQL 9.2.14 / Rails 4.2.5 / Ruby 2.3.0

> 参考文档：
> 
> [Deploying a Ruby app with Passenger to production - Passenger Library](https://www.phusionpassenger.com/library/walkthroughs/deploy/ruby/)  
> [Configuration reference for Nginx - Passenger Library](https://www.phusionpassenger.com/library/config/nginx/reference/)  
> [Configuring NGINX Plus as a Web Server - NGINX Admin Guide](https://www.nginx.com/resources/admin-guide/nginx-web-server/)  
> [The Asset Pipeline - Ruby on Rails Guides](http://guides.rubyonrails.org/asset_pipeline.html)
