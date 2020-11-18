[TOC]

### 一  安装Redis

#### 1  下载安装tcl

````shell
wget http://downloads.sourceforge.net/tcl/tcl8.6.1-src.tar.gz
tar -xzvf tcl8.6.1-src.tar.gz
cd  /usr/local/tcl8.6.1/unix/
./configure  
make && make install
````

#### 2  下载Redis

````shell
wget https://download.redis.io/releases/redis-6.0.9.tar.gz
tar xzf redis-6.0.9.tar.gz
cd redis-6.0.9
make && make test && make install
````

### 二  redis的生产环境启动方案

（1）redis utils目录下，有个redis_init_script脚本

（2）将redis_init_script脚本拷贝到linux的/etc/init.d目录中，将redis_init_script重命名为redis_6379，6379是我们希望这个redis实例监听的端口号

（3）修改redis_6379脚本的第6行的REDISPORT，设置为相同的端口号（默认就是6379）

（4）创建两个目录：/etc/redis（存放redis的配置文件），/var/redis/6379（存放redis的持久化文件）

（5）修改redis配置文件（默认在根目录下，redis.conf），拷贝到/etc/redis目录中，修改名称为6379.conf

（6）修改redis.conf中的部分配置为生产环境

````xml
daemonize	yes							让redis以daemon进程运行
pidfile		/var/run/redis_6379.pid 	设置redis的pid文件位置
port		6379						设置redis的监听端口号
dir 		/var/redis/6379				设置持久化文件的存储位置
````

（7）启动redis，执行

````shell
cd /etc/init.d
chmod 777 redis_6379
./redis_6379 start
````

（8）确认redis进程是否启动，ps -ef | grep redis

（9）让redis跟随系统启动自动启动

````shell
sudo update-rc.d redis_6379 defaults
````

