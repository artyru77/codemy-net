+ [Mina for Deployment Part 1](http://www.youtube.com/watch?v=W2Lt1Hjz2vw)
+ [Mina for Deployment Part 2](http://www.youtube.com/watch?v=8RX6vftLokg)

In this episode I am going to be showing you how to setup mina in your rails app and configure your server to work with mina.

+ [Install Mina](#install-mina)
+ [Initialize Mina in your Project](#initialize-mina-in-your-project)
+ [Setting up our server to work with github](#setting-up-our-server-to-work-with-github)
+ [Configure database.yml](#conigure-databaseyml)
+ [Revisiting Upstart Config](#revisiting-upstart-config)
+ [Making shopper.conf a user job](#making-shopperconf-a-user-job)
+ [Enabling Upstart User Jobs](#enabling-upstart-user-jobs)
+ [Configuring our shopper.conf](#configuring-our-shopperconf)
+ [Configuring unicorn.rb](#configuring-unicornrb)
+ [Configuring nginx](#configuring-nginx)

## Install Mina

```bash
gem install mina
```

## Initialize Mina in your Project

```bash
mina init
```

Once you type in `mina init` it should generate a `deploy.rb` in your config folder

Lets configure this file

```ruby
require 'mina/bundler'
require 'mina/rails'
require 'mina/git'
require 'mina/rbenv'  # for rbenv support. (http://rbenv.org)
# require 'mina/rvm'    # for rvm support. (http://rvm.io)

# Basic settings:
#   domain       - The hostname to SSH to.
#   deploy_to    - Path to deploy into.
#   repository   - Git repo to clone from. (needed by mina/git)
#   branch       - Branch name to deploy. (needed by mina/git)

set :domain, 'cheqin.me'
set :deploy_to, '/opt/www/cheqin.me'
set :repository, 'git@github.com:codemy/shopper.git'
set :branch, 'develop'
set :user, 'deployer'

# Manually create these paths in shared/ (eg: shared/config/database.yml) in your server.
# They will be linked in the 'deploy:link_shared_paths' step.
set :shared_paths, ['config/database.yml', 'log']

# Optional settings:
#   set :user, 'foobar'    # Username in the server to SSH to.
#   set :port, '30000'     # SSH port number.

# This task is the environment that is loaded for most commands, such as
# `mina deploy` or `mina rake`.
task :environment do
  # If you're using rbenv, use this to load the rbenv environment.
  # Be sure to commit your .rbenv-version to your repository.
  invoke :'rbenv:load'

  # For those using RVM, use this to load an RVM version@gemset.
  # invoke :'rvm:use[ruby-1.9.3-p125@default]'
end

# Put any custom mkdir's in here for when `mina setup` is ran.
# For Rails apps, we'll make some of the shared paths that are shared between
# all releases.
task :setup => :environment do
  queue! %[mkdir -p "#{deploy_to}/shared/log"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/log"]

  queue! %[mkdir -p "#{deploy_to}/shared/config"]
  queue! %[chmod g+rx,u+rwx "#{deploy_to}/shared/config"]

  queue! %[touch "#{deploy_to}/shared/config/database.yml"]
  queue  %[echo "-----> Be sure to edit 'shared/config/database.yml'."]
end

desc "Deploys the current version to the server."
task :deploy => :environment do
  deploy do
    # Put things that will set up an empty directory into a fully set-up
    # instance of your project.
    invoke :'git:clone'
    invoke :'deploy:link_shared_paths'
    invoke :'bundle:install'
    invoke :'rails:db_migrate'
    invoke :'rails:assets_precompile'

    to :launch do
      queue "restart shopper"
    end
  end
end

# For help in making your deploy script, see the Mina documentation:
#
#  - http://nadarei.co/mina
#  - http://nadarei.co/mina/tasks
#  - http://nadarei.co/mina/settings
#  - http://nadarei.co/mina/helpers
```

Once we're done configuring our `deploy.rb` we run 

```bash
mina setup
```

## Setting up our server to work with github

Now that we are using mina, our server needs to be able to pull our source code from github directly. We're no longer going to be pushing code from our computer to the server. In fact whats going to happen is now mina is going to pull our code directly from github. So we need our server to be able to access github.

```bash
ssh-keygen -t rsa -C deployer@cheqin.me
```
This is going to save our public key in `~/.ssh/id_rsa.pub` so lets copy and paste the content of this file into our github account

```bash
cat ~/.ssh/id_rsa.pub
```

Once we've installed our public key on github we need to authenticate our server with github

```bash
ssh git@github.com
```

## Configure database.yml

Now that we've setup our server to be able to communicate with github we need to configure database.yml to connect to our database

```ruby
production:
  adapter: postgresql 
  encoding: unicode
  reconnect: false
  database: shopper_production
  pool: 5
  username: deployer
  password: 12345678
  host: localhost
```

Now our application is all setup lets run a deploy

```bash
mina deploy
```

Once we succeed with this we can now continue with automation. Getting our code up into the server is one part, the other part is to automate. We need our server to be restarted once the new code base is up and running.

## Revisiting Upstart Config

In the previous 11 episodes we setup our server in a way that the deployer user required sudo privileges to start stop and restart the application. However the deployer user should be able to do all that without the sudo priveleges. The whole reason for having the deployer user is so that he/she can manage his / her own application without bothering with the sudo privileges.

## Making shopper.conf a user job

Lets start with our shopper.conf. Originally we put shopper.conf into /etc/init we now need to move this to `~/.init/`

```bash
mkdir ~/.init
cp /etc/init/shopper.conf ~/.init
rm /etc/init/shopper.conf
```

We should now have shopper.conf in ~/.init

## Enabling Upstart User Jobs

By default your configuration may have already enabled user jobs however if you haven't when you try to run an upstart job using `start shopper` you may get an error.If you do you need to modify `/etc/dbus-1/system.d/Upstart.conf` ensure that you have this block in your `Upstart.conf`

```xml
<policy context="default">
  <allow send_destination="com.ubuntu.Upstart"
      send_interface="org.freedesktop.DBus.Introspectable" />
  <allow send_destination="com.ubuntu.Upstart"
      send_interface="org.freedesktop.DBus.Properties" />
  <allow send_destination="com.ubuntu.Upstart"
      send_interface="com.ubuntu.Upstart0_6" />
  <allow send_destination="com.ubuntu.Upstart"
      send_interface="com.ubuntu.Upstart0_6.Job" />
  <allow send_destination="com.ubuntu.Upstart"
      send_interface="com.ubuntu.Upstart0_6.Instance" />
</policy>
```

This will allow user to start upstart jobs. Once you've modified your upstart job you need to restart dbus

```bash
sudo service dbus restart
```

## Configuring our shopper.conf

Using **mina** means certain things are going to change. Mina setup is just like capistrano, it sets up shared folder and stores many revision of your application and it has one `current` version that will be running. So this changes a few things in our `shopper.conf`

The way mina handles gems and bundles are also different, now every app's bundler is stored in `./vendor/bundle` and all the binaries are in `./bin` directory. Which means our path needs to change

Another thing is when we move `shopper.conf` into `~/.init` upstart will no longer log its output, so you won't be able to see any errors, you'll need to modify your `shopper.conf`

So there is a few things we need to do to get shopper.conf to work right Lets configure `shopper.conf`

```bash
#~/.init/shopper.conf
description "Shopper App"
author "Zack Siri <zack@artellectual.com>"

start on virtual-filesystems
stop on runlevel [06]

env PATH=/opt/www/cheqin.me/current/bin:/usr/local/rbenv/shims:/usr/local/rbenv/bin:/usr/local/bin:/usr/bin:/bin

env RAILS_ENV=production
env RACK_ENV=production

setuid deployer
setgid admin

chdir /opt/www/cheqin.me

pre-start script
  exec >/home/deployer/shopper.log 2>&1
  exec /opt/www/cheqin.me/current/bin/unicorn -D -c /opt/www/cheqin.me/current/config/unicorn.rb --env production
end script

post-stop script
  exec kill `cat /tmp/unicorn.shopper.pid`
end script
```

## Configuring unicorn.rb

Since our path to our application has changed we need to change our unicorn.rb configuration as well

```ruby
# Sample verbose configuration file for Unicorn (not Rack)
#
# This configuration file documents many features of Unicorn
# that may not be needed for some applications. See
# http://unicorn.bogomips.org/examples/unicorn.conf.minimal.rb
# for a much simpler configuration file.
#
# See http://unicorn.bogomips.org/Unicorn/Configurator.html for complete
# documentation.

# Use at least one worker per core if you're on a dedicated server,
# more will usually help for _short_ waits on databases/caches.
worker_processes 2

# Since Unicorn is never exposed to outside clients, it does not need to
# run on the standard HTTP port (80), there is no reason to start Unicorn
# as root unless it's from system init scripts.
# If running the master process as root and the workers as an unprivileged
# user, do this to switch euid/egid in the workers (also chowns logs):
# user "unprivileged_user", "unprivileged_group"

# Help ensure your application will always spawn in the symlinked
# "current" directory that Capistrano sets up.
working_directory "/opt/www/cheqin.me/current" # available in 0.94.0+

# listen on both a Unix domain socket and a TCP port,
# we use a shorter backlog for quicker failover when busy
listen "/tmp/unicorn.shopper.sock", :backlog => 64
listen 8080, :tcp_nopush => true

# nuke workers after 30 seconds instead of 60 seconds (the default)
timeout 30

# feel free to point this anywhere accessible on the filesystem
pid "/tmp/unicorn.shopper.pid"

# By default, the Unicorn logger will write to stderr.
# Additionally, ome applications/frameworks log to stderr or stdout,
# so prevent them from going to /dev/null when daemonized here:
stderr_path "/opt/www/cheqin.me/current/log/unicorn.stderr.log"
stdout_path "/opt/www/cheqin.me/current/log/unicorn.stdout.log"

# combine Ruby 2.0.0dev or REE with "preload_app true" for memory savings
# http://rubyenterpriseedition.com/faq.html#adapt_apps_for_cow
preload_app true
GC.respond_to?(:copy_on_write_friendly=) and
  GC.copy_on_write_friendly = true

# Enable this flag to have unicorn test client connections by writing the
# beginning of the HTTP headers before calling the application.  This
# prevents calling the application for connections that have disconnected
# while queued.  This is only guaranteed to detect clients on the same
# host unicorn runs on, and unlikely to detect disconnects even on a
# fast LAN.
check_client_connection false

before_fork do |server, worker|
  # the following is highly recomended for Rails + "preload_app true"
  # as there's no need for the master process to hold a connection
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.connection.disconnect!

  # The following is only recommended for memory/DB-constrained
  # installations.  It is not needed if your system can house
  # twice as many worker_processes as you have configured.
  #
  # # This allows a new master process to incrementally
  # # phase out the old master process with SIGTTOU to avoid a
  # # thundering herd (especially in the "preload_app false" case)
  # # when doing a transparent upgrade.  The last worker spawned
  # # will then kill off the old master process with a SIGQUIT.
  # old_pid = "#{server.config[:pid]}.oldbin"
  # if old_pid != server.pid
  #   begin
  #     sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
  #     Process.kill(sig, File.read(old_pid).to_i)
  #   rescue Errno::ENOENT, Errno::ESRCH
  #   end
  # end
  #
  # Throttle the master from forking too quickly by sleeping.  Due
  # to the implementation of standard Unix signal handlers, this
  # helps (but does not completely) prevent identical, repeated signals
  # from being lost when the receiving process is busy.
  # sleep 1
end

after_fork do |server, worker|
  # per-process listener ports for debugging/admin/migrations
  # addr = "127.0.0.1:#{9293 + worker.nr}"
  # server.listen(addr, :tries => -1, :delay => 5, :tcp_nopush => true)

  # the following is *required* for Rails + "preload_app true",
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.establish_connection

  # Sidekiq.configure_client do |config|
  #   config.redis = ConnectionPool.new(size: 1, timeout: 1, &$sidekiq_redis)
  # end 
  # if preload_app is true, then you may also want to check and
  # restart any other shared sockets/descriptors such as Memcached,
  # and Redis.  TokyoCabinet file handles are safe to reuse
  # between any number of forked children (assuming your kernel
  # correctly implements pread()/pwrite() system calls)
end
```

Once we finish configuring our `unicorn.rb` we should be able to start / stop / restart our application without sudo permission. Lets try this out

```bash
start shopper
restart shopper
stop shopper
```

Once this is setup every time you do a deployment to mina your application will automatically be restarted.

## Configuring Nginx

We now need to make nginx use our new working path for our application. This is what our new nginx configuration looks like.

```nginx
upstream unicorn { 
  server unix:/tmp/unicorn.shopper.sock fail_timeout=0;
}

server { 
  server_name cheqin.me www.cheqin.me;
  listen 80 default deferred;
  
  root  /opt/www/cheqin.me/current/public;

  try_files $uri/system/maintenance.html $uri/index.html $uri @unicorn;

  location @unicorn { 
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unicorn;
  }

  location ^~ /assets/ { 
    gzip_static on;
    expires max;
    add_header Cache-Control public;
    add_header Access-Control-Allow-Origin *;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 4G;
  keepalive_timeout 10;
} 
```

## Conclusion

This concludes our episode on using mina for deployment. You may be wondering why I switched to mina over git, well originally I was going to write a git post-receive hook to automate everthing however mina does a lot of things right right out of the box and saves us a lot of time. I didn't like capistrano because the output was ugly and difficult to read and it was slow. Mina on the other hand has a beautiful output thats easy to read and is extremely fast. So I've decided to ditched my original git deploy for mina.