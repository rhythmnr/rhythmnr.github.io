---
title: gorm打印SQL语句
date: 2022-11-23 10:00:00
categories:
- 数据库
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
	return &defaultGopacketDataModel{
		db:    db,
		table: "`tableA`",
	}, nil
}
```

这里连接的是clickhouse数据库，可以参考这个写法，主要就是newLogger和下面的db, err := gorm.Open里指定的logger。这样就可以在gorm的日志（默认是终端）看到对数据库执行的SQL语句是什么了。

