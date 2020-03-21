## 概述

> groupcache is a distributed caching and cache-filling library, intended as a replacement for a pool of memcached nodes in many cases.

groupcache是memcached作者Brad Fitzpatrick的另外一个key/value键值存储项目,是一个轻量级的分布式缓存库,也被称为golang版的memcached,在某些方面可以替代memcached,是goper必读源码入门级项目之一。

## 主要特点
|  |  |
|--|--|
| 存储   |内存
| 移植性   |目前只支持go语言
| 数据  |多节点备份热数据
| 集群  |支持
| 功能  |支持Get,不支持常见的Update/Set/Expired/TTL
| 支持数据类型  |byte、string、proto
| 统计  |缓存命中、远端缓存命中，本地缓存命中、错误信息、网络请求相关统计

## 基本过程
1.获取本地缓存和热点缓存
2.在本地没有获取到缓存的情况下，使用一致性Hash算法,从远端peer获取数据,这里同时会生成热点数据。
3.在远端peer没有获取到缓存情况下，从数据源(mysql,文件等)中获取数据。
4.在第2,3步骤中,groupcache有自己的并发控制器,当对于同一个key的多个请求,保证只有一个请求能从远端或者数据源(mysql)中获取或更新缓存，这个请求取到数据后做统一返回,一定程度上减少不必要网络请求和缓存击穿。
5.从数据源(mysql,文件)获取的数据设置到本地缓存

groupcache具有自己的优点,既是作为客户端库又是做为服务端，省去繁重的服务集部署,缺点是只适用于更新频率较低的数据。

## 简单使用:

```go
package main

import (
	"errors"
	"flag"
	"log"
	"net/http"
	"strings"

	"github.com/golang/groupcache"
)

//相当于DB
var Store = map[string][]byte{
	"red":   []byte("#FF0000"),
	"green": []byte("#00FF00"),
	"blue":  []byte("#0000FF"),
}

//初始化Groupcache对象并设置数据源
var Group = groupcache.NewGroup("foobar", 64<<20, groupcache.GetterFunc(
	func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
		log.Println("looking up", key)
		v, ok := Store[key]
		if !ok {
			return errors.New("color not found")
		}
		dest.SetBytes(v)
		return nil
	},
))

func main() {
	addr := flag.String("addr", ":8080", "server address")
	peers := flag.String("pool", "http://localhost:8080", "server pool list")
	flag.Parse()
	
	//路由
	http.HandleFunc("/color", func(w http.ResponseWriter, r *http.Request) {
		color := r.FormValue("name")
		var b []byte
		err := Group.Get(nil, color, groupcache.AllocatingByteSliceSink(&b))
		if err != nil {
			http.Error(w, err.Error(), http.StatusNotFound)
			return
		}
		w.Write(b)
		w.Write([]byte{'\n'})
	})
	
	//注册连接池peer
	p := strings.Split(*peers, ",")
	pool := groupcache.NewHTTPPool(p[0])
	pool.Set(p...)
	http.ListenAndServe(*addr, nil)
}
```
在浏览器中可以多次访问,查看结果:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200311080020270.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3NzMxMDU2,size_16,color_FFFFFF,t_70)
下节课将为同学详细讲一下groupcache的架构分析


