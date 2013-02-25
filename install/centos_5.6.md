**This installation guide was created for CentOS 5.6 in combination with gitlab 4-2-stable and tested on it.**


## Note ##


**Note about accounts:**
In most cases you are required to run commands as the 'root' user.
When it is required you should be either the 'git' or 'gitlab' user it will be indicated with a line like this

*logged in as gitlab*

The best way to become that user is by logging in as root and typing

    su - gitlab


----------

# Software list

These sortware list need to be installed:


1. Development Tools
2. Python 2.5+, easy_install and pip
3. Ruby 1.9.3, RubyGems and bundle
4. Git 1.7+
5. Apache2
6. Gitolite gl-v321
7. GitLab 4-2-stable

----------

# Service list

1. Mysql 5
2. Redis 2.6.10

----------


# GitLab directory structure

This is the directory structure you will end up with following the instructions in the Installation Guide.

    |-- home
    |   |-- git
    |   |   |-- bin
    |   |   |-- gitolite
    |   |   |-- .gitolite
    |   |   |-- .gitolite.rc
    |   |   |-- projects.list
    |   |   |-- repositories
    |   |   |-- .ssh
    |   |-- gitlab
    |   |   |-- gitlab
    |   |   |-- gitlab-satellites
    |   |   |-- .ssh

You can change them in your `config/gitlab.yml` file.

----------

# Config files


    |-- home
    |   |-- gitlab
    |   |   |-- gitlab
    |   |   |   |-- config
    |   |   |   |   |-- database.yml
    |   |   |   |   |-- gitlab.yml
    |   |   |   |   |-- resque.yml
    |   |   |   |   |-- unicorn.rb


----------

# Overview

The GitLab installation consists of setting up the following components:

1. Packages / Dependencies
2. System Users
3. Gitolite
4. GitLab


----------

# 1. Packages / Dependencies

