# Erlang 项式存储 (ETS)

Erlang 项式存储 (Erlang Term Storage，通常简称 ETS) 是 OTP 中内置的一个功能强大的存储引擎，我们在 Elixir 中也可以很方便地使用。本文将介绍如何使用 ETS 以及如何在我们的应用中使用它。

Table of Contents

* 概览
* 建表
  * 表的类型
  * 访问控制
* 插入数据
* 获取数据
  * 查询键
  * 简单的匹配
  * 高级的查询
* 删除数据
  * 删除记录
  * 删除表
* ETS 的用例

## 概览

ETS 是一个针对 Elixir 和 Erlang 对象的健壮的内存 (in-memory) 存储，并且内置于 OTP 中。ETS 可以存储大量的数据，同时维持常数时间的数据访问。

ETS 中的「表」 (table) 是由单独的进程创建并拥有的。当这个进程退出时，这张表也就销毁了。

您可以创建任意数量的 ETS 表。唯一的限制是服务器内存。可以使用环境变量 ERL_MAX_ETS_TABLES 来指定限制。

## 建表

新的表由 new/2 创建，该函数接受一个表名以及一组选项作为参数，返回一个表标识符 (table identifier)，用之于接下来的操作。

我们创建一个通过昵称来存取用户的表来做例子：

```erlang
table = :ets.new(:user_lookup, [:set, :protected])
```

类似 GenServer，我们也可以直接通过名字而不是标识符来访问 ETS 表。这需要我们添加 :named_table 选项。然后我们就可以用名字来访问这张表了：

```erlang
ets.new(:user_lookup, [:set, :protected, :named_table])
```

## 表的类型

ETS 提供了四种类型的表：

* set - 默认的表类型。每个键(key)对应一个值(value)。键是唯一的。
* ordered_set - 与 set 类似，但是按照 Erlang/Elixir 项式来排序。需要注意的是这里键的比较方式。键可以不同，只要｢相等｣即可，例如 1 和 1.0 就是｢相等｣的。
* bag - 每个键可以包括多个对象，但一个对象在一个键中只能有一个实例。
* duplicate_bag - 每个键可以包括多个对象，也允许对象重复。

## 访问控制

ETS 提供的访问控制机制跟模块差不多：

* public - 所有进程都可以读／写。
* protected - 所有进程都可读。只有拥有者可以写。这是默认的配置。
* private - 只有拥有者可以读／写。

## 资源竞争（Race Conditions）

如果多于一个进程写入数据到一个表 - 不管是通过 :public 访问，或者通过拥有者进程接收消息 - 资源竞争都是可能发生的。比如，两个进程每个都尝试读取一个值为 0 的计数器，自增，然后写入 1；最后的结果就只反映了一次自增。

对于计数器来说，:ets.update_counter/3 提供了原子性的读和写操作。对于其它场景，拥有者进程可能还是必须根据收到的消息，自己实现原子性的操作，比如 “把当前值，添加到列表里面键为 :results 的位置”。

## 插入数据

ETS 没有模式 (Schema) 的概念。唯一的限制是数据需要以元组的形式存放，并且将第一个元素作为键。我们使用 insert/2 来添加新数据：

```erlang
ets.insert(:user_lookup, {"doomspork", "Sean", ["Elixir", "Ruby", "Java"]})
```

在 set 或 ordered_set 上直接执行 insert/2 会覆盖掉已经存在的数据。使用 insert_new/2 可以避免数据覆盖的情况，该函数会在键已经存在时返回 false：

## 基于磁盘的 ETS (DETS)

---

我们现在了解了 ETS 这个内存存储，那有没有基于磁盘的存储呢？没错，我们有「基于磁盘的项式存储」 (Disk Based Term Storage)，简称 DETS。ETS 和 DETS 的 API 基本上是通用的，只有创建表的方式有些许不同。DETS 使用 open_file/2 而且不需要 :named_table 选项：
