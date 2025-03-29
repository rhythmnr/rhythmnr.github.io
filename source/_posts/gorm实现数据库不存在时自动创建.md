---
title: gorm实现数据库不存在时自动创建
date: 2024-1-4 10:00:00
categories:
- 数据库
---

可以参考下面的写法

```go
func ConnMysql() *gorm.DB {
	checkDatabase()
	dsn := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=%s&collation=%s&%s",
		config.Conf.Mysql.Username,
		config.Conf.Mysql.Password,
		config.Conf.Mysql.Host,
		config.Conf.Mysql.Port,
		config.Conf.Mysql.Database,
		config.Conf.Mysql.Charset,
		config.Conf.Mysql.Collation,
		config.Conf.Mysql.Query,
	)
	// 隐藏密码
	showDsn := fmt.Sprintf(
		"%s:******@tcp(%s:%d)/%s?charset=%s&collation=%s&%s",
		config.Conf.Mysql.Username,
		config.Conf.Mysql.Host,
		config.Conf.Mysql.Port,
		config.Conf.Mysql.Database,
		config.Conf.Mysql.Charset,
		config.Conf.Mysql.Collation,
		config.Conf.Mysql.Query,
	)
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		// 禁用外键(指定外键时不会在mysql创建真实的外键约束)
		DisableForeignKeyConstraintWhenMigrating: true,
	})
	if err != nil {
		logger.Log.Panicf("初始化mysql数据库异常: %v", err)
	}
	// 开启mysql日志
	if config.Conf.Mysql.LogMode {
		db.Debug()
	}
	logger.Log.Infof("初始化mysql数据库完成! dsn: %s", showDsn)
	return db
}

func checkDatabase() {
	dsn := fmt.Sprintf("%s:%s@tcp(%s:%d)/?charset=%s&collation=%s&%s",
		config.Conf.Mysql.Username,
		config.Conf.Mysql.Password,
		config.Conf.Mysql.Host,
		config.Conf.Mysql.Port,
		config.Conf.Mysql.Charset,
		config.Conf.Mysql.Collation,
		config.Conf.Mysql.Query,
	)
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		DisableForeignKeyConstraintWhenMigrating: true,
	})
	if err != nil {
		logger.Log.Panicf("初始化mysql数据库异常: %v", err)
	}
	sqlDB, err := db.DB()
	if err != nil {
		logger.Log.Panicf("获取SQL DB失败: %v", err)
	}
	defer sqlDB.Close()
	createDBStatement := fmt.Sprintf("CREATE DATABASE IF NOT EXISTS %s", config.Conf.Mysql.Database)
	_, err = sqlDB.Exec(createDBStatement)
	if err != nil {
		logger.Log.Panicf("验证数据库信息失败: %v", err)
	}
}
```

