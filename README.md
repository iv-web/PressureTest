## 压测工具
ab 

## windows 安装 ab 的方法 ##
1. 安装 [git bash](https://git-scm.com/downloads)  
2. 安装 [apache](https://httpd.apache.org/download.cgi)
3. $path 变量中 添加 ${apache安装的路径}\Apache24\bin;

运行 bash，执行 ab

## ab 压测命令 ##

- 压测命令

```
ab -H 'Accept-Encoding: gzip' -n 1000 -c 500 -l  'http://now.qq.com/h5/record.html?vid=1055_24965905613032497100000000000000&_bid=2336&_wv=16778241&from=timeline&now_n_r=1' 
```

测试动态页面时 必须带有 `-l` 选项，否则会出现因为 content-length 不一致而统计为请求失败
-n 1000 意义为 1000个请求  
-c 500 意义为 500个并发请求  
-H 为添加的头部，这里添加 gzip 头  

- 持续压测命令

while [ 1 ]
do
    ab -H 'Accept-Encoding: gzip' -n 1000 -c 500  'http://now.qq.com/h5/record.html?vid=1055_24965905613032497100000000000000&_bid=2336&_wv=16778241&from=timeline&now_n_r=1' 
    sleep 60
done

每分钟进行一次压测，适用于观察服务器是否存在内存泄露。

## 压测结果含义 ##

```
Document Path: /phpinfo.php
#测试的页面
Document Length: 50797 bytes
#页面大小

Concurrency Level: 1000
#测试的并发数
Time taken for tests: 11.846 seconds
#整个测试持续的时间
Complete requests: 4000
#完成的请求数量
Failed requests: 0
#失败的请求数量
Write errors: 0
Total transferred: 204586997 bytes
#整个过程中的网络传输量
HTML transferred: 203479961 bytes
#整个过程中的HTML内容传输量
Requests per second: 337.67 [#/sec] (mean)
#最重要的指标之一，相当于LR中的每秒事务数，后面括号中的mean表示这是一个平均值
Time per request: 2961.449 [ms] (mean)
#最重要的指标之二，相当于LR中的平均事务响应时间，后面括号中的mean表示这是一个平均值
Time per request: 2.961 [ms] (mean, across all concurrent requests)
#每个连接请求实际运行时间的平均值
Transfer rate: 16866.07 [Kbytes/sec] received
#平均每秒网络上的流量，可以帮助排除是否存在网络流量过大导致响应时间延长的问题
Connection Times (ms)
min mean[+/-sd] median max
Connect: 0 483 1773.5 11 9052
Processing: 2 556 1459.1 255 11763
Waiting: 1 515 1459.8 220 11756
Total: 139 1039 2296.6 275 11843
#网络上消耗的时间的分解，各项数据的具体算法还不是很清楚

Percentage of the requests served within a certain time (ms)
50% 275
66% 298
75% 328
80% 373
90% 3260
95% 9075
98% 9267
99% 11713
100% 11843 (longest request)
#整个场景中所有请求的响应情况。在场景中每个请求都有一个响应时间，其中50％的用户响应时间小于275毫秒，66％的用户响应时间小于298毫秒，最大的响应时间小于11843毫秒。对于并发请求，cpu实际上并不是同时处理的，而是按照每个请求获得的时间片逐个轮转处理的，所以基本上第一个Time per request时间约等于第二个Time per request时间乘以并发请求数。
```

## 压测关注点 ##
1. 并发请求上限  
  上限是指，超出该并发值后，服务器会出现响应失败的错误, 以统计结果中的 
  Failed requests 作为标准，并且计算 成功率在99.9% 和 99.99%的并发值

2. 固定响应时间内的最大并发数  
   如，所有请求都需要在600ms以内访问，并发量要在多少以内，Time per request 的值要小于 600ms 时，最大的并发数， 第二个 Time per request 代表一个并发事务所消耗的时间。

3. 绘制各个并发下的吞吐量 ，绘制曲线图，推算最大吞吐量的并发值。


4. 持续压测时，服务器相关的数据
    在 [moniter.server.com](moniter.server.com) 搜索服务器的 ip，查看相关视图。 如果视图 `应用程序使用内存数` ，在持续压测期间没有出现指数级增长，则说明不存在内存泄露。

    

## 会遇到的问题 ##

1. 并发量超过压测工具上限  
ab 并发测试的上线一般是 20000 左右， 而如果是node集群服务器的话，承载量往往会高于这个量，为了测集群的并发上限，
一般做法是先测某台单机（如果是pm2 的话，就不启用 cluster 模了）的并发上限，
然后乘以集群的数量。

2. tgw 防御 DDoS, 导致无法压测  
直接设置 host 为目标机器的内网 ip，绕过tgw。

3. 大量的request fail  content-length 类型错误  
命令中添加 -l 参数。