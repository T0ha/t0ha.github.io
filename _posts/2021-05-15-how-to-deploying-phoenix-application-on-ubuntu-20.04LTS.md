---
layout: post
locale: en_US
title: "How To: Deploying Phoenix Application on Ubuntu 20.04 LTS"
description: Master the Art of Deploying Phoenix Applications on Ubuntu 20.04LTS! Our expert guide will equip you with the knowledge and skills needed to smoothly navigate the deployment process. Get ready to embark on a journey towards application success!
image: /assets/images/posts/phoenix_ubuntu.png
tags:
  - devops
  - elixir
  - phoenix
  - phoenix-framework
  - web-development
  - ubuntu
status: advertise
categories:
  - en
  - devops
  - elixir
published: true
---
![Top image](/assets/images/posts/phoenix_ubuntu.png)
Phoenix is cool, right? You've just created your brilliant web
application and it works fine on your local machine. It's awesome, but
nobody knows about it. Stop, your friend knows, because you've told him
in the bar a week ago and he is tied waiting to try it! Hey, it's time
to deploy the app and show it to the entire world...

## Prerequisite

For the purpose of this article we assume that you have the following
things ready:

-   a Simple [Phoenix](http://phoenixframework.org) application
    working locally
-   [git](https://git-scm.com) repository for the app, hosted on
    [GitHub](https://github.com/) (it can be hosted anywhere, but I'll
    describe configuration process for GitHub)
-   [Ubuntu](https://www.ubuntu.com), preferably 22.04-LTS, installed
    and you have a sudo'er user ssh access to it

For the purpose of this article I use several environmental variables:

-   `$PROJECT` --- your project name, eg. my_project
-   `$REPO` --- your project's git repository ssh url
-   `$EMAI` --- your email address
-   `$DOMAIN` --- domain for your app

## Let's Prepare the Ground

In order to work properly Phoenix application needs 3 things:

-   [PostgreSQL](http://postgresql.org) or other database to store data
-   [Nginx](http://nginx.org) (or other web server) to serve HTTP(S)
    requests quickly and secure
-   [Erlang](http://www.erlang.org) and [Elixir](http://elixir-lang.org)
    as runtime environment for Phoenix

Let's discuss each point in more details.

## Installing and Configuring PostgreSQL

First of all, let's install PostgreSQL from Ubuntu software repository.
```console
$ sudo apt-get update

$ sudo apt-get install -y postgresql postgresql-contrib
```

After that, we create a role for our app. Don't use "postgres" role with
your app as it gives too much privileges to our app.

```console
$ sudo -u postgres createuser --superuser -dP ${PROJECT}_app
```

Then we need to change authentication method that is used to connect to
Postgres. In either case we'll get connection errors from postgrex. You
can edit this config file with your favourite editor if you like, but
this way is quicker if you have default config from repositories.
```console
$ sudo sed -i "s/peer$/md5/" /etc/postgresql/*/main/pg_hba.conf
```


Optionally, you can tune other Postgres options here. But it's out of
spec for this article.

After all restart Postgres to apply changes in configs.
```console
$ sudo systemctl restart postgresql
```


## Installing Erlang and Elixir

If you just try to install Erlang and Elixir from system repositories
you get it a bit outdated, so I suggest to use ones from [Erlang
Solutions](https://www.erlang-solutions.com). Firstly, we need to add
them to apt, then update repository and install.
```console

$ wget https://packages.erlang-solutions.com/erlang-solutions_2.0_all.deb

$ sudo dpkg -i erlang-solutions_2.0_all.deb

$ sudo apt-get update

$ sudo apt-get install -y esl-erlang elixir
```

In order to successfully build and manage the project we need a bit more
packages.

```console
$ sudo apt-get install -y tmux git make gcc
```

## Brining up the Project

It's time to get your project from git, build it and make running, but,
firstly, I want to mention that it's very bad habit to run web projects
from "root" user. So let's create a user for our app and switch to .

```terminal
$ sudo useradd -m -s /bin/bash $PROJECT
```

Next step is to create a folders for our project and give our user
rights to utilise it. It's up to you where to create it, but I suggest
*/opt* folder as it looks suitable from the point of convention.

We need 2 folders: one for **code** from git and one for **server
specific configuration**.
```console
$ sudo mkdir -p /opt/$PROJECT

$ sudo chown -R $PROJECT:$PROJECT /opt/$PROJECT
```

Now we switch to the user we created for our project and complete most
of project bootstrapping work from it.

```terminal
$ sudo su $PROJECT

(PROJECT) $ ssh-keygen -t rsa -b 4096 -C “$EMAIL”
```

Here you need to add public key from */home/\$PROJECT/.ssh/id_rsa.pub*
to authorised keys that can have access to your git repo through ssh.
For GitHub it is called "Deploy keys" and can be added from repo
settings page. After that, you can clone, tune and build your project.
```terminal
(PROJECT) $ cd /opt/$PROJECT

(PROJECT) $ git clone ssh://$REPO repo
```

Now we need to create `/opt/$PROJECT/repo/.env` and add
our Postgres connection settings there and may be other stuff.

```bash
PHX_HOST=$DOMAIN
PHX_SERVER=true
 
DATABASE_URL="ecto://${PROJECT}_app:${POSTGRES_PASSWORD}@localhost/${PROJECT}_db"

```

Replace all variables here with actual values. And go on with
compilation and bringing it up.

```terminal
(PROJECT) $ cd repo

(PROJECT) $ set -a

(PROJECT) $ source .env

(PROJECT) $ mix deps.get

(PROJECT) $ mix compile

(PROJECT) $ echo "SECRET_KEY_BASE=$(mix phx.gen.secret)" >> .env

(PROJECT) $ mix ecto.setup

(PROJECT) $ mix assets.deploy

(PROJECT) $ iex -S mix phx.server
```

If we don't get an error here, we can press `Ctrl-C` twice, return to our
default user (press `Ctrl-D` or type exit `<CR>`) and go on.

## Postgres Security Fix
Due to `CREATE EXTENSION ...` work properly in Ecto migrations we've created PostgreSQL role with `superuser` privileges. It's time to fix it.

```terminal
$ sudo -u postgres psql ${PROJECT}_db -c "ALTER ROLE ${PROJECT}_app nosuperuser" 
```

## Configuring system.d to Start the App

System.d is the core Ubuntu initialising and service monitoring system.
To make our app start automatically on boot we need to configure it.
Here is a config for our app.

```ini
[Unit]
Description=$PROJECT Phoenix application
After=network.target postgresql.service
Requires=postgresql.service
Wants=nginx.service

[Service]
Type=simple
User=$PROJECT
Group=$PROJECT
EnvironmentFIle=/opt/$PROJECT/repo/.env
WorkingDirectory=/opt/$PROJECT/repo
ExecStart=/usr/bin/mix phx.server
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
You need to change variables here to actual values to make it work.

After you put this file to */etc/systemd/system/\$PROJECT.service*, run
the following commands to initialise and start it.

```console
$ sudo systemctl enable $PROJECT

$ sudo systemctl start $PROJECT
```

## Setting up Nginx and certbot for HTTPS

To finish our journey and make our application look professional we need
to make it work through secure HTTPS protocol and listen on standard 443
port. To achieve it we need to install and configure Nginx proving our
app through ssl and caching static content to enhance load speed and
certbot to get free SSL certificated for us.

```console
$ sudo apt-get install -y certbot python3-certbot-nginx nginx
```


Let's configure Nginx to serve our app. Here I assume that Phoenix app
is configured to listen on 4000 port (as default configuration). Add
this config to */etc/nginx/sites-available/\$PROJECT*.

```nginx
server {
    listen 0.0.0.0:80;
    server_name $DOMAIN;

    location / {
        proxy_pass http://localhost:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

As in all previous config files you need to replace variables with
their values.

```terminal
$ sudo ln -s /etc/nginx/sites-available/$PROJECT /etc/nginx/sites-enabled/$PROJECT

$ sudo systemctl restart nginx
```

After that, Nginx is ready to work for us trough insecure http protocol
on 80 port. Let's start it and check if it's working.

Now point your browser to the \$DOMAIN and check if you see your apps
pages. If everything is ok, we are moving to last stage: adding SSL.

This command should do the job.
```terminal

$ sudo certbot --nginx
```

Now we have our application deployed and working in correct way. To be
even more secure it's recommended to configure firewall to allow only
SSH and HTTP(S) ports, but it's a topic of the other article.

{% include calendly.md %}
