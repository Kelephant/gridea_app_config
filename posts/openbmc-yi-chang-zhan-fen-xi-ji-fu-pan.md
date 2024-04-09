---
title: 'openbmc 异常栈分析及复盘'
date: 2024-04-09 22:53:33
tags: [openbmc,coredump,debug]
published: true
hideInList: false
feature: 
isTop: false
---
Openbmc ipmi在运行之后，出现段错误
<!-- more -->

# 背景

Openbmc ipmi在运行之后，出现段错误，段错误的dump信息如下

通过堆栈可以看到异常在openssl库里面，再往上追踪已经没有栈了

# 分析

Openssl库正常是不会有问题的，可以先暂时先排除。再往上就是栈被破坏了，谁调用EVP_Q_mac不知道，分析到这里就懒得再往下追踪了。后续通过替换openbmc版本，ipmi版本尝试解决，做了很多无用功。最面，通过分析github上的log才找到真正的问题。如果当初硬着头皮，往下想想，或许能更快发现问题，该反思反思。

1. 谁调用了EVP_Q_mac？
    1. Ipmi并没有直接调用该函数，那就是间接调用，如果库没有带符号表，那问题复杂度更高了。
        
        **（1）没有符号表**
        
        1. **如果是直接调用，我们直接打印参数内容**
        2. **间接调用，需要找到EVP_Q_mac的各种引用函数，再一个个追踪引用函数**
        
        **（2）有符号表：通过设置断点，在栈被破坏前，把函数停住，把栈先打出来，这样就清楚异常的具体位置了。**
        
        1. 在异常的位置设置一个断点
        2. run
        3. 在运行到断点的时候自动停止，执行bt，通过这个方式就能看到异常的点在read password了
        ![](https://kelephant.github.io/gridea/post-images/1712674496787.png)

    2. 栈为什么被破坏？被破坏的栈有没有办法恢复，或者调试手段防止栈被破坏？
        1. Gdb在执行时是否能加参数，保护栈不被破坏？似乎没有找到解决办法
        2. Gcc编译参数，参考下面的文章：https://outflux.net/blog/archives/2014/01/27/fstack-protector-strong/

# 64位系统地址翻译

1. 找到异常地址对应的段（/proc/{pid}/maps）
2. 将地址与段的起始地址，作差，即可得到异常的位置
3. 如果是.so文件，通过objdump -d libcrypto.so.3 | grep 1b02a0(这个是作差后的值)
4. 如果是可执行文件，通过addr2line即可

# 其他

## 动态库加载情况

info sharedlibrary

## 所有线程的bt

thread apply all bt

使用该办法可以定位死锁问题