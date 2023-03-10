代码时序图

时序图中主要列举重点节点函数名称，帮助更快速理解代码调用逻辑。
# CodeDetails
## main函数
ccache 首先对于当前输入参数进行解析，如果是ccache的命令行命令 形如
```
ccache -V // 显示ccache 版本
ccache -s // 显示统计信息
```
直接执行对应操作后退出。

而对于 `ccache g++ -c hello.c -o hello.o`等实际编译命令操作，则进行缓存操作处理。

![main](./images/01-main.png)

ccache_compilation即实际缓存流程,先对整体流程有一个了解。
![ccache整体流程](../usage/pic/2-原理介绍/ccache编译流程.png)
## 初始化
如果是编译命令
ccache先进行初始化操作，初始化内容有 读取配置文件、日志模块初始化、本地/远程仓库初始化以及找到实际编译器
> 找到实际编译是指，对于ccache的两种使用方式的 ccache g++ 以及ccache软链接形式的g++的转换，此时需要从环境变量中找到实际的g++编译器

![初始化](./images/02-%E5%88%9D%E5%A7%8B%E5%8C%96.png)

## 生成Manifest

初始化后，ccache对cwd、配置选项、命令输入参数、编译器、输入源文件(.cpp)...等进行hash。
最终得到一个hash值即manifest路径，并对其进行查询本地是否存在。

![生成manifest](./images/03-%E7%94%9F%E6%88%90Manifest.png)

> 由于时序图仅标注重点流程，实际在获取Manifest的过程中还有一些本地、远程缓存保存细节。

如下流程图

![获取manifest](./images/%E8%8E%B7%E5%8F%96%E7%BC%93%E5%AD%98%E9%80%BB%E8%BE%91.png)

在获取不同类型的远端缓存通过抽象工厂的方式实现，其类图关系如下：

![stroage类图](./images/storage类图.png)

## 直连命中

上述流程走到了从本地获取Manifest路径，此时返回有两种情况 获取/未获取到。

对于获取到Manifest文件,且本地仓库存在其result文件，其获取时序图如下

![直连命中时序图](./images/04-%E7%9B%B4%E8%BF%9E%E5%91%BD%E4%B8%AD%E6%97%B6%E5%BA%8F%E5%9B%BE.png)


## 预处理命中

而对于上述生成Manifest路径后，对于以下三种场景会使用预处理器进行预处理编译

- 关闭直连命中，只开启预处理命中;
- 通过查询缓存路径，本地不存在Manifest;
- 通过查询缓存路径，本地存在Manifest文件，但是在result_matches中校验头文件失败(**头文件存在修改**)，没找到适合的result（实际走到miss）。

其代码时序图如下：

![预处理命中时序图](./images/05-%E9%A2%84%E5%A4%84%E7%90%86%E5%91%BD%E4%B8%AD%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

## Miss

对上述的场景3，从本地无法获取到Result，则需要重新归档Manifest文件及Result文件
![Miss时序图](./images/06-Miss%E5%91%BD%E4%B8%AD%E6%97%B6%E5%BA%8F%E5%9B%BE.png)


