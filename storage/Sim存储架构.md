# 存储数据
长序列数据，<K,V>
## 存储介质
AEP/SSD/SHM
## 数据类型
```cpp
field_types = {
    'int8_t': 1,
    'uint8_t': 1,
    'int16_t': 2,
    'uint16_t': 2,
    'int32_t': 4,
    'uint32_t': 4,
    'int64_t': 8,
    'uint64_t': 8,
    'float': 4,
    'double': 8,
    'char': 1
}
```

# 数据存储
- Base： 存在SSD(目前存在AEP/Mem中)
- Delta:    存在内存中
 一个 partition 对应一个 MultiMemKV，其中一个 part 最多存多少 key
 multimemkv包含多个slab，每个slab大小不同

## Delta
使用行式存储
Delta数据满时（达到水位线），触发rerange,然后delta合并base并且清空delta数据

# Base
使用列式存储，大批量的value压缩后，写入
- 越大的value 压缩

# 单边特征存储
只使用Delta层存储


# 请求特征
1. 大部分不会读取全部的列
2. 