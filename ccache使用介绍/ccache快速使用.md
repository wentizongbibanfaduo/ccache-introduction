
@[toc]

# ccache是什么
   ccache -- “compiler cache”的缩写，是一个gcc/g++的c语言编译器缓存。
# ccache能做什么
   简单来说，ccache将基于单条gcc编译命令级别颗粒，将本次编译命令的产物，复制进行压缩一份到缓存目录中，下次编译的时候， <font color='red'>如果检测到相同编译命令，并且没有修改输入的源文件（当前c/cpp或依赖的头文件） </font>，则直接读取缓存目录中上次编译流程，省去编译时间，从而优化编译时长。

 对项目工程而言，一次代码修改仅会改变极少量的源文件，使用ccache只会重新编译修改部分的代码相关的源文件，而未进行修改的源文件则可以直接使用缓存优化了编译时长。
# ccache的效率如何
## 小试牛刀
  
使用ccache编译一个简单的cpp

- 原始编译
    ```
    time g++ hello.cpp -o hello.o
    ```
- 首次通过ccache 进行编译
  
    `time ccache g++ hello.cpp -o hello.o`

    ![原始编译](./pic/1-%E5%85%A5%E9%97%A8%E4%BD%BF%E7%94%A8/%E5%8E%9F%E5%A7%8B%E7%BC%96%E8%AF%91gcc.jpg)


- 二次通过ccache 进行编译
  
    `time ccache g++ hello.cpp -o hello.o`

    ![gcc二次命中](./pic/1-%E5%85%A5%E9%97%A8%E4%BD%BF%E7%94%A8/gcc%E4%BA%8C%E6%AC%A1%E5%91%BD%E4%B8%AD.jpg)

|编译方式     | 编译时长|
|-------- | -----|
|原始构建  | 0.24s|
|使用ccache首次构建  | 0.30s|
|使用ccache二次构建  | 0.02s|


**使用ccache首次编译时长 > 原始编译时长 >> 使用ccache二次编译时长**

原因是使用ccache会对编译命令做缓存处理需要花费一些耗时。
（具体原因放在之后再更新）
## 实际工程表现
以使用虚拟机1U2G规格编译redis-6.0.16：

原始直接make: 

![redis原始编译](./pic/1-%E5%85%A5%E9%97%A8%E4%BD%BF%E7%94%A8/redis%E5%8E%9F%E5%A7%8B%E7%BC%96%E8%AF%91.jpg)

make clean后使用ccache首次编译：

![redis首次编译](./pic/1-%E5%85%A5%E9%97%A8%E4%BD%BF%E7%94%A8/redis%E9%A6%96%E6%AC%A1%E7%BC%96%E8%AF%91.jpg)

make clean后使用ccache二次编译：

![redis二次命中](./pic/1-%E5%85%A5%E9%97%A8%E4%BD%BF%E7%94%A8/redis%E4%BA%8C%E6%AC%A1%E5%91%BD%E4%B8%AD.jpg)

编译方式     | 编译时长
-------- | -----
原始构建  | 35s
使用ccache首次构建  | 43s
使用ccache二次构建  | 2.8s

可以看到，对于重复编译，无论是单命令编译，还是实际工程编译，使用ccache构建速度都压缩了十倍，极大的提升了编译效率！
# ccache 如何使用
  ## 安装ccche
  * apt直接安装
    ```shell
    # 安装ccache
    apt install ccache
    # 查看ccache版本
    ccache -V 
    ```

  * 源码安装
  目前ccache遵循GPL-3.0在github上开源，并且处于开源软件的成长期，迭代较快，apt源并不能实时同步最新版本，可通过源码编译安装的方式，获取最新版本。
  	[github： https://github.com/ccache/ccache](https://github.com/ccache/ccache)


## ccache的使用方式
* ccache 拉起编译命令
```shell
    ccache g++ -c hello.cpp -o hello.o
```
* ccache伪装成编译器
```shell
    # 找到ccache的安装路径
    which ccache 
    # 安装实际路径为： /usr/bin/ccache 
    # 创建软连接，将ccache伪装成g++编译器
    ln -s /usr/bin/ccache g++
    # 使用伪装后的g++ 进行编译
    ./g++ -c hello.cpp -o hello.o
```
可是问题来了
>在实际工程当中，可能是由Makefile或者CMake进行管理编译，不可能手动修改所有的编译命令，我们应该将ccache使用上？ 
 
最为常用的方式是：
无论是在Makefile还是Cmake都可以通过声明环境变量的方式来使用ccache

```shell
# 在构建脚本中增加
export CC="ccache gcc"
export CXX="ccache g++"
# 再进行编译make/cmake 
make 
# cmake: 
cmake -B output && cmake --build output
```
* make

对于Makefile工程，我们常会通过会在Makefile中，通过CC/CXX指定编译器，因此在编译前通过配置缓存变量CC/CXX来指定要是用编译器即可。

![makefile中使用](./pic/1-%E5%85%A5%E9%97%A8%E4%BD%BF%E7%94%A8/Makefile%E7%BC%96%E8%AF%91%E4%BD%9C%E4%B8%BACXX.jpg)

* cmake
  
 Cmake隐式使用CC和CXX两个缓存变量，通过export 配置CC/CXX一样能使ccache生效。

![cmake中使用](./pic/1-%E5%85%A5%E9%97%A8%E4%BD%BF%E7%94%A8/Cmake%E4%B8%AD%E4%BD%9C%E4%B8%BACXX.jpg)

# ccache使用情况
如上在使用ccache跑一个CMake工程的时候，我们如果不注意标准输出内容，就比较难判断本次编译是否使用了ccache。

> 有没有一种更好的方式来帮助我们判断当前工程有没有使用ccache？
 
ccache提供了查询缓存命中能力

执行 `ccache -s -v`

![查看最后编译时间](./pic/1-%E5%85%A5%E9%97%A8%E4%BD%BF%E7%94%A8/%E9%80%9A%E8%BF%87%E7%BB%9F%E8%AE%A1%E5%AE%9E%E9%99%85%E5%88%A4%E6%96%AD%E6%98%AF%E5%90%A6%E4%BD%BF%E7%94%A8ccacche.jpg)

可以详细的看到


* 缓存存放目录
Cache directory
* 缓存配置文件路径
Primary config
* 最近一次使用缓存的时间
Status updated
*  缓存总体命中的情况
 	- Hits表示命中缓存
 	- Misses表示未命中缓存，但是生成了缓存，下次可以命中
* 缓存目录使用大小
	Cache size

注意：
>ccache 会通过LRU自动清理较长未访问的缓存文件，使得缓存总大小小于Cache size。不要手动清理Cache directory目录,造成缓存丢失。


如果默认缓存目录会影响到你的正常使用磁盘，可通过 在/etc/profile 增加ccache配置缓存目录的环境变量，改变缓存目录并使其一直生效。
```shell
# 将ccache的缓存目录设置为了/usr1/zhou/ccache
vim /etc/profile
# 追加环境变量
export CCACHE_DIR=/usr1/zhou/ccache 
# 让环境变量生效
source /etc/profile
```

# 小结
如果你只是想简单了解并使用ccache， 通过上述的介绍，已经可以满足日常使用ccache帮助自己的工程更快的进行编译。

如果还想对ccache有更深的了解，在后续内容中会介绍ccache 其他好用的配置项功能以及ccache的源码实现。