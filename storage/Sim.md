数据结构修改的可行性
1. 长序列数据基本都是基本类型，1024很容易存下来

不可行性：
- 单block提供的数据信息有限，容易压缩比上不去
- 需要多种light压缩算法组合，配置管理复杂
- 由于是对整个value做操作，当把value进行切分后，就会使得存储更加复杂
- delta的数据不是严格新于base数据
	- 这导致delta数据基本都要跟base合并
	- 那delta的数据意义不是很大
	- delta单纯让写性能更高
- 原地更新问题比较大
	- delta似乎数据是最新的

实时数据

加载实时数据
https://docs.corp.kuaishou.com/d/home/fcABXQMm0WxzMqyIepUfyImvu

