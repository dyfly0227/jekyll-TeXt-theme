---
title: 'gin连接mysql和redis'
excerpt: 'GORM连接MYSQL，GIN连接REDIS'
tags: 'go'
---

### Mysql
安装依赖
```bash
go get "github.com/jinzhu/gorm"
go get "github.com/jinzhu/gorm/dialects/mysql"
```
主体内容在`init`函数中，`main.go`直接引入即可
```go
// mysql.go
package Mysql

import (
	"fmt"
	"github.com/jinzhu/gorm"
	"go-service/src/Config"
	"strings"
)
import _ "github.com/jinzhu/gorm/dialects/mysql"

var DB *gorm.DB

func init() {
	path := strings.Join([]string{Config.MYSQL_URL.UserName, ":", Config.MYSQL_URL.Password, "@tcp(", Config.MYSQL_URL.Ip, ":", Config.MYSQL_URL.Port, ")/", Config.MYSQL_URL.DataBase, "?charset=utf8"}, "")
	var err error
	DB, err = gorm.Open("mysql", path)
	if err != nil {
		DB.Close()
		fmt.Println(err.Error())
	}
	fmt.Println("已连接")
	// 设置连接池，空闲连接
	DB.DB().SetMaxIdleConns(50)
	// 打开链接
	DB.DB().SetMaxOpenConns(100)

	// 表明禁用后缀加s
	DB.SingularTable(true)
}
```
### Redis
安装依赖
```bash
go get "github.com/go-redis/redis/v8"
```
与mysql同步
```go
// redis.go
package Redis

import (
	"context"
	"fmt"
	"github.com/go-redis/redis/v8"
	"go-service/src/Config"
	"time"
)

var (
	Redis *redis.Client
)

var ctx = context.Background()

func init() {
	Redis = redis.NewClient(&redis.Options{
		Addr:     Config.REDIS_URL.Ip + ":" + Config.REDIS_URL.Port,
		Password: "", // no password set
		DB:       0,  // use default DB
	})
}

func SetRedis(key string, value string, t int) bool {
	expire := time.Duration(t) * time.Second
	if err := Redis.Set(ctx, key, value, expire).Err(); err != nil {
		fmt.Println(err)
		return false
	}
	return true
}

func GetRedis(key string) string {
	result, err := Redis.Get(ctx, key).Result()
	if err != nil {
		fmt.Println(err)
		return ""
	}
	return result
}

func DelRedis(key string) bool {
	_, err := Redis.Del(ctx, key).Result()
	if err != nil {
		fmt.Println(err)
		return false
	}
	return true
}
// 延长过期时间
func ExpireRedis(key string, t int) bool {
	expire := time.Duration(t) * time.Second
	if err := Redis.Expire(ctx, key, expire).Err(); err != nil {
		fmt.Println(err)
		return false
	}
	return true
}
```
### 运行
```go
package main

import (
	_ "go-service/src/Mysql"
	_ "go-service/src/Redis"
)
```