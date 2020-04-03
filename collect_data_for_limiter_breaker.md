# 熔断/限流相关数据的采集方式

- 限流中间件采集数据，用于限制内部调用的频率。主要通过 golang.org/x/time/rate 这个三方库实现，用到了三方库中的函数 func (*rate.Limiter).Allow() bool 。

- 熔断中间件通过 "/sys/fs/cgroup" 获取cpu信息，结合滚动窗口的请求失败率判断是否熔断。 滚动窗口的实现，复用prom的采集数据，采用 client 端的失败率数据。