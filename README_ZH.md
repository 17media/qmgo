# Qmgo

`Qmgo` 是一款`Go`语言的`MongoDB` `dirver`，它基于[Mongo官方driver](https://github.com/mongodb/mongo-go-driver)开发实现，同时使用了更好的接口设计，其中接口设计主要参考[mgo](https://github.com/go-mgo/mgo)（比如[mgo](https://github.com/go-mgo/mgo)的链式调用）。

因为[Mongo官方driver](https://github.com/mongodb/mongo-go-driver)接口设计不够友好，而设计上出色的`driver`[mgo](https://github.com/go-mgo/mgo)，早已不维护。

所以我们希望`qmgo`能让用户以更优雅的姿势使用`MongoDB`的新特性。同时，也希望`qmgo`是从`mgo`迁移到新`MongoDB driver`的第一选择。

## 要求

- `Go 1.10` 及以上。
- `MongoDB 2.6` 及以上。

## 安装

推荐方式是使用`go mod`，通过在源码中`import github.com/qiniu/qmgo` 并`build` 来自动安装依赖。

当然，通过下面方式同样可行：

```
go get github.com/qiniu/qmgo
```

## Usage

- 开始，`import`并新建连接
```go  
import(
    "context"
    
    "github.com/qiniu/qmgo"
)	
ctx := context.Background()
client, err := qmgo.NewClient(ctx, &qmgo.Config{Uri: "mongodb://localhost:27017"})
db := client.Database("class")
coll := db.Collection("user")
      
```

如果你的连接是指向固定的database和collection，我们推荐使用下面的更方便的方法初始化连接，后续操作都基于`cli`而不用再关心database和collection

```go
cli, err := qmgo.Open(ctx, &qmgo.Config{Uri: "mongodb://localhost:27017", Database: "class", Coll: "user"})
```

***后面都会基于`cli`来举例，如果你使用第一种传统的方式进行初始化，根据上下文，将`cli`替换成`client`、`db`、`coll`即可***

在初始化成功后，请`defer`来关闭连接 

```go
defer func() {
    if err = cli.Close(ctx); err != nil {
        panic(err)
    }
}()
```

- 创建索引

做操作前，我们先初始化一些数据：

```go
type BsonT map[string]interface{}

type UserInfo struct {
	Name   string `bson:"name"`
	Age    uint16 `bson:"age"`
	Weight uint32 `bson:"weight"`
}

var oneUserInfo = UserInfo{
	Name:   "xm",
	Age:    7,
	Weight: 40,
}	
```

创建索引

```go
cli.EnsureIndexes(ctx, []string{"name"}, []string{"age", "name,weight"})
```

- 插入一个文档

```go
// insert one document
result, err := cli.Insert(ctx, oneUserInfo)
```

- 查找一个文档

```go
	// find one document
one := UserInfo{}
err = cli.Find(ctx, BsonT{"name": oneUserInfo.Name}).One(&one)
```

- 删除文档

```go
err = cli.Remove(ctx, BsonT{"age": 7})
```

- 插入多条数据

```go
// batch insert
var batchUserInfoI = []interface{}{
    UserInfo{Name: "wxy", Age: 6, Weight: 20},
    UserInfo{Name: "jZ", Age: 6, Weight: 25},
    UserInfo{Name: "zp", Age: 6, Weight: 30},
    UserInfo{Name: "yxw", Age: 6, Weight: 35},
}
result, err = cli.Collection.InsertMany(ctx, batchUserInfoI)
```

- 批量查找、`Sort`和`Limit`

```go
// find all 、sort and limit
batch := []UserInfo{}
cli.Find(ctx, BsonT{"age": 6}).Sort("weight").Limit(7).All(&batch)
```

## 功能

- 已经支持
  - 文档的增删改查
  - 索引配置
  - 查询`Sort`、`Limit`、`Count`

- TODO:
  - 事务
  - 聚合`Aggregate`
  - 操作支持`Options` 

## `qmgo` vs `mgo` vs `go.mongodb.org/mongo-driver`

下面我们举一个多文件查找、`sort`和`limit`的例子, 说明`qmgo`和`mgo`的相似，以及对`go.mongodb.org/mongo-driver`的改进

官方`Driver`需要这样实现

```go
// go.mongodb.org/mongo-driver
// find all 、sort and limit
findOptions := options.Find()
findOptions.SetLimit(7)  // set limit
var sorts bson.D
sorts = append(sorts, bson.E{Key: "weight", Value: 1})
findOptions.SetSort(sorts) // set sort

batch := []UserInfo{}
cur, err := coll.Find(ctx, BsonT{"age": 6}, findOptions)
cur.All(ctx, &batch)
```

`Qmgo`和`mgo`更简单，而且实现相似：

```go
// qmgo
// find all 、sort and limit
batch := []UserInfo{}
cli.Find(ctx, BsonT{"age": 6}).Sort("weight").Limit(7).All(&batch)

// mgo
// find all 、sort and limit
coll.Find(BsonT{"age": 6}).Sort("weight").Limit(7).All(&batch)
```

## contributing

非常欢迎您对`Qmgo`的任何贡献，非常感谢您的帮助！
