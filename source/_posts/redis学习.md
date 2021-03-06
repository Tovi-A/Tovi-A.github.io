---
title: redis学习
date: 2019-04-18 10:06:37
tags:
- redis
categories: 数据库
comments: true
mathjax: true
---
# RDB持久化
将服务器中的非空数据库以及它们的键值对统称为数据库状态。redis是内存数据库，它将自己的数据库状态储存在内存里面，所以如果不想办法将储存在内存中的数据库状态保存到磁盘里面，那么一旦服务器进程退出，服务器中的数据库状态也会消失不见。由此，redis提供了RDB持久化功能，它可以将redis在内存中的数据库状态保存到磁盘里面，避免数据意外丢失。   
RDB持久化功能所生成的RDB文件是一个经过压缩的二进制文件，通过该文件可以还原数据库状态，如图所示：
![image](https://media.githubusercontent.com/media/Tovi-A/tovi-a.github.io/hexo/Additional_Resources/redis_study/1.png)
## RDB文件的创建与载入   
生成RDB文件命令：SAVE、BGSAVE     
- SAVE命令会阻塞redis服务器进程，直到RDB文件创建完毕为止，在阻塞期间，服务器不能处理任何命令请求。  
- GBSAVE命令会派生出一个子进程，然后由子进程负责创建RDB文件，服务器进程（父进程）继续处理命令请求。   
只要redis服务器在启动时候检测到RDB文件的存在，就会自动载入RDB文件。  
由于AOF文件的更新频率通常比RDB文件的更新频率高，所以：  
- 如果服务器有开启AOF持久化功能，那么会优先使用AOF文件来还原数据库状态。   
- 只有AOF持久化功能处于关闭状态时，服务器才会使用RDB文件来还原数据库状态。  
服务器判断该用哪个文件来还原数据库状态的流程以及rdbLoad与rdbSave函数之间的关系如图：  
![image](https://media.githubusercontent.com/media/Tovi-A/tovi-a.github.io/hexo/Additional_Resources/redis_study/2.png)
服务器只有在执行完SAVE命令之后，客户端发送的命令才会被处理。    
BGSAVE执行期间，redis服务器可以处理客户端的请求。   
- BGSAVE执行期间，客户端发送的SAVE命令会被服务器拒绝，服务器禁止SAVE与BGSAVE命令同时执行是为了避免父进程（服务器进程）和子进程同时执行两个rdbSave调用，防止产生竞争条件。  
- BGSAVE执行期间，客户端发送的BGSAVE会被拒绝，同时执行两个BGSAVE也会产生竞争条件。  
- BGREWRITEAOF和BGSAVE两个命令不能同时执行。    
服务器在载入RDB文件时候，一直处于阻塞状态，知道载入完成。
## 自动间隔性保存
SAVE由父进程执行保存工作，而BGSAVE由子进程执行保存工作。   
因为BGSAVE命令可以在不阻塞服务器进程的情况下执行，所以redis允许用户通过设置服务器配置save选项，让服务器每隔一段时间自动执行一次BGSAVE。   
### redis服务器是如何根据save选项设置的保存条件，自动执行BGSAVE命令？
dirty计数器和lastsave属性：
- dirty计数器的值为123，则表示服务器在上次保存之后，对数据库状态共进行了123次修改。   
- lastsave属性则记录了服务器上次执行保存操作的时间。   
![image](https://media.githubusercontent.com/media/Tovi-A/tovi-a.github.io/hexo/Additional_Resources/redis_study/3.png)
