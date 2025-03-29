---
title: gorm打印SQL操作日志
date: 2022-12-25 11:04:00
categories:
- golang
---
```
func newModel(opts ...Option) (*defaultDataModel, error) {
	e := &Options{
		Addr:     "127.0.0.1:9999",
		Database: "database",
	}
	for _, o := range opts {
		o(e)
	}

	connStr := fmt.Sprintf(
		"tcp://%s?database=%s&username=%s&password=%s&read_timeout=10&write_timeout=20",
		e.Addr,
		e.Database,
		e.Username,
		e.Password)
	newLogger := logger.New(
		log.New(os.Stdout, "\r\n", log.LstdFlags), // io writer
		logger.Config{
			// SlowThreshold: time.Microsecond, // 慢 SQL 阈值
			LogLevel: logger.Info, // Log level
			Colorful: false,       // 禁用彩色打印
		},
	)

	db, err := gorm.Open(clickhouse.Open(connStr), &gorm.Config{
		Logger: newLogger,
	})
	if err != nil {
		return nil, err
	}
	return &defaultDataModel{
		db:    db,
		table: "`table`",
	}, nil
}
```

主要就是newLogger和下面的db, err := gorm.Open

另外一个：把慢查询日志单独放到一个文件里

```go
var logFile *os.File
	fileName := "gorm.log"
	_, err2 := os.Stat(fileName)
	if err2 != nil {
		if os.IsNotExist(err2) {
			logFile, _ = os.Create(fileName)
		} else {
			logFile, _ = os.Open(fileName)
		}
	}
	ormLog := logger.New(
		log.New(logFile, time.Now().Format("2006-01-02 15:04:05 "), log.Lshortfile),
		logger.Config{
			SlowThreshold: 200 * time.Millisecond,
			LogLevel:      logger.Info, // Log level
			Colorful:      false,       // 禁用彩色打印
		},
	)
	connStr := fmt.Sprintf(
		"tcp://%s?database=%s&username=%s&password=%s&read_timeout=10&write_timeout=20",
		e.Addr,
		e.Database,
		e.Username,
		e.Password)
	db, err := gorm.Open(clickhouse.Open(connStr), &gorm.Config{
		Logger: ormLog,
	})
```

参考上面的写法，主要就是gorm.Open里只指定包含Logger的gorm Config，可以看到打印的效果：

```log
2022-12-13 15:45:05 logger.go:133: /Users/u/go/pkg/mod/gorm.io/driver/clickhouse@v0.3.3/clickhouse.go:55
[info] replacing callback `gorm:create` from /Users/u/go/pkg/mod/gorm.io/driver/clickhouse@v0.3.3/clickhouse.go:55
2022-12-13 15:45:05 logger.go:172: /Users/u/code/service/model/datamodel.go:222 SLOW SQL >= 200ms
[725.062ms] [rows:900] SELECT u.ct, untuple(arrayJoin(arraySlice(arraySort((x, y)->-y, arrayMap((x, y, z) -> (x, y, z), groupArray(u.src_ip), groupArray(u.dst_ip), groupArray(u.p)), groupArray(u.p)), 1, 5))) AS res FROM (SELECT toStartOfInterval(create_time, INTERVAL 10 minute) AS ct, src_ip, dst_ip, SUM(pack_size) AS p FROM `table` WHERE create_time < FROM_UNIXTIME(1669968000) AND create_time >= FROM_UNIXTIME(1669860000) GROUP BY ct, src_ip, dst_ip) AS u GROUP BY `ct` ORDER BY ct
```
