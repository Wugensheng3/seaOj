# 功能展示
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/f7e8dfa0-4225-425e-8558-53185c4e4d82)
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/e7bfe953-0741-458c-84cd-34f849886184)
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/3b15c924-d53a-455e-8456-9f186f6e9a94)
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/56b548ca-424d-459d-9ee7-e80329544726)
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/4fccc738-c1d5-4f3d-99b1-f966926c4431)
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/8d583769-0c8b-482f-bad8-66b89a32b121)
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/46f3d036-e7bc-4c36-b6a9-486e56e2e2f5)
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/2cb49d69-d4d1-4aab-a0b3-d58b6c1e6eed)

![image](https://github.com/Wugensheng3/seaOj/assets/130813027/66bcb91f-739f-480f-8b7d-f53e5157d84f)


### 前端

使用的是acro design pro 以及使用到了calheatmap(这里同时感谢一位大佬的指点[yqchilde (Yqchilde) (github.com)](https://github.com/yqchilde))之后我就懒得特意弄个月份的提交记录,直接在大佬的组件上修改了

以及使用到了vue-image-cropper(一位playground图片是蕾姆的大佬)

### 系统设计上(后端)

主要在代码提交以及运行上做了于自身有所成长的设计

#### 考虑到的问题如下:

##### 1. 运行代码相对时间比较长,要保持长连接吗?让前端一直阻塞等待结果?

可以但是没必要,类似的设计例如说百度网盘的扫码(**长轮询(超时时间设的很长)**),手机一扫码,电脑马上刷新,但是他们这么设计的情况下是用户扫码的时间不会特别长,而我们运行代码可未必,所以否决

归根到底我们需要实现的效果就是类似一个服务端往客户端推送的效果,tcp本来就是全双工的,只不过http在上层不采取这个方式罢了

那么websocket可以吗?不行,相对而言,websocket建立可比http消耗的资源多,我们运行代码通常是两次(1. 运行,2.运行成功后提交,如果失败的话,通常会修改代码之后再运行),websocket通常是用在网页游戏上频繁交互上,才使用的

> 顺便提一下websocket和http的区别,相当于是升级,正如**header 头**中的
>
> ```http
> Connection: Upgrade
> Upgrade: WebSocket
> Sec-WebSocket-Key: T2a6wZlAwhgQNqruZ2YUyg==\r\n
> 
> ```
>
> **（Sec-WebSocket-Key）是随机生成的 base64 码**,之后的流程就是服务器方和浏览器方根据相同的算法以及这个base64码来进行算出一个str,如果相等则建立(相当于对暗号),所以显而易见websocket的建立需要2次http

所以我们最终采用短轮询的方式

##### 2.后端与代码沙箱的交互是怎么样的?

这里有点失误,原本我以为代码沙箱可能不简单,但是一叶障目了,之后发现很容器,所以导致之前设计时,考察到了远程沙箱,所以代码中还有远程沙箱,也就导致说我并没有采用rabbitmq彻彻底底的worker模型

大致是这样的

**运行**
![image](https://github.com/Wugensheng3/seaOj/assets/130813027/0407517d-02b1-48d7-b803-9b4a19a0fde6)


**提交**

![![image](https://github.com/Wugensheng3/seaOj/assets/130813027/c101c013-31e3-42be-9031-3234f1631d03)


lock对应的方法其实就是限制一个用户只能有一个待执行的代码或者正在执行中的代码

然后根据是commit和run分别转到不同的rabbitmq的queue中

然后使用work模型来读取,不过读取出来后是交给线程池来处理

由于这里主要是网络IO密集型所以,在这里线程池数量设计为cpu核心*2+1

然后在得到判题结果后,这里使用了策略以及工厂以及模板设计模式

给每个部分进行分层

![image](https://github.com/Wugensheng3/seaOj/assets/130813027/97ef6a46-e909-4b28-bd51-a52dd2d7b402)


大致是这样的,这里是模板的部分,由于下面的endchain需要注入所以于是有了工厂,而且还要考虑到不同的语言例如说sql或者其他如c,或java这类的

例如说sql题目:好比说查询表格,那么可能我们第一步就不同,我们需要先建表以及准备对应的数据

![image](https://github.com/Wugensheng3/seaOj/assets/130813027/1bb8d6d7-3684-4181-bb8e-91833614597d)


工厂方法这里利用的是spring的ApplicationContext来注入

##### 3.点赞和评论系统要怎么设计

原定是各个模块都要有点赞评论的,所以打算做成一个中台,可以最后又碍于时间不足暑假实习招聘时间快截止了,所以在各模块直接分表设计

例如说评论目前是题解有对应的评论

不过也是类似中台的设计了,把点赞功能上集成到redis中,通过业务key带上各个模块来进行划分,防止ID重复thumb_count,以及做了用户是否点赞的统计(点赞模块是没有数据库表的,基本上是基于redis的)

然后评论上的数据库表设计参考了b站的设计,同时我加上了刷屏的限制(基于redis的zset),以及敏感词处理ac自动机

