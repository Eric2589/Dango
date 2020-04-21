部署基于centos7+ngix+uwsgi+python3+django
版本centos7.7    python3.6    django2.1.5
1.服务器环境部署
	* 
更新系统软件包



yum updata -y
	* 
安装软件管理包和可能使用到的依赖



yum -y groupinstall "Development tools"
yum install openssl-devel bzip2-devel expat-devel gdbm-devel readline-devel sqlite-devel psmisc libffi-devel
	* 
下载python3到/usr/local目录



cd /usrr/local

wget https://www.python.org/ftp/python/3.6.6/Python-3.6.6.tgz
解压

tar  -zxvf Python-3.6.6.tgz
进入Python-3.6.6路径

cd Python-3.6.6
编译安装到指定路径
./configure --prefix=/usr/local/python3
安装python3
make 
make install
建立软连接，方便在终端中使用python3
ln -s /usr/local/python3/bin/python3.6 /usr/bin/python3
给pip3也建立软连接（非必须)

ln -s /usr/local/python3/bin/pip3.6 /usr/bin/pip3
查看python3与pip3版本

python3 -V
pip3 -V
	* 
安装virtualenv


pip3  install virtualenv
建立软连接（非必须）

ln -s /usr/local/python3/bin/virtualenv /usr/bin/virtualenv
安装成功之后建立两个文件夹，存放env与网站文件（可根据习惯自行选定）

mkdir -p /data/env
mkdir -p /data/wwwroot
	* 
切换到/data/env,创建指定版本的虚拟环境,名称自定，如blog



virtualenv --python=/usr/bin/python3 blog
进入/data/env/blog/bin,启动虚拟环境

source activate
	* 
虚拟环境中用pip3安装django和uwsgi


pip3 install Django==2.1.5
pip3 install uwsgi
注：uwsgi在系统中装一次，在虚拟环境中装一次
建立软连接，方便使用

ln -s /usr/local/python3/bin/uwsgi /usr/bin/uwsgi

2.本地项目搬迁
	* 
切换到/data/wwwroot,使用git工具（也可用其他工具)将项目克隆到该目录下



git clone  项目地址
	* 
安装依赖包



pip freeze > requirements.txt
使用python manage.py runserver 检查项目是否正常运行
注：此处本地使用的是sqlite数据库，如果使用mysql



#导出Mysql,django为你的数据库mysqldump -uroot -ppassword django>django.sql
#把django.sql上传到服务器，在服务器里用下面命令导入
mysql -uroot -ppassword
use dajngo;
source your Path\django.sql
	* 
在项目根目录添加uwsgi配置文件


1）ini格式，在项目中创建uwsgi.ini,添加以下内容



#添加配置选择
[uwsgi]
#配置和nginx连接的socket连接
socket=127.0.0.1:8997
#配置项目路径，项目的所在目录chdir=/data/wwwroot/myBlog/
#配置wsgi接口模块文件路径,也就是wsgi.py这个文件所在的目录名
wsgi-file=mysite/wsgi.py
#配置启动的进程数
processes=4
#配置每个进程的线程数
threads=2
#配置启动管理主进程
master=True
#配置存放主进程的进程号文件
pidfile=uwsgi.pid
#配置dump日志记录
daemonize=uwsgi.log`

启动uwsgi


uwsgi  --ini  uwsgi.ini
显示 [uWSGI] getting INI configuration from uwsgi.ini 表明uwsgi运行成功
可能通过ps -ef|grep uwsgi   查看确认是否uwsgi启动.
ini配置其他相关命令：



#停止运行uwsgi，通过包含主进程编号的文件设置停止项目uwsgi --stop uwsgi.pid
#重启uwsgi
uwsgi --reload uwsgi.pid
2）xml配置
路径是 /data/wwwroot/myBlog/,在项目根目录下创建
myBlog.xml文件，输入如下内容：


<uwsgi>    
   <socket>127.0.0.1:8997</socket> <!-- 内部端口，自定义 -->
   <chdir>/data/wwwroot/myBlog/</chdir> <!-- 项目路径 -->            
   <module>mysite.wsgi</module>  <!-- mysite为wsgi.py所在目录名-->
   <processes>4</processes> <!-- 进程数 -->     
   <daemonize>uwsgi.log</daemonize> <!-- 日志文件 --></uwsgi>

保存
注意<module>里的mysite，为wsgi.py所在的目录名。

这种方式的配置,可以用下面的命令启动.


#启动uwsgluwsgi -x mysite.xml
#uwsgi有没有启动成功,可以用下面的命令查看
ps -ef|grep uwsgi
#如果想重启uwsgi,先使用下面的命令杀掉进程,再启动uwsgi
killall -9 uwsgi
	* 
安装nginx,配置nginx.conf文件


进入home目录，执行下面命令

cd /home/
wget http://nginx.org/download/nginx-1.13.7.tar.gz
下载完成后，执行解压命令：

tar -zxvf nginx-1.13.7.tar.gz
	* 
进入解压后的nginx-1.13.7文件夹，依次执行以下命令：



./configure
make
make install

nginx一般默认安装好的路径为/usr/local/nginx
在/usr/local/nginx/conf/中先备份一下nginx.conf文件，以防意外。


cp nginx.conf nginx.conf.bak
	* 
然后打开nginx.conf，把原来的内容删除，直接加入以下内容：




events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    server {
        listen 80;
        server_name  域名; #改为自己的域名，没域名修改为127.0.0.1:80
        charset utf-8;
        location / {
           include uwsgi_params;
           uwsgi_pass 127.0.0.1:8997;  #端口要和uwsgi里配置的一样
           uwsgi_param UWSGI_SCRIPT mysite.wsgi;  #wsgi.py所在的目录名+.wsgi
           uwsgi_param UWSGI_CHDIR /data/wwwroot/myBlog/; #项目路径
           
        }
        location /static/ {
        alias /data/wwwroot/myBlog/static/; #静态资源路径
        }
    }
}

要留意备注的地方，要和UWSGI配置文件mysite.xml，还有项目路径对应上。
进入/usr/local/nginx/sbin/目录
执行./nginx -t命令先检查配置文件是否有错，没有错就执行以下命令：


./nginx
部署完成

遇见的一些问题：
1.internal server error
导致这个问题的原因很多，可以尝试重启uwsgi 
2.后台管理样式丢失
1、在settings.py尾部：

STATIC_ROOT  = os.path.join(BASE_DIR, 'static')#指定样式收集目录
#或STATIC_ROOT = '/www/mysite/mysite/static'  #指定样式收集目录
2、收集CSS样式，在终端输入：

python manage.py collectstatic
自动将后台css样式收集到/static/目录下，刷新页面即可
