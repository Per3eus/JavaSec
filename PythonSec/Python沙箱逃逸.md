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
