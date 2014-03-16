[go-mysql](https://github.com/siddontang/go-mysql)已经支持golang database/sql接口，并通过[https://github.com/bradfitz/go-sql-test](https://github.com/bradfitz/go-sql-test)测试用例。

现在[go-mysql](https://github.com/siddontang/go-mysql)可以直接通过golang sql接口使用，如下：

	import _ "github.com/siddontang/go-mysql/mysql"
	import "database/sql"
	
后续的使用，可以直接参考相关golang sql的教程，譬如[这个](http://go-database-sql.org/index.html)。

golang sql接口的兼容主要在driver.go的文件中，

[go-mysql](https://github.com/siddontang/go-mysql)支持的dsn格式为：

    <username>:<password>@<host>:<port>/<database>
    
因为在实现[go-mysql](https://github.com/siddontang/go-mysql)的过程中，我就有意识的将一些接口设计成能跟database/sql进行适配。除了Rows接口的适配，因为我总觉得使用起来不方便，但是通过Resultset进行适配Rows也很方便。因为Resultset存储了Query之后所有的数据，所以我们可以通过一个int型iterator，就可以模拟Rows接口：
    
    func (r *mysqlRows) Close() error {
        r.iter = -1
        return nil
    }
    
    func (r *mysqlRows) Next(dest []driver.Value) error {
        if r.iter >= r.r.RowNumber() {
            return io.EOF
        }
    
        data := r.r.Data[r.iter]
        
        for i := range data {
            //set data[i] to dest[i]
        }
    
        r.iter++
        return nil
    }