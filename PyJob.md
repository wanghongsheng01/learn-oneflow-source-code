# 一个 user_job 的执行过程
回顾一下 启动一个 job 时，数据在 Python 端的流转过程
主要是将用户定义的 function 通过 oneflow 的接口 @oneflow_function_config 添加为 FunctionConfig 类的属性。

## 用户定义的一个 user_job
==> pad_Job
如 test_compile_process.py
```.py
import oneflow as flow
import oneflow.typing as tp
import numpy as np

@flow.global_function()
def pad_Job(x: tp.Numpy.Placeholder((2, 1, 3, 3))) -> tp.Numpy:
    with flow.scope.placement("cpu", "0:0"):
        loss = flow.reflection_pad2d(x, padding=1)
        return loss

x = np.arange(18).reshape((2, 1, 3, 3)).astype(np.float32)
y = pad_Job(x)
print("in:\n", x, "y:\n", y)

```
从 `@flow.global_function() -> function_util.py:def api_oneflow_function()`


## 初始化 FunctionConfig 类
==> oneflow_function_config
将 72 个 类似 @oneflow_function_config("default_data_type") 特性添加为 FunctionConfig 类的属性
function_util.py: 72 个 @oneflow_function_config("xxx")

```.py
"""
将 72 个 类似 @oneflow_function_config("default_data_type") 特性添加为 FunctionConfig 类的属性
将每个 @oneflow_function_config("xxx") 设置为 FunctionConfig 类对象的属性，然后将该属性添加为 FunctionConfig 类的属性
"""
def oneflow_function_config(*field_paths):
    def Decorator(func):
        global _class_property2return_obj_class
        for field_path in field_paths:
            fields = field_path.split(".")
            assert len(fields) > 0
            cls = FunctionConfig
            for index, field in enumerate(fields):
                assert field != "function_desc"
                assert re.match("^[_\w]+[_\w\d]*$", field)
                if (cls, field) not in _class_property2return_obj_class:
                    class_name = ".".join(["function_config"] + fields[: index + 1]) # function_config.use_xla_jit

                    def Init(self, function_desc):
                        self.function_desc = function_desc

                    config_class = type(class_name, (object,), dict(__init__=Init)) # <class 'oneflow.python.framework.function_util.function_config.use_xla_jit'>
                    setattr(cls, field, _MakeInnerJobConfigClassProperty(config_class)) # 将 function_config.use_xla_jit 添加为 FunctionConfig 类的属性
                    _class_property2return_obj_class[cls, field] = config_class
                cls = _class_property2return_obj_class[cls, field]
            cls.__call__ = _MakeLeafJobConfigCall(func)
        return func

    return Decorator


_class_property2return_obj_class = {}


def _MakeInnerJobConfigClassProperty(return_obj_class):
    return property(lambda self: return_obj_class(self.function_desc)) # 返回 function_config.use_xla_jit 属性
```

## 构建 Session 对象
==> session_context.py：def GetDefaultSession
```.py
class SessionStatus:
    OPEN = "OPEN"

    RUNNING = "RUNNING"

    CLOSED = "CLOSED" # 启动时状态

```

```.py
"""
 def GetDefaultSession():
     
 构建一个 Session 字典，{sess_id : Session 对象}，每启动一个 Session，是通过
 `oneflow._oneflow_internal.GetDefaultSessionId()` 生成默认的 seess_id，
 再根据该 id 从 Seesion 字典里获取对应的 Session 对象。

"""
def GetDefaultSession():
    global _sess_id2sess # 是一个 key 为 sess_id，value 为 Session 对象的字典
    default_sess_id = oneflow._oneflow_internal.GetDefaultSessionId() # 生成默认的 default_sess_id
    assert default_sess_id in _sess_id2sess # 检查当前 seess_id 是否存在于 Session 字典里
    return _sess_id2sess[default_sess_id] # 根据 default_sess_id，获取字典对应的 value值， Session 对象

```
Debug Mode:
_sess_id2sess 是一个 key 为 sess_id，value 为 Session 对象的字典
0 : <oneflow.python.framework.session_util.Session object at 0x7f1d9c056910>
default_sess_id : 0

