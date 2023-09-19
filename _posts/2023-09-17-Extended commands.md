---
layout: post
title: 扩展命令
subtitle: 意外发现的命令
categories: markdown
tags: [ Linux ]
---

## mkpasswd 命令生成随机复杂密码

mkpasswd命令生成随机复杂密码，前提安装expect，然后执行mkpasswd命令即可生成随机的密码。

一、基本的命令安装
安装expect：   yum install -y expect


    -l #      (密码的长度定义, 默认是 9)
    -d #      (数字个数, 默认是 2)
    -c #      (小写字符, 默认是 3)
    -C #      (大写字符, 默认是 2)
    -s #      (特殊字符, 默认是  1)
    -v        (详细。。。)
    -p prog   (程序设置密码, 默认是 passwd)

详细参数，用如下命令查看：

创建了一个长度为20位,包括数字个数5个，包含小写字母个数5个，包含大写字母个数5个，包含特殊符号个数5个。

```
mkpasswd  -l 20 -d 5 -c 5 -C 5 -s 5 
Z}K7hp0UPJ6v@&,c5{d3
```

随机密码最短只能7位（两个数字，两个小写字母，两个大写字母，一个特殊符号）

```
mkpasswd -l 5
impossible to generate 5-character password with 2 numbers, 2 lowercase letters, 2 uppercase letters and 1 special characters.

```



## bc 计算器

bc 命令是任意精度计算器语言，通常在linux下当计算器用。它类似基本的计算器, 使用这个计算器可以做基本的数学运算。

- -i：强制进入交互式模式；
- -l：定义使用的标准数学库
- ； -w：对POSIX bc的扩展给出警告信息；
- -q：不打印正常的GNU bc环境信息；
- -v：显示指令版本信息；
- -h：显示指令的帮助信息。

+ quit： 退出

**通过管道符**

```
$ echo "15+5" | bc
20
```

scale=2 设小数位，2 代表保留两位:

```
$ echo 'scale=2; (2.777 - 1.4744)/1' | bc
1.30
```

bc 除了 scale 来设定小数位之外，还有 ibase 和 obase 来其它进制的运算:

```
$ echo "ibase=2;111" |bc
7
```

**进制转换**

```
#!/bin/bash

abc=192 
echo "obase=2;$abc" | bc
```

执行结果为：11000000，这是用bc将十进制转换成二进制。

```
#!/bin/bash 

abc=11000000 
echo "obase=10;ibase=2;$abc" | bc
```

执行结果为：192，这是用bc将二进制转换为十进制。

计算平方和平方根：

```
$ echo "10^10" | bc 
10000000000
$ echo "sqrt(100)" | bc
10
```


## ack 文本搜索工具

比grep好用的文本搜索工具

