
![image.png|400](https://cloudflare-imgbed-4ea.pages.dev/file/paper/database/1760782447402_image.png)
有个知名梗图（etcd自己的仓库上的），能看出etcd的重要性。 是一个开源的、分布式的、强一致性的**键值存储系统**，一般来说主要跟zookeeper进行对比。
- etcd采用Raft
- 支持KV操作（提供接口）
其他的，还有 go语言实现
# etcd KV实现：boltdb
一个嵌入式的KV数据库
- 采用B+树实现，数据放入一个文件中
- 
