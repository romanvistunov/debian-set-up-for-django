# Debian Server Set Up for Django Instruction

In this guide we will set up clean Debian server for Python and Django projects. We will configure secure SSH connection, install from Debian repositories and from sources all needed packages and ware it together for working Debian Django server.

[Youtube video guide (in Russian)](https://www.youtube.com/watch?v=FLiKTJqyyvs)

## Create user, setup SSH

Connect through SSH to remote Debian server and update repositories and install some initial needed packages:

```
apt-get update
apt-get install sudo
sudo apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make
```

Configure SSH:

```
sudo vim /etc/ssh/sshd_config
    AllowUsers www
    PermitRootLogin no
    PasswordAuthentication no
```

Restart SSH server, change `www` user password:

```
sudo service ssh restart
sudo passwd www
```

## Init — must-have packages & ZSH

```
sudo apt-get install -y zsh tree redis-server nginx  libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python3-dev python3-lxml libxslt-dev python-libxml2 python-libxslt1 libffi-dev libssl-dev python-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor
```

Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh):

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
chsh -s $(which zsh)
```

## Install python 3.7

mkdir ~/code

Build from source python 3.7, install with prefix to ~/.python folder:

```
wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tgz ; \
tar xvf Python-3.7.* ; \
cd Python-3.7.3 ; \
mkdir ~/.python ; \
./configure --enable-optimizations --prefix=/home/www/.python ; \
make -j8 ; \
sudo make altinstall
```

Now python3.7 in `/home/www/.python/bin/python3.7`. Update pip:

```
sudo /home/www/.python/bin/python3.7 -m pip install -U pip
```

Next step

```
mkdir project
python3 -m pip install -U pip
sudo apt-get install python3-pip
python3 -m venv env
apt-get install python3-venv
. ./env/bin/activate
pip install Django
```

Ok, now we can pull our project from Git repository (or create own), create and activate Python virtual environment:

```
cd code
git pull project_git
cd project_dir
python3.7 -m venv env
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

Change settings.py:

```
DATABASES = {
'default': {
'ENGINE': 'django.db.backends.postgresql_psycopg2',
'NAME': 'dbname',
'USER': 'username',
'PASSWORD': 'userpass',
'HOST': '127.0.0.1',
'PORT': '5432'
}
}
```

Install psycopg2:

```
pip install psycopg2-binary
```

Make migrations and create superuser:

```
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
```

## Install and configure supervisor

Now recommended way is using Systemd instead of supervisor. If you need supervisor — welcome:

```
sudo apt install supervisor

vim /home/www/code/project/bin/start_gunicorn.sh
	#!/bin/bash
	source /home/www/code/project/env/bin/activate
	source /home/www/code/project/env/bin/postactivate
	exec gunicorn  -c "/home/www/code/project/gunicorn_config.py" project.wsgi

chmod +x /home/www/code/project/bin/start_gunicorn.sh

nano /etc/supervisor/conf.d/PROJECTNAME.conf
	[program:gunicorn]
	command=/home/www/code/project/bin/start_gunicorn.sh
	user=www
	process_name=%(program_name)s
	numproc=1
	autostart=true
	autorestart=true
	redirect_stderr=true
```


Nginx config /sites-enabled/default
```
server {
  listen 80 default_server;
  listen [::]:80 default_server;
  root /var/www/html;
  location / {
  proxy_pass http://127.0.0.1:8001;
  proxy_set_header X-Forwarded-Host $server_name;
  proxy_set_header X-Real-IP $remote_addr;
  add_header P3P 'CP="ALL DSP COR PSAa PSDa OUR NOR ONL UNI COM NAV"';
  add_header Access-Control-Allow-Origin *;
  }
}

```

If you need some Gunicorn example config — welcome:

```
command = '/home/www/code/project/env/bin/gunicorn'
pythonpath = '/home/www/code/project/project'
bind = '127.0.0.1:8001'
workers = 3
user = 'www'
limit_request_fields = 32000
limit_request_field_size = 0
raw_env = 'DJANGO_SETTINGS_MODULE=project.settings'
```


Настройка HTTPS

Для настройки будем использовать Cerbot
```
sudo apt-get install certbot python-certbot-nginx
sudo certbot –nginx
```
И следуем подсказкам на экране. Если появились ошибки, то устраните и повторите. Например, ошибка может возникнуть если вы указали доменное имя, не относящееся к данному серверу.
Для автоматического обновления сертификата введите команду.
```
sudo certbot renew --dry-run 
```
Теперь тестим сайт и всё должно работать!

Если статика так и не отобразилась, то попробуйте открыть какой-нибудь css файл указав полный адрес до него, если выдал ошибку 403, то всё отлично, но проблема в правах доступа нужно экспериментировать с ними, попробуйте сбросить все настройки прав для www-data на каталог /var/www. Если ошибка 404, то нужно смотреть в сторону настроек location в настройках NGINX.
