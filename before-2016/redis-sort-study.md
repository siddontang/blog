[LedisDB][1]本来是没有sort命令的，而且实际我们也没有使用过该命令，但一位用户给我反应他迫切需要这个功能，我决定首先考察一下redis相关的实现，再看是否提供。

然后我一看redis的sort命令，真的是震惊了，这可能算得上redis里面最复杂的一个命令了，命令原型如下：

```
SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination]
```

如果不仔细看文档，或者看源码，一下子真的不知道这个命令怎么用。首先我们可以去掉LIMIT offset count这个选项，这个很容易理解，就是排好序之后取偏移数据。ASC和DESC这个也比较容易，就是正向和逆向排序。STORE destination这个其实就是将排好序的数据放到destination这个list里面，也比较容易理解。好了，去掉这些，那么sort的原型就是这个样子了:

```
SORT key [BY pattern] [GET pattern [GET pattern ...]] [ALPHA]
```

key里面存储的就是需要排序的东西，所以key只能是list，set或者zset类型，我们以list为例。假设做如下操作：

```
redis> lpush a 1 2 3
redis> lrange a 0 -1
1) "3"
2) "2"
3) "1"
```
如果使用sort，则排序结果如下:

```
redis> sort a
1) "1"
2) "2"
3) "3"
```

呢么ALPHA是什么意思呢？我们可以做如下操作解释：

```
redis> lpush b a1 a2 a3
redis> sort b
(error) ERR One or more scores can't be converted into double
redis> sort b alpha
1) "a1"
2) "a2"
3) "a3"
```

我们在b里面压入的是字符串，所以不能直接sort，必须指定alpha方式。所以alpha就是明确告知sort使用字节序排序，不然sort就会尝试将需要排序的数据转成double类型。

理解了alpha，我们再来看看by的含义，如下例子：

```
redis> set w_1 3
redis> set w_1 30
redis> set w_2 20
redis> set w_3 10
redis> sort a by w_*
1) "3"
2) "2"
3) "1"
127.0.0.1:6379>
```

如果有by了，sort就会首先取出对应的数据，也就是1，2，3，然后跟by的pattern进行组合，变成w_1，w_2，w_3，然后以这个作为key去获取对应的值，也就是30，20，10，在按照这些值进行排序。上面这个例子，1对应的by值最大，为30，所以升序排列的时候在最后。

说完了by，我们再来说说get，get是不参与排序的，只是在拍完序之后，将排好序的值依次跟get的pattern组合，获取对应的数据，进行返回，如下例子：

```
redis> set o_1 10
redis> set o_2 20
redis> set o_3 30
redis> sort a get o_*
1) "10"
2) "20"
3) "30"
```

再来一个多个get的例子：

```
redis> set oo_1 100
redis> set oo_2 200
redis> set oo_3 300
redis> sort a get o_* get oo_*
1) "10"
2) "100"
3) "20"
4) "200"
5) "30"
6) "300"
```

从上面可以看到，如果有多个get，那么sort的做法是对于排好序的一个值，依次通过get获取值，放到结果中，然后在处理下一个值。

如果有get，我们就能获取到相关的值，但这时候我们还需要返回原有的值怎么办？只需要`get #`就成了，如下：

```
redis> sort a get o_* get #
1) "10"
2) "1"
3) "20"
4) "2"
5) "30"
6) "3"
```

好了，折腾了这么久，我算是终于理解了sort的原理，然后在看完sort的实现，依葫芦画瓢在[LedisDB][1]里面支持了sort。当然在一些底层细节上面还是稍微跟redis不一样的。

[1]: https://github.com/siddontang/ledisdb "A fast NoSQL"