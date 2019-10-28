# MongoDB 模型设计

## intro

[@ref: MongoDB 进阶模式设计](http://mongoing.com/mongodb-advanced-pattern-design)

[@ref: 50 Tips and Tricks for MongoDB Developers by Kristina Chodorow](https://www.oreilly.com/library/view/50-tips-and/9781449306779/ch01.html)

### MongoDB 特点

`高可用`

`分布式`

`灵活模式`

`文档数据库` -> 与关系型数据库的最大区别

- 关系模型需要你把一个数据对象，**拆分**成零部件，然后存到各个相应的表里，需要的是最后把它拼起来。举例子来说，假设我们要做一个 CRM 应用，那么要管理客户的基本信息，包括客户名字、地址、电话等。**由于每个客户可能有多个电话，那么按照第三范式，我们会把电话号码用单独的一个表来存储，并在显示客户信息的时候通过关联把需要的信息取回来。**

- 而 MongoDB 的文档模式，与这个模式大不相同。由于我们的**存储单位是一个文档**，可以支持数组和嵌套文档，所以很多时候你直接用一个这样的文档就可以涵盖这个客户相关的所有个人信息。关系型数据库的关联功能不一定就是它的优势，而是它能够工作的必要条件。 而在 MongoDB 里面，利用富文档的性质，很多时候，关联是个伪需求，可以通过合理建模来避免做关联。

  | 关系模型     | 共同点   | 文档模型           |
  | ------------ | -------- | ------------------ |
  | 单值         | 动态查询 | 富文档, 数组, 内嵌 |
  | 多文档事务性 | 二级索引 | 单文档事务性       |
  | 关联         | 聚合     | 基本不支持关联     |

- 文档模型优点

  1. **读写效率高** - 由于文档模型把相关数据集中在一块，在普通机械盘上读数据的时候不用花太多时间去定位磁头，因此在 IO 性能上有先天独厚的优势；
  2. **可扩展能力强** - 关系型数据库很难做分布式的原因就是多节点海量数据关联有巨大的性能问题。如果不考虑关联，数据分区分库，水平扩展就比较简单；
  3. **动态模式** - 文档模型支持可变的数据模式，不要求每个文档都具有完全相同的结构。对很多异构数据场景支持非常好；
  4. **模型自然** - 文档模型最接近于我们熟悉的对象模型。从内存到存储，无需经过 ORM 的双向转换，性能上和理解上都很自然易懂。

### 文档模式设计经典问题:: 内嵌 or 引用?

- 首先考虑内嵌

  适合一对一, 一对多

  局限性: 单文档最大 16M, 数组过大会影响性能.

- 再考虑引用

  适合多对多, 两个对象均为主要对象.

  局限性: 多次查询, 写入. 无跨表事务性.

1. 直接根据对象模型来设计数据模型.
2. 一般的一对一, 一对多关系均可以使用内嵌.
3. 若文档的某个数组可能长度过大, 需要使用引用, 进行分表.

   此时一般需要两次查询才能拿到完整数据, 并且 mongo 不支持跨表的事务性, 对于强事务性的应用场景需要慎重考虑.

## 案例

### 购物车

- 一个文档 -> 一个购物车
- 动态模式可以支持车内不同商品的分类描述 ???
- 已于水平扩展 ???
- TTL 索引自动删除过期数据

```js
{
    _id: ObjectId("xxxxxxxxxx"),
    user_id: 123123,
    last_activity: ISODate(),
    status: "active",
    items: [
        {
            item_id: 1234,
            title: "bread",
            price: 2.5,
            quantity: 10,
            img_url: "bread.jpg",
        },
        {
            item_id: 1111,
            title: "water",
            price: 1.0,
            quantity: 10,
            img_url: "water.jpg",
        },
    ],
}
```

### 社交网络

- 考虑因素

  - 关注, 被关注
  - 朋友圈/微博墙 -> 名人效应

    ```js
    {
        _id: ObjectId("xxxxxxxxxx"),
        user_id: 123123,
        full_name: "Alice",
        followers: ["Bob", "Coco"],
        following: ["Daniel", "Elian"],
    }
    ```

- 存在问题

  - 单文档 16M, 可能超出大小限制
  - 字段数组太大, 影响性能(`C`R`U`D)

- 解决方案

  - 拆表, 关注被关注单独一个表(引用)

  ```
  { user: "Alice",      following: "Daniel",    }
  { user: "Alice",      following: "Elian",     }
  ```

- 关注数

  - 每次使用 `count()` 计算浪费资源
  - 直接在用户信息中添加 `count_following` 字段
  - 实时调整此字段的增减

### 朋友圈/微博墙

- 扇出读

  - 用户发布状态只需要写一次
  - 查看朋友圈从多个地方读取
  - 节省存储, 牺牲性能

- 扇出写

  - 发布状态, 写到所有好友的朋友圈中
  - 查看朋友圈, 只读取一次
  - 提高性能, 浪费存储

| -      | 优点               | 缺点                     |
| ------ | ------------------ | ------------------------ |
| 扇出读 | 实现简单           | 读取时需要去多个节点     |
| 扇出读 | 无需额外存储空间   | 浪费资源在不活跃的用户上 |
| 扇出写 | 读取高效           | 写操作昂贵, 需要 N 次    |
| 扇出写 | 有效过滤不活跃用户 | 空间大                   |
| 扇出写 | 工作集较小         | -                        |

### IOT 设备

- 特点

  - 写密集型
  - 异构数据(数据类型繁多)

    ```js
    {
        _id: ObjectId,
        plain_id: ObjectId,
        ts: TimeStamp,
        metrics: {
            key: Any,
            key: Any,
            key: Any,
        }
    }
    ```

- 对 doc.metrics.xxx 设置 sparse 索引, 忽略字段为空的条目

- 性能提升: `分桶`

  - 将采集的数据按小时分桶, 每小时一条数据
  - 每条数据按照分钟来分字段

    ```js
    {
        ...
        metrics:{
            key:{
                "0":...,
                "1":...,
                "2":...,
                ...
                "59":...
            }
        }
    }
    ```