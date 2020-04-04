# elastic 的go client使用示例

- 定义 Dao 结构体，其中包含 elastic client，并创建 Dao 结构体。通过多次 req.WhereEq 限定 elastic 存储对象的筛选条件。通过 req.Order 确定返回数据的排序方式。采用 req.Scan 提交查询的请求，并返回结果。
```go
// Dao dao
type Dao struct {
	c *conf.Config
	// elastic client
	EsCli *elastic.Elastic
}

// New init Elastic
func New(c *conf.Config) (dao *Dao) {
	dao = &Dao{
		c: c,
		// elastic client
		EsCli: elastic.NewElastic(c.Elastic),
	}
	return
}

// SearchSubtitle .
func (d *Dao) SearchSubtitle(c context.Context, arg *model.SubtitleSearchArg) (res *model.SearchSubtitleResult, err error) {
	var (
		req    *elastic.Request
		fields []string
	)
	fields = _subtitleFields
	req = d.esCli.NewRequest("dm_subtitle").Index("subtitle").Fields(fields...).Pn(int(arg.Pn)).Ps(int(arg.Ps))
	if arg.Aid > 0 {
		req.WhereEq("aid", arg.Aid)
	}
	if arg.Mid > 0 {
		req.WhereEq("mid", arg.Mid)
	}
	if arg.Oid > 0 {
		req.WhereEq("oid", arg.Oid)
	}
	if arg.Mid > 0 {
		req.WhereEq("mid", arg.Mid)
	}
	if arg.Status > 0 {
		req.WhereEq("status", arg.Status)
	}
	if arg.UpperMid > 0 {
		req.WhereEq("up_mid", arg.UpperMid)
	}
	if arg.Lan > 0 {
		req.WhereEq("lan", arg.Lan)
	}
	req.Order("mtime", "desc")
	if err = req.Scan(c, &res); err != nil {
		log.Error("elastic search(%s) error(%v)", req.Params(), err)
		return
	}
	return
}
```