Note that during the installation you use the *"Configure Network"* option (it's a button in the same screen wher you speciify the hostname) to enable the *"Connect automatically"* option for the network interface and hand (usually eth0). 
**If you forget this option the network will NOT start at boot.**


## Install the required tools for gitlab and gitolite

### Add EPEL repository

*logged in as root*

	rpm -Uvh http://dl.fedoraproject.org/pub/epel/5/x86_64/epel-release-5-4.noarch.rpm

*logged in as root*

    yum install -y gcc* apr-devel apr-util-devel openssh-server openssl-devel \
    			   curl-devel httpd-devel python-devel python-setuptools \
    			   libicu-devel ruby-devel libxml2 libxml2-devel libxslt \
    			   libxslt-devel libyaml-devel libyaml-devel libffi-devel \
    			   libjpeg-devel patch readline readline-devel zlib zlib-devel \
    			   make automake autoconf bzip2  git giflib-devel \
    			   freetype-devel bison mysql-devel
    
## Install Python2.5+

Install the python2.5+, easy_install and pip if there is not.

3.x is not supported at the moment.

Install the pygments if there is not:

*logged in as root*

	pip install pygments


## Install Ruby1.9.3

Install the ruby1.9.3 if there is not.

## Install Apache 2

Install the Apache 2 if there is not.

## Configure httpd

We want to be able to reach gitlab using the normal http ports (i.e. not the :3000 thing)
So we create a file called **/etc/httpd/conf.d/gitlab.conf** with this content (replace the git.example.org with your hostname!!). 

*logged in as root*

    <VirtualHost *:80>
      ServerName git.example.org
      ProxyRequests Off
        <Proxy *>
           Order deny,allow
           Allow from all
        </Proxy>
        ProxyPreserveHost On
        ProxyPass / http://localhost:3000/
        ProxyPassReverse / http://localhost:3000/
    </VirtualHost>

----------

# 2. System Users

## Create users for Git and Gitolite
*logged in as root*

    useradd --shell /bin/bash --comment 'git version control' -m git

    useradd --shell /bin/bash --comment 'gitlab user' -m gitlab

    usermod -a -G git gitlab 

Because the gitlab user will need a password later on, we configure it right now, so we are finished with all the user stuff.

*logged in as root*

    passwd gitlab # please choose a good password :)  

*logged in as root*

    # Generate the SSH key
    sudo -u gitlab -H ssh-keygen -q -N '' -t rsa -f /home/gitlab/.ssh/id_rsa


    
----------

# 3. Gitolite

## Clone GitLab's fork of the Gitolite source code:

*logged in as root*

    cd /home/git
    
    sudo -u git -H git clone -b gl-v320 https://github.com/gitlabhq/gitolite.git /home/git/gitolite

## Setup Gitolite with GitLab as its admin:

**Important Note:**
GitLab assumes *full and unshared* control over this Gitolite installation.

*logged in as root*

    # Add Gitolite scripts to $PATH
    sudo -u git -H mkdir /home/git/bin
    
    sudo -u git -H sh -c 'printf "%b\n%b\n" "PATH=\$PATH:/home/git/bin" "export PATH" >> /home/git/.profile'
    
    sudo -u git -H sh -c 'gitolite/install -ln /home/git/bin'

    # Copy the gitlab user's (public) SSH key ...
    cp /home/gitlab/.ssh/id_rsa.pub /home/git/gitlab.pub
    
    chmod 0444 /home/git/gitlab.pub

    # ... and use it as the admin key for the Gitolite setup
    sudo -u git -H sh -c "PATH=/home/git/bin:$PATH; gitolite setup -pk /home/git/gitlab.pub"

### Fix the directory permissions for the configuration directory:

*logged in as root*

    # Make sure the Gitolite config dir is owned by git
    chmod 750 /home/git/.gitolite/
    
    chown -R git:git /home/git/.gitolite/

### Fix the directory permissions for the repositories:

*logged in as root*

    # Make sure the repositories dir is owned by git and it stays that way
    chmod -R ug+rwXs,o-rwx /home/git/repositories/
    
    chown -R git:git /home/git/repositories/

    # Make sure the gitlab user can access the required directories
    chmod g+x /home/git

### Make the git account known and allowed to the gitlab user

*logged in as root*

    su - gitlab

*logged in as **gitlab***    
    
    ssh git@localhost  # type 'yes' and press <Enter>.

The expected behaviour is that you get a message similar to this and then immediately the connection is closed again:

    PTY allocation request failed on channel 0
    hello gitlab, this is git@gitlab running gitolite3 v3.2-gitlab-patched-0-g2d29cf7 on git 1.7.1

## Test if everything works so far

*logged in as **gitlab***

    # Clone the admin repo so SSH adds localhost to known_hosts ...
    # ... and to be sure your users have access to Gitolite
    git clone git@localhost:gitolite-admin.git /tmp/gitolite-admin

    # If it succeeded without errors you can remove the cloned repo
    rm -rf /tmp/gitolite-admin

**Important Note:**
If you can't clone the `gitolite-admin` repository: **DO NOT PROCEED WITH INSTALLATION**!
Check the [Trouble Shooting Guide](https://github.com/gitlabhq/gitlab-public-wiki/wiki/Trouble-Shooting-Guide)
and make sure you have followed all of the above steps carefully.

----------

# 4. GitLab

*logged in as gitlab*

    # We'll install GitLab into home directory of the user "gitlab"
    cd /home/gitlab
    mkdir /home/gitlab/gitlab-satellites

## Clone the Source

*logged in as **gitlab***

    # Clone GitLab repository
    git clone https://github.com/gitlabhq/gitlabhq.git gitlab

    # Go to gitlab dir 
    cd /home/gitlab/gitlab
   
    # Checkout to stable release
    git checkout 4-2-stable



## Configure it

### Copy the example GitLab config

*logged in as **gitlab***

    cp /home/gitlab/gitlab/config/gitlab.yml.example /home/gitlab/gitlab/config/gitlab.yml

Edit the gitlab config to make sure to change "localhost" to the fully-qualified domain name of your host serving GitLab where necessary. Also review the other settings to match your setup.

*logged in as **gitlab***

    vim /home/gitlab/gitlab/config/gitlab.yml

### Copy the example Unicorn config

*logged in as **gitlab***

	cp /home/gitlab/gitlab/config/unicorn.rb.example /home/gitlab/gitlab/config/unicorn.rb

Edit the unicorn config

*logged in as **gitlab***

	vim /home/gitlab/gitlab/config/unicorn.rb

Change the listen parameter so that it reads:

	listen "127.0.0.1:3000"  # listen to port 3000 on the loopback interface

Also review the other settings to match your setup.

### Configure GitLab DB settings

*logged in as **gitlab***

    # MySQL
    cp /home/gitlab/gitlab/config/database.yml.mysql /home/gitlab/gitlab/config/database.yml

Edit the database config and set the correct username/password

*logged in as **gitlab***

    vim /home/gitlab/gitlab/config/database.yml

The config should look something like this (where supersecret is replaced with your real password):

    production:
      adapter: mysql2
      encoding: utf8
      reconnect: false
      database: gitlabhq_production
      pool: 5
      username: root
      password: 
      host: localhost
      # socket: /tmp/mysql.sock

Update the config of production env to the mysql-server ip, port, database, username and password
    
## Configure Redis(resque)

*logged in as **gitlab***

    cp /home/gitlab/gitlab/config/resque.yml.example /home/gitlab/gitlab/config/resque.yml
    
Edit the resque config and set the correct redis ip and port

*logged in as **gitlab***

	vim /home/gitlab/gitlab/config/resque.yml

Update the config of production env to the redis-server ip and port
    
## Install Gems

*logged in as **gitlab***

    logout

*logged in as **root***

    cd /home/gitlab/gitlab

	gem update --system
	gem install bundle
    gem install charlock_holmes --version '0.6.9'

    su - gitlab

*logged in as **gitlab***

    cd /home/gitlab/gitlab

    # For mysql db
    bundle install --deployment --without development test postgres

## Configure Git

GitLab needs to be able to commit and push changes to Gitolite. In order to do
that Git requires a username and email. (We recommend using the same address
used for the `email.from` setting in `config/gitlab.yml`)

*logged in as gitlab*

    git config --global user.name "GitLab"
    git config --global user.email "gitlab@localhost"

## Setup GitLab Hooks

*logged in as **gitlab***

    logout

*logged in as **root***

    cd /home/gitlab/gitlab
    cp ./lib/hooks/post-receive /home/git/.gitolite/hooks/common/post-receive
    chown git:git /home/git/.gitolite/hooks/common/post-receive

## Initialise Database and Activate Advanced Features

*logged in as **root***

    su - gitlab

*logged in as **gitlab***

    cd /home/gitlab/gitlab
    
    bundle exec rake gitlab:setup RAILS_ENV=production

The previous command will ask you for the root password of the mysql database and create the defined database and user.

## Install Init Script

Download the init script (will be /etc/init.d/gitlab)

*logged in as **gitlab***

    logout

*logged in as root*

    curl https://raw.github.com/gitlabhq/gitlab-recipes/master/init.d/gitlab-centos > /etc/init.d/gitlab
    
    chmod +x /etc/init.d/gitlab
    
    chkconfig --add gitlab

Make GitLab start on boot:

*logged in as **root***

    chkconfig gitlab on

Start your GitLab instance:

*logged in as **root***

    service gitlab start
    # or
    /etc/init.d/gitlab start

## Check Application Status

Check if GitLab and its environment is configured correctly:

*logged in as **root***

    su - gitlab

*logged in as **gitlab***

    cd /home/gitlab/gitlab
    bundle exec rake gitlab:env:info RAILS_ENV=production

To make sure you didn't miss anything run a more thorough check with:

*logged in as **gitlab***

    cd /home/gitlab/gitlab
    bundle exec rake gitlab:check RAILS_ENV=production

If you are all green: congratulations, you successfully installed GitLab!
Although this is the case, there are still a few steps to go.

# Done!

Visit YOUR_SERVER for your first GitLab login.
The setup has created an admin account for you. You can use it to log in:

    admin@local.host
    5iveL!fe

**Enjoy!**


# Others

## Prompts git@server password

If there prompts git@server password, then please fix it using this command:

*logged in as **gitlab***

    cd /home/gitlab/gitlab
    bundle exec rake gitlab:gitolite:update_keys RAILS_ENV=production

## Please check status after creating a repo

*logged in as **gitlab***

    cd /home/gitlab/gitlab
    bundle exec rake gitlab:check RAILS_ENV=production

