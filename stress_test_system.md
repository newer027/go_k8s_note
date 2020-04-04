# 网络服务的压测系统

更新压测脚本的参数：
通过平台对外接口的REST API提交这次压测的参数，平台将这些参数填入预先设置好的代码生成器模板当中，得到新的压测素材。

```go
"html/template"
var buff           *template.Template
buff = buff.Funcs(template.FuncMap{"unescaped": unescaped})
buff.Execute(io.Writer(file), scene)
```

多个压测脚本的集合形成一个分布式应用的全链路压测的场景，覆盖到应用的所有接口。并根据接口参数依赖，计算出接口分组与执行顺序。
```go
//DoPtestByJmeter do ptest by jmeter
func (s *Service) DoPtestByJmeter(c context.Context, ptestParam model.DoPtestParam, testNameNicks []string) (resp model.DoPtestResp, err error) {
	//获取 Debug,CPUCore,command
	Debug, CPUCore, command = s.CreateCommand(ptestParam)
}

//CreateCommand Create Command
func (s *Service) CreateCommand(ptestParam model.DoPtestParam) (debug, CPUCore int, command string) {
	// 调试逻辑
	if ptestParam.IsDebug {
		CPUCore = s.c.Paas.CPUCoreDebug
		debug = 1
		command = cpJar + " mkdir /data/jmeter-log & jmeter -n -t " + ptestParam.FileName + " -j " + ptestParam.JmeterLog + " -l " + ptestParam.ResJtl + " -F ;" + pingString
	} else {
		CPUCore = s.c.Paas.CPUCore
		debug = -1
		command = cpJar + " mkdir /data/jmeter-log & jmeter -n -t " + ptestParam.FileName + " -j " + ptestParam.JmeterLog + " -l " + ptestParam.ResJtl + " -F"
	}
	return
}
```

- 命令行的拼接的参考文档如下：
https://jmeter.apache.org/usermanual/get-started.html
>-n

>This specifies JMeter is to run in cli mode

>-t

>[name of JMX file that contains the Test Plan].

>jmeter -n -t my_test.jmx -l log.jtl -H my.proxy.server -P 8000
