# Thread 
## 生产者 Actor 与 消费者 Actor 之间消息传递，msg 的流动轨迹
![image](https://user-images.githubusercontent.com/31394900/125795762-ae38987f-8574-4687-987c-dacb22ea08be.png)

当 Actor a 给 Actor b 发消息时，会判断 Actor b 是否在当前线程内，<br>
如果是，则将 Actor a 发送的消息直接压入当前 Thread 的 Local Msg Queue 中；<br
否则 Actor a 将消息发送给当前机器的 Actor Msg Bus （每个 Machine 都有自己的一个 Actor Msg Bus），Actor Msg Bus 判断接收者的 Actor 的 Thread 是否在本机上，<br>
如果是，则 Actor Msg Bus 会在本机上找到该 Thread，将消息传给该 Thread 的 Msg channel。<br>
如果不是，Actor a 将 msg 发送给本机 Actor Msg Bus，Actor Msg Bus 再把 msg 发送给本机传输数据的 Common Net，由 Actor b 机器的 Common Net 接收 msg，
传递给接收机的 Actor Msg Bus，Actor Msg Bus 再找到接收 Actor 的 Thread，将消息压入该 Thread 的 Msg channel 消息队列中。<br>

## 术语：
*  Actor Msg Bus ：每个 Machine 都有自己的一个 Actor Msg Bus，当同一机器跨 Thread 或不同机器上传递 msg 时，需要 Actor Msg Bus 传递数据
*  Local Msg Queue ：本机上发送 Actor 所在的当前 Thread 中存放 msg 的队列容器
* Msg channel : 本机上跨线程的 Thread 或不同机器上的 Thread 中存放 msg 的队列容器
* Common Net : 跨机传输 msg 的 “班车”，由发送/接收 Actor 所在机器的 Actor Msg Bus 负责与 Common Net  对接 msg 数据

## 总结：
生产者 Actor 与 消费者 Actor 之间消息传递，msg 流动轨迹总共分为 3 种情形：
1. 发送 Actor 与 接收 Actor 同 Thread: 将 msg 直接压入当前 Thread 的 Local Msg Queue 消息队列中，即发送 Actor -> msg -> 压入本机 Local Msg Queue 消息队列中。<br>
2. 同机不同 Thread: 发送 Actor -> msg -> 本机 Actor Msg Bus -> 找到接收 Actor 的 Thread -> 压入 Msg channel 消息队列中。<br>
3. 不同机器不同 Thread: 发送 Actor -> msg -> 本机 Actor Msg Bus -> 本机 Common Net -> 接收机 Common Net -> 接收机 Actor Msg Bus -> 接收 Thread -> 压入 Msg channel 消息队列中<br>

## Actor msg 的消费
从当前 Thread 的 Local Msg Queue 消息队列中取 msg，如果 Local Msg Queue 队列为空，则从 Msg channel 中取。

Thread 类成员：
每个 Thread 内部都有一个轮询线程 actor_thread_，负责轮询消息队列 PollMsgChannel
std::thread actor_thread_ ：轮询消息队列的线程

void PollMsgChannel(const ThreadCtx& thread_ctx); // 轮询消息队列 PollMsgChannel

Thread 类做了啥？<br>

thread.h<br>
声明了存储本线程 TaskProto 和 Actor 的 HashMap 容器，都与线程 id 绑定，如 id2task_、id2actor_ptr_
声明了本线程需要处理的消息队列，如 local_msg_queue_、msg_channel_
声明了本线程的轮询消息队列的线程，如 actor_thread_



```.cpp
namespace oneflow {

class Thread {
 public:
  OF_DISALLOW_COPY_AND_MOVE(Thread);
  virtual ~Thread();

  void AddTask(const TaskProto&);

  Channel<ActorMsg>* GetMsgChannelPtr() { return &msg_channel_; } // 获取消息队列
  void EnqueueActorMsg(const ActorMsg& msg); // 将 ActorMsg 压入消息队列中

  void JoinAllActor() { actor_thread_.join(); } // 启动本线程的轮询线程，阻塞主线程

 protected:
  Thread() = default;
  std::thread& mut_actor_thread() { return actor_thread_; } // 返回该 thread 的轮询线程
  void PollMsgChannel(const ThreadCtx& thread_ctx); // 轮询消息队列 PollMsgChannel
  void set_thrd_id(int64_t val) { thrd_id_ = val; }

 private:
  void ConstructActor(int64_t actor_id, const ThreadCtx& thread_ctx); // 创建接收消息的 Actor

  HashMap<int64_t, TaskProto> id2task_; // 保存本线程多个 TaskProto
  std::mutex id2task_mtx_; // 本线程的互斥量

  // 每个 Thread 内部都有一个轮询线程 actor_thread_，负责轮询消息队列 PollMsgChannel
  std::thread actor_thread_; // 轮询消息队列的线程
  Channel<ActorMsg> msg_channel_; // 消息队列，接收跨线程的 ActorMsg
  HashMap<int64_t, std::unique_ptr<Actor>> id2actor_ptr_; // 保存本线程的多个 Actor，与 id2task_ 中的多个 TaskProto 对应，一个 Actor 接收 一个 TaskProto
  std::queue<ActorMsg> local_msg_queue_; // 消息队列，接收本线程的 ActorMsg

  int64_t thrd_id_;
};

}  // namespace oneflow

#endif  // ONEFLOW_CORE_THREAD_THREAD_H_

```
成员变量：
1. 申请保存本线程的多个 TaskProto 的 HashMap 容器：HashMap<int64_t, TaskProto> id2task_;
2. 申请本线程的多线程互斥量：std::mutex id2task_mtx_;
3. 申请本线程的轮询消息队列的线程：std::thread actor_thread_;
4. 申请本线程接收跨线程 ActorMsg 的队列容器：Channel<ActorMsg> msg_channel_; 
5. 申请保存本线程处理多个 Actor 的 HashMap 容器：HashMap<int64_t, std::unique_ptr<Actor>> id2actor_ptr_;
   本线程中的多个 Actor 与 id2task_ 中的多个 TaskProto 一一对应。
6. 申请消息队列，接收本线程的 ActorMsg：std::queue<ActorMsg> local_msg_queue_;

成员方法：
1. 
 
 
 
 ## ThreadPool <br>
	ThreadPool 类定义了一个存储所有任务队列的 vec 容器，和与任务队列一一对应的线程 vec 容器。
	每个任务队列里存储了多个任务 work，每个线程负责处理对应的一个任务队列，一个任务队列包含多个任务 work。

	ThreadPool 类还定义了往队列里添加任务的行为：将所有任务平均地分配到各个队列（将当前任务 work 写入分配的对应队列中）

 * 一个队列容器 vec 存储多个队列<br>
 * 一个线程容器 vec 存储多个线程<br>
 * 线程与队列一一对应，一个线程负责一个队列<br>
 * 一个线程完成一个队列<br>
 * 一个队列存储多个任务<br>
 
 thread_pool.h
 ```.h
	namespace oneflow {

	class ThreadPool final {
	 public:
		OF_DISALLOW_COPY_AND_MOVE(ThreadPool);
		ThreadPool() = delete;

		/**
		遍历 thread_num = 48 个线程，从任务队列的 vec 容器 ThreadPool::work_chans_ 中取出每个线程对应的任务队列 queue_,
		为执行当前任务队列 chan 中的任务 work 创建线程 std::thread，然后将该线程新增到线程 vec 的容器成员 threads_ 中，
		新创建的线程内容为从任务队列 chan 中取出任务 work，并执行 work
		*/
		ThreadPool(int32_t thread_num); 
		~ThreadPool();

		int32_t thread_num() const { return threads_.size(); } // 获取线程的总数，也是任务队列总数
		// 将所有任务 work 均匀地分配到（写入）（48个）任务队列中
		void AddWork(const std::function<void()>& work); // 添加任务，每个任务都放在函数模版类 std::function 中

	 private:
	 /**
	 threads_ 存储多个线程的容器 vec 
	 work_chans_ 存储多个任务队列的容器 vec，这里的一个任务队列对应 threads_ 中的一个线程

	 */

		// 分别声明了队列的容器 vec、线程容器 vec，一个线程管理一个队列，所以多个待完成任务的队列，与threads_中的工作线程一一对应
		std::vector<Channel<std::function<void()>>> work_chans_; // 存放任务队列的 vec 容器，与 threads_ 中的工作线程对应
		std::vector<std::thread> threads_; // 多个工作线程容器 vec

		std::atomic<size_t> work_cnt_; // 任务总数
	};

	}  // namespace oneflow

	#endif  // ONEFLOW_CORE_THREAD_THREAD_POOL_H_

 ```
	
 thread_pool.cpp
 ```.cpp
 #include "oneflow/core/thread/thread_pool.h"

	namespace oneflow {

	ThreadPool::ThreadPool(int32_t thread_num)
			: work_chans_(thread_num), threads_(thread_num), work_cnt_(0) {
		FOR_RANGE(int32_t, i, 0, thread_num) { // thread_num = 48，有 48 个线程
		/**
		work_chans_：任务队列 vec 容器，存放任务队列（queue_）元素
		当前线程池处理 thread_num = 48 个线程，每个线程对应一个任务队列 queue_
		*/
			Channel<std::function<void()>>* chan = &(work_chans_.at(i)); // 从队列容器中取出当前任务队列 queue_，chan：queue_
			threads_[i] = std::thread([chan]() { // 为执行当前任务队列 chan 中的任务 work 创建线程 std::thread，然后将该线程新增到线程 vec 的容器 threads_ 中
				std::function<void()> work;
				while (chan->Receive(&work) == kChannelStatusSuccess) { work(); } // 从任务队列 chan 中取出任务 work，并执行 work
			});
		}
	}

	ThreadPool::~ThreadPool() {
		FOR_RANGE(int32_t, i, 0, work_chans_.size()) { // 遍历任务队列 vec 容器  work_chans_
			work_chans_.at(i).Close(); // 一个任务队列使用完毕
			threads_.at(i).join();     // 线程 vec 容器中对应的新增线程 threads_[i] 启动，主线程阻塞。
		}
	}

	/**
	将所有 work 均匀地分配到（写入） work_chans_.size() 个队列中：当前任务队列的索引 = 任务总数 % 任务队列总数

	一个队列容器 vec 放多个队列
	一个线程容器 vec 放多个线程
	线程与队列一一对应
	一个线程放一个队列
	一个队列放多个任务
	*/
	void ThreadPool::AddWork(const std::function<void()>& work) {
		const size_t cur_chan_idx = // 当前 work 所分配到的当前任务队列的索引，cur_chan 指当前任务队列，一个任务队列有多个任务 work
				work_cnt_.fetch_add(1, std::memory_order_relaxed) % work_chans_.size(); // work_chans_.size()：任务队列总数
		work_chans_.at(cur_chan_idx).Send(work); // 将待完成的任务写入任务队列 vec 容器中，按索引顺序
		// work_chans_.at(cur_chan_idx) 在队列容器 vec 中，找到当前任务 work 对应的队列
		// .Send(work) 再将当前任务 work 写入队列
	}

	}  // namespace oneflow

 ```
  
 
 
 
 
 
 ## Common 里的工具类——线程队列<br>
 channel.h<br>
 线程队列主要维护了一个带线程的队列 std::queue<T>，包括了入队、出队、整个队列转移到新队列的方法。<br>
 ```.cpp
 namespace oneflow {

enum ChannelStatus { kChannelStatusSuccess = 0, kChannelStatusErrorClosed };

template<typename T>
class Channel final {
 public:
  OF_DISALLOW_COPY_AND_MOVE(Channel);
  Channel() : is_closed_(false) {}
  ~Channel() = default;

  ChannelStatus Send(const T& item);
  ChannelStatus Receive(T* item);
  ChannelStatus ReceiveMany(std::queue<T>* items);
  void Close();

 private:
  std::queue<T> queue_; // 队列
  mutable std::mutex mutex_; // 互斥量
  bool is_closed_;
  std::condition_variable cond_; // 条件变量，使用 wait，unique_lock 用到
};

/**
写数据：
Channel<T>::Send(const T& item)
1. 将 Channel::mutex_ 上锁
2. 将 item 元素压入队列 Channel::queue_
3. 释放锁，条件变量 Channel::cond_ 唤醒 wait 的线程，释放 mutex_
*/
template<typename T>
ChannelStatus Channel<T>::Send(const T& item) {
  std::unique_lock<std::mutex> lock(mutex_); // 上锁，阻塞主线程
  if (is_closed_) { return kChannelStatusErrorClosed; } // 校验队列是否打开，当队列打开时，往队列中压入元素
  queue_.push(item); // 将 item 压入队列
  cond_.notify_one(); // 将 wait 的线程唤醒，wait 线程可获取该互斥量锁
  return kChannelStatusSuccess; 
}

/**
读数据：
Channel<T>::Receive(T* item)
将 Channel::queue_ 队列中值为 item 的元素弹出队列，并存储在 *item 中
*/
template<typename T>
ChannelStatus Channel<T>::Receive(T* item) {
  std::unique_lock<std::mutex> lock(mutex_);
  // 若互斥量 lock 被锁定，且 lambda 函数返回值为 true，则 wait 阻塞。必须同时满足，否则不会阻塞。
  // 只要其它线程调用 notify_one() 函数，且 lambda 为 false 时，wait() 一直处于阻塞状态。
  cond_.wait(lock, [this]() { return (!queue_.empty()) || is_closed_; });  
  if (queue_.empty()) { return kChannelStatusErrorClosed; }
  *item = queue_.front();
  queue_.pop();
  return kChannelStatusSuccess;
 /**
  写法二：
  当队列不为空（或不失效）时，执行读数据；否则处于等待状态
  while((!queue_.empty()) || is_closed_ == false)
  { 
    cond_.wait(lock);
  }
  *item = queue_.front();
  queue_.pop();
  return kChannelStatusSuccess;

*/
}

/**
读数据：
Channel<T>::ReceiveMany(std::queue<T>* items) ：
将队列 Channel::queue_ 中的元素全部转移到参数队列 items 中

*/
template<typename T>
ChannelStatus Channel<T>::ReceiveMany(std::queue<T>* items) {
  std::unique_lock<std::mutex> lock(mutex_);
  cond_.wait(lock, [this]() { return (!queue_.empty()) || is_closed_; });
  if (queue_.empty()) { return kChannelStatusErrorClosed; }
  while (!queue_.empty()) {
    items->push(std::move(queue_.front()));
    queue_.pop();
  }
  return kChannelStatusSuccess;
}

template<typename T>
void Channel<T>::Close() {
  std::unique_lock<std::mutex> lock(mutex_);
  is_closed_ = true; // Channel::is_closed_ 为 true，队列 Channel 失效标志
  cond_.notify_all();
}

}  // namespace oneflow

#endif  // ONEFLOW_CORE_COMMON_CHANNEL_H_

 ```
 
 
 
 
 
 ## C++ 知识：<br>
 
 1. std::function<br>
 https://en.cppreference.com/w/cpp/utility/functional/function<br>
 
 2. std::condition_variable::wait<br>
 void wait( std::unique_lock<std::mutex>& lock);<br>
 https://en.cppreference.com/w/cpp/thread/condition_variable/wait<br>

3. 完美转发 std::forward 
	
4. 原子操作 std::atomic<T>::fetch_add(另一个加数， std::memory_order_relexed)
 
