# Setup and Deploy Ruby On Rails on Ubuntu 14.04 Trusty Tahr

A guide to setting up a Ruby on Rails production environment

## Overview

This will take about 60-120 minutes.

We will be setting up a Ruby on Rails production environment on Ubuntu 14.04 LTS Trusty Tahr.

Using Ubuntu LTS in production allows you to continue receiving security updates which is important for your production server(s).

We're going to be setting up a Droplet on Digital Ocean for our server. It costs $5/mo and is a great place to host your applications.

If you sign up with [my Digital Ocean referral link](https://www.digitalocean.com/?refcode=0a9ec2d1d5b0), you'll get 2 months ($10) free credit to try it out.


## Creating a Virtual Private Server (VPS)

Since we are going to be using [DigitalOcean](https://www.digitalocean.com/?refcode=0a9ec2d1d5b0), the first thing we are going to need to do is create a new droplet.

First give the new droplet a name. `rails-server-workshop` will do.

![Droplet name](http://i.imgur.com/iRhDHA5.png)

For this workshop, I suggest you choose the cheapest $5 droplet size.

![Droplet size](http://i.imgur.com/sEVF233.png)

The region we select isnt super important, Ill go with Singapore as it is likely to have the least latancy to New Zealand.

![Droplet location](http://i.imgur.com/CxHBMlk.png)

Go ahead and select Ubuntu 14.04 x64 as our image.

![Droplet image](http://i.imgur.com/XYpuk1T.png)

Only thing left to do now is to create the droplet and drink some coffee while we wait for it to finish.

![Droplet creation](http://i.imgur.com/SNP0ae7.png)

Once finished you should get an email from DigitalOcean with instructions for logging into your new droplet.

Login now so we can setup the user we will use for the rest of the workshop:


```
sudo adduser deploy
sudo adduser deploy sudo
su deploy
```

Now that we have a user that is part of the sudo group, lets add our key to the server and attempt a login that way.

On your local machine, copy your public key and paste the contents into `~/.ssh/authorized_keys` on the server.

Now trying logging in: `ssh deploy@IPADDRESS` We will be spending the rest of the tutorial logged in as this user.

## Installing Ruby

First thing we are going to want to do is install the dependancies we are going to need for ruby:

```
sudo apt-get update
sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev
```

Once the above is done, we will install Rbenv and ruby-build so we can manage the installed ruby versions on the server with ease:

```
cd
git clone git://github.com/sstephenson/rbenv.git .rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL

git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
exec $SHELL

git clone https://github.com/sstephenson/rbenv-gem-rehash.git ~/.rbenv/plugins/rbenv-gem-rehash

rbenv install 2.2.1
rbenv global 2.2.1
ruby -v
```

If everything worked correctly you should now see ruby 2.2.1 when you check for the current ruby versions `ruby -v`

When bundler installs gems it will also install documentation for the gem also (`rdoc`, `ri` etc), so lets disable that and install bundler:

```
echo "gem: --no-ri --no-rdoc" > ~/.gemrc
gem install bundler
```

## Installing Nginx and Passenger

Install Phusion's PGP key to verify packages
```
gpg --keyserver keyserver.ubuntu.com --recv-keys 561F9B9CAC40B2F7
gpg --armor --export 561F9B9CAC40B2F7 | sudo apt-key add -
```

Add HTTPS support to APT
```
sudo apt-get install apt-transport-https
```

Add the passenger repository
```
sudo sh -c "echo 'deb https://oss-binaries.phusionpassenger.com/apt/passenger trusty main' >> /etc/apt/sources.list.d/passenger.list"
sudo chown root: /etc/apt/sources.list.d/passenger.list
sudo chmod 600 /etc/apt/sources.list.d/passenger.list
sudo apt-get update
```

Install nginx and passenger
```
sudo apt-get install nginx-full passenger
```

Start nginx
```
sudo service nginx start
```

Now lets open the nginx config `sudo vim /etc/nginx/nginx.conf` and uncomment these lines:
```
passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
passenger_ruby /home/deploy/.rbenv/shims/ruby;
```

With the correct version of ruby now configured, lets go ahead and restart nginx `sudo service nginx restart`

## PostgreSQL Database setup

Install postgresql
```
sudo apt-get install postgresql postgresql-contrib libpq-dev
```

Now we want to create a user for our rails app

```
sudo su - postgres
createuser --interactive
```

Once you have done that, exit back out to our `deploy` user by typing `exit`

## Installing Node

We are going to need node so that when we deploy our rails app, sprokets can use node to compile our assets.

```
sudo apt-get update
sudo apt-get install nodejs
```

Just like ruby, node has a version manager. If you were wanting to have better control over the node version on the server, consider using `NVM` (Node version manager).

For this workshop we don't really need this so I am not including it.

## Capistrano Setup

We are almost ready to deploy our app, but first we are going to have to setup capistrano.

Add the gems we are doing to need to the `Gemfile`

```
gem 'capistrano', '~> 3.1.0'
gem 'capistrano-bundler', '~> 1.1.2'
gem 'capistrano-rails', '~> 1.1.1'
gem 'capistrano-rbenv', github: "capistrano/rbenv"
```

Now run `bundle` to install and once done add binstubs `bundle --binstubs`

Once we have the binstubs, we can run our capify tasks: `cap install STAGES=production`

We will need to add this configuration to our `Capfile`

```
require 'capistrano/bundler'
require 'capistrano/rails'
require 'capistrano/rbenv'

set :rbenv_type, :user # or :system, depends on your rbenv setup
set :rbenv_ruby, '2.2.1'
```

Once we have done this, we will need to update

`config/deploy.rb`:

```
set :ssh_options, {
  forward_agent: true
}

set :application, 'workshop'
set :repo_url, 'git@github.com:charlespeach/workshop-app.git'

set :deploy_to, '/home/deploy/workshop'

set :linked_files, %w{config/database.yml}
set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system}

namespace :deploy do

  desc 'Restart application'
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      execute :touch, release_path.join('tmp/restart.txt')
    end
  end

  after :publishing, 'deploy:restart'
  after :finishing, 'deploy:cleanup'
end
```

`config/deploy/production.rb`:

```
set :stage, :production

# Replace 127.0.0.1 with your server's IP address!
server '127.0.0.1', user: 'deploy', roles: %w{web app}
```


There is going to be a few extra steps to do now. To check out deploy we will need to run:

`bundle exec cap production deploy:check`

This will start the deploy process and check all the nessasary dependancies are met for a successful deploy. Along the way it will create directories that dont exist, but will not create files for you if you have set them for symlink etc

After the first run, we will need to add a `database.yml` file: `touch /home/deploy/workshop/shared/config/database.yml` and add your database details.

We will also need to make sure that our SSH key is being forwarded when we communicate over SSH with the server. This is so that the server can authenticate with github and clone the repository.

To add your key for forwarding on your laptop: `ssh-add ~/.ssh/id_rsa`

To ensure the key is added for forwarding when you are starting a session with the server:

`~/.ssh/config`:

```
Host workshop
  HostName 127.0.0.1 # replace with your servers IP address
  User deploy
  ForwardAgent yes
```

Now if you login to your server: `ssh workshop` you can test your key with github by typing `ssh git@github.com`

# Final steps


