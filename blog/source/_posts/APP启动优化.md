---
title: APP启动优化
date: 2019-04-09 15:35:38
tags: 学习
categories: iOS
---

### APP启动

> 冷启动：App 点击启动前，它的进程不在系统里，需要系统新创建一个进程，需要系统新创建一个进程分配给他。一次完整的启动过程；
>
> 热启动：app在后台，进程还在系统中。



------



### 冷启动优化

app启动3阶段：

> 1. main()函数执行前；
> 2. main()函数执行后；
> 3. 首屏渲染完成后。

#### main()函数执行前

系统操作：

- 加载可执行文件（.0文件），动态连接器首先会检查共享缓存看看是否存在，如果存在，就直接从共享缓存中拿出来使用。每一个进程都把这个共享缓存映射到自己的地址空间中。

- 加载动态链接库，进行rebase指针调整和bind符号绑定

  > 动态度的加载过程：
  >
  > ①分析所依赖的动态库
  > ②找到动态库的mach-o文件
  > ③打开文件
  > ④验证文件
  > ⑤在系统核心注册文件签名
  > ⑥对动态库的每一个segment调用mmap()

- Objc运行时初始处理（相关类注册、category注册、selector唯一性检查）

- 初始化，执行+load()方法、attribute修饰的函数的调用、创建c++静态全局变量

优化：

- 减少动态库加载，尽量将多个动态库合并
- 减少加载启动后不会去使用的类或者方法
- 减少使用+load()方法，使用+initialize()方法替换。在一个 +load() 方法里，进行运行时方法替换操作会带来4毫秒的消耗
- 控制c++全局变量的数量



> 通过在工程的scheme中添加环境变量`DYLD_PRINT_STATISTICS`，设置值为1，App启动加载时Xcode的控制台就会有pre-main各个阶段的详细耗时输出。但是`DYLD_PRINT_STATISTICS`变量打印时间是iOS10以后才支持的功能，所以需要用iOS10系统及以上的机器来做测试。

#### mian()函数执行

![img](http://www.zoomfeng.com/images/2018/07/5.png)

#### mian()函数执行后

mian()函数执行-->didFinishLaunchingWithOptions 方法里首屏渲染相关方法执行完成

执行：

> - 首屏初始化所需配置文件的读写操作
> - 首屏数据读取
> - 首屏渲染的大量计算

优化：

> didFinishLaunchingWithOptions方法里面制作必要的操作。其它的操作放到对应的功能去初始化

#### 首屏渲染完成后

首屏渲染完成-->didFinishLaunchingWithOptions方法结束

执行：

> 非首屏其它业务模块的初始化、监听的注册、配置文件的读取等



#### 功能级别的启动优化

![img](https://static001.geekbang.org/resource/image/f3/19/f30f438d447e81132dd520e657427419.png)



#### 方法级别的启动优化

objc_msgSend ：在运行时根据对象和方法的selector去找到对应的函数指针，然后执行。能够控制所有的oc方法。

[源码](<https://opensource.apple.com/source/objc4/objc4-723/runtime/Messengers.subproj/>)

> objc_msgSend逻辑：先获取对象对应类的信息，再获取方法的缓存，根据方法的selector查找函数指针，经过异常错误处理后，最后跳到对应函数的实现。



资料：

> [来源](<https://time.geekbang.org/column/article/85331?code=0eTznNzpAbVisw%252525252FesJ9iM32u2ctcY8OqwgMuqSlv5OE%252525253D>)
>
> [iOS启动时间优化](<http://www.zoomfeng.com/blog/launch-time.html>)
>
> [汇编](<http://www.ruanyifeng.com/blog/2018/01/assembly-language-primer.html>)

