---
title: go爬虫学习
date: 2025-04-12 16:57:22
tags: Golang
---

# golang爬虫学习

目的是学习一下爬虫的流程，顺便熟悉一下go的语法和使用

代码仓库见[我的github仓库](https://github.com/fishwww-ww/go-spider-mundo)

---

## 项目介绍
本项目爬取的是[mundo](https://mundo.trancecho.top/)中的组队信息，从而快速了解组队的概况，以便进一步筛选适合且人数未满的队伍

使用的工具：
- golang原生http库等网络请求、解析相关的库
- gorm
- docker

---

## 开发流程

### 1. 分析网页

打开[mundo组队](https://mundo.trancecho.top/teamup)页面,F12打开开发者工具，找到网络栏

在document类型的请求中找到名为teamup的请求,查看它的响应,

发现里面没有数据,说明这些信息是通过api请求得到的,无法在DOM中获取

再筛选xhr类型,找到 allteam?service=mundo ,正是所需要的响应

接下来通过伪造api请求,来获取它的响应,从而实现数据爬取

### 2. 实现请求

将响应的json转换为go的结构体
可以直接复制json数据,交给ai处理,也可以用爬虫工具处理
得到响应结构体
```golang
type Response struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Data    Data   `json:"data"`
}
```

使用go的http库,创建一个客户端,并将方法和url填入,构造一个请求

```golang
client := &http.Client{}
req, err := http.NewRequest("GET", "https://qgdoywhgtdnh.sealosbja.site/timerme/api/allteam?service=mundo", nil)
```

填入一些请求头,大型网站都有反爬机制,请求头越详细越好,这里只填入部分信息

```golang
req.Header.Set("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/135.0.0.0 Safari/537.36 Edg/135.0.0.0")
req.Header.Set("Accept", "application/json, text/plain, */*")
req.Header.Set("Accept-Language", "zh-CN,zh;q=0.9")
req.Header.Set("Accept-Encoding", "gzip, deflate, br, zstd")
```

执行请求,并获取响应,别忘了最后关闭请求

```golang
resp, err := client.Do(req)
defer resp.Body.Close()
```

读取响应,并反序列化,便于后期操作数据
```golang
body, err := ioutil.ReadAll(resp.Body)
err = json.Unmarshal(body, &response)
```

### 3. 存入数据库

我是在docker中下的mysql,启动后新建一个schema,这个schema要和代码中的DBNAME对应上

直接用gorm框架来连接mysql
```golang
DB, err = gorm.Open(mysql.Open(path), &gorm.Config{})
```

然后用AutoMigrate方法自动建表
```golang
err = DB.AutoMigrate(&Content{})
```

最后回到处理响应的部分,将数据插入数据库中
```golang
result := DB.Create(&response.Data.Team.Content)
```

---

## 项目收获
- 学会了分析网页和实现爬虫程序的流程
- 熟悉了go,docker和数据库连接,把请求,响应,解析,存储等流程串联起来

## 后续学习
- 学习go的并发编程,提高爬虫效率
- 学习docker的目录挂载机制,如何将数据挂载在宿主机中,实现数据的持久化