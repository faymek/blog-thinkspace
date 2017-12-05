---
layout: post
title: "阿里云部署 django gunicorn nginx mysql ubuntu python3"
comments: true
description: "阿里云部署 django gunicorn nginx mysql ubuntu python3"
keywords: django, nginx
---

摘要: Django + mysql + nginx + gunicorn + supervisord + ubuntu

因为毕业设计需要个服务器来提供api，于是乎弄了个阿里云的ecs。平时都是在本机上运行的开发服务器，所以部署还是花了不少时间。真的有不少坑，这里我也把遇到的坑顺带提了一下。

## 主要用下面这些东西

1. Ubuntu：因为大家都说Ubuntu简单方便，我就选它了
2. Django：python上的web开发我喜欢用它，方便好用
3. Gunicorn : 一个Python WSGI UNIX的HTTP服务器，按我的理解，它的作用可能就是用来代替django自带server的。有了它就不用自带的runserver了，自带的server只能单线程运行，而这个能并发多线程。
4. Nginx：高性能的HTTP和反向代理服务器，按我的理解主要干这几件事
   - 缓冲请求：直到收完整个请求，再转发给Gunicorn，避免Gunicorn一直占用进程等待接收完成。
   - 负载均衡：由nginx占用80端口，而gunicorn同时运行多个程序占用不同的端口。Nginx根据客户端发来的不同的url请求，把请求分发给不同的程序。这样就达到一台服务器运行多个网站的目的呢
   - 访问控制：限制流量，限制ip，限制连接数量
   - 处理静态文件：对图片css之类的静态文件请求不用经过Python服务器，直接由Nginx处理，可能更快吧
5. Supervisord：python2.x写的进程管理程序，用他来管理gunicorn的运行情况，当gunicorn的进程挂了，可以自动重新运行gunicorn。
6. git：把服务器设置成git服务器，以后每次修改了代码只用push一次就完事了。
7. Mysql：没什么好说的

> 总的来说运行起来就是:
> 浏览器 <--> Nginx服务器 <-socket-> Gunicorn <--> Django

## 安装配置环境

> 安装过程中会遇到了很多loc什么什么的错误，那是因为vps没有设置好，关于这点，很郁闷，网上的各种文章都没提到这个。
> 解决方法就是设置环境变量，把`export LC_ALL=C`加到.bash里面去

> 另外网上的教程都是用的virtualenv来弄虚拟环境，可是我配置的时候怎么都弄不好，总是报错，所以干脆就用python3自带的虚拟环境了，值得说的是他会提示你安装python3-venv 安装了不行的话 可以尝试安装python3.5-venv，

> 注意一定要记得把上面提到的环境变量设置好，不然venv狂报错，我反正当时google好半天都没找出这个原因。

```
#更新VPS的apt-get里的东西
sudo apt-get update
sudo apt-get upgrade

#给本机的python2安装pip，关于pip我也是很服气的。。
sudo apt-get install python-pip
 
#安装Supervisor
sudo apt-get install supervisor
 
#安装Nginx
sudo apt-get install nginx

#安装git
sudo apt-get install git

#安装python3和venv,安装后输入python3就运行的python3，输入pip3 运行的就是python3的pip
sudo apt-get install python3
sudo apt-get install python3-venv
sudo apt-get install python3-pip  

```

### 配置git仓库

把自己电脑上`~/.ssh/id_ras.pub`里ras公钥告诉服务器，这样他就知道你是自己人，允许你去上传东西

把公钥复制在这个文件最后面

```
vim ~/.ssh/authorized_keys

```

在服务器上创建用来提交代码的空仓库

```
cd /srv
mkdir xxxx.git
cd xxxx.git
git init --bare

```

指定工作区为网站的代码目录，以后网站的源码就放在/webapps/xxxx/code里面了

```
git clone /srv/xxxx.git /webapps/xxxx/code

```

添加git钩子，作用呢就是当有代码push的时候，可以自动复制到刚才设置的工作区去

```
cd /srv/xxxx.git/hooks/
vim post-receive

```

修改并复制下面的内容进去

```
#!/bin/sh

# This file is to be used by git repo on server
# It should be a file called hooks/post-receive in the bare git repo

set -xe 

echo "Running Post Receive Hook"

export GIT_WORK_TREE=/webapps/xxxx/code
export GIT_DIR=${GIT_WORK_TREE}/.git

cd ${GIT_WORK_TREE}
git fetch origin
git reset --hard origin/master

```

修改post-receive的权限

```
chmod u+x post-receive

```

回到本机上传代码

```
git remote add aliyun root@XXXXXXXX:/srv/xxx.git
git push -u aliyun master

```

### 配置虚拟环境

现在去/srv/webapps里面看看代码上传好没，然后配置下虚拟环境

```
 sudo python3 -m venv xxxx

```

开启虚拟环境，

```
cd /sev/webapps/xxxx
source bin/activate

```

现在就进入虚拟环境中了，运行python和pip命令都是虚拟环境中的python和pip，不管怎么弄都不会影响系统中自带的python，所以现在就可以用pip安装所需的库了。

安装完后用下面的语句退出环境

```
deactivate

```

\###安装mysql

```
sudo apt-get install mysql-server mysql-client

```

重新进下虚拟环境，在虚拟环境中运行`pip install mysqlclient`

*因为我的项目是python3的所以用mysqlclient，好像python2得用另一个，我也是真的是懒得管了，我已经完全服气了，部署个服务器在网上到处搜教程照着一模一样的做，结果不是这里报错就是那里报错，满世界google，什么破东西*

