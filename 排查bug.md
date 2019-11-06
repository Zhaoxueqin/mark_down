前因

CouponBirds codes 页面功能大幅修改, guozhao 进行全站的404 检测, 发现有很大部分 url 无法连接,返回异常 如下图
![image-20180724110932055](https://wx3.sinaimg.cn/mw1024/5faa347dly1ftkrw8qnzij21kw045jxv.jpg)


Scrapy 快速的三次retry 之后give up   然后 接着就是一堆url 都出现这个问题.



排查

浏览器挂上代理能访问,但是本地IP 无法访问.

看Nginx log  日志如下

![image-20180724112232078](https://wx2.sinaimg.cn/mw1024/5faa347dly1ftkscmztozj21kw05f0yp.jpg)

发现 有状态码499 但是,只有 访问一次这个url 会出现俩三个499 的response,然后一段时间内, 整个站就都不能访问了,没有反应了, Nginx 日志里面也没有记录到任何东西了, 说明数据没有到Nginx  应该是Tcp   层的网络有问题,

然后 用tcpdump+Wireshark 抓包检查果不其然发现

    # tcpdump 抓包 , 用Wirsshark 分析
    sudo tcpdump tcp -i eth0 -t -s 0 -c 1000 and dst port ! 22 -w ./target.cap

结果如下图:
![image-20180724111544243](https://wx4.sinaimg.cn/mw1024/5faa347dly1ftks5juaqkj21kw0nrqod.jpg)

出现了很多的RST 导致TCP连接中断, 仔细看, 发现 里面的ACK 完全和上一个包的Seq 对不上, 我们客服端的ACK 的是一个巨大的随机数. 导致服务器端返回RST .

访问咱们站正常的抓包情况 如下图:

![image-20180724112639491](https://wx3.sinaimg.cn/mw1024/5faa347dly1ftksgy8m6mj21kw0mlhak.jpg)



因为一直好奇讲道理,项目是HTTP应用层的 不会影响到TCP层,所以,Google 关键字  tcp reset + blogspot.com 看到reddit 一篇8年前的讨论. 讲到了 关于中国防火墙.  中国GFW会有根据tcp协议里面的关键字来进行屏蔽.然后会通过reset修改 来屏蔽整个站几分钟.

https 协议的不会

测试了俩个国外的网站

http://site.aace.org/conf/blogspot.com

http://www.motogp.com/blogspot.com

加入了 blogspot.com 就无法访问了.



结论

所以问题找到了 中国GFW 会通过url 里面的敏感字进行封锁网站 有篇具体分析的文章 . 线上因为用的https   协议所以内容加密了, 没有被block , 解决方法:

1. 将测试服务器换成https 然后就可以访问了.
2. 通过代理访问.






