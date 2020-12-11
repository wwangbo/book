# 一次pprof线上问题分析

先来介绍一下go tool pprof: 两种方式: 1.runtime/pprof 2.net/http/pprof 使用方式，以net/http/pprof为例:

![](.gitbook/assets/image%20%2813%29.png)

cpu分析:go tool pprof -http=:8081二话不说，\`go tool pprof -http=:8081\`profile\_10.0.74.90\_20201208-010137.pb.gz\`,打开pprof的ui。CPU分析:go tool pprof -http=:32000 http://127.0.0.1:6060/debug/pprof/profile?second=10s内存分析:go tool pprof -http=:32000 [http://127.0.0.1:6060/debug/heap](http://127.0.0.1:6060/debug/heap)Top:

- Flat：函数自身运行耗时

- Flat%: 函数自身耗时比例

- Sum%: 累计耗时比例

- Cum: 函数自身+其调用函数耗时

- Cum%: 函数自身+其调用函数耗时比例

- Name：函数名

![](.gitbook/assets/image%20%288%29.png)

Source: 在Top中选中1-n行，点击view-source可以查看对应代码位置，以Syscall为例:

![](.gitbook/assets/image%20%284%29.png)

Graph： Graph呈现出程序函数的调用树，方框越大代表函数自身消耗越大。

![](.gitbook/assets/image%20%2815%29.png)

火焰图:

![](.gitbook/assets/image%20%2814%29.png)

x轴显示的是在该性能指标分析中所占用的资源量，横向越宽，占用资源越多。 y轴代表调用栈，每一层都是一个函数，调用栈越深，火焰图就越高。 内存分析： 可视化分配的字节数或分配的对象数 alloc\_objects:收集自程序启动以来，累计的分配对象数 alloc\_space:收集自程序启动以来，累计分配的空间 inuse\_objects:收集统计实时，正在使用的分配对象数 inuse\_space：收集统计实时，正在使用的分配空间。

![](.gitbook/assets/image%20%285%29.png)

下面分析一下遇到的问题： 某天收到了报警，好家伙，CPU利用率百分之90多。

![](.gitbook/assets/image%20%289%29.png)

首先看一下Graph:

![](.gitbook/assets/image%20%2810%29.png)

其实在这个图里大体上就猜出来了，应该是unmarshal的问题。 接下来看看我们的火焰图：

![](.gitbook/assets/image%20%2811%29.png)

我们知道，火焰图中x轴表示CPU耗时，越宽代表占用越多，y轴代表函数栈深度，尖刺越高代表调用栈越深。从图中很容易看出来火焰图中显示是Umarshal吃了CPU。 下面来看看top图：

![](.gitbook/assets/image.png)

从top图中可以看待代码的位置，点击上面的列名可以排序跟着火焰图的调用栈或者top图或者Graph,很容易定位到具体的代码。

我们这里的问题是，把用户的permission全部塞到了context里面，这一块业务比较复杂，需要多次调用数据库服务（目前CRUD还是单独的服务），调用之前需要进行permission检查，这块业务需要掉比较多次数据库服务。所以会有频繁的permission的umarshal,umarshal的结果存到一个map里面，从图中我们也可以看到malloc gc，umarshal本身就是一个很吃cpu的操作，在umarshal的过程中map还会疯狂扩容、gc，就造成了High CPU的问题。（这一块本来是有缓存的，但是后面写这块代码同事估计不知道，所以去unmarshal的）

第二天，感到办公室，CPU又高了，经理让我查一下是不是和昨天一样的问题。用类似的方法查了一下是我们图片处理的时候，decode和encode的时候造成的问题，查起来非常简单。不得不说，pprof真的是神器啊！

