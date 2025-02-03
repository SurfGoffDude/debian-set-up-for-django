# Debian Server Set Up for Django Instruction

In this guide we will set up clean Debian server for Python and Django projects. We will configure secure SSH connection, install from Debian repositories and from sources all needed packages and ware it together for working Debian Django server.

[Youtube video guide (in Russian)](https://www.youtube.com/watch?v=FLiKTJqyyvs)

## Create user, setup SSH

Connect through SSH to remote Debian server and update repositories and install some initial needed packages:

```
sudo apt-get update ; \
sudo apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make
```

Add new user:
```
adduser artem
usermod -aG sudo artem
```

Configure SSH:

```
sudo vim /etc/ssh/sshd_config
    AllowUsers artem
    PermitRootLogin no
    PasswordAuthentication no
```

Restart SSH server, change `artem` user password:

```
sudo service ssh restart
sudo passwd artem
```

## Init — must-have packages & ZSH

```
sudo apt-get install -y zsh tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python-imaging python3-lxml libxslt-dev python-libxml2 python-libxslt1 libffi-dev libssl-dev python-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor
```

if doesn't work then use modified command:

```
sudo aptitude install -f zsh tree redis-server nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev liblzma-dev python3-dev python3-lxml libxslt-dev libffi-dev libssl-dev  gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor
```

Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh):

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

Configure some needed aliases:

```
vim ~/.zshrc
    alias cls="clear"
```

## Install python 3.13.1

mkdir ~/code

Build from source python 3.13.1, install with prefix to ~/.python folder:

```
wget https://www.python.org/ftp/python/3.13.1/Python-3.13.1.tgz ; \
tar xvf Python-3.13.* ; \
cd Python-3.13.1 ; \
mkdir ~/.python ; \
./configure --enable-optimizations --prefix=/home/artem/.python ; \
make -j8 ; \
sudo make altinstall
```

If you meet this ```configure: WARNING: pkg-config is missing. Some dependencies may not be detected correctly.``` you should do this:

```
sudo apt install pkgconf
sudo apt install build-essential
./configure --enable-optimizations --prefix=/home/artem/.python ; \
make -j8 ; \
sudo make altinstall
```

Now python3.13.1 in `/home/artem/.python/bin/python3.13.1`. Update pip:

```
sudo /home/artem/.python/bin/python3.13 -m pip install -U pip
```

If you have ssl issue, use this [guide](https://gist.github.com/SurfGoffDude/52651ec4532b3cc54bfa1a56cf02d9ea)

Ok, now we can pull our project from Git repository (or create own), create and activate Python virtual environment:

```
cd code
git pull project_git
cd project_dir
python3.13.1 -m venv env
. ./env/bin/activate
```

## Install and configure PostgreSQL

Install PostgreSQL 11 and configure locales.

```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - ; \
RELEASE=$(lsb_release -cs) ; \
echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | sudo tee  /etc/apt/sources.list.d/pgdg.list ; \
sudo apt update ; \
sudo apt -y install postgresql-11 ; \
sudo localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
export LANGUAGE=ru_RU.UTF-8 ; \
export LANG=ru_RU.UTF-8 ; \
export LC_ALL=ru_RU.UTF-8 ; \
sudo locale-gen ru_RU.UTF-8 ; \
sudo dpkg-reconfigure locales
```

Add locales to `/etc/profile`:

```
sudo vim /etc/profile
    export LANGUAGE=ru_RU.UTF-8
    export LANG=ru_RU.UTF-8
    export LC_ALL=ru_RU.UTF-8
```

Change `postges` password, create clear database named `dbms_db`:

```
sudo passwd postgres
su - postgres
export PATH=$PATH:/usr/lib/postgresql/11/bin
createdb --encoding UNICODE dbms_db --username postgres
exit
```

Create `dbms` db user and grand privileges to him:

```
sudo -u postgres psql
postgres=# ...
create user dbms with password 'some_password';
ALTER USER dbms CREATEDB;
grant all privileges on database dbms_db to dbms;
\c dbms_db
GRANT ALL ON ALL TABLES IN SCHEMA public to dbms;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public to dbms;
GRANT ALL ON ALL FUNCTIONS IN SCHEMA public to dbms;
CREATE EXTENSION pg_trgm;
ALTER EXTENSION pg_trgm SET SCHEMA public;
UPDATE pg_opclass SET opcdefault = true WHERE opcname='gin_trgm_ops';
\q
exit
```

Now we can test connection. Create `~/.pgpass` with login and password to db for fast connect:

```
vim ~/.pgpass
	localhost:5432:dbms_db:dbms:some_password
chmod 600 ~/.pgpass
psql -h localhost -U dbms dbms_db
```

Run SQL dump, if you have:

```
psql -h localhost dbms_db dbms  < dump.sql
```

## Install and configure supervisor

Now recommended way is using Systemd instead of supervisor. If you need supervisor — welcome:

```
sudo apt install supervisor

vim /home/artem/code/project/bin/start_gunicorn.sh
	#!/bin/bash
	source /home/artem/code/project/env/bin/activate
	source /home/artem/code/project/env/bin/postactivate
	exec gunicorn  -c "/home/artem/code/project/gunicorn_config.py" project.wsgi

chmod +x /home/artem/code/project/bin/start_gunicorn.sh

vim project/supervisor.salesbeat.conf
	[program:www_gunicorn]
	command=/home/www/code/project/bin/start_gunicorn.sh
	user=artem
	process_name=%(program_name)s
	numprocs=1
	autostart=true
	autorestart=true
	redirect_stderr=true
```

If you need some Gunicorn example config — welcome:

```
command = '/home/artem/code/project/env/bin/gunicorn'
pythonpath = '/home/artem/code/project/project'
bind = '127.0.0.1:8001'
workers = 3
user = 'artem'
limit_request_fields = 32000
limit_request_field_size = 0
raw_env = 'DJANGO_SETTINGS_MODULE=project.settings'
```
