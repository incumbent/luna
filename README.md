# 关于`luna`

## 怎么加到工程中去

直接将`luna.h`, `luna.cpp`两个文件加到工程中去即可.

值得注意的是,你得编译器必须支持C++14.

我自己测试过的环境如下(都是64位编译,32位没测试过):

Windows: Visual studio 2015

Mac OS X: Apple (GCC) LLVM 7.0.0, 注意在编译参数中加入选项: `-std=c++1y`

GCC 5.3

## 创建luna运行时环境:

初始化luna运行时环境: `lua_setup_env(L)`
该函数会创建一个运行时数据结构,以userdata的形式存放到lus\_State中去.
在lua\_State被close时,会通过userdata的gc方法自动释放.

``` c++
lua_State* L = luaL_newstate();
luaL_openlibs(L);
lua_setup_env(L);
```

有必要的话,你可以设置以下即可回调函数:
`lua_set_error_func`: 错误回调函数,可以用来写日志之内的,默认调用puts.
`lua_set_file_time_func`: 获取文件的最后修改时间.
`lua_set_file_size_func`: 获取文件的大小.
`lua_set_file_data_func`: 一次性读取文件数据.
运行中可以调用`la_reload_update`重新加载已变更的文件.

## 导出函数

当函数的参数以及返回值是基本类型或者是导出类时,可以直接用`lua_register_function`导出:

``` c++
int func_a(const char* a, int b);
int func_b(some_export_class* a, int b);
some_export_class* func_c(float x);

lua_register_function(L, func_a);
lua_register_function(L, func_b);
lua_register_function(L, func_c);
```

当然,你也可以导出lua标准的C函数.

## 导出类

首先需要在你得类声明中插入导出声明:

``` c++
class my_class
{
	// ... other code ...
	int func_a(const char* a, int b);
	int func_b(some_export_class* a, int b);
    char m_name[32];
  	// 插入这句:
	DECLARE_LUA_CLASS(my_class);
};
```

在cpp中加入导出代码:

``` c++
EXPORT_CLASS_BEGIN(my_class)
EXPORT_LUA_FUNCTION(func_a)
EXPORT_LUA_FUNCTION(func_b)
EXPORT_LUA_STRING(m_name)
EXPORT_CLASS_END()
```

可以用带`_AS`的导出宏指定别名导出,用带`_R`的宏指定导出为只读变量.

-  不要对导出对象memset.
-  已导出的对象,应该在关闭虚拟机之前析构,或者提前调用`lua_clear_ref`解除引用.

## lua中访问导出对象

lua代码中直接访问导出对象的成员/方法即可.

``` lua
local obj = get_obj_from_cpp();
obj.func("abc", 123);
obj.name = "new name";
```

另外,也可以在lua中重载(覆盖)导出方法.

### C++中调用lua函数

比如我们要调用脚本test.lua中的一个函数,如下说示:

``` lua
function some_func(a, b)
  	return a + b, a - b;
end
```

那么,可以在C++中这样调用:

```cpp
int x, y;
lua_guard_t g(L);
// x,y用于接收返回值
call_file_function(L, "test.lua", "some_func", std::tie(x, y), 11, 2);
```

无返回,无参数:

```cpp
call_file_function(L, "test.lua", "some_func");
```

无返回, 有a,b,c三个参数:

```cpp
call_file_function(L, "test.lua", "some_func", std::tie(), a, b, c);
```

### 沙盒环境
每个文件具有独立的环境,文件之间通过`import`相互访问,如:

```lua
local m = import("some_one.lua");
m.some_function();
```

实际上,import返回目标文件的环境表.
如果目标文件未被加载,则会自动加载.
已经import的文件不会重复加载.


## 可能的问题

1. DECLAREi\_LUA\_CLASS宏在对象中插入了成员,在某些环境下可能是不能接受的,比如对象数量级非常大的时候.
2. 如果类之间有继承关系,同时导出基类和子类时,要小心操作指针,比如用基类指针push子类对象,结果可能就是非预期的.

对于1,可以采用将对象与lua table的对应关系另外存一张表的方式解决,当然这会产生一点性能消耗.
这个额外的对应表可以是C++层面维护,也可以在lua中维护.
但是还得另外处理C++对象析构时清除引用的问题.

对于2,问题的根本原因是C++对象的子类指针和其基类指针可能是不一样的,而luna在lua影子对象中存储了对象的指针.
之前的版本使用了虚函数来取得导出表之类的元数据,但是并不能完全解决问题,所以在新版中去掉了虚函数.
使用者需要了解这一点,以在实践中有针对性的规避这一问题了.



