# 介绍

[go-mysql](https://github.com/siddontang/go-mysql)是一个用go写的mysql driver，使用接口类似于go自身的database sql，但是稍微有一点不同，现阶段还不支持集成进go database/sql中，但实现难度并不大，后续可能会接入。

[go-mysql](https://github.com/siddontang/go-mysql)最先开始于[mixer](https://github.com/siddontang/mixer)（一个用go实现的mysql proxy）中，随着mixer的演化，我觉得有必要将其mysql模块独立出来使用。对于[mixer](https://github.com/siddontang/mixer)，后续我会详细介绍。

为什么要自己实现一套新的接口，而不是go自身的sql接口呢？最主要的原因在于我很不习惯使用Query的查询方式。go自身的query例子：

    age := 27
    rows, err := db.Query("SELECT name FROM users WHERE age=?", age)
    if err != nil {
        log.Fatal(err)
    }
    for rows.Next() {
        var name string
        if err := rows.Scan(&name); err != nil {
            log.Fatal(err)
        }
        fmt.Printf("%s is %d\n", name, age)
    }
    if err := rows.Err(); err != nil {
        log.Fatal(err)
    }
    
可以看到，使用起来非常的繁琐复杂，如果代码里面select语句很多（恰恰我们代码里面n多select），那么如果每次select都要靠这种方式得到我们需要的结果，那我可能写代码会写崩溃的。所以势必我们需要提供一套封装用来简化select的结果集获取。

# Resultset

[go-mysql](https://github.com/siddontang/go-mysql)跟go自身的sql接口最大的不一样在于Query的时候直接返回了一个resultset，而这个resultset定义如下：
    
    type Field struct {
        Name []byte
        Type uint8
        Flag uint16
    }

    type Resultset struct {
        Status uint16 //server status for this query resultset
        
        Fields     []Field
        FieldNames map[string]int
        
        Data [][]interface{}
    }
    
Field用来表示查询的时候返回的数据每列的名字，数据类型以及一些特定的Flag。而resultset中的Data则是存放了query结果中对于的实际数据。因为对于mysql select来说，它返回的是一个**"m x n"**的结果集，我们直接使用[][]interface{}在go中表示。

Resultset提供了非常方便的接口用于select数据的获取：

    //指定某一行，某一列获取数据，结果为string
    func (r *Resultset) GetString(row, column int) (string, error)
    //执行某一行，某一列的名字获取数据，结果为string
    func (r *Resultset) GetStringByName(row int, columnName int) (string, error)
    
# 接口

[go-mysql](https://github.com/siddontang/go-mysql)除了query之外，几乎提供了与go database/sql一样的接口使用方式：

    //创建一个db，最大允许保活16个空闲连接
    //dsn格式为：<username>:<password>@<host>:<port>/<database>
    db := NewDB("qing:admin@127.0.0.1:3306/mixer", 16)

    //ping一下，看mysql server是不是还是活的
    db.Ping()

    //执行exec，包括insert,update,delete,replace
    r, err := db.Exec("insert into mixer_conn (id, str) values (1, `abc`)")
    println(r.LastInsertId(), r.RowsAffected())

    //执行exec，语句中包含 ？占位符，需要传递相应的参数
    r, err := db.Exec("insert into mixer_conn (id, str) values (?, ?)", 2, "efg")
    println(r.LastInsertId(), r.RowsAffected())
    
    //执行select，得到结果集，并获取相关数据
    r, err := db.Query("select str from mixer_conn where id     = 1")
    str, _ = r.GetString(0, 0)
    str, _ = r.GetStringByName(0, "str")

    //开始一个事物
    tx, err = db.Begin()

    //在事物里面执行语句
    tx.Exec("insert into mixer_conn (id, str) values (3, `abc`)")

    //提交事物
    tx.Commit()

    //创建一个prepare statement
    s, err := db.Prepare("insert into mixer_conn (id, str) values(?, ?)")
    
    //执行 prepare statement
    s.Exec(5, "abc")
    
    //关闭 prepare statement
    s.Close()
    
不光如此，[go-mysql](https://github.com/siddontang/go-mysql)还提供单独获取一个conn，用于给外部额外使用的功能，譬如:

    conn, err := db.GetConn()
    
    //我需要设置conn的字符集为gb2312
    conn.SetCharset("gb2312")    
    
    conn.Close()
    
可以看到，[go-mysql](https://github.com/siddontang/go-mysql)的使用非常简单，现阶段正逐渐在我们项目中使用，也非常期待您的反馈。