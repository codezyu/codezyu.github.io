3FS是高性能的分布式文件系统，场景用于AI训练和推理，存储主要是通过SSD和RDMA实现
# 分布式文件系统实现

Cluster Manager
	- 相当于中心化的节点管理
MetaService
	- 提供元数据服务 以KV存储的方式在FoundationDB 
	- 这个用法好像比较常见，在我之前看的文章里面`singularfs: a billion-scale distributed file system using a single metadata server`也有类似的做法
Storage Service
    - 提供块存储服务
Client
	- fuse客户端，原生客户端
### 
