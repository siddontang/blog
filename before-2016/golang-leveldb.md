leveldb是一个很强悍的kv数据库，自然，我也希望能在go中使用。

如果有官方的go leveldb实现，那我会优先考虑，譬如[这个](https://code.google.com/p/leveldb-go/)，但是该库文档完全没有，并且在网上没发现有人用于实战环境，对其能否在生产环境中使用打上问号，保险起见，我还是决定不使用。

因为leveldb有c的接口，所以通过cgo能很方便的进行集成，所以我决定采用该种方式，幸运的是，已经有人做了cgo的版本，也就是[levigo](https://github.com/jmhodges/levigo)。

使用levigo，需要编译安装leveldb，如果需要压缩支持还需要编译snappy，为了简单，我写了一个构件脚本，如下：

    #!/bin/bash
    #refer https://github.com/norton/lets/blob/master/c_src/build_deps.sh

    #你必须在这里设置实际的snappy以及leveldb源码地址
    SNAPPY_SRC=./snappy
    LEVELDB_SRC=./leveldb

    SNAPPY_DIR=/usr/local/snappy
    LEVELDB_DIR=/usr/local/leveldb

    if [ ! -f $SNAPPY_DIR/lib/libsnappy.a ]; then
        (cd $SNAPPY_SRC && \
            ./configure --prefix=$SNAPPY_DIR && \
            make && \
            make install)
    else
        echo "skip install snappy"
    fi

    if [ ! -f $LEVELDB_DIR/lib/libleveldb.a ]; then
        (cd $LEVELDB_SRC && \
            echo "echo \"PLATFORM_CFLAGS+=-I$SNAPPY_DIR/include\" >> build_config.mk" >> build_detect_platform &&
            echo "echo \"PLATFORM_CXXFLAGS+=-I$SNAPPY_DIR/include\" >> build_config.mk" >> build_detect_platform &&
            echo "echo \"PLATFORM_LDFLAGS+=-L $SNAPPY_DIR/lib -lsnappy\" >> build_config.mk" >> build_detect_platform &&
            make SNAPPY=1 && \
            make && \
            mkdir -p $LEVELDB_DIR/include/leveldb && \
            install include/leveldb/*.h $LEVELDB_DIR/include/leveldb && \
            mkdir -p $LEVELDB_DIR/lib && \
            cp -af libleveldb.* $LEVELDB_DIR/lib)
    else
        echo "skip install leveldb"
    fi

    function add_path()
    {
      # $1 path variable
      # $2 path to add
      if [ -d "$2" ] && [[ ":$1:" != *":$2:"* ]]; then
        echo "$1:$2"
      else
        echo "$1"
      fi
    }

    export CGO_CFLAGS="-I$LEVELDB_DIR/include -I$SNAPPY_DIR/include"
    export CGO_LDFLAGS="-L$LEVELDB_DIR/lib -L$SNAPPY_DIR/lib -lsnappy"
    export LD_LIBRARY_PATH=$(add_path $LD_LIBRARY_PATH $SNAPPY_DIR/lib)
    export LD_LIBRARY_PATH=$(add_path $LD_LIBRARY_PATH $LEVELDB_DIR/lib)

    go get github.com/jmhodges/levigo 

对于leveldb在go里面的使用，levigo做了很好的封装，但是有一点我不怎么习惯，在leveldb中，对于read和write的操作，都需要传入一个Option的东西，这玩意大多数时候都是一个默认Option对象，没必要每次在go里面进行创建删除。所以我对其进行了封装，提供了如下的接口，这样使用的都是默认的option。

    func (db *DB) Put(key, value []byte) error 
    func (db *DB) Get(key []byte) ([]byte, error)
    func (db *DB) Delete(key []byte) error 
    
同时对于iterator，我参考c++的模型，提供了iterator以及reverse_iterator两种模式，如下：

    func (db *DB) Iterator(begin []byte, end []byte, limit int) *Iterator 
    func (db *DB) ReverseIterator(rbegin []byte, rend []byte, limit int) *Iterator 

具体的代码在[这里](https://github.com/siddontang/golib/tree/master/leveldb)。
