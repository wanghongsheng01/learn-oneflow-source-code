register

术语：<br>
1. CUDA页锁定内存（Pinned Memory）<br>
对CUDA架构而言，主机端的内存被分为两种，一种是可分页内存（pageable memroy）和页锁定内存（page-lock或 pinned）。可分页内存是由操作系统API malloc()在主机上分配的，页锁定内存是由CUDA函数cudaHostAlloc()在主机内存上分配的，页锁定内存的重要属性是主机的操作系统将不会对这块内存进行分页和交换操作，确保该内存始终驻留在物理内存中。<br>
GPU知道页锁定内存的物理地址，可以通过“直接内存访问（Direct Memory Access，DMA）”技术直接在主机和GPU之间复制数据，速率更快。由于每个页锁定内存都需要分配物理内存，并且这些内存不能交换到磁盘上，所以页锁定内存比使用标准malloc()分配的可分页内存更消耗内存空间。<br>

页锁定内存的内配、操作和可分页内存的对比：<br>

```.cpp
#include "cuda_runtime.h"
#include "device_launch_parameters.h"
#include "iostream"
#include <stdio.h>
 
using namespace std;
 
float cuda_host_alloc_test(int size, bool up)
{
	//耗时统计
	cudaEvent_t start, stop;
	float elapsedTime;
	cudaEventCreate(&start);
	cudaEventCreate(&stop);
 
	int *a, *dev_a;
 
	//在主机上分配页锁定内存
	cudaError_t cudaStatus = cudaHostAlloc((void **)&a, size * sizeof(*a), cudaHostAllocDefault);
	if (cudaStatus != cudaSuccess)
	{
		printf("host alloc fail!\n");
		return -1;
	}
 
	//在设备上分配内存空间
	cudaStatus = cudaMalloc((void **)&dev_a, size * sizeof(*dev_a));
	if (cudaStatus != cudaSuccess)
	{
		fprintf(stderr, "cudaMalloc failed!\n");
		return -1;
	}
 
	//计时开始
	cudaEventRecord(start, 0);
 
	for (int i = 0; i < 100; i++)
	{
 
		//从主机到设备复制数据
		cudaStatus = cudaMemcpy(dev_a, a, size * sizeof(*dev_a), cudaMemcpyHostToDevice);
		if (cudaStatus != cudaSuccess)
		{
			fprintf(stderr, "cudaMemcpy Host to Device failed!\n");
			return -1;
		}
 
		//从设备到主机复制数据
		cudaStatus = cudaMemcpy(a, dev_a, size * sizeof(*dev_a), cudaMemcpyDeviceToHost);
		if (cudaStatus != cudaSuccess)
		{
			fprintf(stderr, "cudaMemcpy Device to Host failed!\n");
			return -1;
		}
	}
	cudaEventRecord(stop, 0);
	cudaEventSynchronize(stop);
	cudaEventElapsedTime(&elapsedTime, start, stop);
 
	cudaFreeHost(a);
	cudaFree(dev_a);
	cudaEventDestroy(start);
	cudaEventDestroy(stop);
 
	return (float)elapsedTime / 1000;
 
}
 
float cuda_host_Malloc_test(int size, bool up)
{
	//耗时统计
	cudaEvent_t start, stop;
	float elapsedTime;
	cudaEventCreate(&start);
	cudaEventCreate(&stop);
 
	int *a, *dev_a;
 
	//在主机上分配可分页内存
	a = (int*)malloc(size * sizeof(*a));
 
	//在设备上分配内存空间
	cudaError_t	cudaStatus = cudaMalloc((void **)&dev_a, size * sizeof(*dev_a));
	if (cudaStatus != cudaSuccess)
	{
		fprintf(stderr, "cudaMalloc failed!\n");
		return -1;
	}
 
	//计时开始
	cudaEventRecord(start, 0);
 
	for (int i = 0; i < 100; i++)
	{
 
		//从主机到设备复制数据
		cudaStatus = cudaMemcpy(dev_a, a, size * sizeof(*dev_a), cudaMemcpyHostToDevice);
		if (cudaStatus != cudaSuccess)
		{
			fprintf(stderr, "cudaMemcpy Host to Device failed!\n");
			return -1;
		}
 
		//从设备到主机复制数据
		cudaStatus = cudaMemcpy(a, dev_a, size * sizeof(*dev_a), cudaMemcpyDeviceToHost);
		if (cudaStatus != cudaSuccess)
		{
			fprintf(stderr, "cudaMemcpy Device to Host failed!\n");
			return -1;
		}
	}
	cudaEventRecord(stop, 0);
	cudaEventSynchronize(stop);
	cudaEventElapsedTime(&elapsedTime, start, stop);
 
	free(a);
	cudaFree(dev_a);
	cudaEventDestroy(start);
	cudaEventDestroy(stop);
 
	return (float)elapsedTime / 1000;
}
 
int main()
{
	float allocTime = cuda_host_alloc_test(100000, true);
	cout << "页锁定内存: " << allocTime << " s" << endl; // 0.01 s
	float mallocTime = cuda_host_Malloc_test(100000, true);
	cout << "可分页内存: " << mallocTime << " s" << endl; // 0.02 s
	getchar();
	return 0;

}
```
对比效果，页锁定内存的访问时间约为可分页内存的访问时间的一半.<br>

