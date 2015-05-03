# Use Hashicorp Raft to build a Redis sentinel

## Redis Sentinel

We use Redis not only for cache, but also storing important data, and we build up a Master/Slave replication topology to guarantee data security.

Master/Slave architecture works well, but sometimes we need a more powerful high availability solution. If master is down, we must check this immediately, reselect a new master from the slaves and do failover.

The official Redis supplies a solution named redis-sentinel, which is very powerful to use. But I still want to build my own sentinel solution, why?

+ I want to monitor not only Redis but also LedisDB, maybe other services using Redis serialization Protocol too.
+ I want to embed it into xcodis or other go service easily.
+ I want to study some consensus algorithms and use them in practice.
Sentinel Cluster and Election

Building a single sentinel application is easy: checking master every second, and do failover when master is down. But if the sentinel is down too, how do we do?

Using sentinel cluster is a good choice, if one sentinel is down, other sentinel will still work. But let’s consider below scenario, if two sentinels in the cluster both see the master is down, and do failover at same time, sentinel1 may select slave1 as master, but sentinel2 may select slave2 as master, this may be a horrible thing for us.

A common use way is to elect a leader sentinel in the cluster and let it monitor and do failover. So we need a consensus algorithm helping us do this thing.

## Paxos and Raft

Paxos may be the most famous consensus algorithm in the world, many companies use it in their distributed system. However, Paxos is very hard to understand and if you write a paxos lib by yourself, you even cann’t testify its correctness easily. Luckly, we have zookeeper, an open source centralized service based on Paxos. We can use zookeeper to manage our clusters like electing a leader.

Raft was born on 2013 in Stanford, it’s very new but awesome. Raft is easy to understand, everyone reading the Raft paper can write its own Raft implentation easily than Paxos. Now many projects use Raft, like Etcd, InfluxDB, CockroachDB, etc…

The above projects I list using Raft all use Go, and I will develop my own redis sentinel with Go too, so I decide to use Raft.

## Use Hashicorp Raft

There are some Go raft projects, but I prefer Hashicorp Raft which is easy to be integrated in other project, and this package is used in Consul product and has already been tested in production environment (maybe!).

The create raft function declaration is below:

```
func NewRaft(conf *Config, fsm FSM, logs LogStore, stable StableStore, snaps SnapshotStore, peerStore PeerStore, trans Transport) (*Raft, error)
```

Although it looks a little complex, it’s still easy to use, we only need do following things:

+ Create a configuration using raft own DefaultConfig function. We should know that raft should be used with at least three nodes, but if we just want to try it with only one node, or first start a raft node, than add others later, we must set EnableSingleNode to true.
+ Define our own FSM struct, FSM is a state machine applying replicated log, generating point-in-time snapshot, and restoring from a snapshot. In our sentinel, the only data need to care is all Redis masters, whenever we add a master, remove a master or reset all masters, we should let all sentinels know. So my FSM struct is very easy, like below:

```
type masterFSM struct {
    sync.Mutex
    
    // below holding all Redis master addresses
    masters map[string]struct{}
} 
```

+ Define our own FSMSnapshot struct. In our sentinel, this is a list of masters at some point. The struct like this:

```
type masterSnapshot struct {
    masters []string
}
```

+ Create a log storage storing and retrieving logs and a stable storage storing key configurations. Hashicorp supplies a LMDB lib and a BoltDB lib for both storage, we use BoltDB because of the pure Go implementation.
+ Create a snapshot storage saving FSM snapshot, we use raft own NewFileSnapshotStore generating a file saving this.
+ Create a peer storage storing all raft nodes, we use raft own NewJSONPeers generating a file saving all nodes with JSON format.
+ Create a transport allowing a raft node to communicate with other nodes, we use raft own NewTCPTransport generating a TCP transport.

After do that, we can create a raft, we can use LeaderCh and Leader function to check whether a raft node is leader or not. Only the leader node can handle operations. If the leader is down, raft can re-elect a new leader.

You can see the source [here](https://github.com/siddontang/redis-failover/blob/master/failover/raft.go) for more information.

## Summary

Our redis sentinel is named redis-failover, although it looks a little simple and needs improvement, it still the first trial and later we will use raft in more projects, maybe instead of zookeeper.

redis-failover: [https://github.com/siddontang/redis-failover](https://github.com/siddontang/redis-failover)