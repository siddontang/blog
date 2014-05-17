ledisdb现在可以支持嵌入式使用。你可以将其作为一个独立的lib（类似leveldb）直接嵌入到你自己的应用中去，而无需在启动单独的服务。

ledisdb提供的API仍然类似redis接口。首先，你需要创建db对象：

    import "github.com/siddontang/ledisdb/ledis"
    
    configJson = []byte('{
        "data_db" : 
        {
            "path" : "/tmp/testdb",
            "compression":true,
            "block_size" : 32768,
            "write_buffer_size" : 2097152,
            "cache_size" : 20971520
        }    
    }
    ')
    
    db, _ := ledis.OpenDB(configJson)
    
data_db就是数据存储的leveldb位置，简单起见，所有的size配置全部使用byte作为单位。
    
下面是一些简单的例子：

## kv

    db.Set(key, value)
    db.Get(key)
    db.SetNX(key, value)
    db.Incr(key)
    db.IncrBy(key, 10)
    db.Decr(key)
    db.DecrBy(key, 10)

    db.MSet(KVPair{key1, value1}, KVPair{key2, value2})
    db.MGet(key1, key2)

## list

    db.LPush(key, value1, value2, value3)
    db.RPush(key, value4, value5, value6)
    
    db.LRange(key, 1, 10)
    db.LIndex(key, 10)
    
    db.LLen(key)

## hash

    db.HSet(key, field1, value1)
    db.HMSet(key, FVPair{field1, value1}, FVPair{field2, value2})
    
    db.HGet(key, field1)
    
    db.HGetAll()
    db.HKeys()

## zset

    db.ZAdd(key, ScorePair{score1, member1}, ScorePair{score2, member2})
    
    db.ZCard(key)
    
    //range by score [0, 100], withscores = true and no limit
    db.ZRangeByScore(key, 0, 100, true, 0, -1)

    //range by score [0, 100], withscores = true and limit offset = 10, count = 10
    db.ZRangeByScore(key, 0, 100, true, 10, 10)
    
    db.ZRank(key, member1)
    
    db.ZCount(key, member1)
    

ledisdb的源码在这里[https://github.com/siddontang/ledisdb](https://github.com/siddontang/ledisdb)，欢迎反馈。
