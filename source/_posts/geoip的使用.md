---
title: geoip的使用
date: 2023-2-7 14:00:00
categories:
- golang
---
Maxminds的官网：https://www.maxmind.com/en/geoip2-services-and-databases

## GeoIP2和GeoIPLite2的对比

两者都提供了获取某个指定IP的信息的功能。

GeoIP2和GeoIPLite2都提供了数据库和web服务。

GeoIPLite2数据库和web服务提供了指定IP的免费地理定位和有限的网络数据。通过GeoIP2获取的数据比通过GeoIP2获取的数据更为准确，也就是准确性更高。

下载GeoLite2数据库或者访问GeoLite2的web服务都需要先注册GeoLite2的账号。

GeoLite2 使用与 GeoIP2相同的集成方法和文档，并进行了一些小的修改。

GeoIP2比GeoLite2的数据更为准确，每个月的第一个周二，这两个数据库都会有更新。

GeoIP2 数据库比 GeoLite2 数据库更准确，因为前者是使用更多数据构建的。

GeoIP2 在国家层面上的准确率为 99.8%。 GeoLite2 在国家层面上应该是差不多的。

## 自动更新数据库

可以使用MaxMind的GeoIP更新程序（参考https://github.com/maxmind/geoipupdate）或者直接下载数据库。前一种方法更为推荐，下面介绍前一种方法，以Mac为例：

1. 安装geoipupdate程序：

   ```
   brew install geoipupdatec
   ```

2. 在https://www.maxmind.com/en/accounts/current/license-key生成license key信息， 复制https://www.maxmind.com/en/accounts/current/license-key/GeoIP.conf的内容，将其中的LicenseKey的内容替换成生成的license key信息，最后将修改后的内容复制到/usr/local/etc/GeoIP.conf，最终/usr/local/etc/GeoIP.conf的内容格式为：

   ```
   # GeoIP.conf file for `geoipupdate` program, for versions >= 3.1.1.
   # Used to update GeoIP databases from https://www.maxmind.com.
   # For more information about this config file, visit the docs at
   # https://dev.maxmind.com/geoip/updating-databases?lang=en.
   
   # `AccountID` is from your MaxMind account.
   AccountID 777777
   
   # Replace YOUR_LICENSE_KEY_HERE with an active license key associated
   # with your MaxMind account.
   LicenseKey IXXXXXXXX
   
   # `EditionIDs` is from your MaxMind account.
   EditionIDs GeoLite2-ASN GeoLite2-City GeoLite2-Country
   ```

3. 运行：

   ```
   geoipupdate
   ```

## 获取IP信息

```go
package main

import (
	"fmt"
	"log"
	"net"

	"github.com/oschwald/geoip2-golang"
)

func main() {
	db, err := geoip2.Open("GeoLite2-City.mmdb")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
	// If you are using strings that may be invalid, check that ip is not nil
	ip := net.ParseIP("81.2.69.142")
	record, err := db.City(ip)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Portuguese (BR) city name: %v\n", record.City.Names["zh-CN"])
	if len(record.Subdivisions) > 0 {
		fmt.Printf("English subdivision name: %v\n", record.Subdivisions[0].Names["zh-CN"])
	}
	fmt.Printf("country name: %v\n", record.Country.Names["zh-CN"])
	fmt.Printf("ISO country code: %v\n", record.Country.IsoCode)
	fmt.Printf("Time zone: %v\n", record.Location.TimeZone)
	fmt.Printf("Coordinates: %v, %v\n", record.Location.Latitude, record.Location.Longitude)
}
```

运行结果为：

```shell
Portuguese (BR) city name: 
English subdivision name: 英格兰
country name: 英国
ISO country code: GB
Time zone: Europe/London
Coordinates: 53.3022, -2.2312
```
