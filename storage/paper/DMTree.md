DMTree相比Sherman的核心创新在于：

1. **指纹表技术**：通过FPTable快速定位，避免读取整个页面
	1. FPTable存储每个键的指纹（fingerprint），用于快速定位叶子节点中的具体Entry位置：
2. **计算端协同**：将索引辅助结构放在计算端，减少网络通信
	1. 指FPTable
3. **多级缓存**：本地FP缓存 + DMCache + KeyCache的三级缓存架构
4. **写合并机制**：减少重复写入，提高写性能
