# 介绍
ccache为Github 上1.7K Star的开源项目。

* [Ccache官方链接](https://ccache.dev/)

* [Github链接](https://github.com/ccache/ccache/)

内容c++ 14实现，技术栈涉及设计模式、Redis、Http使用。

用于提升C++工程的编译速度，对于一次提交仅有少量修改的话，能压缩编译用时大概10倍左右。

内容分两个部分
- [x] 如何使用
- [x] 源码剖析
  
# 如何使用
  
从[介绍与入门使用（点击查看）](./usage/01-ccache快速使用.md)、
    [缓存原理（点击查看）](./usage/02-ccache原理介绍.md)、
    [高级使用（点击查看）](./usage/03-ccache的进阶使用.md)都进行介绍，花费20分阅读即可对c++工程编译进行10倍的提速！
# 源码剖析
  
通过绘制代码时序图深度分析ccache的代码实现，学习优质c++代码设计。

[代码时序图（点击查看）](./codeDetails/README.md)
> 本仓库为个人记录学习ccache，所有内容都为原创。
