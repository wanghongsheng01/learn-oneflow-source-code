编译期的 C++ 

oneflow.h




## oneflow.cpp::FilterOpName2ParallelBlobConf<br>

FilterOpName2ParallelBlobConf 干了啥？<br>
获取当前 job 中需要内存复用的 OpBlob 的信息，将当前 job 的 (op_conf，parallel_blob_conf）信息添加到 op_name2parallel_blob_conf 集合里。<br>

```.cpp
/**
FilterOpName2ParallelBlobConf：获取当前 job 中需要内存复用的 OpBlob 的信息，将当前 job 的 (op_conf，parallel_blob_conf）信息添加到 op_name2parallel_blob_conf 集合里。
判断当前 job 的 op_conf.op_type_case() 是否在匹配集合 match 对象 （OperatorConf::OpTypeCase 类型）里：
如果有，获取当前 job 中需要内存复用的 OpBlob 的信息。再判断当前 job 的 op_conf 是否在 op_name2parallel_blob_conf 集合里：
如果有，则校验两者的 parallel_blob_conf 是否相等；
如果没有，则 op_name2parallel_blob_conf 添加当前 job 的信息。

*/
void FilterOpName2ParallelBlobConf(
    const HashSet<OperatorConf::OpTypeCase>& match, const std::vector<std::shared_ptr<Job>>& jobs,
    HashMap<std::string, ParallelBlobConf>* op_name2parallel_blob_conf) {
  FOR_RANGE(int64_t, job_id, 0, jobs.size()) {
    JobBuilder job_builder(jobs.at(job_id).get()); // 根据 job 对象，构造了 job_builder 对象
    for (const OperatorConf& op_conf : jobs.at(job_id)->net().op()) {
      if (match.find(op_conf.op_type_case()) == match.end()) { continue; } // 若当前 job 的 op_conf.op_type_case() 不在 OperatorConf::OpTypeCase 匹配集合 match 里，则跳过进入下一个 job 循环
      ParallelBlobConf parallel_blob_conf; 
      GetMemSharingOpBlobInfo(job_builder, op_conf.name(), &parallel_blob_conf); // 获取当前 job 中需要内存复用的 OpBlob 的信息
      auto iter = op_name2parallel_blob_conf->find(op_conf.name());
      if (iter == op_name2parallel_blob_conf->end()) { // 判断 op_name2parallel_blob_conf 是否存在当前 job 的 (op_conf，parallel_blob_conf）信息
        CHECK(op_name2parallel_blob_conf->emplace(op_conf.name(), parallel_blob_conf).second); // 如果没有，则 op_name2parallel_blob_conf 添加当前 job 的信息
      } else {
        CHECK(parallel_blob_conf == iter->second); // 如果有，则校验两者的 parallel_blob_conf 是否相等
      }
    }
  }
}
```
