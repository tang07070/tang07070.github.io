title: iOS持久化（一）：介绍
date: 2017-04-17 22:48:10
tags: [iOS, 持久化]
categories: iOS
---

iOS在本地持久化上很很多种方式可以选择，在使用上大体上分为以下三种：

+ File
+ CoreData
+ SQLite

这里先不展开说，后面会依次详细介绍，下面是一些前置基础知识。

## SandBox
在文件存储方面，相比其他操作系统，iOS、macOS的一大特色就是其沙盒机制，每个APP只能访问到自己APP的存储空间，其他APP的存储空间是访问不到的（iOS8后可以通过APP Groups来共享数据），苹果这么做的最主要原因：**安全**。恩，很符合苹果的理念。

![](https://developer.apple.com/library/content/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/art/ios_app_layout_2x.png)
<center> *图1：app 沙盒目录* </center >

## 目录结构
下面说的目录结构是指沙盒中的Data Container，有三个目录：

+ Documents
+ Library
+ tmp

每个目录都有每个目录的目的，别乱放就好。

### Documents
用来存储用户生成的文件，可以通过设置file sharing让用户可以读取，所以这里只放一些可以展示给用户的文件。

整个目录默认都**会**被iTunes和iCloud自动同步。

### Library
用来存储APP需要，但是不需要对用户暴露的文件，默认有两个子目录：*Application Support* 和 *Caches*，如果需要的话也可以新建子目录

+ Application Support
+ Caches

*Application Support* 用来存放APP生成的一些支持文件，如配置文件之类的，默认**会**被iTunes和iCloud自动同步

*Caches* 用来存放APP生成的缓存文件，如日志，视频缓存之类的，系统会在磁盘空间不足时删除这个目录的内容来释放空间，默认**不会**被iTunes和iCloud自动同步

### tmp
用来存储临时文件，使用完后记得自己删掉，在APP没有运行时系统也会定期清理这个目录，**不会**被iTunes和iCloud自动同步