2. RDMA 远程直接内存访问<br>

3. RegstMgr：负责创建所有的 Regst （ Mgr 是 Manager 的缩写）<br>
   oneflow/core/register/ 路径<br>
   在初始化全局对象时，会创建 Global 对象 RegstMgr，`Global<RegstMgr>` 类是 `RegstMgr` 类的 `private friend class`，单例模式<br>
   每台机器上的 RegstMgr 管理了所有的 Regst。<br>
   RegstMgr 在初始化时就会根据 Plan 申请所有的本机上的内存：HostMemory、HostPinnedMemory（For CUDA CopyH2D）、DeviceMemory、LockedMemory（For RDMA）等。<br>
   并根据 Plan 中的 Regst 配置信息分配相应的内存地址给 Regst。Regst 的内存地址是固定的，直到运行时结束 Regst 的内存地址和大小都不会变化。OneFlow 的静态内存管理是 Runtime 启动时统一分配，    Runtime 结束时统一销毁。运行时的内存调度开销是 0。<br>
   
4. regist<br>
   regist 是一段存储空间，可能在 CPU/GPU/Device 上。runtime 时有 actor 处理数据，actor 输出完的数据写入 register，其它 actor 从 register 里读取数据。<br>
   register 里包含一个 blob，一个 blob 是一个输入。<br>
   
 5. blob<br>
    存储描述数据属性，shape、 datatype、shape_dynamic<br>
    
 6. danamic <br>
    编译期不能确定 shape 具体是大小，但可以确定变化范围。编译期将最大范围的值交给运行时，运行时分配最大的空间去运行。<br>
    
