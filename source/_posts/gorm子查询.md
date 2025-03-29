---
title: gorm子查询的方法
date: 2022-11-23 10:00:00
categories:
- 数据库
---

可以参考下面的写法

```
		subQuery := m.db.Table(m.table).Select(fmt.Sprintf("%s AS ct, src_ip, dst_ip, SUM(pack_size) AS p", format)).
			Where("create_time < FROM_UNIXTIME(?) AND create_time >= FROM_UNIXTIME(?)", endTime, startTime).
			Group("src_ip, dst_ip, ct")
		res := fmt.Sprintf("untuple(arrayJoin(arraySlice(arraySort((x,y)->-y, arrayMap((x, y, z) -> (x, y,z), groupArray(u.src_ip),groupArray(u.dst_ip), groupArray(u.p)) , groupArray(u.p)),1,%d)))", topNumber)
		db := m.db.Table("(?) as u", subQuery).
			Select(fmt.Sprintf("u.ct, %s AS res", res)).
			Group("ct").Order("ct")
		var r []map[string]interface{}
		if err := db.Find(&r).Error; err != nil {
			return nil, err
		}
```

上传到 github打码

> PS: 有的时候查询结果是一些预期之外的为止格式，此时可以将结果Scan到一个map中或者map数组中，然后查看内容后通过mapstructure将结果转换结构体，比如：
>
> ```
> type ResultByTime struct {
> 	Time  time.Time `mapstructure:"ct"`
> 	SrcIP string    `mapstructure:"res.1"`
> }
> ```
>
> ```
> 		db := m.db.Table("(?) as u", subQuery).
> 			Select(fmt.Sprintf("u.ct, %s AS res", res)).
> 			Group("ct").Order("ct")
> 		var r []map[string]interface{}
> 		if err := db.Find(&r).Error; err != nil {
> 			return nil, err
> 		}
> 		results := make([]ResultByTime, 0, len(r))
> 		if err := mapstructure.Decode(r, &results); err != nil {
> 			return nil, err
> 		}
> 		return results, nil
> 	}
> ```
