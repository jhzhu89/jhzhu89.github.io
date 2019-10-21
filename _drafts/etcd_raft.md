raft

raft这个结构实现了状态机

rawnode封装了raft来实现Node这个interface

node进一步封装rawnode，从而可以自己run？？



每个server节点在收到请求，propose给raft的时候，会用id来注册请求，这个id是由idutil.Generator来生成的，保证在每个节点上的递增。

当applier收到请求，将commit的entry apply了之后，会trigger，server节点就会知道这个id的请求已经在这个节点上完成，这时候就可以返回结果了。

请求是被设置了超时时间的，所有就保证了不会在极端情况下一直hang