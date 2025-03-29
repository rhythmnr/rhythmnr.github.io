---
title: golang时间相关计算等
date: 2022-12-27 18:00:00
tags:
- 原创
categories:
- golang
---
## 计算时间所在天，所在周，所在月的开始时间

```go
// timeByInterval 根据interval，计算t基于interval的开始时间是什么，可以按照天，月，周对时间统计，比如计算每周的时间有多少
func timeByInterval(t time.Time, interval string) time.Time {
	if interval == "day" {
    // 当天的开始时间
		return time.Date(t.Year(), t.Month(), t.Day(), 0, 0, 0, 0, t.Location())
	}
	if interval == "week" {
    // 当周的开始时间
		t = t.AddDate(0, 0, int(t.Weekday())-1)
		return time.Date(t.Year(), t.Month(), t.Day(), 0, 0, 0, 0, t.Location())
	}
	if interval == "month" {
    // 当月的开始时间
		return time.Date(t.Year(), t.Month(), 0, 0, 0, 0, 0, t.Location())
	}
	// interval默认值是day
	return time.Date(t.Year(), t.Month(), t.Day(), 0, 0, 0, 0, t.Location())
}
```

