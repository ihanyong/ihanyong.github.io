zookeeper.md

# 
## DataModel
hierrchal name space, like a file sysytem, each node can have data

ZNodes
    - stat structure
        - version
        - acl
        - timestamps
    - Wathces
    - Data Access
    - Ephemerl Nodes
    - Sequence Nodes -- Unique Naming
    - Conatiner Nodes
    - TTL Nodes

Time in Zookeeper
- Zxid
- Version numbers
- Ticks
- Real time

Zookeeper Stat Structure
- czxid
- mzxid
- pzxid
- ctime
- mtime
- verion
- cversion
- aversion
- ephemeralOwner
- DataLenght
- numChildren







## Sessions
## Watches
## Consistency Guarrantees


## Operations
## Bindings
## Program Structure
## Gotchas: Common Problems and Troubeshooting


### 原理
https://www.cnblogs.com/buyuling/p/7265943.html

https://www.cnblogs.com/stateis0/p/9062133.html


ZAB 协议
Raft 协议

https://www.cnblogs.com/stateis0/p/9062123.html