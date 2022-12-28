Ccache的进阶使用

# §Cccache的文档地址
如果单从搜索引擎查找ccache的文档是非常少的，更多的还是在官网上进行查找

* [ccache官方链接](https://ccache.dev/)

* [github链接](https://github.com/ccache/ccache/)
  
> 官网中的使用相关的内容集中在 ccache/doc/MANUAL.adoc

ccache所有的配置项都可以通过export环境变量进行修改,conf文件中缺省ccache前缀，仅保留后边的即可。

优先级: 环境变量> 配置文件

如 
* 通过环境变量
  
    `export CCACHE_RECACACHE=true`

* 通过配置文件
  
    conf中写入 `recache = true`， 即有相同作用。
# §CCache的几个好用的配置项
在MANUAL.adoc，详细且细致的介绍了所有的配置项，但是很多配置项是比较少使用的，在此介绍几个总结几个相对常用的
* CCACHE_MAX_SIZE
    
    缓存目录最大缓存容量，一般无需手动清理缓存，ccache在运行时会自动通过清理缓存。
* CCACHE_CONFIGPATH
  
  指定配置文件路径，对于首次安装的时候，对于ccache并不了解，很多时候都选择了默认安装，有时路径不是那么符合预期,因此ccache提供了修改配置文件的路径的能力.

  通过 `export CCACHE_CONFIGPATH=/usr1/ccache/ccache.conf`,放入/etc/profile中，让其永久生效。
* CCACHE_RECACHE

    有时因为一些意外，可能导致了Manifest文件损坏，一直命中不了编译效率恶化，可以通过CCACHE_RECACHE重新生成Manifest和Result文件，就和重启一样解决99%命中的问题。

    `export CCACHE_RECACHE=true`
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
 ccache在 4.4版本提供了远端缓存能力，如果别人在开发过程中归档了到了远端缓存目录（nfs、redis、http）等，那么别人编译过即相当于你也编译过，无需再重复编译其他人的修改。如果多位开发者都mnt同一个nfs路径,ccache除了会对于CCACHE_DIR存一份Manifest/Result文件，同时也会在nfs路径中存一份，当本地CCACHE_DIR不存在时，则会去访问远端路径，同时将远端路径下的M/R文件拷贝下自己的CCACHE_DIR（本地主缓存路径），这样无需每次都访问远端仓库，而造成使用缓存不稳定。
## nfs
1. 挂载Nfs
 
  本次直接使用 虚拟机的共享目录 /mnt/hgfs/,并将*权限修改为777*

2. 声明远端仓库
   
  ```
  export CCACHE_SECONDARY_STORAGE=file:/mnt/hgfs/
  export CCACHE_DEBUG=1
  ```
3. 进行编译
   
  
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
存储到db 0上

2. 设置远程仓库
```
export CCACHE_SECONDARY_STORAGE="redis://123456@192.168.1.5:6379/0|connect-timeout=50"
export CCACHE_DEBUG=1
```

3. 进行首次编译
   
   make

4. 远端归档
![redis首次编译](./pic/redis%E9%A6%96%E6%AC%A1%E7%BC%96%E8%AF%91.png)

通过ccache的日志，可以看到已经成功存储到了redis中

5. 使用远端缓存
 - 删除本地主仓缓存
    `rm /usr1/cache/* -rf`
 - make
![redis远端命中](./pic/redis%E8%BF%9C%E7%AB%AF%E5%91%BD%E4%B8%AD.png)

可以看到，在远端命中的同时将缓存文件转存到本地的缓存目录当中，下次会直接本地缓存目录命中


* http
```
export CCACHE_SECONDARY_STORAGE=http://localhost

export CCACHE_SECONDARY_STORAGE=http://someusername:p4ssw0rd@example.com/cache/

export CCACHE_SECONDARY_STORAGE=http://localhost:8080|layout=bazel|connect-timeout=50
```
  