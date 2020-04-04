# prometheus中如何分析监控数据

代码来源：vendor/github.com/prometheus/client_golang/prometheus/summary.go

正态分布的窗口统计：
```go
import "github.com/beorn7/perks/quantile"
for i := uint32(0); i < opts.AgeBuckets; i++ {
    s.streams = append(s.streams, s.newStream())
}

func (s *summary) newStream() *quantile.Stream {
	return quantile.NewTargeted(s.objectives)
}
```
