# 集群架构



## nginx

Nginx是一个高性能的HTTP和反向代理web服务器，同时也提供IMAP/POP3/SMTP服务。



- 反向代理
- 通过简单配置，即可实现集群和负载均衡
- 静态资源虚拟化





### 反向代理

何为反向代理







### 安装

nginx下载：http://nginx.org/en/download.html



- 安装gcc环境

```shell
yum install gcc-c++
```

- 安装PCRE库，解析正则表达式

```shell
yum install -y pcre pcre-devel
```

- zlib压缩和解压缩依赖

```shell
 yum install -y zlib zlib-devel
```

- SSL安全的加密的套接字协议层，用于HTTP安全传输，也就是https

```shell
 yum install -y openssl openssl-devel
```



- 解压安装包

```shell
# tar -zxvf nginx-1.16.1.tar.gz
```

- 编译之前创建临时目录，否则启动过程中会报错

```shell
# mkdir /var/temp/nginx -p
```

- 配置nginx

```shell
./configure     --prefix=/usr/local/nginx    --pid-path=/var/run/nginx/nginx.pid     --lock-path=/var/lock/nginx.lock     --error-log-path=/var/log/nginx/error.log  --http-log-path=/var/log/nginx/access.log    --with-http_gzip_static_module    --http-client-body-temp-path=/var/temp/nginx/client    --http-proxy-temp-path=/var/temp/nginx/proxy     --http-fastcgi-temp-path=/var/temp/nginx/fastcgi     --http-uwsgi-temp-path=/var/temp/nginx/uwsgi     --http-scgi-temp-path=/var/temp/nginx/scgi

```

- make编译

```shell
make
```

- 安装

```shell
make install
```

- 进入sbin目录启动nginx，刚才在配置时已经执行制定--prefix，则这个路径为安装目录

```shell
[root@localhost sbin]# pwd
/usr/local/nginx/sbin
```

- 访问nginx服务

![1671941287836](G:\bookmark\docs\assets\1671941287836.png)





### 进程模式

master进程：主进程，为指挥者，可向worker发送信号

worker进程：工作进程，可配置多个worker，真正做事的进程



- 修改nginx.conf

```shell
#user  nobody;
worker_processes  2;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}
//省略部分
```



重启nginx后，可以看到多个worker进程了：

```powershell
[root@localhost conf]# ps -ef|grep nginx
root      6801     1  0 11:58 ?        00:00:00 nginx: master process ./nginx
nobody    6967  6801  0 12:12 ?        00:00:00 nginx: worker process
nobody    6968  6801  0 12:12 ?        00:00:00 nginx: worker process
root      6970  3317  0 12:12 pts/0    00:00:00 grep --color=auto nginx

```



### worker抢占机制

![1671942012020](G:\bookmark\docs\assets\1671942012020.png)

worker争夺accept_mutex锁



### 传统服务器事件处理

![1671942149098](G:\bookmark\docs\assets\1671942149098.png)

同步阻塞状态，如果并发量高，不利于客户端的请求处理



### nginx事件处理

![1671942260561](G:\bookmark\docs\assets\1671942260561.png)

异步非阻塞



- 涉及配置nginx.conf

```shell
events {
    # 默认在linux环境下使用的epoll
	use epoll; 
	# 每个worker进程允许的最大连接数
    worker_connections  1024;
}

```



扩展知识：epoll模型



### nginx.conf配置结构

![1671942522707](G:\bookmark\docs\assets\1671942522707.png)



### 核心配置文件详解

nginx.conf



指定worker进程的用户

**user nobody**

进程数量，参考可设置为cpu数量-1

**worker_processes 2;**  

日志路径

**error_log  logs/error.log**





```shell
http{
	#包含文件
	include mime.types;

    #高效传输文件
	sendfile on;

	#压缩，针对html/css/js
    gzip on;

    # 虚拟主机
    server{
		
    }

}
```



### nginx.pid找不到

1、文件夹(var/run/nginx)不存在，创建目录,之后会提示 

invalid PID number



解决方式： 指定nginx的配置文件地址 

./nginx -c /usr/local/nginx/confg/nginx.conf



### nginx常用命令



./nginx -s stop 暴力关闭

./nginx -s quit  优雅停止

./nginx -t 检测配置文件合法性

./nginx -v nginx版本

./nginx -V nginx详情信息



### nginx日志切割-手动

1、编写bash脚本

```shell
#!/bin/bash
LOG_PATH="/var/log/nginx/"
RECORD_TIME=$(date -d "yesterday" +%Y-%m-%d+%H:%M)
PID=/var/run/nginx/nginx.pid
mv ${LOG_PATH}/access.log ${LOG_PATH}/access.${RECORD_TIME}.log
mv ${LOG_PATH}/error.log ${LOG_PATH}/error.${RECORD_TIME}.log

#向Nginx主进程发送信号，用于重新打开日志文件
kill -USR1 `cat $PID`
```

将sh脚本添加权限

```shell
chmod +x cut_my_log.sh
```



### nginx日志切割-自动

1、利用定时任务crontabs，安装定时任务组件

```shell
yum install crontabs
```

2、`crontab -e` 编辑并且添加一行新的任务：

```shell
*/1 * * * * /usr/local/nginx/sbin/cut_my_log.sh
```

以上表达式意思是每隔一分钟，执行一次sh脚本。

3、重启定时任务

```
service crond restart
```



附：常用定时任务指令

```shell
service crond start         //启动服务
service crond stop          //关闭服务
service crond restart       //重启服务
service crond reload        //重新载入配置
crontab -e                  // 编辑任务
crontab -l                  // 查看任务列表
```



**定时任务表达式**

Cron表达式通常分为5或者6个域，每个域代表一个含义：

|          | 分   | 时   | 日   | 月   | 星期几 | 年（可选） |
| -------- | ---- | ---- | ---- | ---- | ------ |----|
| 取值范围 | 0~59 | 0~23 | 1~31 | 1~12 | 1~7 | 2019/2020/.... |





### 静态资源服务器



































## 虚拟机virsualBox

- 启动失败，提示如下：

![1671931932405](G:\bookmark\cluster\assets\1671931932405.png)

以管理员身份修改虚拟机启动类型：

C:\WINDOWS\system32>bcdedit /set hypervisorlaunchtype off



