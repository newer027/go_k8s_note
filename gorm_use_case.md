# 优先采用gorm的场景和gorm典型用例

- queryAddIsDeleted 在 sql 语句中添加 is_deleted = 0 ，可以选定数据库中通过字段标注没删除的对象。
- FetchUserList 中调用 FetchUserList，获取符合条件的对象的总数和对象列表。
```go
func queryAddIsDeleted(query string) string {
	if len(query) != 0 {
		query += " AND is_deleted = 0"
	} else {
		query = "is_deleted = 0"
	}
	return query
}

// FetchUserList FetchUserList
func (d *Dao) FetchUserList(ctx context.Context, query string, from, limit int) (users []*model.UserSetting, total int, err error) {
	var (
		user *model.UserSetting
	)
	query = queryAddIsDeleted(query)
	err = d.db.Table(user.TableName()).Where(query).Count(&total).Error
	if err != nil {
		return
	}
	err = d.db.Table(user.TableName()).Order(sort).Offset(from).Where(query).Limit(limit).Find(&users).Error
	return
}
```




