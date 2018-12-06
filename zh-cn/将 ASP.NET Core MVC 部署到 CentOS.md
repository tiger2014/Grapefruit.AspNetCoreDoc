**一、安装 .NET Core SDK or .NET Core Runtime**

1. 在服务器上注册 Microsoft 秘钥

   ```bash
   sudo rpm -Uvh https://packages.microsoft.com/config/rhel/7/packages-microsoft-prod.rpm ## 每台机器安装时都需要注册
   ```

2. 更新可供安装的版本

   ```
   sudo yum update
   ```

3. 安装 .NET Core SDK or .NET Core Runtime

   ```bash
   sudo yum install aspnetcore-runtime-2.2 ## 如果需要在 Linux 上进行开发，则安装 aspnetcore-sdk-2.2
   ```

4. 查看版本信息，确定是否安装正确

   ```bash
   dotnet --info
   ```

**二、安装 MySQL Server**

1. 使用 rpm -ivh 下载 MySQL 安装包

   ```bash
   wget https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
   rpm -ivh mysql80-community-release-el7-1.noarch.rpm
   ```

2. 校验 MD5 与官网是否一致

   ```
   md5sum mysql80-community-release-el7-1.noarch.rpm
   ```

3. 安装 MySQL Server

   ```bash
   sudo yum install mysql-server
   ```

4. 启动 MySQL Server 守护程序

   ```bash
   sudo systemctl start mysqld
   ```

   上面的命令执行之后，不会有任何的反应，所以为了确保我们已经执行成功，我们可以使用下面的命令来查看是否启用了 MySQL 服务。此时，如果我们的 MySQL 服务已经启动了，则会输出我们的执行信息

   ```bash
   sudo systemctl status mysqld
   ```

5. 重新设置 root 用户密码

   在安装 MySQL 的过程中，会自动的为 MySQL root 用户生成一个临时的密码，通过下面的命令找到这个默认的密码

   ```bash
   sudo grep 'temporary password' /var/log/mysqld.log
   ```

   使用下面的命令来修改 root 用户密码，输入默认密码正确后就可以自己设置新的密码

   ```bash
   sudo mysql_secure_installation
   ```

   重新设置的密码至少包含一个大写字母，一个小写字母，一个数字和一个特殊字符的12个字符

   在整个设置密码的过程中，总共有五步：设置 root 密码；是否禁止 root 账号远程登录；是否禁止匿名账号（anonymous）登录；是否删除测试库；是否确认修改

6. 设置允许远程登录

   ```bash
   mysql -h localhost -u root -p ## 登录数据库，输入修改后的密码完成登录
   use mysql; ## 选择 mysql 表
   select user,host from user; ## 查询当前的用户
   update user set host='%' where user='root'; ## 设置允许使用 root 账户进行远程登录
   flush privileges; ## 刷新设置
   exit; ## 退出 mysql
   ```

**三、部署 ASP.NET Core 程序**

1. 将程序放置到我们的服务器上

   可以使用 mkdir 命令创建路径或者直接使用 WinSCP 之类的工具可视化的新增路径，之后将部署的程序使用 WinSCP 放置到指定的目录下

   ```bash
   mkdir /usr/local/wwwroot/psu ## 这里设置的地址是部署程序存放的根目录，即直接存放程序的 dll 或其它的资源文件
   cd /usr/local/wwwroot/psu/ ## 最后一定要加上 /
   ```

2. 使用 dotnet 命令运行我们的程序

   ```bash
   dotnet PSU.Site.dll
   ```

   Linux 对于大小写是敏感的，部署程序存放的路径以及主程序的 dll 名称都需要和实际相同

**四、安装 Nginx 服务器及配置反向代理**

1. 下载 Nginx 安装包并解压

   这里采用源码编译安装 Nginx 的方式，需要提前安装 gcc 环境

   ```bash
   sudo yum install gcc gcc-c++ ## 为服务器安装 gcc 环境 
   wget -c http://nginx.org/download/nginx-1.9.9.tar.gz ## 下载 1.9.9 版本的 Nginx 安装包
   tar -zxvf nginx-1.9.9.tar.gz ## 解压
   ```

2. 进入解压后的 Nginx 目录下，创建默认的配置文件

   ```bash
   cd nginx-1.9.9 ## 进入 Nginx 目录
   ./configure ## 创建默认的配置文件
   ```

3. 添加插件去完善 Nginx 的服务

   ```bash
   sudo yum install -y pcre pcre-devel ## 使用 pcre 来解析正则表达式
   sudo yum install -y zlib zlib-devel ## 使用 zlib 对 http 包的内容进行 gzip（对于静态资源的压缩与解压缩）
   sudo yum install -y openssl openssl-devel ## OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及 SSL 协议，完善 Nginx 对于 HTTPS 协议的支持
   ```

