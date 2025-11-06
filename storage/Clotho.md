Delta数据：刚更新的热数据，数据量小（2%）且离散
Base数据：较老的冷数据，数据量大（98%）且紧凑，blob kv分离

# compaction
1. 后台compaction： 只对delta进行compaction
> RocksDB的compaction只会在**同一个列族内部**进行文件合并和压缩：
2. 主动compaction:  只在读取时进行内存中的临时合并

注册了不同数据类型的merge算子处理Merge写入