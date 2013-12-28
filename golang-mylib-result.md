go提供了一套统一操作database的sql接口，任何第三方都可以通过实现相应的driver来访问感兴趣的数据库。譬如我们项目中使用的[Go-MySQL-Driver](https://github.com/go-sql-driver/mysql)。

go提供了一套很好的机制来处理数据库的查询操作，譬如官方的例子：

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
    
在上面的例子中，Query会返回一个rows，我们通过Next获取下一行数据，然后使用Scan将每行的数据设置到相应的变量上面去。

虽然这样操作查询结果集很方便，但是我们在使用过程中，发现当sql query语句过多，如果每一个查询都按照这种写法，代码量太大了。所以，我们自然就想到了封装。

db result主要进行的工作就是提供一套统一的接口供外部方便的使用查询的结果集，它主要提供了如下几个接口：

    func (*Result) GetInt(row, col int) (int64, error)
    func (*Result) GetIntByName(row int, colName string) (int64, error)
    
    func (*Result) GetFloat(row, col int) (float64, error)
    func (*Result) GetFloatByName(row int, colName string) (float64, error)
    
    func (*Result) GetBool(row, col int) (bool, error)
    func (*Result) GetBoolByName(row int, colName string) (bool, error)
   
    func (*Result) GetString(row, col int) (string, error)
    func (*Result) GetStringByName(row int, colName string) (string, error)
    
    func (*Result) GetBuffer(row, col int) ([]byte, error)
    func (*Result) GetBufferByName(row int, colName string) ([]byte, error)
    
可以看到，result的“Get\*”接口通过结果集的row和col的索引来访问数据，而"Get\*ByName"接口则是通过row以及col的名字来访问数据。

因为go支持很多的数据类型，为了简单起见，所有int类型的我们统一使用int64代替，外部在进行相应的数据转换。同理float类型也是用float64代替。如果查询的结果某个字段为空，result只是返回该字段默认的数值。通常情况下，我都会要求数据库中的字段都为not null，所以查询字段为null的情况这里没有过多考虑。

一个很简单的例子：
    
    //msg_id bigint
    //msg varchar(256)
    res, err := Query("is", "select msg_id, msg from msg_table where msg_id in (?, ?)", 1, 2)
    
    var id1 int64
    id1, err = res.GetInt(0, 0)
   
    var msg2 string
    msg2, err = res.GetStringByName(1, "msg")
    
在上面的例子中，我们在查询的时候返回了一个result，然后通过相关的函数获取到了相应的信息。这里我们需要特别注意的就是Query后面的第一个参数**“is”**。

在[Go-MySQL-Driver](https://github.com/go-sql-driver/mysql)中，会两种结果集模式，一个是text rows，另一个则是binary rows（使用stmt query的结果集），在text rows中，所有的数据都是[]byte格式，而在binary rows中，则会根据stmt对应的结果类型转换成相应的数据。为了统一，我们通过在query的时候手动指定column types，在row scan的时候直接创建对应的类型数据，供scan设置。如下：

    func (res *Result) newValue(column int) (interface{}, error) {
        t := res.ColumnTypes[column]
    
        switch t {
        case Column_String:
            return new(string), nil
        case Column_Int:
            return new(int64), nil
        case Column_Bool:
            return new(bool), nil
        case Column_Float:
            return new(float64), nil
        case Column_Buffer:
            return new([]byte), nil
        default:
            return nil, ErrColumnTypes
        }
    }

    //遍历结果集的时候，我们根据column types生成指定的value，并通过Scan设置
    
    for rows.Next() {
        dest := make([]interface{}, len(res.ColumnNames))

        for i, _ := range dest {
            dest[i], err = res.newValue(i)
            if err != nil {
                return err
            }
        }

        err = rows.Scan(dest...)
        if err != nil {
            return err
        }
  
 如果大家使用过mysql_stmt_bind_result，就可以发现，column types的概念其实就跟设置MYSQL_BIND差不多，只是result为了简单，只支持int64，float64，bool，string，以及[]byte这几种类型。
 
 具体的代码在这里[https://github.com/siddontang/qpush/blob/master/go/src/lib/db/result.go](https://github.com/siddontang/qpush/blob/master/go/src/lib/db/result.go)。