## Session 添加 function_desc 中的 signature 属性

==> function_util.py : api_oneflow_function
```.py
"""
api_oneflow_function 根据用户传入的 type（=train/predict）参数，通过 lazy_oneflow_function 获取 func_config(包含 job 对象地址)。

"""
@oneflow_export("global_function")
def api_oneflow_function(
    type: str = "predict", function_config: FunctionConfig = None,
) -> Callable[[Callable], Callable]:
    r"""Creates a callable OneFlow global function from a Python function.

    For instance::

        @oneflow.global_function(flow.FunctionConfig())
        def train():
            # your model

    Args:
        function_config (FunctionConfig, optional): a `FunctionConfig` object. Defaults to FunctionConfig().

    Returns:
        Callable[[Callable], Callable]: a callable which is called to execute the compiled function
    """
    if isinstance(type, FunctionConfig):
        function_config = type
        print(
            """WARNING: flow.global_function(func_config) is deprecated. Please replace it with flow.global_function(type, func_config).
            """
        )
        print(traceback.format_stack()[-2])
    else:
        assert type in ["train", "predict"]
        if function_config is None:
            function_config = FunctionConfig()
        if type == "train":
            function_config.function_desc.job_config_proto.mutable_train_conf()
        else:
            function_config.function_desc.job_config_proto.mutable_predict_conf()
    api = enable_if.unique([eager_oneflow_function, lazy_oneflow_function])
    return api(function_config)

```

==> function_util.py : lazy_oneflow_function
* 将 signature 属性添加到当前 job 中。
* 启动一个默认 Session 对象，并将当前 job 添加到 Session 中。


```.py
def lazy_oneflow_function(function_config=FunctionConfig()):
    assert isinstance(function_config, FunctionConfig)

    def Decorator(job_func): # job_func : <function pad_Job at 0x7fd2b9603710>
        if not hasattr(job_func, "__oneflow_function_signature__"):
            job_func.__oneflow_function_signature__ = inspect.signature(job_func) # 为当前 job 添加 signature 属性
        oft_util.CheckGlobalFunctionAnnotation(job_func.__oneflow_function_signature__)
        sess = session_ctx.GetDefaultSession() # 启动一个默认 Session 对象

        @functools.wraps(job_func)
        def Func(*args, **kwargs):
            return _RunLazyJob(sess, job_func, *args, **kwargs)

        sess.AddJob(_CloneFunctionDesc(function_config.function_desc, job_func)) # 当前 Session 添加 function_desc
        for x in dir(job_func): # job_func : <function pad_Job at 0x7f047078c320>
            if x.startswith("__oneflow_"): # __oneflow_function_signature__
                setattr(Func, x, getattr(job_func, x)) # getattr 获取 pad_Job 的 signature 属性，并将该属性添加为 Func 的属性
        return Func

    return Decorator
```


==> function_util.py : lazy_oneflow_function -> sess.AddJob
==> session_util.py :: AddJob
将  'pad_Job' 转换为 FunctionDesc 对象，存储在 job_name2function_desc_ 字典中。
```.py
"""
    AddJob 将 function_desc 添加到 job_name2function_desc_ 字典中
    job_name2function_desc_:{'pad_Job': <oneflow.python.framework.function_desc.FunctionDesc object at 0x7f047078b9d0>}


    """
    def AddJob(self, function_desc):
        assert self.status_ is SessionStatus.OPEN
        assert isinstance(function_desc, FunctionDesc)
        self.job_name2function_desc_[function_desc.job_func.__name__] = function_desc
```


==> function_util.py : lazy_oneflow_function -> return Func -> Func -> _RunLazyJob(sess, job_func, *args, **kwargs)
==> function_util.py :: _RunLazyJob

```.py
def _RunLazyJob(session, job_func, *args, **kwargs): # 已获取到用户 Python 端输入的参数
    return session.TryInit().LazyRun(job_func, *args, **kwargs)
```

==> session_util.py :: TryInit
```.py
def TryInit(self):
        if self.status_ is SessionStatus.OPEN:
            self.Init()
        return self
```
==> session_util.py :: Init





[打印 Plan 图方法](https://github.com/Oneflow-Inc/OneTeam/issues/224)

