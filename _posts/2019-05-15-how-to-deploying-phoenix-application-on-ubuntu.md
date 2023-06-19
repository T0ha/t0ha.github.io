---
layout: post
locale: en_US
title: "How To: Deploying Phoenix Application on Ubuntu"
description: Description
image: /assets/images/posts/image.png
tags:
  - devops
  - elixir
  - phoenix
  - phoenix-framework
  - web-development
status: edit
categories:
  - en
  - devops
  - elixir
published: false
---

#### Simple way

Phoenix is cool, right? You've just created your brilliant web
application and it works fine on your local machine. It's awesome, but
nobody knows about it. Stop, your friend knows, because you've told him
in the bar a week ago and he is tied waiting to try it! Hey, it's time
to deploy the app and show it to the entire world...

### **Prerequisite**

For the purpose of this article we assume that you have the following
things ready:

-   a Simple [Phoenix](http://phoenixframework.org) application
    working locally
-   [git](https://git-scm.com) repository for the app, hosted on
    [GitHub](https://github.com/) (it can be hosted anywhere, bu I'll
    describe configuration process for GitHub)
-   [Ubuntu](https://www.ubuntu.com), preferably 22.04-LTS, installed
    and you have a sudo'er user ssh access to it

For the purpose of this article I use several environmental variables:

-   **\$PROJECT** --- your project name, eg. my_project
-   **\$REPO** --- your project's git repository ssh url
-   **\$EMAIL** --- your email address
-   **\$DOMAIN** --- domain for your app

### **Let's prepare the ground**

In order to work properly Phoenix application needs 3 things:

-   [PostgreSQL](http://postgresql.org) or other database to store data
-   [Nginx](http://nginx.org) (or other web server) to serve HTTP(S)
    requests quickly and secure
-   [Erlang](http://www.erlang.org) and [Elixir](http://elixir-lang.org)
    as runtime environment for Phoenix

Let's discuss each point in more details.

### **Installing and Configuring PostgreSQL**

First of all, let's install PostgreSQL from Ubuntu software repository.

    $ sudo apt-get update

    $ sudo apt-get install -y postgresql postgresql-contrib

After that, we create a role for our app. Don't use "postgres" role with
your app as it gives too much privileges to our app.

    $ sudo -u postgres createuser -dP ${PROJECT}_app

Then we need to change authentication method that is used to connect to
Postgres. In either case we'll get connection errors from postgrex. You
can edit this config file with your favourite editor if you like, but
this way is quicker if you have default config from repositories.

    $ sudo sed -i "s/peer$/md5/" /etc/postgresql/*/main/pg_hba.conf

Optionaly, you can tune other Postgres options here. But it's out of
spec for this article.

After all restart Postgres to apply changes in configs.

    $ sudo systemctl restart postgresql

### **Installing Erlang and Elixir**

If you just try to install Erlang and Elixir from system repositories
you get it a bit outdated, so I suggest to use ones from [Erlang
Solutions](https://www.erlang-solutions.com). Firstly, we need to add
them to apt, then update repository and install.

    $ wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb

    $ sudo dpkg -i erlang-solutions_1.0_all.deb

    $ sudo apt-get update

    $ sudo apt-get install -y esl-erlang elixir

In order to successfully build and manage the project we need a bit more
packages.

    $ sudo apt-get install -y tmux git make gcc npm webpack

### **Brining up the Project**

It's time to get your project from git, build it and make running, but,
firstly, I want to mention that it's very bad habit to run web projects
from "root" user. So let's create a user for our app and switch to .

    $ sudo useradd -m -s /bin/bash $PROJECT

Next step is to create a folders for our project and give our user
rights to utilise it. It's up to you where to create it, but I suggest
*/opt* folder as it looks suitable from the point of convention.

We need 2 folders: one for **code** from git and one for **server
specific configuration**.

    $ sudo mkdir -p /opt/$PROJECT/config

    $ sudo chown -R $PROJECT:$PROJECT /opt/$PROJECT

Now we switch to the user we created for our project and complete most
of project bootstrapping work from it.

    $ sudo su $PROJECT

    (PROJECT) $ ssh-keygen -t rsa -b 4096 -C “$EMAIL”

Here you need to add public key from */home/\$PROJECT/.ssh/id_rsa.pub*
to authorised keys that can have access to your git repo through ssh.
For GitHub it is called "Deploy keys" and can be added from repo
settings page. After that, you can clone, tune and build your project.

    (PROJECT) $ cd /opt/$PROJECT

    (PROJECT) $ git clone ssh://$REPO repo

    (PROJECT) $ cp repo/config/prod.secrets.exs.sample config/prod.secrets.exs

Now we need to create */opt/\$PROJECT/config/prod.secrets.exs* and add
our Postgres connection settings there and may be other stuff.

<https://medium.com/media/d63bc74ec774e4d63a54922f423c6b14/href>

Replace all variables here with actual values. And go on with
compilation and bringing it up.

    (PROJECT) $ ln -s /opt/$PROJECT/config/prod.secret.exs repo/config/prod.secret.exs

    (PROJECT) $ cd repo

    (PROJECT) $ export MIX_ENV=prod

    (PROJECT) $ mix deps.get

    (PROJECT) $ mix compile

    (PROJECT) $ mix do ecto.create, ecto.migrate

    (PROJECT) $ cd assets && npm install && webpack --mode production && cd ..

    (PROJECT) $ mix phx.digest

    (PROJECT) $ iex -S mix phx.server

If we don't get an error here, we can press Ctrl-C twice, return to our
default user (press Ctrl-D or type exit \<CR\>) and go on.

### **Configuring system.d to Start the App**

System.d is the core Ubuntu initialising and service monitoring system.
To make our app start automatically on boot we need to configure it.
Here is a config for our app.

<https://medium.com/media/8a70617eb13a300b2e5080d49657558f/href>

You need to change variables here to actual values to make it work.

After you put this file to */etc/systemd/system/\$PROJECT.service*, run
the following commands to initialise and start it.

    $ sudo systemctl enable $PROJECT

    $ sudo systemctl start $PROJECT

### **Setting up Nginx and certbot for HTTPS**

To finish our journey and make our application look professional we need
to make it work through secure HTTPS protocol and listen on standard 443
port. To achieve it we need to install and configure Nginx proving our
app through ssl and caching static content to enhance load speed and
certbot to get free SSL certificated for us.

    $ sudo apt-get install software-properties-common

    $ sudo add-apt-repository universe

    $ sudo add-apt-repository ppa:certbot/certbot

    $ sudo apt-get update

    $ sudo apt-get install -y certbot python-certbot-nginx nginx

Let's configure Nginx to serve our app. Here I assume that Phoenix app
is configured to listen on 4000 port (as default configuration). Add
this config to */etc/nginx/sites-available/\$PROJECT*.

<https://medium.com/media/d58da2ac7e9edebaf941db2ce9c3ea4b/href>

As in all previous config files you need to replace variables with
their values.

    $ sudo ln -s /etc/nginx/sites-available/$PROJECT /etc/nginx/sites-enabled/$PROJECT

    $ sudo systemctl restart nginx

After that, Nginx is ready to work for us trough insecure http protocol
on 80 port. Let's start it and check if it's working.

Now point your browser to the \$DOMAIN and check if you see your apps
pages. If everything is ok, we are moving to last stage: adding SSL.

This command should do the job.

    $ sudo certbot --nginx

Now we have our application deployed and working in correct way. To be
even more secure it's recommended to configure firewall to allow only
SSH and HTTP(S) ports, but it's a topic of the other article.

#### PS

Dear readers, thank you for your interest and feedback. You are welcome
to support this article by clapping, subscribing to the publication or
following [\@T0ha666](http://twitter.com/T0ha666).

Do you have a tech startup idea? Or do you need some help with web,
mobile or IoT programming?

Visit [3∑
website](https://3epsilon.pro?utm_source=medium&utm_medium=post&utm_compaign=footer&utm_content=293645f38145)
and get a great offer from us.

You can contact me by Email (<t0hashvein@gmail.com>), Twitter
([\@t0ha666](http://twitter.com/t0ha666)), Reddit (/u/t0ha) or Telegram
(t.me/war1and).\
See you next time...

![](https://medium.com/_/stat?event=post.clientViewed&referrerSource=full_rss&postId=293645f38145){width="1"
height="1"}

------------------------------------------------------------------------

[How To: Deploying Phoenix Application on
Ubuntu](https://medium.com/3-elm-erlang-elixir/how-to-deploying-phoenix-application-on-ubuntu-293645f38145)
was originally published in [3∑: Elm, Erlang &
Elixir](https://medium.com/3-elm-erlang-elixir) on Medium, where people
are continuing the conversation by highlighting and responding to this
story.
