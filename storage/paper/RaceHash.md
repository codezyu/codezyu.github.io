基于Dash 
CoalescedHash 中，哈希表 bucket 被编号为1-N，其中前 M 个 bucke t用于哈希函数的可寻址区域，剩余的 N-M 个 bucket 专门用于存放冲突记录（后称为Cellar Region）。当 Cellar Region 被占满时，后续的冲突记录必须占用前 M 个可寻址区域的空 bucket

  - sephash
写入性能差
- adaptive hash


- batch
- index data