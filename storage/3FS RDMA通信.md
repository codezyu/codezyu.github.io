为了实现和TCP统一的接口，使用Socket的抽象接口
```cpp
class Socket {
 public:
  virtual ~Socket() = default;

  // describe this socket.
  virtual std::string describe() = 0;
  virtual folly::IPAddressV4 peerIP() = 0;

  // file descriptor monitored by epoll.
  virtual int fd() const = 0;

  using Events = uint32_t;
  constexpr static auto kEventReadableFlag = (1u << 0);
  constexpr static auto kEventWritableFlag = (1u << 1);
  // poll read and/or write events for this socket.
  virtual Result<Events> poll(uint32_t events) = 0;

  // receive a piece of buffer. return the received size.
  // 这里的folly::MutableByteRange 相当于只是提供了视图类似string_view 但是可变
  virtual Result<size_t> recv(folly::MutableByteRange buf) = 0;
  // send a batch of buffers. return the send size.
  virtual Result<size_t> send(struct iovec iov[], uint32_t len) = 0;
  // flush the write buffers.
  virtual Result<Void> flush() = 0;
  // check the liveness of the socket.
  virtual Result<Void> check() = 0;
};
```
而继承该Socket的IBSocket增加了RDMA读写接口
```cpp
/** RDMA operations. */
  CoTryTask<void> rdmaRead(const RDMARemoteBuf &remoteBuf, RDMABuf &localBuf) {
    co_return co_await rdmaRead(remoteBuf, std::span(&localBuf, 1));
  }
  CoTryTask<void> rdmaRead(const RDMARemoteBuf &remoteBuf, std::span<RDMABuf> localBufs) {
    co_return co_await rdma(IBV_WR_RDMA_READ, remoteBuf, localBufs);
  }

  CoTryTask<void> rdmaWrite(const RDMARemoteBuf &remoteBuf, RDMABuf &localBuf) {
    co_return co_await rdmaWrite(remoteBuf, std::span(&localBuf, 1));
  }
  CoTryTask<void> rdmaWrite(const RDMARemoteBuf &remoteBuf, std::span<RDMABuf> localBufs) {
    co_return co_await rdma(IBV_WR_RDMA_WRITE, remoteBuf, localBufs);
  }
```

## 建联过程
先通过TCP建联
设备上，是支持IB和ROCE两种链路
- 网口上是轮询使用
最后，发条空消息触发连接完成
### 连接数量限制
QP默认配置中，对于每个地址默认QP数为1
```
static constexpr uint32_t kDefaultMaxConnections = 1;
```
### 内存拷贝
使用IBSocket的Send则会直接拷贝内容到Send buf
```
memcpy(sendBuf.data(), buf.data(), wsize);
```