7. RegstMgr<br>
   RegistMgr 管理所有 regist，把所有 regist 目录下的 regist 串起来，最终完成空间的分配。<br>
   RegistMgr 完成就是这个存储空间的设计。<br>
  
   MemBlock与Chunk<br>
   这是 OneFlow 的多级内存设计：Chunk -> MemBlock -> Regst<br>
   
   ![三级存储](https://user-images.githubusercontent.com/31394900/126730293-17a34bc7-508a-4191-8ce1-b5307999d56e.png)
   
   分配 chunk:
   分配出一个 chunk。比如一个 Train Job，编译时计算一下需要多少存储空间，500M，一开始就一次性分配好 500M。比如还有 Eval Job，如果不到 500M，根据分时复用原则，会分配最大的 500M。
   
   二级分配 chunk-> 
   将 chunk 划分出 MemBlock，MemBlock 划分出 register。每个 register 有一个 id，每一个 regist 用在什么地方，编译期就分配好。
   
   从 chunk 分出来的 MemBlock 可分时复用，否则不能。
   
   register 里包含一个 blob，一个 blob 是一个输入。<br>
   register 里包含一个 blob，一个 blob 是一个输入。<br>
   
   一个 actor 可以写多个 regist
   
   
   register_manager.cpp -> `RegstMgr::RegstMgr`
   ```.cpp
   char* chunk_ptr = Global<MemoryAllocator>::Get()->Allocate(chunk.mem_case(), chunk.mem_size());
   ```
   
   memory_allocator.cpp -> `void* MemoryAllocatorImpl::Allocate(MemoryCase mem_case, size_t size)`
	 
   ```.cpp
    void* MemoryAllocatorImpl::Allocate(MemoryCase mem_case, size_t size) {
		void* ptr = nullptr;
		// 如果是 host，则 cudaMallocHost 分配 chunk 内存空间
		if (mem_case.has_host_mem()) 
		{
			if (mem_case.host_mem().has_cuda_pinned_mem()) {
	#ifdef WITH_CUDA
				if (Global<ResourceDesc, ForSession>::Get()->enable_numa_aware_cuda_malloc_host()) {
					NumaAwareCudaMallocHost(mem_case.host_mem().cuda_pinned_mem().device_id(), &ptr, size);
				} else {
					OF_CUDA_CHECK(cudaMallocHost(&ptr, size));
				}
	#else
				UNIMPLEMENTED();
	#endif
			} else {
				ptr = malloc(size);
				CHECK_NOTNULL(ptr);
			}
		} 
		// 如果是 CUDA，则 cudaMalloc 分配 chunk 内存空间
		else if (mem_case.has_device_cuda_mem()) 
		{
	#ifdef WITH_CUDA
			CudaCurrentDeviceGuard guard(mem_case.device_cuda_mem().device_id());
			OF_CUDA_CHECK(cudaMalloc(&ptr, size));
	#else
			UNIMPLEMENTED();
	#endif
		} else {
			UNIMPLEMENTED();
		}
		return ptr;
	}
   ```
	 
register_manager.cpp -> `void RegstMgr::NewRegsts(const RegstDescProto& regst_desc_proto, std::function<void(Regst*)> OneRegstDone)` 
new 出来的 regist，最终回调到 RegstMgr::NewRegsts 的调用者里 actor.cpp 的 lambda 函数里。最终 regist 保存到 Actor::produced_regsts_ 成员变量(Actor 写的 regist 成员变量)

```.cpp
void RegstMgr::NewRegsts(const RegstDescProto& regst_desc_proto,
                         std::function<void(Regst*)> OneRegstDone) {
  const int64_t regst_desc_id = regst_desc_proto.regst_desc_id();
  const RegstDescTypeProto& regst_desc_type = regst_desc_proto.regst_desc_type();
  const RtRegstDesc* rt_regst_desc = regst_desc_id2rt_regst_desc_.at(regst_desc_id).get();
  char* main_mem_ptr = nullptr;
  char* separated_header_mem_ptr = nullptr;
  int64_t mem_block_id = regst_desc_proto.mem_block_id();
  int64_t header_block_id = regst_desc_proto.separated_header_mem_block_id();
  if (mem_block_id != -1 && mem_block_id2ptr_.find(mem_block_id) != mem_block_id2ptr_.end()) {
    main_mem_ptr = mem_block_id2ptr_.at(mem_block_id) + regst_desc_proto.mem_block_offset();
  }
  if (header_block_id != -1 && mem_block_id2ptr_.find(header_block_id) != mem_block_id2ptr_.end()) {
    separated_header_mem_ptr = mem_block_id2ptr_.at(header_block_id);
  }
  std::vector<LbiBlobDescPair> lbi_pairs;
  if (regst_desc_type.has_data_regst_desc()) {
    for (const LbiBlobDescPair& pair : regst_desc_type.data_regst_desc().lbi2blob_desc()) {
      lbi_pairs.push_back(pair);
    }
    std::sort(lbi_pairs.begin(), lbi_pairs.end(), &CompareLbiBlobDescPair);
    CHECK(!lbi_pairs.empty());
  }
  for (int64_t i = 0; i < rt_regst_desc->register_num(); ++i) {
    Regst* regst = new Regst;
    regst->set_regst_desc(rt_regst_desc);
    if (regst_desc_type.has_data_regst_desc()) {
      NewBlobsInOneRegst(lbi_pairs, regst, rt_regst_desc, main_mem_ptr, separated_header_mem_ptr);
      if (rt_regst_desc->mem_case().has_host_mem()
          && rt_regst_desc->mem_case().host_mem().used_by_network()) {
        CheckBlobInRegstNotDisabled(regst_desc_proto);
        regst->comm_net_token_ = Global<CommNet>::Get()->RegisterMemory(
            main_mem_ptr, rt_regst_desc->MainByteSize4OneRegst());
      }
      if (main_mem_ptr != nullptr) { main_mem_ptr += rt_regst_desc->MainByteSize4OneRegst(); }
      if (separated_header_mem_ptr != nullptr) {
        separated_header_mem_ptr += rt_regst_desc->SeparatedHeaderByteSize4OneRegst();
      }
    } else if (regst_desc_type.has_ctrl_regst_desc()) {
      // do nothing
    } else {
      UNIMPLEMENTED();
    }
    OneRegstDone(regst); // new 出来的 regist，最终回调到 RegstMgr::NewRegsts 的调用者里 actor.cpp 的 lambda 函数里
    /*
    [this](Regst* regst) {
      produced_regsts_[regst->regst_desc_id()].emplace_back(regst); // Actor::produced_regsts_ 成员变量，Actor 的写的 regist 成员变量
    }
    */
  }
}
```