好，安装好mysql，登陆进去建个数据库，一定要记得设置编码呀

```
create database 数据库名 default charset utf8 collate utf8_general_ci

```

然后出来配置下Django的`setting.py` 弄成类似这样

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': '刚才建的数据库名',
        'USER':'用户名',
        'PASSWORD':'密码',
        'HOST':'127.0.0.1',
        'PORT':3306,
    }
}

```

### 使用gunicorn

先打开虚拟环境，在虚拟环境中安装gunicorn

```
sudo pip install gunicorn

```

在项目目录里面然后测试下gunicorn是否正常运行（项目目录就是manage.py所在的目录 因为我们用的虚拟环境所以要这样

```
GUNICORN地址 --chdir 项目目录地址 --pythonpath PYTHON地址 -w4 -b127.0.0.1:8000 xxxx.wsgi

```

其中GUNICORN地址和PYTHON地址可以开启虚拟环境后分别用`which gunicorn`和`which python`得到，而项目地址就是manage.py的那个目录地址，可以切换过去用`pwd`得到

如果没用虚拟环境的话，直接这样就行了

```
gunicorn -w4 -b127.0.0.1:8000 xxxx.wsgi

```

现在你应该就能从服务器ip的8000端口访问网站了。ctrl+c结束

如果浏览器返回 Bad Request (400）那多半是你setting.py里面ALLOWED_HOSTS数组里没设置好咯，把自己的ip或者域名设置进去，或者直接设置成`ALLOWED_HOSTS=['*',]`就行了

### 使用supervisord

生成 supervisor 默认配置文件

```
echo_supervisord_conf > /etc/supervisord.conf

```

打开supervisor.conf在最后添加这些配置

```
[program:xxxx] ;这个xxxx可以自己随便取，只是方便标识
command=刚才启动gunicorn输入的那个命令
directory=项目所在地址，和刚才gunicorn里用的那个项目地址一样的
startsecs=0
stopwaitsecs=0
autostart=true
autorestart=true

```

启动supervisor服务端，启动的时候会自动的吧配置文件里刚才设置的程序运行。注意: supervisord启动了服务端程序以后，就只需用supervisorctl客户端来进行操作了

```
supervisord -c /etc/supervisord.conf

```

在supervisor客户端上重启xxxx程序

```
supervisorctl -c /etc/supervisord.conf restart xxxx

```

启动，停止，或重启 supervisor 管理的某个程序 或 所有程序：

```
supervisorctl -c /etc/supervisord.conf [start|stop|restart] [program-name|all]

```

查看程序当前的运行状态：

```
supervisorctl -c /etc/supervisord.conf status

```

## 配置Django

```
#因为django只在开发服务器上处理静态文件，而部署后静态文件是交给nginx处理的，所以这样要把用到的静态文件收集到一个目录里面，方便使用
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'collected_static')

#设置media目录
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')


```

搜集静态文件，这会自动把静态文件收集到刚才设置的collected_static目录里

```
python manage.py collectstatic

```

在全局的urls.py里面，加上处理静态文件的url

```
from django.contrib.staticfiles.urls import staticfiles_urlpatterns

urlpatterns = [.........] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT) + staticfiles_urlpatterns()


```

开发环境与部署环境分离，因为部署后要把debug之类的关了，但是每次都修改很麻烦，我这里采用的是判断hostname的方法，把这个写在setting.py里面，

```
if socket.gethostname() == 'GoGodeMacBook-Pro.local': #本机
    DEBUG=True
    TEMPLATE_DEBUG = True
    DATABASES=........
else: # 服务器
    DEBUG=False
    TEMPLATE_DEBUG = False
    DATABASES=........

```

然后我部署的时候还遇到个问题，他老提示我找不到zh_Hans语言包，原来是中间件MIDDLEWARE里没加`django.middleware.locale.LocaleMiddleware`这个要加在`django.contrib.sessions.middleware.SessionMiddleware`的后面，奇怪的是我本机上并没有报错啊..

### 使用nginx

新建网站配置,sites-avaliable顾名思义就是存放可以使用的网站，但不一定要运行这些网站，还有一个sites-enabled目录存放需要运行的网站，要激活一个网站就软连接到sites-enabled，然后重启nginx就行了

```
sudo vim /etc/nginx/sites-available/xxxxxx.conf

```

添加如下配置：(记得把#开头的备注删了...

```
server {
    listen  80; #外网通过这个端口访问
    server_name xxxxx.com; #域名或者ip地址
    charset     utf-8;
    access_log  /var/log/nginx/pixelMill.log; #日志会记录在这个文件


    client_max_body_size 75M;

    location /media  {
        alias /webapps/xx/code/media;   #django里面的media路径
    }

    location /static {
	expires 30d;
        alias /webapps/xx/code/collected_static;  #django里面的static路径
    }

    #除了media和static以外的请求都给django处理
    location / {
        proxy_pass http://127.0.0.1:8000;   #刚才gunicorn设置的端口
        include     /etc/nginx/uwsgi_params;
    }
}

```

这个配置就告诉了nginx，nginx自己负责media文件和静态文件，其他请求就让Django负责

激活网站：

```
sudo ln -s /etc/nginx/sites-available/xxx.conf /etc/nginx/sites-enabled/xxxx.conf

```

测试配置文件

```
sudo service nginx configtest

```

要是fail的话，其实可以先restart下，然后照他的提示输入`systemctl status nginx.service`然后会告诉你哪行出错了

重启Nginx服务器，注意每次修改了配置文件都需要restart或者reload

```
sudo service nginx restart
#或者
sudo service nginx reload

```

好了完了~~