[原链接](https://jaywcjlove.gitee.io/linux-command/c/ack.html)

[官网](https://beyondgrep.com/)

**安装**

```shell
# ubuntu下要安装ack-grep，因为在debian系中，ack这个名字被其他的软件占用了。
sudo apt-get install ack-grep

# alpine Linux-apk软件包管理器 安装 ack
apk install ack

# Centos7 要准备好epel源
yum install -y ack.noarch
```

**参数**

这些参数在linux上的适用频率是相当高的，尤其是你用vim做为IDE的话

```shell
-n, 显示行号
-l, 显示匹配的文件名
-L, 显示不匹配的文件名
-c, 统计次数
-w, 词匹配
-i, 忽略大小写
-f, 只显示文件名,不进行搜索.
-h, 不显示名称
-v, 显示不匹配
```

**特点**

ack官网列出了这工具的5大卖点：

1. 速度非常快,因为它只搜索有意义的东西。
2. 更友好的搜索，忽略那些不是你源码的东西。
3. 为源代码搜索而设计，用更少的击键完成任务。
4. 非常轻便，移植性好。
5. 免费且开源

**实例**

grep常用操作

```shell
grep -r 'hello_world' # 简单用法
grep '^hello_world' . # 简单正则
ls -l | grep .py # 管道用法
```

**搜索**

简单的文本搜索，默认是递归的。

```
ack hello
ack -i hello
ack -v hello
ack -w hello
ack -Q 'hello*'
```

**搜索文件**

对搜索结果进行处理，比如只显示一个文件的一个匹配项，或者xxx

```shell
ack --line=1       # 输出当前目录下所有文件第一行
ack -l 'hello'     # 包含查询内容的文件名
ack -L 'hello'     # 非包含查询内容的文件名
```

**文件演示**

输出的结果是以什么方式展示呢，这个部分有几个参数可以练习下

```shell
ack hello --pager='less -R'    # 以less形式展示
ack hello --noheading      # 不在头上显示文件
ack hello --nocolor        # 不对匹配字符着色
```

**文件查找**

没错，它可以查找文件，以省去你要不断的结合find和grep的麻烦，虽然在linux的思想是一个工具做好一件事。

```shell
ack -f hello.py     # 查找全匹配文件
ack -g hello.py$    # 查找正则匹配文件
ack -g hello  --sort-files     # 查找然后排序
```

**文件包含/排除**

文件过滤，个人觉得这是一个很不错的功能。如果你曾经在搜索项目源码是不小心命中日志中的某个关键字的话，你会觉得这个有用。

```shell
ack --python hello       # 查找所有python文件
ack -G hello.py$ hello   # 查找匹配正则的文件
```

**ack支持的文件类型**

```shell
ack --help-types
```

**更多扩搜索展**

[知乎上找的]



## pigz  解压缩文件

可以用来解压缩文件，gzip的并行实现升级版

**补充说明**

**pigz命令**可以用来解压缩文件，最重要的是支持多线程并行处理，解压缩比gzip快。主页: http://zlib.net/pigz/

**语法**

```shell
pigz [ -cdfhikKlLmMnNqrRtz0..9,11 ] [ -b blocksize ] [ -p threads ] [ -S suffix ] [ name ...  ]
unpigz [ -cfhikKlLmMnNqrRtz ] [ -b blocksize ] [ -p threads ] [ -S suffix ] [ name ...  ]
```

**参数**

```shell
-0 to -9, -11       # Compression level (level 11, zopfli, is much slower)
--fast, --best      # Compression levels 1 and 9 respectively
-b, --blocksize mmm # Set compression block size to mmmK (default 128K)
-c, --stdout        # Write all processed output to stdout (won't delete)
-d, --decompress    # Decompress the compressed input
-f, --force         # Force overwrite, compress .gz, links, and to terminal
-F  --first         # Do iterations first, before block split for -11
-h, --help          # Display a help screen and quit
-i, --independent   # Compress blocks independently for damage recovery
-I, --iterations n  # Number of iterations for -11 optimization
-J, --maxsplits n   # Maximum number of split blocks for -11
-k, --keep          # Do not delete original file after processing
-K, --zip           # Compress to PKWare zip (.zip) single entry format
-l, --list          # List the contents of the compressed input
-L, --license       # Display the pigz license and quit
-m, --no-time       # Do not store or restore mod time
-M, --time          # Store or restore mod time
-n, --no-name       # Do not store or restore file name or mod time
-N, --name          # Store or restore file name and mod time
-O  --oneblock      # Do not split into smaller blocks for -11
-p, --processes n   # Allow up to n compression threads (default is the number of online processors, or 8 if unknown)
-q, --quiet         # Print no messages, even on error
-r, --recursive     # Process the contents of all subdirectories
-R, --rsyncable     # Input-determined block locations for rsync
-S, --suffix .sss   # Use suffix .sss instead of .gz (for compression)
-t, --test          # Test the integrity of the compressed input
-v, --verbose       # Provide more verbose output
-V  --version       # Show the version of pigz
-Y  --synchronous   # Force output file write to permanent storage
-z, --zlib          # Compress to zlib (.zz) instead of gzip format
--                  # All arguments after "--" are treated as files
```

**实例**

可以结合`tar`使用, 压缩命令

```shell
tar -cvf - dir1 dir2 dir3 | pigz -p 8 > output.tgz
```

解压命令

```shell
pigz -p 8 -d output.tgz
```

如果是gzip格式，也支持用tar解压

```shell
tar -xzvf output.tgz
```
