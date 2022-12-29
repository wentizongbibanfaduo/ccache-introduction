Ccache的进阶使用

# §Ccache配置的使用方式

> 官网中的使用相关的内容集中在 ccache/doc/MANUAL.adoc

ccache所有的配置项都可以通过export环境变量进行修改,conf文件中缺省ccache前缀，仅保留后边的即可。

优先级: 环境变量> 配置文件

如 
* 通过环境变量
  
  一次生效，其他终端无效。

    ```
    export CCACHE_RECACHE=true
    ```

* 通过配置文件
  
  写入配置文件永久生效。
  1. 查看本地config文件路径
  ``` 
  ccache -s -v
  ```
  ![查看config路径](./pic/3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/%E6%9F%A5%E7%9C%8Bconfig%E8%B7%AF%E5%BE%84.png)

  2. 写入配置
   
    conf中写入 `recache = true`， 即有相同作用。

  ![配置文件生效](./pic/3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E7%94%9F%E6%95%88.png)

# §CCache中好用的配置项
在[MANUAL.adoc](https://github.com/ccache/ccache/blob/master/doc/MANUAL.adoc)，详细且细致的介绍了所有的配置项，但是很多配置项是比较少使用的，在此总结几个相对常用的

* CCACHE_CONFIGPATH
  
  指定配置文件路径，对于首次安装的时候，对于ccache并不了解，很多时候都选择了默认安装，有时路径不是那么符合预期,因此ccache提供了修改配置文件的路径的能力.

  通过 `export CCACHE_CONFIGPATH=/usr1/ccache/ccache.conf`,放入/etc/profile中，让其永久生效。
* CCACHE_RECACHE

    有时因为一些意外，可能导致了Manifest文件损坏，一直命中不了编译效率恶化，可以通过CCACHE_RECACHE重新生成Manifest和Result文件，就和重启一样解决99%命中的问题。

    `export CCACHE_RECACHE=true`

* CCACHE_MAXSIZE
    
    缓存目录最大缓存容量，一般无需手动清理缓存，ccache在运行时会自动通过清理缓存。
    
    `export CCACHE_MAXSIZE=20G`

* CCACHE_BASEDIR
  
  假定某条编译命令如下
  ```
  ccache gcc -I/usr/include/example -I/home/alice/project2/include -c /home/alice/project1/src/example.c
  ```

  有时候编译时指定了绝对路径但是下载工程路径可能发生改变（当前cwd为/home/alice/project1/），正常情况下由于源文件路径发生改变，被认为是另外一次工程编译，大大降低了命中率。

  此时我们可以通过设置`export CCACHE_BASEDIR=/home/alice/project1`, ccache会将编译命令改造成使用相对路径的进行编译归档。
  ```
  gcc -I/usr/include/example -I/home/alice/project2/include -c ../src/example.c
  ```
  在工程project2 未变化的情况下，不会由于project1的绝对路径发生改变而造成大量miss，大大提高了ccache在不同目录下的命中率。
  
* 远端缓存
  
    成熟的项目很难由一个人完全Cover住，我们在实际开发中很多时候都是并行开发，再单独合入。有时别人开发了一个重要特性，修改量极大，而你在pull对方的代码后，由于这部分得到了修改，当你进行编译时这部分就没办法进行缓存了需要重新编译。对于大型项目而言（数十个人维护一个C++项目），每天都可能存在大量的修改，这样每天需要重新编译其他人的修改部分也是一个很头疼的地方。

   
# §Ccache的远程仓库
 ccache在 4.4版本提供了远端缓存能力，如果别人在开发过程中归档了到了远端缓存目录（nfs、redis、http）等，那么别人编译过即相当于你也编译过，无需再重复编译其他人的修改。如果多位开发者都mnt同一个nfs路径,ccache除了会对于CCACHE_DIR存一份Manifest/Result文件，同时也会在nfs路径中存一份，当本地CCACHE_DIR不存在时，则会去访问远端路径，<font color='blue'>同时将远端路径下的M/R文件拷贝下自己的CCACHE_DIR（本地主缓存路径），这样无需每次都访问远端仓库，而造成使用缓存不稳定 </font>。
## Nfs
1. 挂载Nfs
 
  挂载自己的nfs （/mnt/nfs/）并将*权限修改为777*

2. 声明远端仓库
   
  ```
  export CCACHE_SECONDARY_STORAGE=file:/mnt/nfs/
  export CCACHE_DEBUG=1
  ```
3. 进行首次编译
   ![nfs首次编译](./pic/3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/nfs%E9%A6%96%E6%AC%A1%E7%BC%96%E8%AF%91.png)
   通过日志可以看到已经归档到了/mnt/nfs/中

4.  使用远端缓存
 - 删除本地主仓缓存
    `rm /usr1/cache/* -rf`
 - 编译
   ![nfs命中](./pic/3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/nfs-%E5%91%BD%E4%B8%AD.png)
   可以看到，在远端命中的同时将缓存文件转存到本地的缓存目录当中，下次会优先命中本地缓存目录。
  
## Redis
  
使用方式有以下几种
```
export CCACHE_SECONDARY_STORAGE=redis://localhost

export CCACHE_SECONDARY_STORAGE=redis://p4ssw0rd@cache.example.com:6379/0|connect-timeout=50

export CCACHE_SECONDARY_STORAGE=redis+unix:/run/redis.sock

export CCACHE_SECONDARY_STORAGE=redis+unix:///run/redis.sock

export CCACHE_SECONDARY_STORAGE=redis+unix://p4ssw0rd@localhost/run/redis.sock?db=0
```
1. 开启redis
   
  对此在另一台192.168.1.5机器上开启redis,设置6379端口并**打开防火墙**，设置密码为123456。


2. 设置远程仓库
   存储到db 0上
```
export CCACHE_SECONDARY_STORAGE="redis://123456@192.168.1.5:6379/0|connect-timeout=50"
export CCACHE_DEBUG=1
```

3. 进行首次编译
![redis首次编译](./pic/3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/redis%E9%A6%96%E6%AC%A1%E7%BC%96%E8%AF%91.png)

通过ccache的日志，可以看到已经成功存储到了redis中

4. 使用远端缓存
 - 删除本地主仓缓存
    `rm /usr1/cache/* -rf`
 - 编译
![redis远端命中](./pic//3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/redis%E8%BF%9C%E7%AB%AF%E5%91%BD%E4%B8%AD.png)


## Http

1. 配置nginx
   
vim /etc/nginx/nginx.conf
输入如下内容
```
user root;
worker_processes auto;

error_log /var/log/nginx/error.log notice;
pid /var/run/nginx.pid;

events {
        worker_connections 1024;
}

http {
        include mime.types;
        default_type application/octet-stream;

        access_log /var/log/nginx/access.log;
        sendfile on;
        keepalive_timeout 65;
        server {
                listen 20080;
                server_name _;
                location /ccache/ {
                        autoindex on;
                        autoindex_exact_size off;
                        autoindex_localtime on;
                        alias /ccache/;
                        log_not_found off;
                        dav_methods PUT DELETE;
                        create_full_put_path on;
                        client_max_body_size 100M;
                }
        }
}

```
2. 启动nginx
   
   直接执行 `nginx `

   如上配置的端口为20080, 本地浏览器输入启动nginx的 ip:20080，可以成功进入才认为启动nginx成功。
   ![nginx启动](./pic/3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/nginx%E5%90%AF%E5%8A%A8.png)

3. 首次编译
```
export CCACHE_SECONDARY_STORAGE="http://192.168.32.128:20080/ccache/"
export CCACHE_DEBUG=1
```
查看ccache日志，可以看到已经成功存储到http仓库中
![http首次编译](./pic/3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/nginx.png)
4. 远端命中
   - 删除本地主仓缓存
`rm /usr1/cache/* -rf`
 - 编译
![http命中](./pic/3-%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8/nginx%E5%91%BD%E4%B8%AD.png)