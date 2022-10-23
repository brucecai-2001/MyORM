# MyORM

> 对象关系映射（Object Relational Mapping，简称ORM）是通过使用描述对象和数据库之间映射的元数据，将面向对象语言程序中的对象自动持久化到关系数据库中。

![](https://raw.githubusercontent.com/brucecai-2001/Image/master/20221019161127.png)

ORM 框架相当于对象和数据库中间的一个桥梁，借助 ORM 可以避免写繁琐的 SQL 语言，仅仅通过操作具体的对象，就能够完成对关系型数据库的操作。 <br>
```Go
type User struct {
    Name string
    Age  int
}

orm.CreateTable(&User{})
orm.Save(&User{"Tom", 18})
var users []User
orm.Find(&users)
```

# database/sql 标准库

Go 语言提供了标准库 database/sql 用于和数据库的交互 <br>

使用 sql.Open() 连接数据库，第一个参数是驱动名称 <br>
Exec() 用于执行 SQL 语句。<br>
查询语句通常使用 Query() 和 QueryRow()，前者可以返回多条记录，后者只返回一条记录。<br>

<br>

# 实现一个简单的 log 库

开发一个框架/库并不容易，详细的日志能够帮助我们快速地定位问题。使用log库 <br>
    ---支持日志分级（Info、Error）。
    ---不同层级日志显示时使用不同的颜色区分。
    ---显示打印日志代码对应的文件名和行号。<br>

1. 创建 2 个日志实例(logger)分别用于打印 Info 和 Error 日志。
2. 支持设置日志的层级(InfoLevel, ErrorLevel, Disabled)。
   
<br>

# 核心结构 Session

用于实现与数据库的交互。Session 结构体目前只包含三个成员变量，第一个是 db *sql.DB，即使用 sql.Open() 方法连接数据库成功之后返回的指针。第二个和第三个成员变量用来拼接 SQL 语句和 SQL 语句中占位符的对应值。用户调用 Raw() 方法即可改变这两个变量的值。<br>
```Go
type Session struct {
	db       *sql.DB
	dialect  dialect.Dialect
	refTable *schema.Schema
	clause   clause.Clause
	sql      strings.Builder
	sqlVars  []interface{}
}
```
1. 完成Session的初始化，清除。
2. 封装 Exec()、Query() 和 QueryRow() 三个原生sql方法，封装有 2 个目的，一是统一打印日志（包括 执行的SQL 语句和错误日志）。
二是执行完成后，清空 (s *Session).sql 和 (s *Session).sqlVars 两个变量。这样 Session 可以复用，开启一次会话，可以执行多次 SQL。

<br>

# 核心结构 Engine

Engine负责交互前的准备工作（比如连接/测试数据库），交互后的收尾工作（关闭连接）。Engine 也是 GeeORM 与用户交互的入口。<br>

NewEngine <br>
    ---连接数据库，返回 *sql.DB。<br>
    ---调用 db.Ping()，检查数据库是否能够正常连接。<br>

NewSession <br>
    ---通过 Engine 实例创建会话，进而与数据库进行交互了。<br>


# 2. Golang对象和数据库表结构映射

不同数据库支持的数据类型也是有差异的，即使功能相同，在 SQL 语句的表达上也可能有差异。ORM 框架往往需要兼容多种数据库，因此我们需要将差异的这一部分提取出来，每一种数据库分别实现，实现最大程度的复用和解耦。这部分代码称之为 dialect，相当于驱动。<br>

Dialect 接口包含 2 个方法：<br>

DataTypeOf 用于将 Go 语言的类型转换为该数据库的数据类型。<br>
TableExistSQL 返回某个表是否存在的 SQL 语句，参数是表名(table)。<br>

RegisterDialect 和 GetDialect 两个方法用于注册和获取 dialect 实例。如果新增加对某个数据库的支持，那么调用 RegisterDialect 即可注册到全局dialect map。<br>

<br>

# 2.1 对 SQLite 的支持。

实现了 init() 函数，包在第一次加载时，会将 sqlite3 的 dialect 自动注册到全局。<br>
DataTypeOf 将 Go 语言的类型映射为 SQLite 的数据类型。TableExistSQL 返回了在 SQLite 中判断表 tableName 是否存在的 SQL 语句。<br>
```Go
switch typ.Kind() 
    case reflect.Bool:
		return "bool"
```

<br>

# 2.2 Schema

Dialect 实现了一些特定的 SQL 语句的转换，接下来我们将要实现 ORM 框架中最为核心的转换——对象(object)和表(table)的转换。给定一个任意的对象，转换为关系型数据库中的表结构。<br>
        --表名(table name) —— 结构体名(struct name) <br>
        --字段名和字段类型 —— 成员变量和类型。<br>
        --额外的约束条件(例如非空、主键等) —— 成员变量的Tag（Go 语言通过 Tag 实现，Java、Python 等语言通过注解实现）<br>

```Go
type User struct {
    Name string `geeorm:"PRIMARY KEY"`
    Age  int
}
```
```SQL
CREATE TABLE `User` (`Name` text PRIMARY KEY, `Age` integer);
```

```Go
// Field represents a column of database
type Field struct {
	Name string
	Type string
	Tag  string
}

// Schema represents a table of database
type Schema struct {
	Model      interface{}
	Name       string
	Fields     []*Field
	FieldNames []string
	fieldMap   map[string]*Field
}
```
Field 包含 3 个成员变量，字段名 Name、类型 Type、和约束条件 Tag <br>
Schema 主要包含被映射的对象 Model、表名 Name 和字段 Fields。<br>
FieldNames 包含所有的字段名(列名)，fieldMap 记录字段名和 Field 的映射关系，方便之后直接使用，无需遍历 Fields。 <br>

实现 Parse 函数，将任意的对象解析为 Schema 实例。<br>

<br>

# 2.3 实现表的一些操作CREATE DROP EXIST

Session实现数据库表的创建、删除和判断是否存在的功能。三个方法的实现逻辑是相似的，利用 RefTable() 返回的数据库表和字段的信息，拼接出 SQL 语句，调用原生 SQL 接口执行。<br>

<br>

# 2.4 Clause

SQL语句由很多子句构成。如果想一次构造出完整的 SQL 语句是比较困难的，因此我们将构造 SQL 语句这一部分独立出来，放在子package clause 中实现。<br>
```Go
type generator func(values ...interface{}) (string, []interface{})

var generators map[Type]generator
```
每一个子句都有一个自己的sql子句生成器，input 参数，ouput sql子句<br>

然后在 clause/clause.go 中实现结构体 Clause 拼接各个独立的子句。<br>
```Golang
type Clause struct {
	sql     map[Type]string
	sqlVars map[Type][]interface{}
}
```
Set 方法根据 Type(操作) 调用对应的 generator，生成该子句对应的 SQL 语句。<br>
```Go
    sql, vars := generators[name](vars...)
	c.sql[name] = sql
	c.sqlVars[name] = vars
```
Build 方法根据传入的 Type 的顺序，构造出最终的 SQL 语句。<br>
```Go
for _, order := range orders {
	if sql, ok := c.sql[order]; ok {
		sqls = append(sqls, sql)
		vars = append(vars, c.sqlVars[order]...)
	}
}
return strings.Join(sqls, " "), vars
```

<br>

# CRUD

实现增删改查询操作，构造 SQL 语句的方式:<br>
1. 将input转为数据库表结构体schema
2. 多次调用 clause.Set() 构造好每一个子句。每个操作Set参数都不一样
3. 调用一次 clause.Build() 按照传入的顺序构造出最终的 SQL 语句。
4. 构造完成后，调用 Raw().Exec() 方法执行。

<br>

# 链式调用(chain)

某个对象调用某个方法后，将该对象的引用/指针返回，即可以继续调用该对象的其他方法。SQL 语句的构造过程就非常符合这个条件。SQL 语句由多个子句构成，典型的例如 SELECT 语句，往往需要设置查询条件（WHERE）、限制返回行数（LIMIT）等。<br>