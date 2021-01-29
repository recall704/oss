# MySQL的MaxIdleConns不合理，会变成短连接
## 1 背景

最近石墨文档线上业务出现了一些性能问题，在突发流量情况下，有个业务性能急剧下降。该服务是依赖于数据库的业务，会批量获取数据库里的数据。在经过一系列的排查过程后，发现该服务到数据库的连接数经常超过 MaxIdleConns，因此怀疑是数据库的配置导致的性能问题，所以以下针对数据库的代码进行了剖析，并做了相关实验。

## 2 配置解读

```cs
maxIdleCount      int                    
maxOpen           int                    
maxLifetime       time.Duration          
maxIdleTime       time.Duration          
```

可以看到以上四个配置，是我们 Go MySQL 客户端最重要的配置。

-   maxIdleCount 最大空闲连接数，默认不配置，是 2 个最大空闲连接
-   maxOpen 最大连接数，默认不配置，是不限制最大连接数
-   maxLifetime 连接最大存活时间
-   maxIdleTime 空闲连接最大存活时间

## 3 源码解析

我们的场景是客户端与 MySQL 建立的连接数经常大于最大空闲连接数，这会导致什么问题？我们看下下图中的源码。

![](https://mmbiz.qpic.cn/mmbiz_png/vbERicIdYZbCBKol8gGV4cA6RMnetxaXCwho95VhibV5LcakyujSUoROKCsRicXYz7ibQ8JchV60p7CHHr2CpL6pBA/640?wx_fmt=png)

我们可以看到，当最大空闲连接数小于客户端与数据库建立的连接数的时候，那么就会返回 false，并且最大连接数关闭计数器加 1。

![](https://mmbiz.qpic.cn/mmbiz_png/vbERicIdYZbCBKol8gGV4cA6RMnetxaXCBNd9kFYyPjLXZLGiaGknLbqnGIh2wy2Qfhe5YbwbibACUltdVn7CBbgQ/640?wx_fmt=png)

然后上图中，我们就可以看到，连接被关闭了（MySQL 源码里也不留点缓冲时间再关闭）。Go 的 MySQL 客户端这个操作，就会导致当突发流量情况下，由于请求量级过大，超过了最大空闲连接数的负载，那么新的连接在放入连接池的时候，会被无情的关闭，变成短连接，导致你的服务性能进一步恶化。

## 4 实验

### 4.1 模拟线上并发数大于 MaxIdConns 情况

测试代码 , 为了检测以上逻辑，假设了以下场景，设置最大连接数为 100，最大空闲连接数为 1，并发数为 10 的 goroutine 来请求数据库。我们通过 MySQL 的 stats 的 maxIdleClosed 的统计，可以看到下图，我们的连接不停的被关闭。

![](https://mmbiz.qpic.cn/mmbiz_png/vbERicIdYZbCBKol8gGV4cA6RMnetxaXCakKGctxNZVIZicyp6yERGrzXB4UGib6W8cXric5WEL2ZQ6mVoTgCVt9mA/640?wx_fmt=png)

### 4.2 模拟线上并发数小于 MaxIdConns 情况

测试代码 ，假设了以下场景，设置最大连接数为 100，最大空闲连接数为 20，并发数为 10 的 goroutine 来请求数据库，可以看到下图中，无 MaxIdleClosed 的关闭统计。

![](https://mmbiz.qpic.cn/mmbiz_png/vbERicIdYZbCBKol8gGV4cA6RMnetxaXCvfDBfKWib8AezHPJobaMh4qMNLPa7JTic22v2mU2icPuD4Ksib4lzAHfOg/640?wx_fmt=png)

### 4.3 抓包验证线上并发数大于 MaxIdConns 情况

测试代码 ，为了验证没有理解错代码，抓个包最稳妥。我们将 main 函数里放个 select{}，程序执行完 mysql 的语句后，看下 tcp 状态和抓包数据。

![](https://mmbiz.qpic.cn/mmbiz_png/vbERicIdYZbCBKol8gGV4cA6RMnetxaXCL0OdBbic46nNQicJvdRMZ8P37n9gogHYf7F3sH5z0qExTpTf6Flx8K4w/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/vbERicIdYZbCBKol8gGV4cA6RMnetxaXCSeVwFRymTj1ia652dHR7Rbt6mpnzmKKFyVU77SoHh8UnTv05w8XauZg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/vbERicIdYZbCBKol8gGV4cA6RMnetxaXCFdPevltWtsGiaZ931ZfBxWcichklCQx74smH3YAibwkaicCV6iblkbRzibvg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/vbERicIdYZbCBKol8gGV4cA6RMnetxaXCx6IpVDGIQicDezO3Dk46yOFsTA2XLYCHhcQiaJcBEx7ibDkpicW47Y3M0Q/640?wx_fmt=png)

可以发现确实是 tcp 的状态统计与 MySQL 客户端的统计是一致的，并且存在 fin 包。

## 5 总结

当突发流量情况下，由于请求量级过大，超过了最大空闲连接数的负载，那么新的连接在放入连接池的时候，会被关闭，将连接变成短连接，导致服务性能进一步恶化。为了避免这种情况，下面列举了，可以优化的措施。

-   提前将 maxIdleConns 设大，避免出现短连接
-   做好 mysql 读写分离
-   提升 mysql 的吞吐量：精简返回字段，没必要的字段不要返回，能够够快复用连接
-   吞吐量的包尽量不要太大，避免分包
-   优化连接池，当客户端到 MySQL 的连接数大于最大空闲连接的时候，关闭能够做一下延迟（官方不支持，估计只能自己实现）
-   读请求的最好不要放 MySQL 里，尽量放 redis 里

## 6 测试代码

-   [https://github.com/gotomicro/test/tree/main/gorm](https://github.com/gotomicro/test/tree/main/gorm) 
    [https://mp.weixin.qq.com/s/zxlgnFkcEwaSDx5uJZO8Ig](https://mp.weixin.qq.com/s/zxlgnFkcEwaSDx5uJZO8Ig)
