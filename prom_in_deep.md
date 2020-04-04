# prometheus监控中prom.Prom的原理

- 定义 Prom 结构体，和创建 Prom 结构体的函数。Prom 的 WithTimer 函数，依然返回 Prom 结构体。WithTimer 函数中调用了 prometheus.NewHistogramVec 和 prometheus.MustRegister。
- 代码0.0
```go
// Prom struct info
type Prom struct {
	timer   *prometheus.HistogramVec
	counter *prometheus.CounterVec
	state   *prometheus.GaugeVec
}

// New creates a Prom instance.
func New() *Prom {
	return &Prom{}
}

// LibClient for mc redis and db client.
LibClient = New().WithTimer("go_lib_client", []string{"method"}).WithState("go_lib_client_state", []string{"method", "name"}).WithCounter("go_lib_client_code", []string{"method", "code"})
// RPCClient rpc client
RPCClient = New().WithTimer("go_rpc_client", []string{"method"}).WithState("go_rpc_client_state", []string{"method", "name"}).WithCounter("go_rpc_client_code", []string{"method", "code"})

func (p *Prom) WithTimer(name string, labels []string) *Prom {
	if p == nil || p.timer != nil {
		return p
	}
	p.timer = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name: name,
			Help: name,
		}, labels)
	prometheus.MustRegister(p.timer)
	return p
}

var (
	hitProm  *prom.Prom
	missProm *prom.Prom
)

hitProm = prom.CacheHit
missProm = prom.CacheMiss
```

- HistogramVec 中包含 metricVec，间接包含 metricMap，从而也包含了 newMetric。newMetric 是一个函数，其返回结果是 Metric。newMetricVec 返回的结果也是 Metric。
- 代码0.0.0
```go
type HistogramVec struct {
	*metricVec
}

type metricVec struct {
	*metricMap

	curry []curriedLabelValue

	// hashAdd and hashAddByte can be replaced for testing collision handling.
	hashAdd     func(h uint64, s string) uint64
	hashAddByte func(h uint64, b byte) uint64
}

type metricMap struct {
	mtx       sync.RWMutex // Protects metrics.
	metrics   map[uint64][]metricWithLabelValues
	desc      *Desc
	newMetric func(labelValues ...string) Metric
}

func NewHistogramVec(opts HistogramOpts, labelNames []string) *HistogramVec {
	desc := NewDesc(
		BuildFQName(opts.Namespace, opts.Subsystem, opts.Name),
		opts.Help,
		labelNames,
		opts.ConstLabels,
	)
	return &HistogramVec{
		metricVec: newMetricVec(desc, func(lvs ...string) Metric {
			return newHistogram(desc, opts, lvs...)
		}),
	}
}
```

- newMetricVec 中包含的 makeLabelPairs 会采集指标。
- 代码0.0.0.1
```go
func makeLabelPairs(desc *Desc, labelValues []string) []*dto.LabelPair {
	totalLen := len(desc.variableLabels) + len(desc.constLabelPairs)
	if totalLen == 0 {
		// Super fast path.
		return nil
	}
	if len(desc.variableLabels) == 0 {
		// Moderately fast path.
		return desc.constLabelPairs
	}
	labelPairs := make([]*dto.LabelPair, 0, totalLen)
	for i, n := range desc.variableLabels {
		labelPairs = append(labelPairs, &dto.LabelPair{
			Name:  proto.String(n),
			Value: proto.String(labelValues[i]),
		})
	}
	labelPairs = append(labelPairs, desc.constLabelPairs...)
	sort.Sort(labelPairSorter(labelPairs))
	return labelPairs
}
```

- 前期代码中 p.timer 的类型是 *HistogramVec ，通过 Collector 接口，p.timer具有 Describe 和 Collect 方法。Describe 函数返回 Metric.Desc()，其类型是 *Desc。r.Register(c) 中调用 r.collectorsByID[collectorID] = c，c 包含将监控指标落盘到TSDB的功能。
- 代码0.0.1
```go
func MustRegister(cs ...Collector) {
	DefaultRegisterer.MustRegister(cs...)
}

// MustRegister implements Registerer.
func (r *Registry) MustRegister(cs ...Collector) {
	for _, c := range cs {
		if err := r.Register(c); err != nil {
			panic(err)
		}
	}
}

type Collector interface {
	Describe(chan<- *Desc)
	Collect(chan<- Metric)
}

type selfCollector struct {
	self Metric
}

// Describe implements Collector.
func (c *selfCollector) Describe(ch chan<- *Desc) {
	ch <- c.self.Desc()
}

func (m *invalidMetric) Desc() *Desc { return m.desc }
```

- goroutinesDesc，threadsDesc，gcDesc分别监控系统的golang程序的并发数，操作系统线程数和golang垃圾清理的间隔时间。
- 代码0.1
```go
func NewGoCollector() Collector {
	return &goCollector{
		goroutinesDesc: NewDesc(
			"go_goroutines",
			"Number of goroutines that currently exist.",
			nil, nil),
		threadsDesc: NewDesc(
			"go_threads",
			"Number of OS threads created.",
			nil, nil),
		gcDesc: NewDesc(
			"go_gc_duration_seconds",
			"A summary of the GC invocation durations.",
			nil, nil),
	}
}
```