4. 重新创建配置文件，并编译安装

   ```bash
   ./configure ## 重新创建配置文件
   make install ## 编译安装
   ```

5. 设置开机自启动及自动重启

   ```bash
   
   ```

   建一个软连接用来指向 Nginx 的执行目录

   ```bash
   
   ```

   创建自启动脚本（注意修改 Nginx 路径地址）

   ```bash
   vim /etc/init.d/nginx
   ```

   ```bash
   #!/bin/sh
   #
   # nginx - this script starts and stops the nginx daemon
   #
   # chkconfig:   - 85 15
   # description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
   #               proxy and IMAP/POP3 proxy server
   # processname: nginx
   # config:      /usr/local/nginx/conf/nginx.conf
   # config:      /etc/sysconfig/nginx
   # pidfile:     /usr/local/nginx/logs/nginx.pid
   # Source function library.
   . /etc/rc.d/init.d/functions
   # Source networking configuration.
   . /etc/sysconfig/network
   # Check that networking is up.
   [ "$NETWORKING" = "no" ] && exit 0
   nginx="/usr/local/nginx/sbin/nginx"
   prog=$(basename $nginx)
   NGINX_CONF_FILE="/usr/local/nginx/conf/nginx.conf"
   [ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
   lockfile=/var/lock/subsys/nginx
   make_dirs() {
      # make required directories
      user=`$nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
      if [ -z "`grep $user /etc/passwd`" ]; then
          useradd -M -s /bin/nologin $user
      fi
      options=`$nginx -V 2>&1 | grep 'configure arguments:'`
      for opt in $options; do
          if [ `echo $opt | grep '.*-temp-path'` ]; then
              value=`echo $opt | cut -d "=" -f 2`
              if [ ! -d "$value" ]; then
                  # echo "creating" $value
                  mkdir -p $value && chown -R $user $value
              fi
          fi
      done
   }
   start() {
       [ -x $nginx ] || exit 5
       [ -f $NGINX_CONF_FILE ] || exit 6
       make_dirs
       echo -n $"Starting $prog: "
       daemon $nginx -c $NGINX_CONF_FILE
       retval=$?
       echo
       [ $retval -eq 0 ] && touch $lockfile
       return $retval
   }
   stop() {
       echo -n $"Stopping $prog: "
       killproc $prog -QUIT
       retval=$?
       echo
       [ $retval -eq 0 ] && rm -f $lockfile
       return $retval
   }
   restart() {
       configtest || return $?
       stop
       sleep 1
       start
   }
   reload() {
       configtest || return $?
       echo -n $"Reloading $prog: "
       killproc $nginx -HUP
       RETVAL=$?
       echo
   }
   force_reload() {
       restart
   }
   configtest() {
     $nginx -t -c $NGINX_CONF_FILE
   }
   rh_status() {
       status $prog
   }
   rh_status_q() {
       rh_status >/dev/null 2>&1
   }
   case "$1" in
       start)
           rh_status_q && exit 0
           $1
           ;;
       stop)
           rh_status_q || exit 0
           $1
           ;;
       restart|configtest)
           $1
           ;;
       reload)
           rh_status_q || exit 7
           $1
           ;;
       force-reload)
           force_reload
           ;;
       status)
           rh_status
           ;;
       condrestart|try-restart)
           rh_status_q || exit 0
               ;;
       *)
           echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
           exit 2
   esac
   ```

   将自启动的脚本赋予脚本可执行权限

   ```bash
   chmod a+x /etc/init.d/nginx
   ```

   将 Nginx 服务加入 chkconfig 管理列表中（chkconfig命令主要用来更新（启动或停止）和查询系统服务的运行级信息）

   ```bash
   chkconfig --add /etc/init.d/nginx
   chkconfig nginx on
   ```

   启动 Nginx 守护程序

   ```bash
   systemctl start nginx
   systemctl status nginx ## 可以查看 Nginx 服务的状态
   ```

   一些常用的 Nginx 命令（这里需要处于 Nginx 的程序执行目录（/usr/local/nginx/sbin/）下时执行，否则使用全路径）：

   ```bash
   ./nginx ## 启动 Nginx 服务
   ./nginx -s stop ## 停止 Nginx 服务(先查出 nginx 的进程 id ，再使用 kill 命令强制杀掉进程)
   ./nginx -s quit ## 停止 Nginx 服务(等待 Nginx 的进程处理完成后再进行停止)
   ./nginx -s reload ## 重新加载 Nginx 配置
   ./nginx -t ## 测试配置文件格式是否正确
   ```

6. 为 ASP.NET Core 程序设置反向代理

   进入 Nginx 的配置文件所在的路径，打开默认的配置文件

   ```bash
   cd /usr/local/nginx/conf/
   vim nginx.conf
   ```

   找到 Server 节点，看到配置文件的默认配置项

   ```bash
   server {
   　　listen        80;
   　　server_name   localhost;
   　　location / {
           root   html;
           index  index.html index.htm;
   　　}
   }
   ```

   当使用 Nginx 时，会根据 server_name 匹配服务器，例如当我们运行 Nginx 成功后，通过浏览器浏览本机的 ip 时，默认会显示 Nginx 的默认页面。而当没有匹配的 server_name 时，Nginx 则会使用默认服务器。 如果没有定义默认服务器，则配置文件中的第一台服务器则成为默认服务器

   修改配置信息，将80端口的请求转发到我们使用 Kestrel 监听的5000端口上的应用上

   ```bash
   server {
   　　listen        80;
   　　server_name   localhost;
   　　location / {
   　　　　# Kestrel
   　　　　proxy_pass            http://localhost:5000;
   　　　　proxy_http_version    1.1;
   　　　　proxy_set_header      Upgrade $http_upgrade;
   　　　　proxy_set_header      Host $http_host;
   　　　　proxy_cache_bypass    $http_upgrade;
       }
   }
   ```

   验证修改的配置格式是否正确，配置正确后执行 reload 命令使配置生效

   ```bash
   ./nginx -t ## 测试配置文件格式是否正确
   ./nginx -s reload ## 重新加载配置
   ```

**五、安装守护程序及配置自启动**

1. 守护进程（Daemon）是一种运行在后台的特殊进程，它独立于控制终端并且周期性的执行某种任务或等待处理某些发生的事件。由于在linux中，每个系统与用户进行交流的界面称为终端，每一个从此终端开始运行的进程都会依附于这个终端，这个终端被称为这些进程的控制终端，当控制终端被关闭的时候，相应的进程都会自动关闭。但是守护进程却能突破这种限制，它脱离于终端并且在后台运行，并且它脱离终端的目的是为了避免进程在运行的过程中的信息在任何终端中显示并且进程也不会被任何终端所产生的终端信息所打断。它从被执行的时候开始运转，直到整个系统关闭才退出

2. 安装 Supervisor 守护程序

   ```bash
   sudo yum install supervisor
   ```

3. 创建存储需要守护的程序配置文件目录并创建默认配置文件

   ```bash
   mkdir -m 700 -p /etc/supervisor
   ```

   创建 Supervisor 的配置文件，将默认的配置文件复制一份到配置文件目录下

   ```bash
   echo_supervisord_conf > /etc/supervisor/supervisord.conf
   ```

   在 /etc/supervisor 目录下我们创建一个存放我们 dotnet core 进程文件的文件夹 conf.d

   ```bash
   mkdir -m 700 /etc/supervisor/conf.d
   ```

   修改配置文件，在配置文件的最后，将路径指向我们创建的 conf.d 文件夹（这里注意需要将配置文件前面的 ; 删除，否则配置节点是被注释的）

   ```bash
   [include]
   files=/etc/supervisor/conf.d/*.conf
   ```

4. 创建一个配置文件来管理需要守护的项目的 dotnet 进程

   ```bash
   cd /etc/supervisor/conf.d/ ## 进入设定的配置文件文件夹
   touch PSU.conf ## 创建需要守护的程序的配置文件
   ```

   ```bash
   [program:PSU.Site]
   command=dotnet PSU.Site.dll  #要执行的命令
   directory=/usr/local/wwwroot/psu/ #命令执行的目录
   environment=ASPNETCORE__ENVIRONMENT=Production #环境变量
   user=root  #进程执行的用户身份
   stopsignal=INT
   autostart=true #是否自动启动
   autorestart=true #是否自动重启
   startsecs=3 #自动重启间隔
   stderr_logfile=/usr/local/wwwroot/logs/psu.err.log #标准错误日志
   stdout_logfile=/usr/local/wwwroot/logs/psu.log #标准输出日志
   ```

   这里指明的日志输出的文件，需要事先创建好

5. 创建 Supervisor 的自启动服务

   ```bash
   touch /etc/systemd/system/supervisor.service ## 创建自启动脚本文件
   vim /etc/systemd/system/supervisor.service
   ```

   ```bash
   Description=supervisor
   
   [Service]
   Type=forking
   ExecStart=/usr/bin/supervisord -c /etc/supervisor/supervisord.conf
   ExecStop=/usr/bin/supervisorctl shutdown
   ExecReload=/usr/bin/supervisorctl reload
   KillMode=process
   Restart=on-failure
   RestartSec=42s
   
   [Install]
   WantedBy=multi-user.target
   ```

   重新载入守护程序设置

   ```bash
   systemctl daemon-reload
   ```

   设置 supervisor.service 服务开机启动

   ```bash
   systemctl enable supervisor.service
   ```

   启动守护程序服务

   ```
   systemctl start supervisor.service
   ```

   查看程序是否成功运行

   ```bash
   ps -ef | grep PSU.Site
   ```