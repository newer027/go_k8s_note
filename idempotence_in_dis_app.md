# 分布式应用中幂等的示例

- 从 kafka 获取的消息中含有消息的 group/topic/partition 表示消息的类别，以及 offset 字段，用 map 结构记录最新的 offset，最新的消息的 offset 会大于之前的 offset。
```go
offset, ok = s.videoupSubIdempotent[msg.Partition]
if !ok {
    if dbus, err = s.arc.DBus(c, s.c.VideoupSub.Group, msg.Topic, msg.Partition); err != nil {
        return
    }
    if dbus == nil {
        if _, err = s.arc.AddDBus(c, s.c.VideoupSub.Group, msg.Topic, msg.Partition, msg.Offset); err != nil {
            return
        }
        offset = msg.Offset
    } else {
        offset = dbus.Offset
    }

}
// if last offset > current offset -> continue
if offset > msg.Offset {
    log.Error("key(%s) value(%s) partition(%s) offset(%d) is too early", msg.Key, msg.Value, msg.Partition, msg.Offset)
    return
}
s.videoupSubIdempotent[msg.Partition] = msg.Offset
```

- 在提交事务之前，通过 RowsAffected 是否返回 err 判断是否需要回滚事务。回滚事务采用 tx.Rollback() 函数。
```go
func (res *mysqlResult) RowsAffected() (int64, error) {
	return res.affectedRows, nil
}
```