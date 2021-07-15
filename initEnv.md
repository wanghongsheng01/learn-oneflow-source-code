# 学习 oneflow 完整运行流程。
* 环境初始化
* Python 端 Job 构图
* 编译期：OneFlow(JobSet)->OneFlow(MergedPlan)
* 编译期：Compiler(Job)->Plan
* 运行时：Runtime(Plan)

## 初始化环境(Env)

path：/oneflow/oneflow/core/control

Python 前端启动时，第一件事就是初始化集群环境，环境由一个配置文件 EnvProto 描述。
包括多少台机器、每台机器 id、控制端口、数据传输端口等。

env.proto
```.proto
message EnvProto {
  repeated Machine machine = 1;
  required int32 ctrl_port = 2;
  optional int32 data_port = 3 [default = -1];
  optional CppLoggingConf cpp_logging_conf = 4;
  optional BootstrapConf ctrl_bootstrap_conf = 5;
  optional bool is_default_physical_env = 6 [default = false];
}

```



