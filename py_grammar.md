正则表达式
```.py
assert re.match("^[_\w]+[_\w\d]*$", field) 

```


为对象添加不存在的属性值
```.py
setattr(cls, field, _MakeInnerJobConfigClassProperty(config_class))

```


property() 函数的作用是在新式类中返回属性值。
```.py
property(lambda self: return_obj_class(self.function_desc))

```